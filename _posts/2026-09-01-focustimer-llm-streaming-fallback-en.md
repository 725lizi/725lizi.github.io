---
layout: post
title: "Integrating an LLM into a HarmonyOS App (III): Streaming Output and Graceful Fallback so the Weekly Report Never Breaks — Offline, Bad Key, or Timeout"
date: 2026-09-01
description: "A production-minded DeepSeek integration on HarmonyOS NEXT / ArkTS: SSE streaming with partial-chunk tolerance, prepare-the-fallback-first degradation, HTTP error classification, sandboxed API-key storage, a tiny Markdown renderer, and a deliberate pure-logic vs device-layer testing strategy (108 unit tests)."
categories: [HarmonyOS, Mobile Development]
tags: [HarmonyOS NEXT, ArkTS, LLM Integration, DeepSeek, SSE Streaming, Graceful Degradation, Unit Testing, Hybrid Architecture]
author: 725lizi
lang: en
---

# Integrating an LLM into a HarmonyOS App (III): Streaming and Graceful Fallback so the Report Never Breaks

> This is the third post in the FocusTimer3 series. The [first post](./2026‑08‑23‑focustimer‑review‑en.html) covers building a **working** app from scratch; the [second post](./2026-08-30-focustimer-refactor-engineering-en.html) refactors it into a **well-structured, thoroughly tested** codebase and deliberately leaves a "replaceable interface". This post cashes in that promise — **integrating a real LLM** — with the focus not on "making the call work" but on "never showing the user a blank screen, no matter what goes wrong".
>
> Source code: [https://github.com/725lizi/FocusTimer](https://github.com/725lizi/FocusTimer)
> 中文版：[FocusTimer3 接大模型实战（三）](./2026-09-01-focustimer-llm-streaming-fallback.html)

## 0. TL;DR

I connected a local focus-timer app to the DeepSeek LLM to generate a weekly study report, holding myself to a **production-minded** bar rather than a tutorial-demo bar:

| Capability | How | What keeps it correct |
|---|---|---|
| Typewriter streaming | Receive SSE chunk by chunk via HarmonyOS `requestInStream` | Pure `SseChunkParser` handles partial chunks / bad JSON, locked by unit tests |
| Works offline | Three-condition gate (toggle + key + network); otherwise use local rules | Pure `ReportStrategy`, every combination tested offline |
| Auto fallback | Network/timeout/bad-key/rate-limit/server errors all fall back to rules | `HttpErrorMapper` status mapping, unit-tested |
| Key safety | Stored only in the app sandbox `preferences`, cached in memory, never hard-coded | Central `AiConfig`, zero hard-coded secrets |
| Clean rendering | A ~40-line Markdown renderer removes the model's `**` stars | Pure function + 8 unit tests |
| Overall quality | **108 unit tests** project-wide, small commits, green CI | Pure logic tested offline; device layer verified on device |

## 1. The hard part of LLM integration is not "sending a request"

A typical tutorial "AI integration" is: tap a button → send one HTTP request → wait a few seconds → dump the returned string into a text box. **That is fine for a demo, but nowhere near enough for something I would put on a résumé and defend in an interview**, because the real world is messy:

- **The network is unreliable**: dropped connections in the subway, packet loss, timeouts;
- **The LLM fails**: wrong key, exhausted balance, rate limiting (429), 5xx server errors, empty responses;
- **Streaming has traps**: the response does not arrive as whole sentences but as byte chunks whose boundaries don't line up with sentences — or even with a single Chinese character;
- **Secrets leak**: hard-coding a key and pushing to a public repo is taping your wallet to the door;
- **Models return raw Markdown**: they use `**bold**` for styling, but a native text component doesn't understand Markdown and shows the stars literally.

So my goal was: **no matter which of these happens, tapping the button always yields a report** — smarter when the LLM is reachable, seamlessly replaced by the local rule-based version otherwise, with the UI honestly labelling who wrote it. This is exactly the seam left in post II: a **rules + LLM hybrid architecture** that hides the difference behind one stable interface.

## 2. Design: split one AI report into seven single-responsibility parts

Rather than one god-object `AiManager` (which would just create a second god file), I kept the post-II layering and cut along one axis — **can it be tested offline?**

```
User taps "Generate with AI"
        |
        v
+-----------------------------------------------+
| AiReportOrchestrator (device-facing)          |
|  1. Compute the rule-based fallback FIRST      |
|  2. ReportStrategy.decideSource three-gate     |
|     |- not satisfied -------------> rule text  |
|     +- satisfied -> DeepSeekClient stream      |
|                       |                        |
|          streamed deltas        any error      |
|                       v              v         |
|                 accumulate AI   shouldFallback?|
|                       |        yes-> fallback  |
|                       v                        |
|              return to UI (with source label)  |
+-----------------------------------------------+

Pure-logic layer (no network/system, unit-tested offline)
  ReportPromptBuilder  prompts, request body, weekly aggregation
  SseChunkParser       split SSE, partial-chunk/bad-JSON handling
  ReportStrategy       decide LLM vs rules, whether to fall back
  HttpErrorMapper      status/framework code -> unified error kind
Device layer (needs real network/system, verified on device)
  DeepSeekClient       requestInStream, decoding
  ApiKeyStore          sandboxed key read/write
  NetState             is the device online
```

The principle I care about most: **extract every "deterministic output from a given input" judgment out of device code into a pure function**. Pure functions need no network or device, run in milliseconds, and make it cheap to write dozens of boundary cases. The classes that actually touch the network become so thin they mostly just ferry data around.

## 3. Streaming: the typewriter effect and the "partial chunk" trap

### 3.1 Why stream at all

With a normal request the user stares at a spinner for 5–10 seconds before the whole text appears at once — clunky. With streaming, the model emits text as it thinks; the server pushes **SSE (Server-Sent Events)** where every snippet is a `data: {...}` text block, and the client displays each as it arrives, producing a typewriter effect.

DeepSeek exposes an **OpenAI-compatible** API (same request/response shape; just a different endpoint and key). Set `stream` to `true`:

```typescript
static buildStreamRequest(summary: WeeklyDataSummary, model: string): ChatCompletionRequest {
  const request: ChatCompletionRequest = {
    model: model,              // deepseek-chat
    messages: ReportPromptBuilder.buildMessages(summary),
    stream: true,              // the key flag
    temperature: 0.7,
    max_tokens: 600
  };
  return request;
}
```

In HarmonyOS's `@kit.NetworkKit`, a normal request uses `request()` (whole response at once); **streaming requires `requestInStream()`**, whose body arrives through `on('dataReceive')`. An easy-to-miss documentation detail: the completion callback's second argument is **only the HTTP status code (a number), not the body** — the body lives entirely in `dataReceive`.

### 3.2 Trap 1: one Chinese character can be split across packets

`dataReceive` hands you an `ArrayBuffer` of raw bytes to decode. A Chinese character is 3 bytes in UTF-8, and the network can split those 3 bytes across two packets. Decoding each packet independently produces a replacement glyph `�` at the seam.

The fix is a **stateful streaming decoder**, `decodeWithStream`, which remembers "half a character" and reassembles it when the next packet arrives:

```typescript
private decoder: util.TextDecoder = util.TextDecoder.create('utf-8');

httpRequest.on('dataReceive', (data: ArrayBuffer) => {
  // decodeWithStream carries the leftover bytes of a split character
  const chunkText: string = this.decoder.decodeWithStream(new Uint8Array(data));
  const deltas: string[] = parser.feed(chunkText);
  // ...append each delta
});
```

### 3.3 Trap 2: a line of JSON can be split too

After decoding, more trouble. SSE separates events with newlines; ideally a callback contains whole lines:

```
data: {"choices":[{"delta":{"content":"Overall"}}]}

data: {"choices":[{"delta":{"content":" situation"}}]}
```

But the network does not guarantee "one callback = whole lines". A callback might contain `data: {"choices":[{"delta":{"content":"hal` — **a half line, a half JSON object**. Calling `JSON.parse` on that throws, guaranteed.

My pure `SseChunkParser` keeps an internal `buffer`: prepend the leftover, split on newlines; **if the last segment doesn't end in a newline it is a partial chunk — stash it for next time**, and parse only whole lines:

```typescript
feed(rawChunk: string): string[] {
  const deltas: string[] = [];
  this.buffer += rawChunk;                       // stitch with the leftover
  const lines = this.buffer.split('\n');
  if (this.buffer.endsWith('\n')) {
    this.buffer = '';                            // everything whole
  } else {
    this.buffer = (lines.pop() as string);       // last one partial; keep it
  }
  for (let line of lines) {
    line = line.trim();
    if (!line.startsWith('data:')) continue;     // skip blanks / event lines
    const payload = line.substring('data:'.length).trim();
    if (payload === '[DONE]') { this.doneFlag = true; continue; } // end marker
    const text = SseChunkParser.extractDelta(payload);
    if (text.length > 0) deltas.push(text);
  }
  return deltas;
}
```

Extraction adds a "never crash" guard: even on malformed/half JSON, `try/catch` returns an empty string and moves on instead of killing the stream:

```typescript
private static extractDelta(jsonLine: string): string {
  try {
    const parsed = JSON.parse(jsonLine) as StreamChunk;
    return parsed.choices?.[0]?.delta?.content ?? '';
  } catch (err) {
    return '';   // half JSON: skip now; the next packet completes it
  }
}
```

> Engineering principle: **every input at a network boundary is untrusted**. Defend against half characters when decoding and half lines / bad JSON when parsing. Because none of this depends on the network, the whole thing is a pure class I can feed deliberately-mangled inputs in unit tests — far more reliable than hoping to reproduce it on a device.

## 4. The core: "prepare the fallback first" degradation

Streaming makes it pretty; degradation makes it unbreakable — and the latter is what separates this from a demo.

### 4.1 Gate: request the LLM only when all three conditions hold

We shouldn't call the LLM every time: the user may have toggled AI off, may not have entered a key, may be offline. This is a pure function — AND the three, fall back to rules if any is missing:

```typescript
static decideSource(input: ReportStrategyInput): ReportSource {
  if (input.llmEnabled && input.online && input.hasApiKey) {
    return 'llm';
  }
  return 'rule';
}
```

When choosing rules, the UI gives an **honest, human-readable** reason (`ruleReason`): "AI disabled", "no API key configured", or "offline — using local rules". The source is always labelled; a rule-based report is never passed off as AI-written — the same honesty value as the renaming in post II.

### 4.2 Collapse messy errors into 8 kinds

Failures come from two systems: HTTP status codes (401/403/429/500…) and HarmonyOS framework codes (e.g. `2300089` = request cancelled, `2300004/2300047` = timeout). Facing raw codes directly makes strategy impossible, so a pure `HttpErrorMapper` **normalizes** them into 8 semantic kinds:

| Kind | Meaning | HTTP status | Framework code |
|---|---|---|---|
| `network` | offline / unreachable | others | unknown codes default here |
| `timeout` | timeout | 408 | 2300004 / 2300047 / 2300999 |
| `unauthorized` | bad key / no balance / forbidden | 401 / 403 | —— |
| `rate_limited` | rate limited | 429 | —— |
| `bad_request` | malformed request | 400 / 404 / 422 | —— |
| `server` | server fault | ≥500 | —— |
| `empty` | empty response | app-level | —— |
| `cancelled` | user cancelled | —— | 2300089 |

Once normalized, each kind gets a human message (`userMessage`; e.g. 401 → "API key invalid or out of balance, please check settings"), and the "should we fall back" decision becomes trivial.

### 4.3 There is exactly one case that does not fall back: user cancellation

```typescript
// Everything except an explicit user cancellation falls back, guaranteeing output
static shouldFallback(kind: LlmErrorKind): boolean {
  return kind !== 'cancelled';
}
```

Why skip fallback only on cancel? Because cancel means "I don't want it anymore" (e.g. the user backed out); silently generating then would be wasteful. **Every other anomaly — offline, timeout, bad key, rate limit, server fault, empty — falls back**, because the user's real goal is "see a report"; whether the LLM or the rules wrote it is secondary.

### 4.4 Why the orchestrator computes the fallback first

This is the detail I'm proudest of. The orchestrator's order is deliberate:

1. **Immediately, before any network call, compute the rule-based report** as a spare tire;
2. Only then run the three-condition gate and decide whether to call the LLM;
3. If streaming succeeds, use the AI text; if anything fails along the way, the spare is already there, so it can be handed over **instantly** — no last-minute computation, no error popup stare-down.

Compare this with "call the LLM first, compute rules only after it fails": my ordering drives the failure path's latency and uncertainty to zero. This is **Graceful Degradation** — when a capability is missing, the system doesn't crash wholesale but steps down to a slightly weaker yet fully working form.

### 4.5 Idempotent teardown, always destroy the request

Streaming has several end paths (`[DONE]`, `dataEnd`, error, cancel); careless handling double-fires callbacks and corrupts UI state. A `settled` boolean guarantees `onComplete/onError` fire **at most once**; on finish/cancel I uniformly `off()` the listeners and `destroy()` the request so no connection idles in the background — the same discipline as fixing the "timer leak" in post II: **whoever acquires a resource releases it on exit**.

## 5. API-key security: never hard-code, sandbox only

The most common, most fatal beginner mistake is hard-coding the key as a string. The moment you push to a public repo it leaks and anyone can burn your balance. My constraints:

- **Zero hard-coded keys**; non-secret config (endpoint, model, timeouts, storage keys) lives in `AiConfig`, but the key never does;
- The user enters the key in an in-app settings sheet; it is stored via `@kit.ArkData` `preferences` in the **app's private sandbox** (other apps can't read it), loaded once into memory at startup;
- The input uses a password field (dots) to prevent shoulder-surfing;
- It is only assembled into the `Authorization: Bearer <key>` header at request time and never reported to any third party.

