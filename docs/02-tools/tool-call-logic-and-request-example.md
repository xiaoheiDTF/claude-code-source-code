# Claude Code 工具调用逻辑 & 完整请求示例

## 一、工具调用处理流程

源码: `src/services/tools/toolExecution.ts` → `checkPermissionsAndCallTool()`

```
Claude API 返回 tool_use block
        │
        ▼
┌─────────────────────────────────┐
│  1. findToolByName()             │  按名称查找工具
│     找不到 → 返回错误给 Claude    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  2. inputSchema.safeParse()      │  Zod schema 验证
│     失败 → 返回 InputValidationError │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  3. tool.validateInput()         │  工具自定义验证
│     (例如 Edit 检查文件是否读过)   │
│     失败 → 返回具体错误信息        │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  4. backfillObservableInput()    │  填充派生字段
│     (例如 expandPath 展开路径)    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  5. runPreToolUseHooks()         │  执行前置 Hooks
│     hook 可修改 input / 拒绝执行  │
│     可返回 stop → 终止执行        │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  6. resolveHookPermissionDecision│  合并 Hook 权限决策
│     hook 已决定 → 使用 hook 结果  │
│     hook 未决定 → 进入正常流程    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  7. canUseTool()                 │  用户权限检查
│     ├─ allow → 直接执行          │
│     ├─ deny → 返回拒绝信息       │
│     └─ ask → 弹窗让用户确认       │
│         用户允许 → 执行           │
│         用户拒绝 → 返回拒绝信息    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  8. tool.call()                  │  实际执行工具
│     ┌───────────────────────┐    │
│     │ onProgress() 回调      │    │  实时进度报告
│     │ (终端输出/文件读取等)   │    │
│     └───────────────────────┘    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  9. runPostToolUseHooks()        │  执行后置 Hooks
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│ 10. 返回 ToolResult              │
│     ├─ tool_result block          │  正常结果
│     │  (文件内容/diff/命令输出)   │
│     ├─ newMessages               │  附带消息 (Skill)
│     └─ contextModifier           │  修改上下文 (工具白名单等)
└─────────────────────────────────┘
```

### 并发控制逻辑

源码: `src/services/tools/toolOrchestration.ts`

```
Claude 返回多个 tool_use blocks
        │
        ▼
  partitionToolCalls()
  按 isConcurrencySafe 分组:
        │
        ├── Batch 1: [Read, Glob, Grep]  (只读, 并发安全)
        │     → runToolsConcurrently()     并行执行
        │
        ├── Batch 2: [Edit]               (写操作, 非并发安全)
        │     → runToolsSerially()         串行执行
        │
        ├── Batch 3: [Read, Read]         (只读)
        │     → runToolsConcurrently()     并行执行
        │
        └── Batch 4: [Write]              (写操作)
              → runToolsSerially()         串行执行
```

## 二、完整 API 请求示例

### 场景: 用户说 "把 foo.ts 的 return 1 改成 return 2"

