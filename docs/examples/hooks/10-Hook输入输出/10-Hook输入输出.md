# Hook: 输入输出机制（stdin / stdout）

> 指南 4.4 + 4.5 — Hook 通过 stdin 接收 JSON 输入，通过 stdout 返回 JSON 控制

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json
│   └── hooks/
│       ├── protected-files.sh      ← PreToolUse: 检查受保护文件
│       ├── parse-output.sh         ← PostToolUse: 解析工具输出
│       ├── redirect-path.sh        ← PreToolUse: 重定向路径
│       ├── coverage-summary.sh     ← PostToolUse: 测试覆盖摘要
│       └── keyword-context.sh      ← UserPromptSubmit: 关键词上下文
├── protected/
│   └── config.json
└── package.json
```

## 说明

```
Hook 输入（stdin JSON）:

  所有事件共有字段:
    hook_event_name: 事件名
    session_id: 会话 ID
    cwd: 当前工作目录

  PreToolUse:
    + tool_name: 工具名
    + tool_input: { file_path, old_string, new_string, command, ... }

  PostToolUse:
    + tool_name: 工具名
    + tool_input: { ... }
    + tool_output: 工具输出文本
    + tool_result: 工具完整结果对象

  UserPromptSubmit:
    + source: 来源（api / cli）
    + prompt: 用户消息文本

Hook 输出（stdout JSON）:

  PreToolUse 可返回:
    permissionDecision: "allow" | "deny" | "ask"
    permissionDecisionReason: "原因文本"
    updatedInput: { ... }  ← 修改工具输入参数

  PostToolUse 可返回:
    additionalContext: "额外上下文信息"

  输出规则:
    - 必须 stdout 输出有效 JSON
    - 非JSON 输出会被忽略（stderr 用于调试）
    - 空 JSON {} 表示不做任何控制
```

## 使用示例

### 示例 1：PreToolUse — 检查受保护文件列表（stdin 读取 + deny）

```bash
#!/bin/bash
# .claude/hooks/protected-files.sh
INPUT=$(cat)

# 从 stdin JSON 提取 file_path
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.path // empty')

# 受保护文件列表
PROTECTED_FILES=(
  "production/config.json"
  ".env.production"
  "docker-compose.prod.yml"
  "kubernetes/deployment.yaml"
)

# 检查是否命中受保护文件
for PROTECTED in "${PROTECTED_FILES[@]}"; do
  if [ "$FILE_PATH" = "$PROTECTED" ] || echo "$FILE_PATH" | grep -q "$PROTECTED"; then
    echo "{\"permissionDecision\": \"deny\", \"permissionDecisionReason\": \"受保护文件: $PROTECTED — 需要 Lead 审批\"}"
    exit 0
  fi
done

# 非受保护文件，允许执行
echo '{"permissionDecision": "allow"}'
```

```
用户: "修改生产环境配置"

→ Claude 调用 Edit: file_path = "production/config.json"

→ Hook 接收 stdin:
  {"hook_event_name":"PreToolUse","tool_name":"Edit",
   "tool_input":{"file_path":"production/config.json",...},"session_id":"..."}

→ 脚本执行:
  1. cat → 读取 stdin JSON
  2. jq 提取 file_path = "production/config.json"
  3. 匹配受保护文件列表 → 命中！
  4. stdout 返回: {"permissionDecision":"deny","permissionDecisionReason":"受保护文件..."}

→ Claude 收到拒绝: "生产环境配置是受保护文件，需要 Lead 审批"

效果:
  → stdin 输入包含完整的工具参数
  → stdout 返回 deny 拦截操作
  → 通过数组维护受保护文件列表
```

### 示例 2：PostToolUse — 解析工具输出注入摘要（tool_output + additionalContext）

```bash
#!/bin/bash
# .claude/hooks/parse-output.sh
INPUT=$(cat)

# 提取 tool_output（PostToolUse 独有字段）
OUTPUT=$(echo "$INPUT" | jq -r '.tool_output // empty')
TOOL=$(echo "$INPUT" | jq -r '.tool_name // empty')

if [ "$TOOL" = "Bash" ]; then
  # 检查是否是测试命令的输出
  if echo "$OUTPUT" | grep -q "Tests:"; then
    PASSED=$(echo "$OUTPUT" | grep -oP '\d+(?= passed)')
    FAILED=$(echo "$OUTPUT" | grep -oP '\d+(?= failed)')
    TOTAL=$(echo "$OUTPUT" | grep -oP '\d+(?= tests)')

    echo "{\"additionalContext\": \"测试摘要: $PASSED/$TOTAL 通过, $FAILED 失败\"}"
  fi
fi
```

```
用户: "运行测试"

→ Claude 调用 Bash: npm test → 成功
  tool_output:
  "Tests:       45 passed, 3 failed, 48 total
   Snapshots:   12 passed
   Time:        12.345s"

→ Hook 接收 stdin:
  {"tool_name":"Bash","tool_output":"Tests: 45 passed, 3 failed...","tool_input":{...}}

→ 脚本执行:
  1. 提取 tool_output 文本
  2. 解析 "45 passed, 3 failed, 48 total"
  3. stdout: {"additionalContext": "测试摘要: 45/48 通过, 3 失败"}

→ Claude 看到精简的摘要，自动开始修复失败的测试

效果:
  → tool_output 包含完整的工具输出文本
  → 脚本解析并提取关键信息
  → additionalContext 注入精简摘要给 Claude
