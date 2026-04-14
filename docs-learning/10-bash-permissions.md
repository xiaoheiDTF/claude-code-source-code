# Deep Dive: `src/tools/BashTool/bashPermissions.ts`

> **Lines:** 2,621 | **Role:** The security gatekeeper for every shell command

This module implements `bashToolHasPermission`, the function that decides whether a proposed Bash command is **allowed**, **denied**, or **ask-the-user** before it ever touches the operating system. It is one of the most security-critical files in the entire codebase, and it shows: extensive AST parsing, shadow testing, deny-rule precedence, sandbox integration, and a Haiku-based classifier.

---

## 1. High-Level Flow

```
input.command
     │
     ▼
┌─────────────────────┐
│ Tree-sitter parse   │◄── optional, gated by feature()
│ (parseCommandRaw)   │
└─────────┬───────────┘
     ┌────┴────┐
     ▼         ▼
 too-complex  simple
     │         │
     ▼         ▼
┌─────────┐ ┌─────────────┐
│ early   │ │ checkSemantics│
│ deny    │ │ (eval, zsh, │
│ check   │ │  etc.)      │
└────┬────┘ └──────┬──────┘
     │             │ fail
     │             ▼
     │        ┌─────────┐
     │        │ early   │
     │        │ deny    │
     │        │ check   │
     │        └────┬────┘
     │             │
     └─────────────┘
                   │
                   ▼
         ┌─────────────────┐
         │ Legacy fallback  │
         │ (shell-quote)    │◄── if tree-sitter unavailable
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Sandbox auto-allow│
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Exact-match rules │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Haiku classifier │◄── deny descriptions vs ask descriptions
         │ (parallel)       │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Prefix / wildcard│
         │ deny rules       │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Path / sed / mode│
         │ validators       │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Suggest rules   │
         │ Return allow/ask│
         └─────────────────┘
```

---

## 2. Tree-Sitter Shadow Mode

The file supports two feature gates:

* `TREE_SITTER_BASH` — use tree-sitter as the primary parser
* `TREE_SITTER_BASH_SHADOW` — run tree-sitter **in parallel** with the legacy path, log divergence, but still trust the legacy result

This is a textbook example of **safe migration** for a security-critical system.

```ts
let astRoot = injectionCheckDisabled
  ? null
  : feature('TREE_SITTER_BASH_SHADOW') && !shadowEnabled
    ? null
    : await parseCommandRaw(input.command)
```

When shadow mode is active, the code computes:

* `available` — did the WASM load?
* `tooComplex` — did the AST contain unanalyzable constructs?
* `semanticFail` — did `checkSemantics()` reject it?
* `subsDiffer` — do tree-sitter subcommands differ from the legacy `splitCommand`?

All of this is logged via `tengu_tree_sitter_shadow` for internal analysis, then the AST result is **discarded** and the legacy path continues.

---

## 3. AST Result Branches

### 3.1 `too-complex`

The parse succeeded, but the command contains structures that static analysis can't reason about: command substitution `$()`, parameter expansion, control flow, or parser differentials.

* **Deny rules are still checked** — a user who wrote `Bash(eval:*)` in their deny list expects it to block even complex invocations.
* If no deny matches, the result is **`ask`** with reason `astResult.reason`.
* In `BASH_CLASSIFIER` builds, a `pendingClassifierCheck` is attached so auto-mode can still fast-track obviously safe commands.

### 3.2 `simple`

The AST is a flat list of `SimpleCommand` objects. Before proceeding, `checkSemantics()` validates each command:

* No `eval`
* No `source` / `.` with untrusted input
* No `zsh` builtins that bypass normal security
* No `python -c` / `node -e` style inline code

If semantics fail, same pattern: early-deny check, then **`ask`**.

### 3.3 `parse-unavailable`

Tree-sitter is disabled, not built, or the killswitch is off. Fall back to the legacy `tryParseShellCommand` pre-check.

---

## 4. Legacy Shell-Quote Pre-Check

```ts
const parseResult = tryParseShellCommand(input.command)
if (!parseResult.success) {
  return { behavior: 'ask', reason: 'malformed syntax ...' }
}
```

This is a lightweight regex-based sanity check. It catches unbalanced quotes and obvious injection attempts without loading a full parser.

---

## 5. Permission-Rule Pipeline

After parsing, the command flows through a cascade of rule checks:

### 5.1 Exact Match
```ts
const exactMatchResult = bashToolCheckExactMatchPermission(input, context)
```

* Fastest path
* If deny → return immediately
* If allow → continue to prefix checks

### 5.2 Haiku Classifier (Parallel Deny + Ask)

If the user has configured **Bash prompt deny/ask rules** (the natural-language classifier), both sets are evaluated in parallel:

```ts
const [denyResult, askResult] = await Promise.all([
  hasDeny ? classifyBashCommand(..., denyDescriptions, 'deny') : null,
  hasAsk  ? classifyBashCommand(..., askDescriptions,  'ask')  : null,
])
```

* **High-confidence deny** → `deny`
* **High-confidence ask** → `ask`
* Otherwise → fall through

The classifier receives the raw command string, current working directory, the descriptions, and the mode ('deny' or 'ask').

### 5.3 Prefix / Wildcard Deny Rules

If no classifier hit, the command is tested against user-defined prefix and wildcard deny rules (e.g. `Bash(rm -rf:*)`).

---

