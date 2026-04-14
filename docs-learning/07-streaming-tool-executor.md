# Deep Dive: `src/services/tools/StreamingToolExecutor.ts`

> **Lines:** 530 | **Role:** Concurrent tool scheduler & result streamer

This module is the **engine room of parallel tool execution**. While the Claude API is still streaming an assistant response, `StreamingToolExecutor` starts running tools the moment their `tool_use` blocks arrive, enforcing concurrency rules, handling cancellation cascades, and yielding progress/results in the correct order.

---

## 1. High-Level Architecture

```
Incoming tool_use blocks (as they stream in)
              │
              ▼
    ┌─────────────────────┐
    │     addTool()       │◄── parses input schema, determines concurrency safety
    └─────────┬───────────┘
              │
              ▼
    ┌─────────────────────┐
    │    processQueue()   │◄── starts tools when slots are free
    └─────────┬───────────┘
              │
         ┌────┴────┐
         ▼         ▼
   ┌─────────┐ ┌─────────┐
   │ Concurrent│ │ Exclusive│◄── non-concurrent tools run alone
   │  safe    │ │  tools   │
   └────┬────┘ └────┬────┘
        │           │
        ▼           ▼
   ┌─────────────────────┐
   │   executeTool()     │◄── wraps runToolUse(), collects results
   └─────────┬───────────┘
             │
             ▼
   ┌─────────────────────┐
   │ getCompletedResults │◄── yields messages in original order
   │ getRemainingResults │
   └─────────────────────┘
```

---

## 2. Core Data Structure: `TrackedTool`

```ts
type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: 'queued' | 'executing' | 'completed' | 'yielded'
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

* **`pendingProgress`** — progress messages (Bash output chunks, Sleep ticks, etc.) are buffered here so they can be yielded **immediately** without waiting for the tool to finish.
* **`contextModifiers`** — some tools (e.g. `FileEditTool`) mutate `readFileState` or `fileHistoryState`. These modifiers are applied to `toolUseContext` *after* the tool completes, but only for **non-concurrent** tools.

---

## 3. Concurrency Rules

```ts
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executing = this.tools.filter(t => t.status === 'executing')
  return (
    executing.length === 0 ||
    (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  )
}
```

| Scenario | Outcome |
|----------|---------|
| No tools running | Anything can start |
| Only concurrent-safe tools running | Another concurrent-safe tool may join |
| A non-concurrent tool is running | Queue is blocked until it finishes |
| A concurrent-safe tool wants to start while a non-concurrent tool runs | It waits |

This guarantees that tools like `Bash` (which can be concurrent-safe for read-only commands) run in parallel, while `FileEditTool` (non-concurrent) gets exclusive access to the working directory state.

---

## 4. The Three Abort Controllers

`StreamingToolExecutor` manages a **hierarchy of cancellation signals**:

1. **`toolUseContext.abortController`** (parent)  
   *Fired when the user interrupts the turn (types a new message) or when the outer query loop decides to abort.*

2. **`siblingAbortController`** (child of parent)  
   *Fired when **any** Bash tool errors. It kills *other* running Bash subprocesses immediately, but **does not** abort the parent — the turn can continue with the remaining tools returning synthetic errors.*

3. **`toolAbortController`** (child of sibling)  
   *Per-tool child. If a permission dialog is cancelled, this bubbles up to the parent so the entire turn ends. If a sibling Bash errors, this receives `reason === 'sibling_error'`.*

```
parent (query.ts)
   └── sibling ( StreamingToolExecutor )
          └── tool-1
          └── tool-2
          └── tool-3
```

---

## 5. Synthetic Error Messages

When a tool is aborted before or during execution, it doesn't run at all. Instead it receives a **synthetic error** so the model still gets a `tool_result` block:

| Reason | Synthetic message |
|--------|-------------------|
| `sibling_error` | `Cancelled: parallel tool call <name> errored` |
| `user_interrupted` | `REJECT_MESSAGE` (shows "User rejected edit" in UI) |
| `streaming_fallback` | `Streaming fallback - tool execution discarded` |

The `thisToolErrored` flag prevents a tool that *itself* produced an error from receiving a duplicate sibling-error message.

---

## 6. Streaming Results: Generators in Action

`query.ts` consumes tools through two generator methods:

### `getCompletedResults()` (sync generator)
Yields any `completed` tools, **in the order they were received**, plus all pending progress messages. Called repeatedly inside the main `for await` loop so the UI can update while tools are still running.

### `getRemainingResults()` (async generator)
Called after the API stream ends. It waits for the queue to drain, yielding results and progress as they become available. The wait strategy:

1. Run `processQueue()` to start any queued tools that are now unblocked
2. Yield all completed results
3. If nothing is ready but tools are still executing, `Promise.race([...executingPromises, progressPromise])`
4. Repeat until every tool status is `yielded`

---

## 7. Progress Message Fast-Path

Progress messages are **not** sent to the LLM; they are UI-only ephemera. Because users want to see Bash output or Sleep countdowns update in real time, they get special treatment:

```ts
if (update.message.type === 'progress') {
  tool.pendingProgress.push(update.message)
  if (this.progressAvailableResolve) {
    this.progressAvailableResolve()   // wake getRemainingResults immediately
    this.progressAvailableResolve = undefined
  }
}
```

This means a long-running Bash command can emit dozens of progress updates while a concurrent Read tool is still executing, and the UI receives both streams without waiting for either to finish.

---

## 8. Interrupt Behavior

Tools can declare an `interruptBehavior`:

* `'cancel'` — if the user presses ESC or sends a new message while this tool is running, it is aborted and the turn continues
* `'block'` — the abort is ignored; the tool runs to completion (default)

The executor tracks whether **all currently executing tools** are cancelable:

```ts
this.toolUseContext.setHasInterruptibleToolInProgress?.(
  executing.every(t => this.getToolInterruptBehavior(t) === 'cancel')
)
```

This boolean drives the REPL's "interruptible" spinner UI state.

---

## 9. Discard Mode

When a **streaming fallback** happens (e.g. model overload triggers a retry with a different model), the executor receives `discard()`. All queued tools are never started, and in-progress tools receive synthetic `streaming_fallback` errors. This prevents stale results from leaking into the retried turn.

---

## 10. Key Code Snippets

### Concurrency gate inside `processQueue`
```ts
for (const tool of this.tools) {
  if (tool.status !== 'queued') continue
  if (this.canExecuteTool(tool.isConcurrencySafe)) {
    await this.executeTool(tool)
  } else {
    if (!tool.isConcurrencySafe) break   // maintain order for exclusive tools
  }
}
```

### Bash error → cascade siblings
```ts
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.erroredToolDescription = this.getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')
}
```

### Permission-cancel bubble-up
```ts
toolAbortController.signal.addEventListener('abort', () => {
  if (
    toolAbortController.signal.reason !== 'sibling_error' &&
    !this.toolUseContext.abortController.signal.aborted &&
    !this.discarded
  ) {
    this.toolUseContext.abortController.abort(toolAbortController.signal.reason)
  }
}, { once: true })
```

---

## 11. Takeaways

* `StreamingToolExecutor` is the **crossroads** between API streaming and tool execution.
* Its concurrency model is deliberately conservative: **parallelism is opt-in** per tool, and non-concurrent tools block the queue.
* The **three-layer abort hierarchy** cleanly separates "user wants to stop everything" from "one Bash failed so kill its siblings" from "this one tool was denied".
* Progress messages bypass the normal "wait for completion" flow, giving the UI real-time updates without breaking the message-order contract.
