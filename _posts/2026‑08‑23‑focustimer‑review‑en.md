
---
layout: post

title: "FocusTimer3: Build a Distributed Focus Timer with HarmonyOS NEXT ArkTS"

date: 2026‑08‑23

description: "End‑to‑end HarmonyOS NEXT practice project review. Implements distributed‑data‑object, offline‑queue fault‑tolerance, ArkTS static‑type programming under Stage Model."

categories: [HarmonyOS, Mobile‑Dev]

tags: [HarmonyOS‑NEXT, ArkTS, Distributed‑Data‑Object, Stage‑Model]

author: 725lizi

lang: en

---

# FocusTimer3 — A Focus‑Timer Application Built on HarmonyOS NEXT

> Date: 2026‑08‑23

> GitHub Repository: [https://github.com/725lizi/FocusTimer](https://github.com/725lizi/FocusTimer)

> Deliverable: Pre‑built HAP installation package for simulator quick‑run

## 1. Project Background & Objectives

Most existing focus‑tracking applications lack native cross‑device collaboration within the HarmonyOS ecosystem. FocusTimer3 is a productivity application built for study scenarios. Beyond basic timer functions, it practices core HarmonyOS capabilities including distributed data objects, local persistent storage, and offline‑queue fault‑tolerance.

**Core Project Goals**

1. Implement a full‑featured focus timer: custom countdown, start / pause control, floating timer card.
   
2. Weekly statistics dashboard, AI‑generated study reports and data visualization for focus duration.
 
3.Cross‑device data synchronization via HarmonyOS distributed data object. Timer status updates propagate between multiple simulators in real‑time.
 
4. Offline fallback mechanism: when distributed connection is unavailable, changes enter an offline queue and auto‑sync once connection recovers with retry logic.
   
5. Improve application robustness: handle edge‑cases including strict ArkTS type constraints, asynchronous lifecycle timing, UI rendering exceptions and object serialization.
  
6. Complete engineering delivery: source code, runtime screenshots, bug‑troubleshooting documentation, pre‑compiled HAP package.

## 2. Technology Stack

- Programming Language: **ArkTS** (static‑typed, no `any` / `unknown` allowed)
  
- UI Framework: ArkUI declarative UI, reusable components wrapped with `@Builder` decorato
  
- Application Model: Stage Model
  
- System Capabilities:
  
  - `distributedDataObject`: cross‑device distributed data storage
    
  - `preferences`: lightweight local persistent storage
    
  - `AppStorage`: global cross‑page application state management
    
  - `router`: page navigation, gesture menu `bindMenu`, chart rendering
    
  -` Version Control Tool`: Git & GitHub

## 3. Projentry/src/main/ets
├── pages # Business UI pages (Timer home, statistics page etc.)

├── common # Manager utilities: distributed manager, offline cache manager

├── model # Data interface definitions, type declarations, constants

├── entryability # Application entry point, responsible for distributed data initializationect Architecture


**Three‑layer Data Architecture**
1. Distributed Data Object: synchronize timer states across devices under normal network conditions.
2. AppStorage: global in‑memory snapshot for UI rendering.
3. Preferences Offline Queue: all modifications are persisted locally when distributed channel is disconnected. Batch flush and retry after connection restoration.

> Key insight: distributed data initialization is asynchronous. The `aboutToAppear` lifecycle of UI pages may execute **before** distributed objects finish creation, which will cause silent write failures if not handled properly.

## 4. Application Demo Screenshots
### 1. Main Focus‑Timer UI
![Main focus‑timer UI of FocusTimer3 application](assets/屏幕截图%202026‑08‑23%20103957.png)
Simulator features: set custom countdown, start timer, pause timer, entry for distributed transfer.

### 2. Floating Timer Popup Card
![Floating timer card popup component](assets/屏幕截图%202026‑08‑23%20104123.png)
Simulator features: popup floating timer card, display remaining time in real‑time.

### 3. Weekly Statistics Page
![Weekly statistics and AI study report page](assets/屏幕截图%202026‑08‑23%20104033.png)
Simulator features: weekly focus‑duration chart, AI‑generated study report, report share popup.

### 4. Multi‑simulator Distributed Sync Demo

![Multi‑simulator distributed data synchronization demo](assets/屏幕截图%202026‑08‑23%20104110.png)
Simulator features: HarmonyOS distributed data flow, timing data updates real‑time between two simulators.

## 5. Core Code Snippets

### 5.1 Strict Typing with Interfaces (ArkTS requirement)
ArkTS enforces explicit interfaces for object literals; implicit untyped objects are forbidden.
```typescript
export interface FocusRecord {
  id: number;
  duration: number;
  subject: string;
  createTime: number;
}
```
### 5.2 Async Distributed Initialization inside EntryAbility

Critical: use await to guarantee initialization finishes before page access.
```typescript
async onCreate() {
  await this.initDistributedData();
}

async initDistributedData() {
  const focusObj = distributedDataObject.create(this.context, { id: DataKeys.FOCUS_DATA });
  focusObj.on("change", (changed) => {
    AppStorage.setOrCreate("remoteFocusSnapshot", changed);
  })
  AppStorage.setOrCreate('focusDataObj', focusObj);
}
```

### 5.3 Offline Queue Serialization Rule

❌ Bad: store callback functions inside persisted queue items. Functions cannot be serialized by JSON and get silently discarded.

✅ Good: persist only plain serializable data; keep callbacks in runtime memory layer, store payload as JSON string.

### 5.4 Defensive Calculation for UI Rendering

Add protection for division‑by‑zero and NaN to avoid broken UI showing undefined%.
```typescript
@State progress: number = 0;

updateProgress(completed: number, total: number) {
  const raw = total > 0 ? (completed / total) * 100 : 0;
  this.progress = isNaN(raw) ? 0 : raw;
}
```
## 6. Development Challenges & Bug‑fix Summary

Full documentation is maintained inside the GitHub repository. Bugs are categorized into four groups: ArkTS compile constraints, module import issues, distributed & preferences runtime errors, UI rendering exceptions.


### Category 1: ArkTS Compile Constraints

1.arkts‑no‑untyped‑obj‑literals

Symptom: compile error when writing raw object literal without corresponding interface.

Root Cause: ArkTS requires every object literal to match defined class / interface.

Solution: define explicit interface for every data structure.

2.Ban of any type & index‑signature syntax

Solution: use Map<string, number> for dynamic key‑value scenarios.

3.Regex & string escape pitfalls

Symptom: literal \\n displayed on UI instead of line‑break.

Root Cause: redundant backslash escaping inside string literals.

Solution: use \n for newlines, enable .multiLine(true) on Text component.

### Category 2: Module Import & Path Problems

1.File system is case‑sensitive; mismatched filename case causes Cannot find module.

2.Forgetting export keyword leads to missing exported members.

3.Best Practice: rely on IDE auto‑import instead of manually writing relative paths.

### Category 3: Distributed Data & Preferences Runtime Bugs

1.Writing to distributed object with uninitialized instance takes no effect.

Root Cause: asynchronous lifecycle ordering between EntryAbility.onCreate and page aboutToAppear.

Solution: await initialization completion; add null‑check before accessing distributed objects inside pages.

2.getPreferencesSync returns null or throws exception.

Root Cause: passing null context inside utility classes.

Solution: context instance is created only in EntryAbility and passed down to utility modules. Do NOT acquire context inside helper classes.

3.Function / Symbol values inside offline queue items get lost during JSON serialization.

Solution: persist pure plain‑form data only; keep callback logic in runtime memory.

### Category 4: UI Rendering Runtime Exceptions

1.Missing @Builder decorator results in broken pop‑up dialog rendering.

2.Division‑by‑zero produces NaN value and corrupt progress display.

3.bindMenu menu renders empty when data array is still uninitialized in aboutToAppear.

## 7. Project Deliverables

### The GitHub repository contains complete artifacts:

1.Full ArkTS source code with layered engineering structure.

2.Multi‑scene simulator runtime screenshots.

3.Standardized bug‑troubleshooting document: "FocusTimer3 High‑frequency Error Record", organized as Phenomenon → Root Cause → Bad‑code Reproduction → Fix Solution.
Pre‑compiled HAP package, directly importable to HarmonyOS simulator without source‑code compilation.

## 8. Limitations & Future Improvements
   
1.Current implementation uses polling to observe remote snapshot; can be refactored to reactive state listening to eliminate polling and reduce power consumption.

2.Deprecated API warnings (e.g. pushUrl) will be migrated to pushPath in later iterations.

3.Refine retry & logging strategy for offline sync queue.

4.Improve multi‑device adaptive layout for tablets and foldable screens.

## 9. Project Takeaways
This end‑to‑end project practices HarmonyOS NEXT Stage‑model programming, distributed data object and local persistent storage.

Strict static typing of ArkTS prevents large amount of runtime null‑value and type‑mismatch bugs. The biggest challenge of HarmonyOS development is not UI component usage, but managing asynchronous lifecycles, cross‑device state synchronization and defensive handling for runtime edge‑cases.

This project is independently developed, covering requirement design, coding, debugging, bug documentation, Git version control and complete repository delivery.


## Related Links
- 📂 [Source Code Repository: FocusTimer on GitHub](https://github.com/725lizi/FocusTimer)
- 📝 [Read Chinese Version: FocusTimer3 鸿蒙专注计时器｜项目复盘](https://725lizi.github.io/2026/08/23/focustimer-review/)
- 🏠 [Back to Homepage](https://725lizi.github.io/)
 








