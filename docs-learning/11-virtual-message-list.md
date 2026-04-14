# Deep Dive: `src/components/VirtualMessageList.tsx`

> **Lines:** 1,082 | **Role:** Virtualized transcript renderer with search & sticky headers

This component is the **performance-critical heart of the fullscreen REPL UI**. It virtualizes a potentially enormous message list (tens of thousands of messages), supports incremental search (`/foo`, `n`/`N`), sticky prompt headers, and keyboard-driven cursor navigation — all on top of a custom Ink-based rendering layer.

---

## 1. Why Virtualization?

Claude Code sessions can run for hours or days, accumulating **27,000+ messages**. Rendering all of them as Yoga nodes would freeze the terminal. `VirtualMessageList` mounts only the messages visible in the viewport (plus overscan), using the `useVirtualScroll` hook to compute heights, offsets, and spacers dynamically.

```
Viewport
├─ topSpacer (unmounted messages above)
├─ MessageRow [start]
├─ MessageRow
├─ ...
├─ MessageRow [end-1]
└─ bottomSpacer (unmounted messages below)
```

---

## 2. Key Props

| Prop | Purpose |
|------|---------|
| `messages` | `RenderableMessage[]` — the full transcript |
| `scrollRef` | Handle to the `ScrollBox` for imperative scroll |
| `columns` | Terminal width; invalidates the height cache on resize |
| `itemKey` | Stable React key for each message |
| `renderItem` | Factory that returns the actual `MessageRow` JSX |
| `extractSearchText` | Pre-lowered search text (zero-allocation `indexOf` loop) |
| `jumpRef` | `useImperativeHandle` exposing search & jump APIs |
| `cursorNavRef` | Keyboard cursor navigation (`j`/`k`, `PgUp`/`PgDn`) |
| `trackStickyPrompt` | Enables the sticky-prompt header tracker |
| `scanElement` / `setPositions` | Position-based search highlight overlay |

---

## 3. `useVirtualScroll` Integration

The component delegates geometry to `useVirtualScroll`:

```ts
const {
  range,        // [start, end) indices currently mounted
  topSpacer,    // height in rows of skipped content above
  bottomSpacer, // height in rows of skipped content below
  measureRef,   // ref callback for measuring each mounted row
  offsets,      // cumulative pixel offsets for every message
  getItemTop,   // pixel top of message i in the content wrapper
  getItemElement, // DOM element for message i (if mounted)
  scrollToIndex // imperative scroll by index
} = useVirtualScroll(scrollRef, keys, columns)
```

### Incremental `keys` Optimization

Rebuilding the full `keys` array on every message append is `O(n)` and caused ~1MB of churn at 27k messages. The component uses an append-only delta strategy:

```ts
if (messages.length < keysRef.current.length || messages[0] !== prevMessagesRef.current[0]) {
  keysRef.current = messages.map(m => itemKey(m))   // full rebuild on compaction/clear
} else {
  for (let i = keysRef.current.length; i < messages.length; i++) {
    keysRef.current.push(itemKey(messages[i]!))     // delta append
  }
}
```

---

## 4. Search Engine (`JumpHandle`)

The component exposes a rich search API via `useImperativeHandle(jumpRef)`:

```ts
type JumpHandle = {
  jumpToIndex: (i: number) => void
  setSearchQuery: (q: string) => void
  nextMatch: () => void
  prevMatch: () => void
  setAnchor: () => void
  warmSearchIndex: () => Promise<number>
  disarmSearch: () => void
}
```

### 4.1 `warmSearchIndex`

Extracts and caches the lowered search text for every message in **500-message chunks** with `await sleep(0)` yields between them. This prevents blocking the event loop on huge transcripts and lets the UI paint an "indexing…" spinner first.

### 4.2 `setSearchQuery`

Performs a linear scan over all messages, counting occurrences with `indexOf`:

```ts
let pos = text.indexOf(lq)
let cnt = 0
while (pos >= 0) {
  cnt++
  pos = text.indexOf(lq, pos + lq.length)
}
```

It builds:

* `matches: number[]` — message indices that contain the query
* `prefixSum: number[]` — cumulative occurrence count for the "3/47" badge

The nearest message to the current scroll position is chosen as the starting `ptr`, and `jump(ptr, wantLast=true)` is invoked.

### 4.3 Two-Phase Seek (`jump` → `useEffect`)

Because Ink/Yoga layout is deferred, we can't scan the DOM until after paint. The seek is split into two phases:

1. **`jump(i, wantLast)`** — scrolls to the message (mounted if needed) and bumps `seekGen`
2. **`useEffect([seekGen])`** — runs **post-paint**, reads the mounted DOM element, runs `scanElement(el)`, and highlights the first (or last) match position

If the element isn't mounted after `scrollToIndex` (rare), it retries once, then auto-advances to the next match.

### 4.4 Phantom-Burst Guard

Sometimes the engine counts a match (`indexOf` found it in the lowered text) but the render scan doesn't find it (ghost messages, collapsed groups, etc.). Auto-advancing blindly could loop forever. A `phantomBurstRef` caps consecutive phantoms at 20 before giving up.

### 4.5 Position-Based Highlight

