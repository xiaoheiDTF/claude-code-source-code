# Hook 类型: Command Hook（执行 shell 命令）

> 指南 4.3「Hook 类型」— Command Hook 执行 shell 命令，是最常用的 Hook 类型

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json              ← Hook 配置
│   └── hooks/
│       ├── first-edit-check.sh    ← once: true 示例
│       ├── async-lint.sh          ← async: true 示例
│       ├── timeout-hook.sh        ← timeout 示例
│       └── powershell-hook.ps1    ← shell: powershell 示例
├── src/
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
            "command": "bash .claude/hooks/first-edit-check.sh",
            "once": true,
            "statusMessage": "Running first-edit check..."
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/async-lint.sh",
            "async": true,
            "timeout": 30,
            "statusMessage": "Linting in background..."
          }
        ]
      }
    ]
  }
}
```

## 说明

```
Command Hook 字段详解:

  command:     要执行的 shell 命令（必填）
  shell:       "bash" 或 "powershell"（默认 bash）
  timeout:     超时秒数（默认无限制）
  statusMessage: 运行时显示给用户的状态文字
  once:        true = 只运行一次后自动移除
  async:       true = 后台运行，不阻塞主流程
  asyncRewake: true = 后台完成后重新唤醒 agent

  执行流程:
    1. Hook 触发 → 通过 stdin 传入 JSON 数据
    2. 执行 command 指定的 shell 命令
    3. 脚本通过 stdout 返回 JSON 控制
    4. 超时则自动 kill 进程
```

## 使用示例

### 示例 1：`once: true` — 只在首次编辑时运行数据库迁移检查

```
settings.json:
  PreToolUse → matcher: "Edit" → once: true, command: "bash check-migration.sh"

用户: "修改用户模型，添加 email 字段"

→ 第 1 次 Edit 调用:
  PreToolUse Hook 触发:
  1. check-migration.sh 执行:
     - 检查 prisma/schema.prisma 和数据库是否同步
     - 检查 pending migrations
     - 返回: {"additionalContext": "发现 2 个待执行的迁移，已自动执行 prisma migrate deploy"}
  2. Hook 自动移除（once: true）✅

→ 第 2 次 Edit 调用:
  PreToolUse Hook 不再触发（已被移除）

→ 第 3 次 Edit 调用:
  同样不触发

效果:
  → 只在首次编辑时检查数据库状态
  → 避免每次编辑都重复检查（浪费时间）
  → once: true 自动清理，无需手动管理
```

### 示例 2：`async: true` — 编辑后后台运行 ESLint（不阻塞）

```
settings.json:
  PostToolUse → matcher: "Write" → async: true, command: "bash async-lint.sh"

用户: "创建新的 API 路由"

→ Claude 调用 Write 写入 src/api/users.ts → 成功

→ PostToolUse Hook 触发（async: true）:
  1. 后台启动: npx eslint src/api/users.ts
  2. Claude 立即继续下一步操作（不等待 eslint 完成）
  3. 屏幕底部显示: "Linting in background..."

→ eslint 完成（3 秒后）:
  4. 返回: {"additionalContext": "ESLint: 通过 ✅"}

→ Claude 在下一个 turn 中看到 lint 结果

效果:
  → Lint 不阻塞 Claude 的编辑操作
  → Claude 可以连续编辑多个文件，lint 全部在后台跑
  → async: true 适合耗时的辅助任务（lint、format、test）
```

### 示例 3：`timeout: 30` — 防止 Hook 卡死

```
settings.json:
  PostToolUse → matcher: "Bash" → timeout: 30, command: "bash slow-check.sh"

用户: "运行 npm install 添加新依赖"

→ Claude 调用 Bash: npm install → 成功

→ PostToolUse Hook 触发（timeout: 30 秒）:
  1. 执行 slow-check.sh:
     - 脚本运行 npm audit（检查安全漏洞）
     - 网络慢，npm audit 卡住...
  2. 30 秒后: 超时！进程被自动 kill
  3. Hook 静默失败（不影响主流程）

→ Claude 正常继续，不受 hook 超时影响

效果:
  → 防止 hook 卡住导致整个会话死锁
  → 超时后自动终止，不阻塞后续操作
  → 对于可能耗时的操作（网络请求、npm 命令）务必设置 timeout
```

### 示例 4：`shell: "powershell"` — Windows 环境 Hook

```
settings.json:
  PreToolUse → matcher: "Edit" → shell: "powershell",
  command: "powershell -File .claude/hooks/encoding-check.ps1"

用户: "修改 README.md"

→ Claude 调用 Edit 修改 README.md

→ PreToolUse Hook 触发（shell: "powershell"）:
  1. 使用 PowerShell 执行 encoding-check.ps1:
     $input = [Console]::In.ReadToEnd()
     $json = $input | ConvertFrom-Json
     $filePath = $json.tool_input.file_path
     $encoding = (Get-Item $filePath).Attributes
     # 检查文件编码是否为 UTF-8 BOM
     $bytes = [System.IO.File]::ReadAllBytes($filePath)
     if ($bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB) {
       echo '{"permissionDecision": "allow"}'
     } else {
       echo '{"permissionDecision": "allow"}'
     }

效果:
  → 在 Windows 环境使用 PowerShell 脚本
  → 可以调用 Windows 特有的 API 和工具
  → shell 字段区分平台: bash (Linux/Mac) vs powershell (Windows)
```

### 示例 5：`asyncRewake: true` — 后台构建完成后通知 Agent

```
settings.json:
  PostToolUse → matcher: "Edit|Write" → async: true, asyncRewake: true,
  command: "bash .claude/hooks/build-watch.sh", timeout: 120

用户: "修改了 3 个组件文件"

→ Claude 连续调用 Edit 3 次:
  Edit 1: src/components/Header.tsx → 成功
  Edit 2: src/components/Sidebar.tsx → 成功
  Edit 3: src/components/Footer.tsx → 成功

→ 第 3 次 PostToolUse Hook 触发（async + asyncRewake）:
  1. 后台启动: npm run build（需要 60 秒）
  2. Claude 继续其他操作（不等待构建）
  3. 构建完成:
     → 构建成功:
       返回 {"additionalContext": "构建成功 ✅ 无错误"}
       → Agent 被重新唤醒，看到构建结果
     → 构建失败:
       返回 {"additionalContext": "构建失败 ❌ 2 个类型错误:\n1. Header.tsx: Type 'string' ..."}
       → Agent 被唤醒，自动修复构建错误

效果:
  → asyncRewake: true 让 Agent 在后台任务完成后自动继续
  → Agent 可以"边改边等"——编辑完代码后做其他事，构建好了自动回来看结果
  → 比 async: true 更智能：不只是后台运行，还会回来处理结果
```