```

### 示例 3：PreToolUse — 修改工具输入参数（updatedInput）

```bash
#!/bin/bash
# .claude/hooks/redirect-path.sh
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# 规则: 所有新文件写入 src/v2/ 目录
if echo "$FILE_PATH" | grep -q "^src/components/" && [ ! -f "$FILE_PATH" ]; then
  # 新文件（不存在）重定向到 v2 目录
  NEW_PATH=$(echo "$FILE_PATH" | sed 's|^src/|src/v2/|')
  echo "{\"permissionDecision\": \"allow\", \"updatedInput\": $(echo "$INPUT" | jq --arg p "$NEW_PATH" '.tool_input | .file_path = $p')}"
  exit 0
fi

echo '{"permissionDecision": "allow"}'
```

```
用户: "创建新的 Button 组件"

→ Claude 调用 Write: file_path = "src/components/Button.tsx"（文件不存在）

→ Hook 接收 stdin:
  {"tool_name":"Write","tool_input":{"file_path":"src/components/Button.tsx","content":"..."}}

→ 脚本执行:
  1. 提取 file_path = "src/components/Button.tsx"
  2. 文件不存在 + 匹配 src/components/ → 重定向
  3. 新路径: "src/v2/components/Button.tsx"
  4. stdout: {"permissionDecision":"allow",
     "updatedInput":{"file_path":"src/v2/components/Button.tsx","content":"..."}}

→ Write 工具实际写入: src/v2/components/Button.tsx

→ Claude 日志显示写入 "src/components/Button.tsx"
  但实际文件在 src/v2/components/Button.tsx

效果:
  → updatedInput 修改工具的输入参数
  → Claude 不知道路径被修改了
  → 强制执行项目目录规范
```

### 示例 4：PostToolUse — 注入测试覆盖率摘要

```bash
#!/bin/bash
# .claude/hooks/coverage-summary.sh
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
OUTPUT=$(echo "$INPUT" | jq -r '.tool_output // empty')

# 检测 npm test 命令
if echo "$COMMAND" | grep -q "npm.*test"; then
  # 查找覆盖率文件
  if [ -f "coverage/coverage-summary.json" ]; then
    LINES=$(jq '.total.lines.pct' coverage/coverage-summary.json)
    BRANCHES=$(jq '.total.branches.pct' coverage/coverage-summary.json)
    FUNCTIONS=$(jq '.total.functions.pct' coverage/coverage-summary.json)

    echo "{\"additionalContext\": \"覆盖率报告 → 行: ${LINES}% | 分支: ${BRANCHES}% | 函数: ${FUNCTIONS}%\"}"
  fi
fi
```

```
用户: "跑一下测试看看覆盖率"

→ Claude 调用 Bash: npm test -- --coverage → 成功

→ Hook 接收 stdin（包含 tool_input.command 和 tool_output）

→ 脚本执行:
  1. 检测到命令包含 "npm test"
  2. 读取 coverage/coverage-summary.json
  3. 提取覆盖率: lines=78.5%, branches=65.2%, functions=82.1%
  4. stdout: {"additionalContext": "覆盖率报告 → 行: 78.5% | 分支: 65.2% | 函数: 82.1%"}

→ Claude 看到覆盖率摘要:
  "测试全部通过。覆盖率: 行 78.5%、分支 65.2%、函数 82.1%。
   分支覆盖率偏低，需要补充边界条件的测试用例。"

效果:
  → Hook 从外部文件读取数据注入上下文
  → Claude 不需要自己查找覆盖率文件
  → 自动分析覆盖率并给出建议
```

### 示例 5：UserPromptSubmit — 解析用户消息注入关键词上下文

```bash
#!/bin/bash
# .claude/hooks/keyword-context.sh
INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.prompt // empty')
CONTEXT=""

# 关键词检测 + 自动上下文加载
if echo "$PROMPT" | grep -qi "数据库\|database\|prisma"; then
  SCHEMA_INFO=$(head -20 prisma/schema.prisma 2>/dev/null || echo "无 schema 文件")
  CONTEXT="${CONTEXT}数据库上下文: ${SCHEMA_INFO}\n"
fi

if echo "$PROMPT" | grep -qi "API\|接口\|endpoint"; then
  API_ROUTES=$(grep -r "router\.\|app\.\(get\|post\|put\|delete\)" src/api/ --include="*.ts" -l 2>/dev/null | head -5)
  CONTEXT="${CONTEXT}API 路由文件: ${API_ROUTES}\n"
fi

if echo "$PROMPT" | grep -qi "部署\|deploy\|CI\|CD"; then
  PIPELINE=$(cat .github/workflows/*.yml 2>/dev/null | grep -E "^  [a-z]+:" | head -10)
  CONTEXT="${CONTEXT}CI/CD 流水线: ${PIPELINE}\n"
fi

if [ -n "$CONTEXT" ]; then
  # 转义换行符用于 JSON
  CONTEXT=$(echo -e "$CONTEXT" | jq -Rs .)
  echo "{\"additionalContext\": ${CONTEXT}}"
fi
```

```
用户: "帮我添加一个用户注册的 API 接口，数据存到数据库"

→ Hook 接收 stdin:
  {"prompt":"帮我添加一个用户注册的 API 接口，数据存到数据库","source":"cli",...}

→ 脚本执行:
  1. 检测关键词: "API" + "数据库" → 两个条件都命中
  2. 加载数据库上下文: 读取 prisma/schema.prisma 前 20 行
  3. 加载 API 上下文: 列出 src/api/ 下的路由文件
  4. stdout: {"additionalContext": "数据库上下文: model User { id Int @id ... }\nAPI 路由文件: src/api/auth.ts src/api/users.ts"}

→ Claude 看到增强后的上下文:
  "了解，你需要添加用户注册 API。我看到 User model 已有基本字段，
   auth.ts 和 users.ts 已有相关路由。我基于现有结构来扩展..."

效果:
  → 根据用户消息关键词自动加载相关上下文
  → 多关键词可同时触发多个上下文加载
  → Claude 不需要先花一轮查询项目结构
```
