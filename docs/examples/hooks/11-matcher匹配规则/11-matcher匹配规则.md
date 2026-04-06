# Hook: matcher 匹配规则

> 指南 4.6 — matcher 字段决定 Hook 作用于哪些工具或来源

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json              ← Hook 配置
│   └── hooks/
│       ├── edit-only.sh           ← 单工具匹配
│       ├── edit-write.sh          ← 多工具匹配
│       ├── all-tools.sh           ← 全工具匹配
│       ├── bash-filter.sh         ← Bash 子命令过滤
│       └── skip-generated.sh      ← 排除特定文件
├── src/
├── generated/
└── package.json
```

## settings.json 配置

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/edit-only.sh"
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/edit-write.sh"
          }
        ]
      },
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/all-tools.sh"
          }
        ]
      }
    ]
  }
}
```

## 说明

```
matcher 匹配规则:

  匹配对象:
    PreToolUse / PostToolUse / PostToolUseFailure → 匹配工具名
    UserPromptSubmit / SessionStart / SessionEnd → 匹配来源（source）

  语法:
    "Edit"         → 只匹配 Edit 工具
    "Edit|Write"   → 匹配 Edit 或 Write（正则 OR）
    "Bash"         → 只匹配 Bash
    ""             → 空字符串 = 匹配所有
    ".*"           → 正则通配符 = 匹配所有
    "Edit|Write|Bash" → 匹配这三种工具

  匹配行为:
    - 使用正则表达式匹配
    - 同一事件可以有多个 matcher 组
    - 每个 matcher 组可以有多个 hooks
    - 所有匹配的 hooks 都会执行
```

## 使用示例

### 示例 1：单工具匹配 — 只在 Edit 时触发

```
settings.json:
  PreToolUse → matcher: "Edit"
  hooks: [{type: "command", command: "bash edit-only.sh"}]

用户: "修改 auth.ts"

→ Claude 调用 Edit: src/auth.ts
  → PreToolUse matcher "Edit" 匹配 ✅ → Hook 触发
  → edit-only.sh 执行: 检查文件是否在编辑黑名单中

→ Claude 调用 Read: src/auth.ts（读取文件）
  → PreToolUse matcher "Edit" 不匹配 Read ❌ → Hook 不触发

→ Claude 调用 Bash: npm test
  → PreToolUse matcher "Edit" 不匹配 Bash ❌ → Hook 不触发

效果:
  → matcher: "Edit" 精确匹配只作用于 Edit 工具
  → Read、Bash、Grep 等工具不受影响
  → 适合只关注文件编辑的场景
```

### 示例 2：多工具匹配 — Edit 和 Write 都触发

```
settings.json:
  PreToolUse → matcher: "Edit|Write"
  hooks: [{type: "command", command: "bash edit-write.sh"}]

用户: "重构 auth 模块，创建新的工具文件"

→ Claude 调用 Edit: src/auth.ts
  → matcher "Edit|Write" 匹配 Edit ✅ → Hook 触发
  → edit-write.sh: 自动 git add + lint

→ Claude 调用 Write: src/utils/jwt.ts（新文件）
  → matcher "Edit|Write" 匹配 Write ✅ → Hook 触发
  → edit-write.sh: 自动 git add + lint

→ Claude 调用 Bash: npm test
  → matcher "Edit|Write" 不匹配 Bash ❌ → Hook 不触发

→ Claude 调用 Read: package.json
  → 不匹配 ❌ → Hook 不触发

效果:
  → "Edit|Write" 用 | 分隔匹配多个工具（正则 OR 语法）
  → 所有文件修改操作都被 Hook 覆盖
  → 非修改操作（Read、Bash）不受影响
```

### 示例 3：空 matcher — 匹配所有工具