#### 第一轮请求 (读文件)

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 16384,
  "stream": true,
  "system": [
    {
      "type": "text",
      "text": "x-anthropic-billing-header: cc_version=1.0.33.abc123; cc_entrypoint=vscode;"
    },
    {
      "type": "text",
      "text": "You are Claude Code, Anthropic's official CLI for Claude.",
      "cache_control": { "type": "ephemeral" }
    },
    {
      "type": "text",
      "text": "You are an interactive agent that helps users with software engineering tasks...\n\n# System\n - All text you output outside of tool use is displayed to the user...\n\n# Doing tasks\n - The user will primarily request you to perform software engineering tasks...\n\n# Executing actions with care\nCarefully consider the reversibility...\n\n# Using your tools\n - Do NOT use the Bash to run commands when a relevant dedicated tool is provided...\n\n# Tone and style\n - Only use emojis if the user explicitly requests it...\n\n# Output efficiency\nIMPORTANT: Go straight to the point...",
      "cache_control": { "type": "ephemeral", "scope": "global" }
    },
    {
      "type": "text",
      "text": "# Session-specific guidance\n - If you do not understand why the user has denied a tool call...\n\n# Environment\nYou have been invoked in the following environment:\n - Primary working directory: /home/user/project\n - Is a git repository: Yes\n - Platform: linux\n - You are powered by the model named Claude Sonnet 4.6..."
    }
  ],
  "tools": [
    {
      "name": "Read",
      "strict": true,
      "description": "Reads a file from the local filesystem. You can access any file directly by using this tool.\nAssume this tool is able to read all files on the machine...\n\nUsage:\n- The file_path parameter must be an absolute path, not a relative path\n- By default, it reads up to 2000 lines starting from the beginning of the file\n- Results are returned using cat -n format, with line numbers starting at 1\n- This tool allows Claude Code to read images\n- This tool can read PDF files (.pdf)\n- This tool can read Jupyter notebooks (.ipynb files)",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": {
            "type": "string",
            "description": "The absolute path to the file to read"
          },
          "offset": {
            "type": "integer",
            "description": "The line number to start reading from",
            "minimum": 0
          },
          "limit": {
            "type": "integer",
            "description": "The number of lines to read",
            "exclusiveMinimum": 0
          }
        },
        "required": ["file_path"]
      }
    },
    {
      "name": "Edit",
      "strict": true,
      "description": "Performs exact string replacements in files.\n\nUsage:\n- You must use your `Read` tool at least once in the conversation before editing...\n- When editing text from Read tool output, ensure you preserve the exact indentation (tabs/spaces) as it appears AFTER the line number prefix...\n- ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.\n- The edit will FAIL if `old_string` is not unique in the file...\n- Use `replace_all` for replacing and renaming strings across the file.",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": {
            "type": "string",
            "description": "The absolute path to the file to modify"
          },
          "old_string": {
            "type": "string",
            "description": "The text to replace"
          },
          "new_string": {
            "type": "string",
            "description": "The text to replace it with (must be different from old_string)"
          },
          "replace_all": {
            "type": "boolean",
            "description": "Replace all occurrences of old_string (default false)"
          }
        },
        "required": ["file_path", "old_string", "new_string"]
      }
    },
    {
      "name": "Write",
      "strict": true,
      "description": "Writes a file to the local filesystem.\n\nUsage:\n- This tool will overwrite the existing file if there is one at the provided path.\n- If this is an existing file, you MUST use the Read tool first...\n- Prefer the Edit tool for modifying existing files — it only sends the diff...\n- NEVER create documentation files (*.md) or README files unless explicitly requested.",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": {
            "type": "string",
            "description": "The absolute path to the file to write (must be absolute, not relative)"
          },
          "content": {
            "type": "string",
            "description": "The content to write to the file"
          }
        },
        "required": ["file_path", "content"]
      }
    },
    {
      "name": "Bash",
      "strict": true,
      "description": "Executes a given bash command and returns its output.\n\nThe working directory persists between commands, but shell state does not...\n\nIMPORTANT: Avoid using this tool to run `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, or `echo` commands...",
      "input_schema": {
        "type": "object",
        "properties": {
          "command": {
            "type": "string",
            "description": "The command to execute"
          },
          "timeout": {
            "type": "number",
            "description": "Optional timeout in milliseconds (up to 600000)"
          },
          "description": {
            "type": "string",
            "description": "Clear, concise description of what this command does"
          }
        },
        "required": ["command"]
      }
    },
    {
      "name": "Glob",
      "description": "Fast file pattern matching tool that works with any codebase size...",
      "input_schema": {
        "type": "object",
        "properties": {
          "pattern": {
            "type": "string",
            "description": "The glob pattern to match files against"
          },
          "path": {
            "type": "string",
            "description": "The directory to search in"
          }
        },
        "required": ["pattern"]
      }
    },
    {
      "name": "Grep",
      "description": "A powerful search tool built on ripgrep...",
      "input_schema": {
        "type": "object",
        "properties": {
          "pattern": {
            "type": "string",
            "description": "The regular expression pattern to search for"
          },
          "path": {
            "type": "string",
            "description": "File or directory to search in"
          },
          "output_mode": {
            "type": "string",
            "enum": ["content", "files_with_matches", "count"]
          },
          "glob": {
            "type": "string",
            "description": "Glob pattern to filter files"
          }
        },
        "required": ["pattern"]
      }
    },
    {
      "name": "Agent",
      "description": "Launch a new agent to handle complex, multi-step tasks autonomously...",
      "input_schema": { "...": "..." }
    },
    {
      "name": "Skill",
      "description": "Execute a skill within the main conversation...",
      "input_schema": { "...": "..." }
    },
    {
      "name": "TodoWrite",
      "description": "Use this tool proactively to track progress...",
      "input_schema": { "...": "..." }
    },
    {
      "name": "AskUserQuestion",
      "description": "Use this tool when you need to ask the user questions...",
      "input_schema": { "...": "..." }
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": "把 foo.ts 的 return 1 改成 return 2"
    }
  ]
}
```

#### 第一轮响应 (Claude 返回)

```json
{
  "id": "msg_01ABC",
  "type": "message",
  "role": "assistant",
  "model": "claude-sonnet-4-6",
  "content": [
    {
      "type": "text",
      "text": "让我先读取文件。"
    },
    {
      "type": "tool_use",
      "id": "toolu_01",
      "name": "Read",
      "input": {
        "file_path": "/home/user/project/src/foo.ts"
      }
    }
  ],
  "stop_reason": "tool_use"
}
```

#### 工具执行结果 (Read)

工具执行后，`tool_result` 作为 user message 追加：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01",
      "content": "     1\tfunction add(a: number, b: number): number {\n     2\t  return 1\n     3\t}\n     4\t\n     5\tconsole.log(add(1, 2))"
    }
  ]
}
```

