---
layout: post
title: "Integrating an LLM into a HarmonyOS App (IV): An On-Device RAG Knowledge Base That Has Only Read My Own Notes"
date: 2026-09-02
description: "On-device RAG on HarmonyOS NEXT / ArkTS: Chinese-aware chunking, OpenAI-compatible embeddings (BGE, 1024-d), local Top-K cosine retrieval, relational persistence, an anti-hallucination planning gate and citations — why DeepSeek has no embeddings API, and why brute-force retrieval is enough at personal scale (168 unit tests)."
categories: [HarmonyOS, Mobile Development]
tags: [HarmonyOS NEXT, ArkTS, RAG, Vector Retrieval, Embedding, BGE, Cosine Similarity, Anti-Hallucination, DeepSeek, Unit Testing]
author: 725lizi
lang: en
---

# Integrating an LLM into a HarmonyOS App (IV): An On-Device RAG Knowledge Base That Has Only Read My Own Notes

> This is the fourth post in the FocusTimer3 series. The [first post](./2026‑08‑23‑focustimer‑review‑en.html) made the app **work**; the [second post](./2026-08-30-focustimer-refactor-engineering-en.html) made it **well-structured and testable** and left a "replaceable AI interface"; the [third post](./2026-09-01-focustimer-llm-streaming-fallback-en.html) integrated an LLM that survives offline conditions. This post tackles a more fundamental problem: **a general-purpose LLM has never read my notes — how do I make it answer from my own data?** The answer is RAG.
>
> Source code: [https://github.com/725lizi/FocusTimer](https://github.com/725lizi/FocusTimer)
> 中文版：[FocusTimer3 接大模型实战（四）](./2026-09-02-focustimer-rag-personal-knowledge-base.html)

## 0. TL;DR

I built a **personal knowledge base** on the phone: my course / English / math notes are chunked and turned into vectors stored locally. On a question, the app first **retrieves** the few most relevant chunks from my own notes, then asks the LLM to answer **using only those chunks**, citing which notes it used. If the notes contain nothing relevant, it honestly says "not found" instead of making things up.

| Stage | Approach | Key point / guarantee |
|---|---|---|
| Chinese chunking | split on sentence punctuation, greedy packing, ≤220 chars/chunk, 30-char overlap | pure `TextChunker`, 8 tests (hard-cut/overlap/empty) |
| Embedding | OpenAI-compatible `/embeddings`, BGE Chinese model, 1024-d | vendor-switchable; dim validation, batch ≤16, tolerant parser (19 tests) |
| Local retrieval | in-memory cosine similarity, Top-5 | pure logic, 10 tests; millisecond-level and fully offline at personal scale |
| Persistence | new `rag_vector_chunk` relational table, vectors as JSON | hydrate on launch; delete-then-insert per source = atomic rebuild |
| Anti-hallucination | similarity gate + char budget + "use only the material" prompt | pure `RagQaPlanner`, 10 tests; grounded vs no_evidence |
| Overall | **168 unit tests** project-wide (60 added this stage) | pure logic offline, thin network layer on-device |

## 1. Why RAG, Not Fine-Tuning or Stuffing Everything into the Prompt

An LLM has read the public internet but never my phone notes. There are three common ways to give it private data, and I made an explicit trade-off:

- **Fine-tuning** keeps training the model on my notes. It needs compute and batches, updates slowly, and I only want to *look up* notes, not change model behavior — overkill, and impossible on-device.
- **Stuffing everything (long context)** concatenates all notes into every prompt. It quickly exceeds context length, is slow and expensive, and suffers from "lost in the middle" attention dilution.
- **RAG (Retrieval-Augmented Generation)** first retrieves the few chunks most relevant to the question and sends only those. **Retrieval finds the evidence; generation phrases the answer.** Updating data only means rebuilding the index — the model never changes.

For a personal study app, RAG is the only approach that is light, current, and explainable. In plain words, it is an **open-book exam: the model is the student, my notes are the textbook, retrieval flips to the relevant pages, and every claim must trace back to those pages.**

## 2. A Selection Trap: DeepSeek Does Not Provide Embeddings

RAG needs two distinct model calls, which beginners often assume one model covers:

1. **Embedding model**: turns text into a vector (numbers) for semantic similarity — it *generates no text*;
2. **Chat model**: generates the answer from material + question.

I checked the official docs before coding: **DeepSeek offers chat completions but no embeddings endpoint.** So the correct combo is "**DeepSeek for chat, a separate OpenAI-compatible service for embeddings**." I wrote the embedding client against the standard OpenAI protocol (`POST /embeddings`, request `{model, input}`, response `data[].embedding`), defaulting to SiliconFlow's `BAAI/bge-large-zh-v1.5` (strong Chinese retrieval, directly reachable in China, free tier for individuals). Switching to Zhipu or Alibaba Bailian later means changing only config:

```typescript
static readonly EMBED_BASE_URL = 'https://api.siliconflow.cn/v1';
static readonly EMBED_PATH = '/embeddings';
static readonly DEFAULT_EMBED_MODEL = 'BAAI/bge-large-zh-v1.5';
static readonly EXPECTED_DIM = 1024;   // fixed output dim, used to validate responses
static readonly MAX_BATCH = 16;        // at most 16 texts per request
```

> An interview-worthy point: **verify capability boundaries first, then program against a protocol rather than a vendor**, locking the variable parts into config so no third-party SDK is welded into business logic. The embedding key and chat key are also stored in two separate sandboxed preferences.

## 3. Data Flow: Two Flows, One Shared Store

![On-device RAG data flow](/assets/focustimer-rag-flow-en.svg)

- **Indexing flow (once when notes change / on manual rebuild)**: local unencrypted notes → Chinese chunking → batched embedding (network) → align chunks with vectors → write to both the in-memory store and the relational table.
- **QA flow (per question)**: prepend BGE instruction → embed the query (one network call) → **local in-memory** Top-K cosine retrieval (offline, milliseconds) → anti-hallucination planning → if grounded, DeepSeek streams an answer with citations; otherwise an honest "not found".

I deliberately made index rebuild a **manual button** rather than "embed on every note save": it saves traffic and free quota, is stable on weak networks, and makes demos controllable. **Encrypted notes are never indexed** — privacy by design at the source.

## 4. Chinese-Aware Chunking: Why Not Hard-Cut by Fixed Length

The unit of embedding and retrieval is a **chunk**, not a whole note. Too large mixes topics and dilutes the vector; too small severs a sentence mid-thought. English has spaces; Chinese does not, and my notes are full of `。；！？` punctuation, so chunking must be Chinese-aware:

1. split into sentences on end punctuation (keeping it);
2. greedy packing: keep adding sentences until 220 chars, then start a new chunk;
3. keep a **30-char overlap** between adjacent chunks so a concept straddling a boundary is still retrievable from both sides;
4. for an ultra-long string with no punctuation, fall back to a sliding window (size 220, step 220−30), guaranteeing **no lost characters and no infinite loop** (overlap is clamped to `[0, maxChars-1]`, so step is never 0).

```typescript
static readonly DEFAULT_MAX_CHARS = 220;
static readonly DEFAULT_OVERLAP_CHARS = 30;
// when overlap >= limit it is clamped so the step is never 0
```

This is pure, device-free logic, locked by 8 tests feeding it normal text, punctuation-free runs, empty strings, and out-of-range overlap. Chunking is the foundation of retrieval quality, so the foundation must be provably correct offline.

## 5. Embedding Client: Keeping Untrustworthy Network Output Out

Embedding is the only batch network request here, and the client does several engineering-grade things:

- **batching**: groups chunks by `MAX_BATCH=16` to avoid oversized request bodies;
- **dimension validation**: every vector must be exactly 1024-d, otherwise it is rejected so no dirty data enters the store;
- **query instruction**: BGE officially recommends a prefix on *queries* but not on *indexed documents*, which improves Chinese matching; the rule lives in config:

```typescript
static buildQueryText(rawQuery: string, isQuery: boolean): string {
  const q = rawQuery.trim();
  return isQuery ? RagConfig.BGE_QUERY_INSTRUCTION + q : q; // prefix on queries only
}
```

- **tolerant parsing**: the JSON may miss fields or return `data` out of order; `EmbeddingParser` (9 tests) extracts safely, realigns by index, and reports errors instead of crashing.

## 6. Local Retrieval: Why Brute Force Is Optimal at Personal Scale

### 6.1 Cosine similarity measures meaning by direction

Each text is a 1024-d vector; the smaller the angle between two vectors, the closer the meaning. Cosine similarity ranges −1..1, closer to 1 meaning more relevant:

```typescript
function cosine(a: number[], b: number[]): number {
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) { dot += a[i]*b[i]; na += a[i]*a[i]; nb += b[i]*b[i]; }
  const d = Math.sqrt(na) * Math.sqrt(nb);
  return d === 0 ? 0 : dot / d;
}
```

Retrieval computes cosine between the query vector and **every** chunk, drops negatives and dimension mismatches, sorts descending, and takes Top-5 (K=5).

### 6.2 Why no dedicated vector database

Industrial RAG deals in millions/ billions of vectors and needs ANN indexes like HNSW. But **how big are personal notes?** 200 notes × 10 chunks is only 2000 chunks. At that scale exact brute force is both simpler and more accurate. I benchmarked the equivalent algorithm in Node on my dev machine (scope: local chunking and cosine only, no network):

| Operation | Scale | Avg time |
|---|---|---|
| chunk one note | 2000 chars → 10 chunks | 0.14 ms |
| local Top-5 retrieval | 200 chunks × 1024-d | 0.17 ms |
| local Top-5 retrieval | 500 chunks × 1024-d | 0.45 ms |
| local Top-5 retrieval | 1000 chunks × 1024-d | 1.07 ms |
| local Top-5 retrieval | 2000 chunks × 1024-d | 2.40 ms |

Even at two thousand knowledge chunks, local retrieval takes a couple of milliseconds, is **fully offline and costs no traffic**; the only network round-trips in a query are embedding the question and generating the answer. This is a deliberate **complexity-matches-scale** decision — don't introduce a heavy component you can't maintain until linear scan actually becomes the bottleneck (YAGNI).

I added lightweight timing logs (a single `[RAG-Perf]` prefix, filterable in DevEco's HiLog) and measured the network part on a real device walkthrough (emulator, SiliconFlow BGE + DeepSeek, 2026-09-02):

| On-device stage | Scale / result | Time |
|---|---|---|
| Rebuild index (read+chunk+embed+persist) | 1 note / 2 chunks | 735 ms |
| Query embedding (network) + local retrieval + planning | 2 hits, mode=grounded | 464 ms |
| Send → first token (TTFT) | —— | 1327 ms |
| Full answer completed | 141 chars | 2105 ms |

Compared with the offline table above, the latency breakdown is clear: **local chunking/retrieval is sub-millisecond to a few milliseconds, and almost all user-perceived wait comes from the two network round-trips.** Query embedding plus local retrieval and planning took 464ms; the first token appeared at 1.3s and the complete answer at 2.1s, which feels smooth with token streaming. The `mode=grounded, used=2` log line is direct evidence that the answer was grounded in *my own two note chunks* rather than a generic response.

### 6.3 In-memory + relational: single source of truth, survives restart

An in-memory `VectorStore` serves retrieval (fast); a new relational table `rag_vector_chunk` persists vectors as JSON with an index on `source_id` (survives restart). On launch the app hydrates the table back into memory. A single facade `RagRuntime` is exposed, so **indexing and retrieval share one data source**, avoiding divergent copies. Rebuilding a note deletes its old chunks then inserts new ones — atomic replacement, no duplicate hits.

## 7. Anti-Hallucination: The Real Differentiator

Where RAG fails most often is not "nothing retrieved" but **retrieving a sliver and then the model inventing facts that aren't in the notes**. A pure planner `RagQaPlanner` applies three gates *before* calling the LLM:

1. **Similarity gate (AnswerDecider)**: if the best score is below 0.2 it is `no_evidence` — never fake a successful retrieval;

```typescript
static readonly DEFAULT_MIN_SCORE = 0.2;
static decide(scored: ScoredChunk[], minScore = 0.2): QaMode {
  if (scored.length === 0 || scored[0].score < minScore) return 'no_evidence';
  return 'grounded';
}
```

2. **Char budget (ContextBuilder.fitBudget)**: context is finite, so fill hits most-relevant-first and stop at 1200 chars (keeping at least one), dropping the tail;
3. **Citation aggregation + prompt constraint**: multiple chunks from the same note merge into one citation (one blue source chip in the UI), and the system prompt "locks the model inside the material":

```typescript
static readonly SYSTEM_GROUNDED =
  'You are the user\'s personal study assistant and may answer ONLY from the provided personal material. ' +
  '1) Do not invent facts absent from the material; 2) if it is insufficient, say so explicitly; ' +
  '4) cite the material id at the end of each sentence, e.g. [material 1].';
// no_evidence: first state "not found in your notes", then give clearly-labelled general advice, never faking a citation
```

The system therefore has two honest end states: **grounded** (answer with sources) and **no_evidence** (state the gap, then give clearly-labelled general advice). Gates two and three are belt-and-suspenders — even if the threshold is off, the prompt still forbids fabrication. Every AI answer lists the exact notes it used — **traceable and verifiable**, the "show your sources" of an open-book exam.

## 8. Testability: Where the 60 New Tests Go

Continuing the "extract pure logic, keep the device layer thin" discipline, this stage adds 60 offline tests:

| Pure module | What it covers | Tests |
|---|---|---|
| `TextChunker` | packing, hard-cut, overlap clamping, empty, no-loss | 8 |
| `VectorMath` | cosine direction, zero vector, orthogonal, dim mismatch | 10 |
| `EmbeddingParser` | normal parse, missing fields, out-of-order realign, bad dim | 9 |
| `VectorStore` + `VectorRetriever` | add/delete, Top-K, negative filter, empty store | 10 |
| Index pipeline (assemble/codec/batch/persist) | alignment, atomic rebuild, JSON round-trip, batch edge | 13 |
| `RagQaPlanner` (budget/gate/citations/prompt) | grounded/no_evidence, budget cut, same-note merge | 10 |

The genuinely networked `EmbeddingClient` and `RagQaService` are thin — send request, call tested pure functions, callback — and are verified on a real device. **Tests pay off most where logic is pure, error-prone, and offline-reproducible**; I can justify each layer's choice rather than chasing a coverage number.

## 9. Two Real-Device UI Bugs (Worth Telling Interviewers)

Neither is about algorithms, but both reveal an understanding of declarative UI:

1. **No back button in the note editor**: long notes pushed the bottom nav off-screen. Fix: conditionally render a "‹ back" in the editor's top bar that exits edit mode without discarding the draft.
2. **The question TextInput was not editable**: I had bound `.enabled(!this.busy())` — a **custom method call** — to a common attribute. ArkUI refreshes UI by directly tracking `@State` during render and **cannot track state read hidden inside a method**, so the input could be judged disabled on first render. Switching to an explicit `@State isAsking` was still flaky; finally, comparing with a known-working sibling layout, I **removed the `enabled` binding entirely** (duplicate-send prevention lives in logic) and it worked.

> The rule that emerged: **in declarative UI, bind `if`/attributes only to `@State` or constant expressions, never to custom method calls**; compute combined judgments at the point state changes and store them in an explicit state. Comparing against a proven-working sibling component is often faster than staring at the docs.

## 10. Interview Perspective

**Q1: RAG vs fine-tuning vs long-context stuffing — why RAG?**
A: Fine-tuning changes model capability at high cost and slow update, wrong for looking up private notes; stuffing everything overflows context and dilutes attention; RAG retrieves only the relevant chunks, updates data by rebuilding the index, and is light and explainable — the best complexity-match for a personal app.

**Q2: Are embedding and chat the same model?**
A: No. Embedding turns text into vectors and generates nothing; chat generates. DeepSeek has no embeddings API, so I use DeepSeek for chat and an OpenAI-compatible BGE model for embeddings, programming against the protocol so switching vendors is config-only.

**Q3: Why overlap in chunking? How is chunk size chosen?**
A: To avoid a concept split at a boundary being missed by both sides; 220 chars balances single-topic focus and semantic completeness, with a no-punctuation hard-cut fallback whose step is never zero and loses no characters.

**Q4: Which vector database did you use?**
A: None. At ~2000 personal chunks, brute-force 1024-d cosine takes ~2.4ms and is fully offline; ANN indexes like HNSW exist for web-scale data and would be over-engineering here — an explicit scale-based trade-off with a clear evolution path.

**Q5: How does RAG prevent hallucination?**
A: Three gates — a 0.2 similarity floor decides no_evidence; a 1200-char budget keeps only the most relevant material; a system prompt forces "material-only, say when insufficient, cite ids", and the UI lists verifiable sources. No-evidence cases honestly state the gap instead of forcing an answer.

**Q6: Where are vectors stored, how do they survive restart, how is read/write consistency kept?**
A: An in-memory VectorStore serves search; a relational `rag_vector_chunk` table persists JSON with a source_id index and hydrates on launch. One `RagRuntime` facade makes indexing and retrieval share a single source; per-source delete-then-insert gives atomic rebuild and no duplicates.

**Q7: Why many unit tests for some modules and few for network modules?**
A: Chunking, vector math, retrieval and planning are pure, error-prone and offline-reproducible, hence high coverage; the thin network layer only sends/forwards already-tested results and is verified on-device — faking mocks there adds little. A deliberate testing investment, not a gap.

## 11. What's Next

With stage 3, the on-device RAG knowledge base closes the loop: for the first time, the AI's answers carry citations from my own notes. Stage 4 will weave RAG together with focus data and the AI weekly report into one coherent story, add real on-device performance numbers, complete the architecture diagrams, and polish résumé and interview narratives — forming a portfolio that progresses from engineering refactor → hybrid AI → on-device RAG, with every layer ready to be probed in depth before junior-year summer internship recruiting. See you in post five.