## 6. `stripSafeWrappers` — The Most Heavily Commented Function

This function strips harmless prefixes so that `timeout 5s ls` is evaluated as `ls`:

* Safe env vars (`NODE_ENV=prod ...`)
* Wrapper commands (`timeout`, `time`, `nice`, `stdbuf`, `nohup`)
* Full-line bash comments

It is implemented in **two phases**:

1. Strip env vars and comments
2. Strip wrapper commands (but NOT env vars — because after a wrapper, `VAR=val cmd` means `VAR=val` is the *command argument* to the wrapper, not a shell-level assignment)

The regexes are paranoid:

* `[ \t]+` instead of `\s+` to avoid matching across newlines (newlines are command separators in bash)
* Enumerated GNU `timeout` flags with an allowlisted character class for values: `[A-Za-z0-9_.+-]`
* `(?:--[ \t]+)?` consumes the wrapper's own `--` separator so it isn't mistaken for a flag to the inner command

> *"SECURITY: flag VALUES use allowlist [A-Za-z0-9_.+-] (signals are TERM/KILL/9, durations are 5/5s/10.5). Previously [^ \t]+ matched $ ( ) ` | ; & — `timeout -k$(id) 10 ls` stripped to `ls`, matched Bash(ls:*), while bash expanded $(id) during word splitting BEFORE timeout ran."*

---

## 7. Safe Environment Variables

Two whitelists exist:

* `SAFE_ENV_VARS` — shipped to all users (~40 vars: `NODE_ENV`, `GOOS`, `RUST_LOG`, `TERM`, `ANTHROPIC_API_KEY`, etc.)
* `ANT_ONLY_SAFE_ENV_VARS` — internal-only (~25 vars: `KUBECONFIG`, `DOCKER_HOST`, `AWS_PROFILE`, `GH_TOKEN`, etc.)

The comment block is explicit:

> *"SECURITY: These must NEVER be added to the whitelist: PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_*, PYTHONPATH, NODE_PATH, CLASSPATH, RUBYLIB, GOFLAGS, RUSTFLAGS, NODE_OPTIONS, HOME, TMPDIR, SHELL, BASH_ENV"*

And for the ant-only list:

> *"This is INTENTIONALLY ANT-ONLY and MUST NEVER ship to external users. DOCKER_HOST redirects the Docker daemon endpoint — stripping it defeats prefix-based permission restrictions."*

---

## 8. Sandbox Auto-Allow Path

If the user has enabled sandboxing and "auto-allow bash if sandboxed", safe sandboxed commands can bypass the full rule pipeline:

```ts
if (SandboxManager.isSandboxingEnabled() &&
    SandboxManager.isAutoAllowBashIfSandboxedEnabled() &&
    shouldUseSandbox(input)) {
  const sandboxAutoAllowResult = checkSandboxAutoAllow(...)
  if (sandboxAutoAllowResult.behavior !== 'passthrough') {
    return sandboxAutoAllowResult
  }
}
```

This respects explicit deny rules — it's an *allow* shortcut, not a security bypass.

---

## 9. Rule Suggestion Engine

When the final verdict is `ask`, the module suggests permission rules the user can save:

```ts
suggestionForExactCommand(input.command)
```

It tries, in order:

1. **Heredoc prefix** — if the command uses `<<`, suggest the prefix before the heredoc operator
2. **First line** — for multiline commands, use the first line as a prefix rule
3. **Two-word prefix** — `git commit`, `npm run`, etc.
4. **Exact command** — fallback if none of the above apply

It also rejects dangerous bare prefixes (`bash:*`, `sh:*`, `sudo:*`, `env:*`, `nice:*`, etc.) that would be too broad to safely suggest.

---

## 10. Key Constants

```ts
const MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50
// ^ prevents ReDoS / event-loop starvation on deeply compound commands

const MAX_SUGGESTED_RULES_FOR_COMPOUND = 5
// ^ caps "Yes, and don't ask again for X, Y, Z…" UI labels
```

---

## 11. Relationship to Other Modules

| Module | Interaction |
|--------|-------------|
| `utils/permissions/permissions.ts` | Calls `bashToolHasPermission` as part of the 7-step `hasPermissionsToUseToolInner` pipeline |
| `utils/bash/ast.ts` | `parseForSecurityFromAst`, `checkSemantics` |
| `utils/bash/parser.ts` | `parseCommandRaw` (tree-sitter WASM) |
| `utils/bash/shellQuote.ts` | `tryParseShellCommand` (legacy regex fallback) |
| `utils/permissions/bashClassifier.ts` | `classifyBashCommand` (Haiku NLP classifier) |
| `utils/sandbox/sandbox-adapter.ts` | `SandboxManager` auto-allow integration |

---

## 12. Takeaways

* `bashPermissions.ts` is a **defense-in-depth fortress**: AST parsing, legacy fallback, sandboxing, exact rules, NLP classifier, prefix rules, and path validators all stack on top of each other.
* **Tree-sitter shadow mode** is a masterclass in migrating security-critical code: run the new system in production, gather telemetry, but keep the old path authoritative until you're certain.
* **`stripSafeWrappers`** contains some of the most paranoid regex engineering in the repo — every character class and whitespace choice is justified by a real security bug.
* The module is acutely aware of its dual audience: **external users** get a tight, safe experience, while **internal (`ant`) users** get extra env-var strippings for power-user convenience.