```
settings.json:
  PreToolUse → matcher: ""
  hooks: [{type: "command", command: "bash all-tools.sh"}]

用户: "分析项目结构"

→ Claude 调用 Glob: "src/**/*.ts"
  → matcher "" 匹配 Glob ✅ → Hook 触发
  → all-tools.sh: 记录工具调用日志

→ Claude 调用 Read: src/app.ts
  → matcher "" 匹配 Read ✅ → Hook 触发
  → all-tools.sh: 记录工具调用日志

→ Claude 调用 Grep: "TODO"
  → matcher "" 匹配 Grep ✅ → Hook 触发
  → all-tools.sh: 记录工具调用日志

→ Claude 调用 Bash: npm run build
  → matcher "" 匹配 Bash ✅ → Hook 触发

→ 所有工具都会触发 Hook

效果:
  → matcher: "" 等于匹配所有工具
  → 适合全局审计、日志记录等场景
  → 注意性能影响（每个工具调用都会触发）
```

### 示例 4：Bash 匹配 + 脚本内过滤特定命令

```
settings.json:
  PreToolUse → matcher: "Bash"
  hooks: [{type: "command", command: "bash bash-filter.sh"}]

bash-filter.sh:
  #!/bin/bash
  INPUT=$(cat)
  COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

  # 只拦截危险命令，放行安全命令
  if echo "$COMMAND" | grep -qiE 'rm\s+-rf|DROP\s+TABLE|truncate|DELETE\s+FROM.*WHERE.*1=1'; then
    echo '{"permissionDecision": "deny", "permissionDecisionReason": "检测到危险命令"}'
    exit 0
  fi

  if echo "$COMMAND" | grep -qiE 'npm\s+publish|docker\s+push|git\s+push.*--force'; then
    echo '{"permissionDecision": "ask", "permissionDecisionReason": "发布/推送操作需要确认"}'
    exit 0
  fi

  echo '{"permissionDecision": "allow"}'

用户操作:

→ Claude 调用 Bash: "npm install axios"
  → matcher "Bash" 匹配 ✅ → bash-filter.sh 执行
  → 命令不在危险列表 → allow ✅

→ Claude 调用 Bash: "rm -rf node_modules"
  → matcher "Bash" 匹配 ✅ → bash-filter.sh 执行
  → 命中 "rm -rf" → deny ❌

→ Claude 调用 Bash: "npm publish"
  → matcher "Bash" 匹配 ✅ → bash-filter.sh 执行
  → 命中 "npm publish" → ask ⚠️ → 弹窗确认

效果:
  → matcher 匹配 Bash 后，脚本内做细粒度过滤
  → 三级控制: allow / ask / deny
  → 安全命令自动放行，危险命令拦截，发布操作需确认
```

### 示例 5：匹配编辑工具但跳过 generated 目录

```
settings.json:
  PostToolUse → matcher: "Edit|Write"
  hooks: [{type: "command", command: "bash skip-generated.sh"}]

skip-generated.sh:
  #!/bin/bash
  INPUT=$(cat)
  FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.path // empty')

  # 跳过自动生成的文件
  if echo "$FILE_PATH" | grep -qE '(generated/|auto-generated|\.pb\.|__generated__)'; then
    exit 0  # 静默跳过
  fi

  # 跳过第三方库文件
  if echo "$FILE_PATH" | grep -qE '(node_modules/|vendor/|third_party/)'; then
    exit 0
  fi

  # 对源码文件执行 lint 和暂存
  if [ -n "$FILE_PATH" ]; then
    npx eslint --fix "$FILE_PATH" 2>/dev/null
    git add "$FILE_PATH" 2>/dev/null
    echo "{\"additionalContext\": \"已 Lint 并暂存: $FILE_PATH\"}"
  fi

用户: "修改组件和自动生成的类型"

→ Claude 调用 Edit: src/components/Button.tsx
  → matcher 匹配 ✅ → skip-generated.sh
  → 路径不包含 generated → 执行 lint + git add ✅

→ Claude 调用 Edit: src/__generated__/api-types.ts
  → matcher 匹配 ✅ → skip-generated.sh
  → 路径包含 "__generated__" → 跳过，不 lint ✅

→ Claude 调用 Edit: node_modules/lodash/index.js
  → matcher 匹配 ✅ → skip-generated.sh
  → 路径包含 "node_modules/" → 跳过 ✅

效果:
  → matcher 匹配宽范围，脚本内做精确过滤
  → 自动生成的文件不需要 lint
  → 第三方库文件不受影响
  → 实现了"匹配 + 排除"的组合逻辑
```