```typescript
// Only non-secret config; the key is never here
static readonly BASE_URL = 'https://api.deepseek.com';
static readonly DEFAULT_MODEL = 'deepseek-chat';
static readonly PREFS_NAME = 'ai_settings';   // key lives in this private store
```

## 6. A small UI trap: the model's Markdown stars

On the first successful device run, the report showed `**Overall**`-style stars — the model follows the prompt's four sections and uses Markdown `**text**` for bold, which ArkUI's `Text` renders literally.

I neither "just deleted the stars" (headings would lose emphasis) nor pulled in a full Markdown library for this. Following **YAGNI (You Aren't Gonna Need It)** — the report needs only bold — I wrote ~40 lines, `MarkdownLite`: a pure function splits text on `**` into `bold`-tagged spans, and the UI uses `Text` with multiple `Span`s (bold/dark for headings, regular/gray for body), so **stars vanish while hierarchy survives**:

```typescript
Text() {
  ForEach(MarkdownLite.parseBoldSpans(this.weeklyReport), (span: TextSpan) => {
    Span(span.text)
      .fontWeight(span.bold ? FontWeight.Bold : FontWeight.Normal)
      .fontColor(span.bold ? '#2D3436' : Constants.COLOR_TEXT_SUB)
  })
}
```

It also handles an unclosed single `**`. Even better, since **the rule and AI reports share one display component**, this one fix covers both sources; the shared plain-text export strips the markers at its boundary too (plain text can't be bold, so it removes them cleanly). The pure function has 8 unit tests.

## 7. Testing trade-offs: 108 tests, not spread evenly

At this stage the project has 108 unit tests, but I did **not** force a test onto every class — I made a deliberate choice:

| Module | Needs network/system? | How verified |
|---|---|---|
| `ReportPromptBuilder` | No, pure | Tests: this/last-week split, delta, empty data, request body |
| `SseChunkParser` | No, pure | Tests: whole/partial/multi-line chunks, `[DONE]`, bad JSON |
| `ReportStrategy` | No, pure | Tests: 8 gate combinations, fallback per error kind |
| `HttpErrorMapper` | No, pure | Tests: classification of every status/framework code |
| `MarkdownLite` | No, pure | Tests: paired/inline/multiline/unclosed, plain strip |
| `DeepSeekClient` | Yes, real network | No forced mocks; verified on device (logic already sunk into the pure classes above) |
| `AiReportOrchestrator` | Yes, orchestrates system abilities | On-device walkthrough; every branch reuses already-tested pure functions |

The interview-ready point: **invest tests where logic is pure, error-prone, and offline-reproducible — the ROI is highest**. For a thin device layer that only "calls a system API and forwards already-tested results", forcing mocks produces brittle, low-information tests. This isn't missing coverage; it's a conscious strategy — and I can justify each layer's choice.

I also hit a meta-lesson about tests themselves: one case failed because I had **copied the expected value from the implementation**, and the implementation's "week start" was wrong — so the test was wrong in lockstep. The rule: **expected values must come from the spec or independent computation, never be copied from the code under test**, or a forever-green test guards nothing.

## 8. Interview angles

**Q1: How do you guarantee UX under weak/no network or API failure on-device?**
A: "Prepare the fallback first" — compute the local rule report up front, gate on toggle+key+online, normalize any streaming error into 8 kinds via `HttpErrorMapper`, and fall back to the ready-made rules for everything except an explicit cancel, honestly labelling the source and reason.

**Q2: How do you handle partial chunks and garbling in streaming?**
A: Two layers. At the byte level, stateful `TextDecoder.decodeWithStream` reassembles split characters; at the text level, `SseChunkParser` buffers non-whole lines, reads `data:` payloads, ends on `[DONE]`, and wraps `JSON.parse` so a bad chunk can't crash the flow. Both are pure and unit-tested.

**Q3: Why classify errors instead of a raw catch?**
A: Raw errors come from two systems (HTTP codes and numeric framework codes); normalizing them to a finite set of 8 semantic kinds makes fallback decisions and user messages uniform, testable, and maintainable.

**Q4: How do you keep the API key from leaking?**
A: No hard-coding; the user enters it in-app, it lives in the private sandbox preferences, is cached in memory, uses a password field, and is only placed in the Authorization header at request time — never reported elsewhere.

**Q5: Why not use an off-the-shelf Markdown library?**
A: YAGNI — only bold is needed; ~40 lines with zero dependencies, fully testable and controllable. A whole library adds bundle size and maintenance; extend gradually if headings/lists are ever truly needed.

**Q6: Why heavily test some classes but not the network one?**
A: Layer by testability — sink judgments into pure functions with high coverage; the thin device layer can't be reproduced offline and forced mocks add little, so it's verified on device while reusing tested pure functions. A deliberate trade-off, not a gap.

## 9. What's next

With Phase 2 (AI MVP), the rules + LLM hybrid is closed on-device: the AI streams a typewriter report when online and steps down to rules smoothly under any anomaly. In Phase 3 I'll pick one direction to go deep — a RAG personal knowledge base or a study-coach agent — so the AI doesn't just "talk" but gives advice **grounded in my own focus data and notes**, which is the real moat against tutorial demos. See you in post IV.
