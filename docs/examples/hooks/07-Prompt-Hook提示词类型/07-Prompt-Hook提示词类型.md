# Hook 类型: Prompt Hook（动态注入提示词）

> 指南 4.3「Hook 类型」— Prompt Hook 动态注入文本提示词到 Claude 的上下文中

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json              ← Hook 配置
│   └── (无需额外脚本文件，Prompt Hook 直接在 JSON 中定义)
├── src/
│   ├── __tests__/
│   │   └── app.test.ts
│   ├── api/
│   │   └── openapi.yaml
│   └── app.ts
└── package.json
```

## settings.json 配置

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Review the last change for security issues before proceeding."
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Review the git diff since last session for any potential issues."
          }
        ]
      }
    ]
  }
}
```

## 说明

```
Prompt Hook 特点:
  格式: {"type": "prompt", "prompt": "..."}
  作用: 将 prompt 文本直接注入到 Claude 的上下文中
  无需脚本: 不需要写 shell 脚本，直接在 JSON 中定义

  与 Command Hook 的区别:
    Command Hook → 执行 shell 命令，可以读取 stdin、返回 JSON 控制
    Prompt Hook → 直接注入文本提示词，不需要执行任何命令

  适用场景:
    - 注入固定的提醒/规则
    - 在特定事件后引导 Claude 的行为
    - 无需编程的轻量级 Hook
```

## 使用示例

### 示例 1：编辑测试文件后提醒运行相关测试

```
settings.json:
  PostToolUse → matcher: "Edit" → type: "prompt"
  prompt: "如果刚修改的是测试文件，请立即运行该测试文件验证修改是否正确。如果是源码文件，检查对应的测试是否需要更新。"

用户: "修改 app.test.ts 的断言"

→ Claude 调用 Edit 修改 src/__tests__/app.test.ts → 成功

→ PostToolUse Prompt Hook 触发:
  注入提示词: "如果刚修改的是测试文件，请立即运行该测试文件验证..."

→ Claude 行为变化:
  "我刚修改了测试文件，让我运行测试验证一下..."
  → 调用 Bash: npx jest src/__tests__/app.test.ts
  → 测试通过 ✅

效果:
  → 编辑测试文件后自动运行测试
  → 无需用户手动提醒 "运行一下测试"
  → Prompt Hook 最简单的用法：一条文字提示
```

### 示例 2：修改 API 文件后检查 OpenAPI 规范

```
settings.json:
  PostToolUse → matcher: "Write" → type: "prompt"
  prompt: "检查是否需要同步更新 OpenAPI/Swagger 规范文件。如果修改了 API 路由、请求参数或响应格式，确保 openapi.yaml 也做了相应更新。"

用户: "添加一个新的 PUT /users/:id 接口"

→ Claude 调用 Write 写入 src/api/users.ts → 成功

→ PostToolUse Prompt Hook 触发:
  注入提示词: "检查是否需要同步更新 OpenAPI/Swagger 规范文件..."

→ Claude 行为:
  "我添加了新的 API 接口，需要检查 OpenAPI 规范是否需要更新..."
  → 读取 src/api/openapi.yaml
  → 发现缺少 PUT /users/:id 的定义
  → 编辑 openapi.yaml 添加新端点

效果:
  → 修改 API 代码时自动提醒更新文档
  → 防止 API 代码和文档不同步
  → 轻量级提醒，不需要写脚本
```

### 示例 3：会话启动时回顾上次修改

```
settings.json:
  SessionStart → matcher: "" → type: "prompt"
  prompt: "请回顾自上次会话以来的所有 git 变更，检查是否有明显的 bug、安全问题或遗漏的 TODO。"

用户启动新会话

→ SessionStart Prompt Hook 触发:
  注入提示词: "请回顾自上次会话以来的所有 git 变更..."

→ Claude 行为:
  "让我查看自上次会话以来的变更..."
  → 调用 Bash: git log --since="yesterday" --oneline
  → 调用 Bash: git diff HEAD~5 --stat
  → 逐个审查变更:
    "发现 2 个问题:
     1. src/auth.ts: JWT secret 硬编码了（安全隐患）
     2. src/api/users.ts: 新增接口缺少输入验证
     建议优先修复第一个安全问题。"

效果:
  → 每次启动自动回顾近期代码变更
  → 充当"代码守门员"角色
  → 发现用户可能遗漏的问题
```

### 示例 4：测试失败后引导修复

```
settings.json:
  PostToolUseFailure → matcher: "Bash" → type: "prompt"
  prompt: "如果失败的是测试命令，请仔细阅读每个失败的测试用例，分析失败原因，然后提出修复方案。不要简单地跳过或忽略失败的测试。"

用户: "运行所有测试"

→ Claude 调用 Bash: npm test → 失败（3 个测试用例失败）

→ PostToolUseFailure Prompt Hook 触发:
  注入提示词: "如果失败的是测试命令，请仔细阅读每个失败的测试用例..."

→ Claude 行为（有 Prompt Hook）:
  "测试失败了，让我逐个分析失败原因..."
  → 读取失败测试的源码
  → 分析断言失败的具体原因
  → 提出修复方案并实施

对比无 Prompt Hook 时:
  Claude 可能只是简单报告 "测试失败了" 就停止

效果:
  → 引导 Claude 主动分析失败原因
  → 防止 Claude 只是报告错误而不修复
  → 通过提示词改变 Claude 的行为模式
```

### 示例 5：每次用户提交时考虑安全影响

```
settings.json:
  UserPromptSubmit → matcher: "" → type: "prompt"
  prompt: "在处理用户请求前，先考虑安全影响。特别关注: SQL 注入、XSS、敏感数据泄露、权限检查缺失。如果发现安全问题，主动指出并提供安全替代方案。"

用户: "写一个搜索接口，接收用户输入的查询词"

→ UserPromptSubmit Prompt Hook 触发:
  注入提示词: "在处理用户请求前，先考虑安全影响..."

→ Claude 行为（有安全 Prompt）:
  "我来实现搜索接口。考虑到安全性，我会:
   1. 使用参数化查询防止 SQL 注入
   2. 对输入进行长度限制和字符过滤
   3. 不直接拼接 SQL 字符串"

  → Write src/api/search.ts（使用参数化查询）

对比无安全 Prompt 时:
  可能直接用字符串拼接: `SELECT * FROM users WHERE name LIKE '%${query}%'`

效果:
  → 每条消息自动注入安全意识
  → Claude 主动考虑安全最佳实践
  → Prompt Hook 作为"安全护栏"
  → 无需编写复杂的安全检测脚本
```
