# Deep Dive: `src/utils/sessionStorage.ts`

> **Lines:** 5,105 | **Role:** The entire session persistence, transcript I/O, and resume engine

This is one of the largest files in the codebase. It is the **persistence layer** for every conversation: writing JSONL transcripts, loading them back, managing metadata sidecars, handling compaction boundaries, and reconstructing conversation chains on resume.

---

## 1. Architecture Overview

At the heart of the module is a **`Project` singleton** that owns:

* **Per-file write queues** (`Map<string, Entry[]>`)
* **A flush scheduler** (`setTimeout`-based, 100ms interval)
* **Pending-write tracking** so callers can `await flush()`
* **Session-scoped metadata caches** (title, tag, mode, worktree, PR links, etc.)

All public functions are thin wrappers around `getProject()`.

```
Public API (saveMessage, loadTranscriptFile, etc.)
              │
              ▼
        ┌───────────┐
        │  Project  │◄── singleton
        │  instance │
        └─────┬─────┘
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 writeQueues  flushTimer  metadata caches
```

---

## 2. The Write Path: Batched, Async, Safe

### `enqueueWrite(filePath, entry)`

Every save operation appends to an in-memory queue rather than hitting the disk immediately.

### `scheduleDrain()`

A 100ms debounce timer batches writes. When it fires:

1. `splice(0)` all queued entries for a given file
2. `jsonStringify` each entry + newline
3. Chunk if the batch exceeds `MAX_CHUNK_BYTES` (100 MB)
4. `fsAppendFile` the chunk
5. Resolve per-entry promises so callers know their write landed

