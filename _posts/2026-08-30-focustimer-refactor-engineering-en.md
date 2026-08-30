---
layout: post
title: "FocusTimer3 Refactor in Practice (II): Turning a Working HarmonyOS Demo into an Engineering-Solid Codebase"
date: 2026-08-30
description: "A HarmonyOS NEXT / ArkTS engineering refactor: breaking up a god page into components, layering pure-logic engines and a repository, reaching 74 unit tests, and the lesson of a green CI that still failed to compile locally."
categories: [HarmonyOS, Mobile Development]
tags: [HarmonyOS NEXT, ArkTS, Refactoring, Componentization, Layered Architecture, Unit Testing, CI]
author: 725lizi
lang: en
---

# FocusTimer3 Refactor in Practice (II): Turning a Working Demo into an Engineering-Solid Codebase

> This is the second post in the FocusTimer3 series. The [first post](./2026‑08‑23‑focustimer‑review‑en.html) explains how I built a **working** app from scratch. This one is about a different problem: turning that app into a codebase that is **well-structured, safe to change, thoroughly tested, and hard to pick apart in an interview** — the foundation for integrating a real LLM in the next phase.
>
> Source code: [https://github.com/725lizi/FocusTimer](https://github.com/725lizi/FocusTimer)
> 中文版：[FocusTimer3 重构实战（二）](./2026-08-30-focustimer-refactor-engineering.html)

## 0. TL;DR

This phase (internally "Phase 1: Engineering Foundation") added **zero new features**. It was entirely about paying down **technical debt** — the shortcuts taken to get something running today that cost interest tomorrow. The result:

| Metric | Before | After |
|---|---|---|
| Timer home page `Index.ets` | A 782-line "god file" mixing UI, timer logic and storage | 566 lines; `build()` only composes components |
| Bottom navigation | Copy-pasted across 4 pages (~166 duplicated lines) | One shared component reused by all 4 pages |
| Home UI blocks | Crammed into one file | 9 extracted components (10 reusable components total) |
| Timer core logic | Inlined in the page, untestable off-device | A pure-logic `TimerEngine`, unit-testable offline |
| Data read/write | Each page talked to storage directly | Funneled through a single `FocusSessionStore` entry point |
| Unit-test cases | 14 for the rule engine | **74 project-wide** (32 rule engine, 18 timer engine, ...) |
| Commits | —— | 10 semantic commits; cloud CI stayed green throughout |

## 1. The Starting Point: Working, but Too Scary to Change

After the first post, the app ran in the emulator, but only I knew how fragile it was:

1. **God page.** The home page was nearly 800 lines: layout, the stopwatch state machine, and persistence were all tangled together. A small change meant scrolling through hundreds of lines, afraid of collateral damage.
2. **Duplication.** The bottom navigation bar was copy-pasted almost identically into four pages. Changing one icon meant editing four places; missing one was a bug.
3. **Half-finished and dead code.** An ambient-light card showing hard-coded fake data, empty methods nobody called, and unused classes.
4. **Misleading naming.** A class that was really a fixed template plus scoring rules was named `AIAnalysisService`, and its output was called an "AI report".

Most importantly: **there were no tests, so I was afraid to change anything.** "It runs" is not "it's safe to evolve". Without automated tests as a safety net, every refactor is tightrope-walking without a harness. That is technical debt in a nutshell — harmless today, but it makes every future iteration slower and scarier.

## 2. The Counter-Intuitive First Move: Remove the "AI" Label

The very first commit wrote almost no logic — it **renamed things honestly**: `AIAnalysisService` became `LearningAnalysisService`, the "AI report" became a "learning analysis report", and the class doc states plainly: **this is currently an on-device rule template; a real LLM arrives in Phase 2.**

Why expose my own weakness? Because **a name is a design decision and a promise.**

- A *rule engine* (fixed logic the programmer writes in advance; same input always yields the same output) and a *large language model* (an LLM such as DeepSeek that generates language) are fundamentally different.
- Keeping the "AI" label to look good on a résumé collapses the moment an interviewer asks which model, what prompt, or how tokens are handled — and then it reads as dishonesty.
- Stating "rules today, a hybrid rules + LLM design tomorrow" is both honest and leaves a clean substitution point for later.

That substitution point is an **interface-driven, swappable design**: callers only ask for "a learning report"; they don't care whether rules or an LLM produced it. Because Phase 1 drew that boundary clearly, Phase 2 can stream from DeepSeek online and fall back to the rule engine offline **without changing a single line in the pages**.

## 3. The Four-Step Layered Refactor

Refactoring is not a rewrite. I worked toward a target architecture where dependencies point only downward, and lower layers never know about upper ones:

![FocusTimer3 layered architecture](assets/focustimer-architecture-en.svg)

### 3.1 Kill duplication first: a shared bottom navigation

I started with the lowest-risk, highest-certainty win: extracting the four navigation copies into one `BottomNavBar` used by all four pages. This removed ~166 duplicated lines and also restored the missing "analysis report" entry on two pages.

> Principle: **Don't Repeat Yourself (DRY).** The second copy of any logic is a future bug nursery.

### 3.2 Componentize the home page: props down, callbacks up

Next I split the ~800-line home page into nine visual components: timer ring, duration selector, subject chips, today's stats, ambient-light card, and so on. The simplest one, the duration selector, illustrates the pattern:

```typescript
@Component
export struct DurationSelector {
  // Data passed DOWN from the parent (props)
  selectedIndex: number = Constants.DEFAULT_INDEX;
  disabled: boolean = false;
  // The child never mutates state itself; it reports the click UP via a callback
  onSelect: (index: number) => void = () => {};

  build() {
    // ...when a chip is tapped:
    .onClick(() => {
      if (!this.disabled) {
        this.onSelect(index);   // emit the action; the parent owns the state change
      }
    })
  }
}
```

This enforces **one-way data flow**: data travels parent → child, and a child can only "ask" the parent to change state, never mutate it behind the scenes. State ownership becomes traceable. Afterwards the home `build()` merely arranges components, and the file dropped from 782 to 571 lines (later 566).

### 3.3 Pull the "brain" out of the UI: the pure TimerEngine

Componentization fixes visual clutter, but the stopwatch *rules* (start, pause, tick every second, reminder threshold, interruption count) were still in the page. I extracted them into a **pure-logic class** `TimerEngine` that imports no UI and touches no system API — deterministic for any given input:

```typescript
export class TimerEngine {
  static readonly PRESET_DURATIONS: number[] = [300, 900, 1500, 2700, 3600]; // 5/15/25/45/60 min
  private currentSecondValue: number;
  private running: boolean = false;
  private pauseCountValue: number = 0;

  start(): void { /* resume; only mutates internal variables, no UI */ }
  pause(): void { /* pause; increment interruption count */ }
  tick(): void  { /* one step per second: remaining -= 1 */ }
  getProgress(): number { /* remaining / total, for the UI to render */ }
}
```

The page's `@State` (ArkUI reactive state that auto-refreshes the view) becomes just a **mirror** of the engine, synced once per second for display. Why is pure logic the easiest to test? You never launch the app or start an emulator — just instantiate an object, call a method, and assert the result in milliseconds. The engine has 18 tests, including recovery from a checkpoint after an abnormal exit.

### 3.4 Funnel data access: the FocusSessionStore repository

Before, the home page and the statistics page each read and wrote storage directly — two entry points to the same data. I introduced a **repository** layer: a single "service desk" for data. Callers request data without knowing which shelf it lives on.

```typescript
export class FocusSessionStore {
  private static instance: FocusSessionStore | null = null; // singleton: one global desk
  private context: Context | null = null;

  public async getAllSessions(): Promise<StudySession[]> {
    if (this.context === null) { return []; }   // fail safe instead of crashing
    try {
      const prefs = await preferences.getPreferences(this.context, 'study_data');
      const raw = await prefs.get('sessions', '[]') as string;
      return JSON.parse(raw) as StudySession[];
    } catch (err) { return []; }                // corrupted data degrades to an empty list
  }
}
```

A deliberate piece of **engineering restraint**: the project historically had two storage paths (a lightweight key-value store and a relational database). This step **wraps without migrating** — unify the entry point first, but don't change how data is physically stored. "Unify the entry point" and "merge the two stores" are independent changes; bundling them makes failures untraceable. Small, individually verifiable steps are what make refactoring safe.

### 3.5 Fix real quality issues along the way

- **Timer leaks.** Timers were created but not symmetrically stopped in `aboutToDisappear` — leaving the tap running after leaving the room. I paired creation and teardown strictly to avoid background battery drain.
- **Fake ambient-light card.** Replaced hard-coded values with a real subscription to the `AMBIENT_LIGHT` sensor (sampled every 2 s, with a fallback on error).
- **Dead code removal.** Deleted uncalled methods, unused classes and imports.

## 4. Testing the Rule Engine: No Decorative Happy Paths

With a safety net in place, I strengthened it. Rather than writing **happy-path** tests that only cover perfect weather, I targeted **boundary values and branches** — the edge and extreme inputs most likely to break.

The focus score is a three-factor weighted sum:

```typescript
const durationScore  = Math.min(100, (avgDuration / 45) * 100); // 45 min = full marks
const interruptScore = Math.max(0, 100 - interruptRate * 100);  // more interruptions, lower
let regularityScore = 80;                                        // steadier daily time, higher
const focusScore = Math.round(
  durationScore * 0.4 + interruptScore * 0.35 + regularityScore * 0.25
);
```

I grew the rule-engine cases from 14 to **32**, covering: empty input, a single record, cross-day de-duplication, fully interrupted (exactly 65) vs. uninterrupted (100), extra-long sessions capped at 95, empty tags, trend up/down, time-slot boundaries (exactly 06:00 and 21:00), pomodoro duration clamped to 15–90 minutes, English terms, stop-word filtering, and week-over-week report deltas.

A detail worth noting: the cloud CI has no HarmonyOS SDK and cannot run ArkTS tests. I first **translated the pure logic to Node.js**, ran 59 assertions to lock down every expected value, then wrote them back into the HarmonyOS hypium framework. This avoids guessing expected values — the tests genuinely catch regressions instead of padding numbers.

## 5. The Most Valuable Lesson: A Green CI That Still Failed to Compile

### 5.1 Green in the cloud, four errors in local DevEco

Every push showed a green CI (continuous integration — a pipeline that auto-checks code on push). Yet my first full local compile in DevEco Studio threw four ERRORs.

**Errors 1–3: `Property 'get/put/flush' does not exist on type 'void & Promise<Preferences>'`**

The store's `this.context` was typed `Context | null`. I called `getPreferences` without first ruling out `null`, so the compiler matched the wrong **overload** — the callback version returning `void`, which naturally has no `.get()`. Following a class in the project that compiled correctly, I added a **null guard** at the top to perform **type narrowing** — eliminate the null branch so the compiler can guarantee non-null afterwards:

```typescript
if (this.context === null) { return []; }   // guard: narrow first, then call the API
const prefs = await preferences.getPreferences(this.context, 'study_data');
```

**Error 4: `Namespace 'sensor' has no exported member 'AmbientLightResponse'`**

I had typed a type name from memory that doesn't exist. The fix is not to guess but to open the SDK's **type declaration file** `@ohos.sensor.d.ts` — the SDK's built-in dictionary of exact API names and signatures. The real callback type is `sensor.LightResponse`.

### 5.2 A stale test: delete it, or make it real?

After the main code compiled, the full test run surfaced four more errors: a test imported a class `CardDataBuilder` that didn't exist, which in turn tripped `arkts-no-any-unknown` (ArkTS forbids the vague `any` type and requires explicit types).

I did **not** take the shortcut of deleting those tests (lowering coverage to force a green result is hiding the problem). Instead I extracted the card-formatting logic — previously inlined in the widget service — into a new pure class `CardDataBuilder`, giving the stale tests a real object to test, rewrote the self-contradicting old cases, added a padding/floor boundary case, and annotated explicit types to remove `any`.

### 5.3 The takeaway: CI is not magic; quality needs layered gates

A green CI does **not** mean the code compiles or runs. Restricted by HarmonyOS SDK distribution, my cloud CI only performed repository-standard checks (encoding, branch, commit messages) — it never compiled ArkTS. Quality therefore relies on **layered gates**, each catching a different class of problem:

| Gate | Where | What it catches |
|---|---|---|
| ① Standard checks | Cloud CI | Encoding, branch, commit message, repo structure |
| ② Full compile + type check | Local DevEco | Type errors, wrong API names, unhandled null (the four errors lived here) |
| ③ Unit tests | Local hypium | Logic regressions, boundary/branch errors |
| ④ Emulator / device | Device | UI, sensors, lifecycle runtime issues |

I also set a personal rule: never silence an error with a forced cast like `as any` — that doesn't fix the problem, it just covers the compiler's mouth and pushes a compile-time bug into production.

## 6. Phase Results at a Glance

Ten semantic commits, each doing one thing and independently verifiable:

| Commit theme | What it did |
|---|---|
| P0 narrative fix | Removed misleading AI packaging; deleted a corrupted duplicate (net -200+ lines) |
| 2.1 shared navigation | Extracted `BottomNavBar`; ~166 fewer duplicated lines |
| 2.2 home components | 9 components; home page 782→571; one-way data flow |
| 2.3 timer engine | Pure `TimerEngine`; 18 unit tests |
| 2.4 repository | `FocusSessionStore` as the single data entry point |
| 2.5 quality fixes | Timer leaks, real sensor, dead-code cleanup |
| 2.6 honest rename | AI service → learning analysis service |
| 2.7 more tests | Rule-engine cases 14→32 |
| compile fixes ×2 | Type narrowing, real SDK type, extracted `CardDataBuilder` |

## 7. Likely Interview Follow-Ups

**Why spend a whole phase refactoring instead of adding features?** This is a long-lived portfolio project meant to host an LLM. On a shaky foundation every feature pays interest; after this phase, adding an AI implementation requires zero page changes. It was a deliberate engineering investment, not rework.

**How do you draw the line between pure logic and UI?** Anything that computes an output from inputs without needing the screen or a device sinks into pure logic (timer rules, scoring, formatting); the UI only renders and relays user actions. The litmus test: can I verify it by instantiating an object without launching the app?

**Why not merge the two storage paths immediately?** Unifying the entry point and merging storage are changes of different risk levels; together they make failures untraceable. I unified first with data flow unchanged and left the merge until tests covered it — small, reversible steps.

**How can CI be green while local compile fails, and how would you improve it?** The cloud CI lacked the HarmonyOS SDK and never compiled. Every pipeline has a coverage boundary that must be stated. The improvement is a build image with the SDK to compile in the cloud; until then a full local compile is a mandatory manual gate.

**How do the rule engine and AI relate?** The rule engine is the deterministic fallback; the LLM is the enhancing implementation; both satisfy the same "produce a learning report" interface. Online, DeepSeek streams richer text; offline, it falls back to rules so the feature always works — exactly Phase 2.

## 8. Next: Phase 2, a Real LLM

With the foundation set, the next phase will: ① connect the DeepSeek API and **stream** a learning report token by token instead of waiting for one blob; ② implement **offline degradation** to the rule engine polished in this phase; ③ handle the engineering details — secret management, loading states, retries. The swappable interface planted in this post will finally pay off.

---

## Related Links
- 📘 First post (English): [FocusTimer3 Project Review](./2026‑08‑23‑focustimer‑review‑en.html)
- 📗 中文版：[FocusTimer3 重构实战（二）](./2026-08-30-focustimer-refactor-engineering.html)
- 📂 Source Code: [FocusTimer GitHub Repo](https://github.com/725lizi/FocusTimer)
- 🏠 Back to [Homepage](/)
