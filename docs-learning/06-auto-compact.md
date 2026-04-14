# Deep Dive: `src/services/compact/autoCompact.ts`

> **Lines:** 351 | **Role:** Context-window health monitor & proactive compactor

This module is the **garbage collector of the conversation context window**. It decides *when* the transcript has grown too large for the current model, and *how* to shrink it before the API throws a `prompt_too_long` error. It sits on the critical path of every turn in `query.ts`.

---

## 1. Design Goals

| Goal | How it's achieved |
|------|-------------------|
| **Proactive prevention** | Trigger compaction well before the hard API limit |
| **Session-memory first** | Try the cheap heuristic prune before paying for an LLM summary |
| **Fail-safe** | Circuit-breaker stops retrying when context is irrecoverably over-limit |
| **Feature-gate hygiene** | Heavy use of `feature()` and `process.env` overrides so internal experiments don't leak into public builds |

---

## 2. Key Constants & Thresholds

```ts
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000   // Reserve headroom for the model to emit a summary
const AUTOCOMPACT_BUFFER_TOKENS       = 13_000 // Fire ~13k tokens early
const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000 // UI "orange" warning
const ERROR_THRESHOLD_BUFFER_TOKENS   = 20_000 // UI "red" warning
const MANUAL_COMPACT_BUFFER_TOKENS    = 3_000  // Hard blocking limit for manual /compact
```

**Effective window** = `contextWindowForModel(model) - min(maxOutputTokens, 20k)`

**Autocompact threshold** = `effectiveWindow - 13_000`

> *Example: For Claude 3.5 Sonnet (200k window) the threshold is ~167k tokens — leaving plenty of room for both the summary and the user's next turn.*

---

## 3. `getEffectiveContextWindowSize(model)`

This is the **canonical source of truth** for "how many tokens can this conversation safely occupy before we must take action". It supports two environment overrides:

* `CLAUDE_CODE_AUTO_COMPACT_WINDOW` — caps the window (useful for local testing with smaller models)
* `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` — sets the threshold as a percentage of the window instead of the fixed 13k buffer

---

## 4. Token Warning State Machine

```ts
calculateTokenWarningState(tokenUsage, model)
```

Returns five booleans:

1. `isAboveAutoCompactThreshold` — should we compact *now*?
2. `isAboveWarningThreshold` — should the UI show a yellow "running low" hint?
3. `isAboveErrorThreshold` — should the UI show a red "critical" hint?
4. `isAtBlockingLimit` — should we *refuse* to send another API call until the user runs `/compact`?
5. `percentLeft` — human-readable headroom for the UI badge

---

## 5. `shouldAutoCompact(...)` — The Guard Tower

Before any expensive work happens, a long list of **early-exit guards** runs:

| Guard | Reason |
|-------|--------|
| `querySource === 'session_memory'` | Avoid deadlock: session-memory compaction is itself a forked agent |
| `querySource === 'compact'` | Same deadlock risk |
| `querySource === 'marble_origami'` | Don't nuke the main thread's committed log from inside the ctx-agent |
| `DISABLE_AUTO_COMPACT` env | User escape hatch |
| `userConfig.autoCompactEnabled === false` | Persistent user preference |
| `REACTIVE_COMPACT` feature + killswitch | When reactive-only mode is on, suppress proactive autocompact and let 413 recovery handle it |
| `CONTEXT_COLLAPSE` feature + enabled | Collapse IS the context-management system; autocompact would race it |

If all guards pass, the function compares `tokenCountWithEstimation(messages) - snipTokensFreed` against the threshold.

> **Why `snipTokensFreed`?**  
> `snipCompact` removes stale messages, but the surviving assistant message still carries the *pre-snip* token count in its usage metadata. Subtracting the rough delta prevents false negatives where we think we're under the limit even though `snip` already did work.

---

## 6. `autoCompactIfNeeded(...)` — The Orchestrator

```ts
if (circuitBreakerTripped) return { wasCompacted: false }
if (!shouldAutoCompact)     return { wasCompacted: false }
```

If compaction is needed, it builds a `RecompactionInfo` object (tracks how many turns since the *last* compact, whether we're in a re-compaction chain, etc.) and then tries **two strategies in order**:

### 6.1 Strategy 1: Session-Memory Compaction
```ts
const sessionMemoryResult = await trySessionMemoryCompaction(...)
```

This is a **cheap heuristic prune**: it deletes tool-result blocks, old progress messages, and other low-signal content *without* calling the LLM. If it frees enough tokens, we're done.

On success:
* `setLastSummarizedMessageId(undefined)` — old UUIDs no longer exist
* `runPostCompactCleanup(querySource)` — resets context-collapse state, cache baselines, etc.
* `notifyCompaction(...)` — analytics only

### 6.2 Strategy 2: LLM Summarization (`compactConversation`)

If session memory didn't save enough tokens, we fall back to the expensive path: asking the model to summarize the conversation into a single system message.

```ts
const compactionResult = await compactConversation(
  messages, toolUseContext, cacheSafeParams,
  true,   // suppressUserQuestions
  undefined, // no custom instructions
  true,   // isAutoCompact
  recompactionInfo,
)
```

On success: failure counter resets to `0`.  
On failure (e.g. `prompt_too_long` *during* compaction): failure counter increments. After **3 consecutive failures** the circuit breaker trips and autocompact is silenced for the rest of the session.

> **BQ 2026-03-10:** *1,279 sessions had 50+ consecutive failures, wasting ~250K API calls/day globally.* The circuit breaker was added directly in response to this telemetry.

---

## 7. Anatomy of a Compaction Turn

```
┌─────────────────┐
│  Start of turn  │
└────────┬────────┘
         │
         ▼
┌──────────────────────────┐
│ shouldAutoCompact?       │◄── guards + token check
└────────┬─────────────────┘
         │ yes
         ▼
┌──────────────────────────┐
│ trySessionMemoryCompaction│◄── cheap prune
└────────┬─────────────────┘
    success│          │fail
         ▼          ▼
┌─────────────┐  ┌─────────────────────┐
│   Done      │  │ compactConversation │◄── LLM summary
└─────────────┘  └────────┬────────────┘
                    success│     │fail
                         ▼     ▼
                    ┌──────┐ ┌─────────────┐
                    │ Done │ │ +1 failure  │
                    │      │ │ (circuit    │
                    │      │ │  breaker?)  │
                    └──────┘ └─────────────┘
```

---

## 8. Relationship to the Rest of the System

| Consumer | How it uses this module |
|----------|------------------------|
| `query.ts` | Calls `autoCompactIfNeeded` at the top of every loop iteration; threads `autoCompactTracking` through `State` |
| `compact.ts` | The heavy LLM summarizer that this module orchestrates |
| `sessionMemoryCompact.ts` | The cheap heuristic pruner |
| `contextCollapse/index.ts` | Mutual exclusion: when collapse is active, autocompact steps aside |
| `postCompactCleanup.ts` | Resets module-level state (collapsed spans, cache baselines, etc.) after any compaction |

---

## 9. Takeaways

* **Autocompact is the first line of defense** against context-window overflow; reactive compact (413 recovery) is the second.
* **Session-memory compaction** is the secret sauce that lets most long conversations survive without ever paying for an LLM summary.
* **The circuit breaker** is a blunt but necessary weapon against infinite retry loops when a session is genuinely wedged.
* **Every public build string** related to internal experiments is wrapped in `feature()` so it gets dead-code-eliminated by Bun.