Search matches are highlighted using **screen-absolute row positions** rather than string indices:

```ts
const positions = scanElement?.(el) ?? []   // { row, col } relative to message top
const rowOffset = vpTop + (top - scrollTop) // message's current screen-top
setPositions?.({ positions, rowOffset, currentIdx: ord })
```

This lets the overlay renderer draw yellow underlines precisely, even for wrapped lines and complex message layouts.

---

## 5. `StickyTracker` — The Floating Header

`StickyTracker` is rendered **as a separate child component** so it can subscribe to scroll at a **finer granularity** than the list itself.

### Why a Separate Component?

The list uses `SCROLL_QUANTUM = 40` to batch Yoga relayouts during fast scrolling. If the sticky header shared that quantum, it would lag by ~one conversation turn. By splitting it out, `StickyTracker` re-renders on every scroll tick while the list stays batched.

### How It Works

1. Walk backward from `end` to `start` to find the first message whose **top is above the viewport target**
2. Continue walking backward to find the **most recent real user prompt** before that visible message
3. If the prompt's `❯` glyph is still visible (`top + 1 >= target`), skip to the *older* prompt (avoid duplication)
4. Call `setStickyPrompt({ text: collapsed, scrollTo: ... })`

### Click-to-Scroll & Two-Phase Correction

When the user clicks the sticky header, it should scroll back to the original prompt. If the prompt is far above the viewport and not currently mounted, the handler:

1. Sets `pending.current = { idx, tries: 0 }`
2. Calls `scrollRef.current?.scrollTo(estimate)` to mount the message
3. A second `useEffect` fires on the **next render**, sees the mounted element, and calls `scrollToElement(el, 1)` for pixel-perfect alignment

A suppression state machine (`'none' | 'armed' | 'force'`) prevents the sticky header from immediately re-appearing while the scroll animation is in flight.

---

## 6. Cursor Navigation (`cursorNavRef`)

Keyboard navigation (Vim-style `j`/`k`, `gg`/`G`) is exposed via another imperative handle:

```ts
{
  enterCursor:   () => scan(messages.length - 1, -1, isUser)
  navigatePrev:  () => scan(selIdx - 1, -1)
  navigateNext:  () => scan(selIdx + 1, 1)   // or exit cursor mode + scrollToBottom
  navigateTop:   () => scan(0, 1)
  navigateBottom:() => scan(messages.length - 1, -1)
}
```

`scan(from, dir, predicate)` walks the message array looking for the next message that satisfies both `isVisible(i)` and the optional predicate. When found, it updates `setCursor` so the UI can render a selection highlight and auto-scroll.

---

## 7. Performance Optimizations

| Technique | Impact |
|-----------|--------|
| Incremental `keysRef` | Avoids `O(n)` rebuild on every streaming append |
| `VirtualItem` stable callbacks | Reduces closure allocation from ~1,800/sec to near-zero during scroll |
| `fallbackLowerCache` WeakMap | Caches lowered search text per message object; auto-GCs on compaction |
| `warmSearchIndex` chunking | Non-blocking index warmup over huge transcripts |
| `StickyTracker` separate component | Decouples fine-grained header updates from coarse list relayouts |
| `useSyncExternalStore` in tracker | Subscribes to raw scroll events without re-rendering the parent |

---

## 8. React Compiler Annotations

The file uses the React Compiler (`_c` cache arrays) extensively inside `VirtualItem`. While the compiler handles memoization automatically, the code still relies on **stable callback identity** through `useRef` patterns:

```ts
const handlersRef = useRef({ onItemClick, setHoveredKey })
const onClickK = useCallback((msg, cellIsBlank) => {
  const h = handlersRef.current
  if (!cellIsBlank && h.onItemClick) h.onItemClick(msg)
}, [])
```

This ensures that `VirtualItem` props don't churn on every parent render, allowing the compiler's memo bailouts to actually fire.

---

## 9. Relationship to Other Modules

| Module | Interaction |
|--------|-------------|
| `hooks/useVirtualScroll.ts` | Computes geometry, range, and scroll offsets |
| `ink/components/ScrollBox.ts` | Imperative scroll API (`scrollTo`, `scrollToElement`, `subscribe`) |
| `components/Messages.tsx` | Parent that decides whether to render `VirtualMessageList` or a plain map |
| `components/messageActions.ts` | `isNavigableMessage`, `stripSystemReminders`, `toolCallOf` |
| `utils/transcriptSearch.ts` | `renderableSearchText` — extracts plain text for indexing |
| `components/FullscreenLayout.tsx` | Provides `ScrollChromeContext` for sticky header state |

---

## 10. Takeaways

* `VirtualMessageList` is a **full-featured virtual-scrolling list** built from first principles on top of Ink's custom reconciler.
* Its search engine is deliberately **simple but robust**: linear scan + `indexOf`, two-phase DOM seek, position-based highlights, and phantom-burst protection.
* **Sticky prompt tracking** is split into its own component to break the coarse-quantum batching that keeps the main list fast.
* Every optimization in this file is driven by **real measured pain**: closure GC pressure, `O(n)` key rebuilds, event-loop blocking during index warmup, and Yoga relayout thrashing.
