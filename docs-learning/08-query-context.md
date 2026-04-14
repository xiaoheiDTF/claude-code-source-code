# Deep Dive: `src/utils/queryContext.ts`

> **Lines:** 179 | **Role:** System-prompt assembler & cache-key builder

This tiny but critical module is the **single source of truth** for how the three pieces of a Claude API cache-key prefix are built:

1. `systemPrompt` (the big instruction block)
2. `userContext` (cwd, git status, etc.)
3. `systemContext` (rare internal additions)

It exists in its own file specifically to **avoid import cycles**: `context.ts` and `constants/prompts.ts` are high in the dependency graph, and placing these helpers inside `systemPrompt.ts` or `sideQuestion.ts` would create circular imports.

---

## 1. `fetchSystemPromptParts(...)`

```ts
export async function fetchSystemPromptParts({
  tools,
  mainLoopModel,
  additionalWorkingDirectories,
  mcpClients,
  customSystemPrompt,
}): Promise<{
  defaultSystemPrompt: string[]
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
}>
```

### Behavior

| `customSystemPrompt` | `defaultSystemPrompt` | `systemContext` |
|----------------------|----------------------|-----------------|
| `undefined` (normal) | Fetched from `getSystemPrompt(...)` | Fetched from `getSystemContext()` |
| Set by user | `[]` (skipped) | `{}` (skipped) |

When the user supplies a **custom system prompt**, it *replaces* the default entirely. Appending `systemContext` to a default that isn't being used would be meaningless, so the function short-circuits both.

All three fetches run in parallel via `Promise.all`.

### Callers

* `QueryEngine.ts` — builds the full prefix before every `submitMessage()`
* `cli/print.ts` — side-question fallback resume path

---

## 2. `buildSideQuestionFallbackParams(...)`

```ts
export async function buildSideQuestionFallbackParams({
  tools,
  commands,
  mcpClients,
  messages,
  readFileState,
  getAppState,
  setAppState,
  customSystemPrompt,
  appendSystemPrompt,
  thinkingConfig,
  agents,
}): Promise<CacheSafeParams>
```

### The Problem It Solves

The SDK can fire a **side question** mid-turn (before the main loop's turn completes). At that point there is **no `stopHooks` snapshot** and therefore `getLastCacheSafeParams()` returns `null`. If we can't rebuild the cache prefix, the side question would fail entirely.

### The Solution

This function **mirrors the system-prompt assembly logic** from `QueryEngine.ts:ask()` so the rebuilt prefix matches what the main loop will send, preserving the prompt-cache hit in the common case.

```ts
const systemPrompt = asSystemPrompt([
  ...(customSystemPrompt !== undefined
      ? [customSystemPrompt]
      : defaultSystemPrompt),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

### Edge Case: In-Progress Assistant Message

If the last message in `messages` is an assistant message with `stop_reason === null`, it means the SDK interrupted us *while the model was still streaming*. That half-finished message must be stripped from the fork context, or the side question would include a malformed assistant block.

```ts
const last = messages.at(-1)
const forkContextMessages =
  last?.type === 'assistant' && last.message.stop_reason === null
    ? messages.slice(0, -1)
    : messages
```

### Trade-Off Acknowledged in Code

> *"May still miss the cache if the main loop applies extras this path doesn't know about (coordinator mode, memory-mechanics prompt). That's acceptable — the alternative is returning null and failing the side question entirely."*

---

## 3. Anatomy of the Built `ToolUseContext`

The function also constructs a **minimal `ToolUseContext`** for the side-question handler:

```ts
const toolUseContext: ToolUseContext = {
  options: {
    commands,
    debug: false,
    mainLoopModel,
    tools,
    verbose: false,
    thinkingConfig: thinkingConfig ?? adaptive/default,
    mcpClients,
    mcpResources: {},
    isNonInteractiveSession: true,   // ← side questions are non-interactive
    agentDefinitions: { activeAgents: agents, allAgents: [] },
    customSystemPrompt,
    appendSystemPrompt,
  },
  abortController: createAbortController(),
  readFileState,
  getAppState,
  setAppState,
  messages: forkContextMessages,
  setInProgressToolUseIDs: () => {},  // no-op stubs for side-question scope
  setResponseLength: () => {},
  updateFileHistoryState: () => {},
  updateAttributionState: () => {},
}
```

Many of the callback fields (`setInProgressToolUseIDs`, etc.) are no-ops because a side question is a **short-lived fork**; it doesn't own the REPL UI state.

---

## 4. Why This File Exists (The Cycle Story)

The file-level comment explains the architectural constraint:

> *"Lives in its own file because it imports from `context.ts` and `constants/prompts.ts`, which are high in the dependency graph. Putting these imports in `systemPrompt.ts` or `sideQuestion.ts` (both reachable from `commands.ts`) would create cycles. Only entrypoint-layer files import from here (`QueryEngine.ts`, `cli/print.ts`)."*

This is a classic **dependency-inversion** move: the low-level prompt builders stay in their own high-level module, and this thin adapter sits above them, reachable only from the true entrypoints.

---

## 5. Takeaways

* `queryContext.ts` is the **canonical assembler** for the cache-key prefix that determines prompt-cache hit rates.
* It knows about **custom system prompts** and handles the "replace vs append" semantics correctly.
* `buildSideQuestionFallbackParams` is a **defensive rebuild** that lets SDK side questions survive even when the main loop hasn't produced a snapshot yet.
* The module's entire reason for being is **cycle avoidance** — a reminder that in large TypeScript codebases, file placement is often an architectural decision.