If `fsAppendFile` fails (directory doesn't exist), it `mkdir -p` and retries.

### `flush()`

Waits for `pendingWriteCount` to reach zero. Used by:

* Cleanup handlers (ensure metadata is on disk before exit)
* Compaction boundaries (must durably persist before truncating in-memory state)

---

## 3. Transcript File Format

Sessions are stored as **JSONL** (one JSON object per line) in:

```
~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl
```

Each line is an `Entry` — one of many types:

| Entry type | Purpose |
|------------|---------|
| `user` / `assistant` / `attachment` / `system` | The actual conversation transcript |
| `summary` | Post-compact summary, keyed by `leafUuid` |
| `custom-title` | User-defined session name |
| `tag` | User-defined tag |
| `agent-name` / `agent-color` / `agent-setting` | Subagent styling |
| `pr-link` | Linked PR metadata |
| `worktree-state` | Cwd worktree isolation state |
| `file-history-snapshot` | File-edit backups for undo |
| `attribution-snapshot` | Code attribution records |
| `content-replacement` | Subagent /resume decisions |
| `marble-origami-commit` | Context-collapse span commits |
| `marble-origami-snapshot` | Context-collapse full snapshot |

> **Progress entries are NOT transcript messages.** Old transcripts may still contain them; `loadTranscriptFile` bridges across them using a `progressBridge` map so parent UUID chains don't break.

---

## 4. The Read Path: `loadTranscriptFile(...)`

This is the most complex function in the file (~250 lines). It performs several optimizations to handle multi-gigabyte session files:

### 4.1 Pre-Compact Skip (`SKIP_PRECOMPACT_THRESHOLD` = 5 MB)

For large files, the function first scans backward from the end of the file looking for the **most recent compact boundary**. Everything *before* the boundary is dead history (it was summarized into a single system message). `readTranscriptForLoad` returns only the post-boundary buffer, skipping potentially hundreds of megabytes.

### 4.2 Pre-Boundary Metadata Recovery

When bytes are skipped, session-scoped metadata (title, tag, PR links, etc.) that was written *before* the boundary would be lost. The function performs a cheap forward scan of the skipped region to recover those metadata lines.

### 4.3 Chain Pruning Before Parse (`walkChainBeforeParse`)

Even after boundary truncation, the remaining buffer can still contain **orphaned fork branches** (dead tool-use paths, abandoned subagents). A second pass walks the `parentUuid` links and strips lines that belong to dead branches. This reduces parsing overhead and memory usage.

### 4.4 Legacy Progress Bridge

Old transcripts contain `type: 'progress'` entries with `uuid` and `parentUuid`. Because progress is no longer considered a transcript message, the loader builds a `progressBridge` map:

```ts
progressBridge.set(progressUuid, nearestNonProgressAncestor)
```

Any later message whose `parentUuid` points into the bridge is rewritten to point to the bridged ancestor.

### 4.5 Preserved-Segment Relinking (`applyPreservedSegmentRelinks`)

Some compact boundaries have a `preservedSegment` — a range of messages that *should* survive the compact (e.g. the most recent turn). These messages are physically pre-boundary on disk but logically post-boundary. The loader fixes their `parentUuid` chains after parsing.

### 4.6 Snip Removals (`applySnipRemovals`)

Messages marked as `snipped: true` by `snipCompact` are removed from the in-memory map.

---

## 5. Leaf Computation & Conversation Reconstruction

After loading, the file must answer: **"Where does the current conversation resume?"**

### Terminal Messages

Find all messages with **no children** (`!parentUuids.has(msg.uuid)`).

### Leaf Walk

From each terminal message, walk backward via `parentUuid` until we hit a `user` or `assistant` message that has **no user/assistant children**. That message is a leaf.

```ts
for (const terminal of terminalMessages) {
  walk backward {
    if (msg.type === 'user' || msg.type === 'assistant') {
      if (!hasUserAssistantChild.has(msg.uuid)) {
        leafUuids.add(msg.uuid)
        break
      }
    }
  }
}
```

This handles the case where the very last message on disk is a progress message, a system reminder, or an attachment — we don't want to resume from those.

### `buildConversationChain(messages, leafMessage)`

Once a leaf is chosen, the chain is rebuilt by following `parentUuid` links backward to the root. The result is the ordered transcript array that the REPL will display.

---

## 6. Agent & Remote-Agent Metadata Sidecars

Subagents and remote tasks write their identity to **sidecar files** so `--resume` can route correctly even when the subagent's `.jsonl` is empty or its `subagent_type` was omitted.

| Function | File path |
|----------|-----------|
| `writeAgentMetadata(agentId, metadata)` | `projects/<cwd>/<sessionId>/subagents/agent-<id>.meta.json` |
| `writeRemoteAgentMetadata(taskId, metadata)` | `projects/<cwd>/<sessionId>/remote-agents/remote-agent-<taskId>.meta.json` |

These files store:

* `agentType` — which agent definition to load
* `worktreePath` — restore cwd if isolation was used
* `description` — original task text (for UI notifications)
* For remote agents: `sessionId`, `title`, `command`, CCR linkage

---

## 7. Key Performance Optimizations

| Technique | Win |
|-----------|-----|
| Write queue + 100ms batching | Reduces fs syscalls from O(messages) to O(turns) |
| 100 MB chunk ceiling | Prevents `fsAppendFile` from choking on giant compaction boundaries |
| Pre-compact skip | Can reduce a 150 MB file read to ~5 MB |
| Pre-boundary metadata scan | Recovers session title/PR links without parsing dead history |
| `walkChainBeforeParse` | Avoids parsing JSON for orphaned fork branches |
| `readTranscriptForLoad` fd-level strip | Skips attribution-snapshot lines at the kernel level, never buffering them into JS |

---

## 8. Notable Edge Cases Handled

* **Cycles in `parentUuid` chains** — detected with a `Set<UUID>` during leaf walk; if found, the branch is treated as a dead end.
* **Multiple compact boundaries** — later boundaries clear earlier `contextCollapseCommits` arrays.
* **Empty session files** — silently returns empty maps; callers handle gracefully.
* **Worktree switches** — `getProjectDir` is memoized by cwd string, so switching worktrees just changes the cache key.
* **Graceful shutdown** — `registerCleanup` flushes queues and re-appends metadata so `--resume` always sees the latest title/tag.

---

## 9. Relationship to Other Modules

| Consumer | How it uses sessionStorage |
|----------|---------------------------|
| `query.ts` | `saveMessage()` on every assistant/user message |
| `QueryEngine.ts` | `loadTranscriptFromFile()` for `/resume` |
| `compact.ts` | Writes compact-boundary marker + summary entry |
| `contextCollapse/index.ts` | Writes `marble-origami-commit` and `-snapshot` |
| `AgentTool.tsx` | `getAgentTranscriptPath()`, `writeAgentMetadata()` |
| `useLogMessages.ts` | Calls `saveCustomTitle`, `saveTag`, etc. |

---

## 10. Takeaways

* `sessionStorage.ts` is the **database layer** of Claude Code. Every message, every compact boundary, every subagent identity flows through here.
* The module treats **transcript files as append-only logs** and uses aggressive **backward scanning + chain pruning** to make resume fast even when files grow to multiple gigabytes.
* The `Project` singleton's write-queue design is a classic **batching optimization** that trades a small amount of latency (≤100ms) for massive I/O efficiency.
* **Sidecar files** (`*.meta.json`) are a clever trick that lets the system resume subagents and remote tasks without changing the JSONL schema.