#### 第二轮请求 (编辑文件)

Claude 收到文件内容后，发出 Edit 调用：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "现在修改返回值。"
    },
    {
      "type": "tool_use",
      "id": "toolu_02",
      "name": "Edit",
      "input": {
        "file_path": "/home/user/project/src/foo.ts",
        "old_string": "  return 1",
        "new_string": "  return 2"
      }
    }
  ]
}
```

#### Edit 工具内部执行过程

```
1. validateInput():
   ✓ old_string ("  return 1") !== new_string ("  return 2")
   ✓ 文件存在
   ✓ readFileState 有记录 (刚才 Read 过)
   ✓ mtime 未变 (新鲜度检查通过)
   ✓ findActualString() 精确匹配
   ✓ 只匹配到 1 处 (唯一性通过)

2. call():
   a. readFileForEdit() → 读取文件内容、编码、换行符
   b. 二次新鲜度检查 → 通过
   c. findActualString() → "  return 1" 精确匹配
   d. preserveQuoteStyle() → 无引号风格变化
   e. getPatchForEdit() → 生成 diff:
      - 2行: "  return 1" → "  return 2"
   f. writeTextContent() → 写入磁盘
   g. LSP 通知 → didChange + didSave
   h. readFileState 更新 → 新的 timestamp + content

3. 返回结果:
   {
     filePath: "/home/user/project/src/foo.ts",
     oldString: "  return 1",
     newString: "  return 2",
     structuredPatch: [{ oldStart: 2, oldLines: 1, newStart: 2, newLines: 1, ... }],
     userModified: false,
     replaceAll: false
   }
```

#### Edit 工具结果返回给 Claude

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_02",
      "content": "The file /home/user/project/src/foo.ts has been updated successfully."
    }
  ]
}
```

#### 第三轮响应 (Claude 完成任务)

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "已将 [foo.ts:2](src/foo.ts#L2) 的 `return 1` 改为 `return 2`。"
    }
  ],
  "stop_reason": "end_turn"
}
```

## 三、错误场景示例

### 场景: 未先 Read 就尝试 Edit

```json
// Claude 请求:
{
  "type": "tool_use",
  "id": "toolu_03",
  "name": "Edit",
  "input": {
    "file_path": "/home/user/project/src/bar.ts",
    "old_string": "hello",
    "new_string": "world"
  }
}

// 工具返回 (validateInput 阶段失败):
{
  "type": "tool_result",
  "tool_use_id": "toolu_03",
  "is_error": true,
  "content": "<tool_use_error>File has not been read yet. Read it first before writing to it.</tool_use_error>"
}
```

### 场景: old_string 不唯一

```json
// Claude 请求:
{
  "type": "tool_use",
  "id": "toolu_04",
  "name": "Edit",
  "input": {
    "file_path": "/home/user/project/src/foo.ts",
    "old_string": "return",
    "new_string": "return 0"
  }
}

// 工具返回:
{
  "type": "tool_result",
  "tool_use_id": "toolu_04",
  "is_error": true,
  "content": "<tool_use_error>Found 3 matches of the string to replace, but replace_all is false. To replace all occurrences, set replace_all to true. To replace only one occurrence, please provide more context to uniquely identify the instance.\nString: return</tool_use_error>"
}
```

### 场景: 文件被外部修改 (新鲜度检查失败)

```json
// 工具返回:
{
  "type": "tool_result",
  "tool_use_id": "toolu_05",
  "is_error": true,
  "content": "<tool_use_error>File has been unexpectedly modified. Read it again before attempting to write it.</tool_use_error>"
}
```
