# settings.json: permissions.allow 白名单

> 指南 7.2「主要配置项」— permissions.allow 自动允许的操作

## 目录结构

```
my-project/
├── .claude/
│   └── settings.json              ← allow 白名单配置
├── src/
└── package.json
```

## settings.json 配置

```json
{
  "permissions": {
    "allow": [
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git status*)",
      "Bash(npm test*)",
      "Bash(npm run lint*)",
      "Bash(npm run build*)",
      "Read",
      "Glob",
      "Grep"
    ]
  }
}
```

## 说明

```
permissions.allow 白名单:
  格式: 字符串数组，支持 glob 模式匹配
  作用: 自动允许匹配的操作（不弹窗确认）

  工具匹配模式:
    "Read"              → 允许所有 Read 操作
    "Edit"              → 允许所有 Edit 操作
    "Write"             → 允许所有 Write 操作
    "Bash(git log*)"    → 允许以 "git log" 开头的 Bash 命令
    "Bash(npm test*)"   → 允许以 "npm test" 开头的命令
    "Glob"              → 允许所有文件搜索
    "Grep"              → 允许所有内容搜索

  Bash 命令模式:
    * 匹配任意字符
    "Bash(git *)"       → 允许所有 git 命令
    "Bash(npm *)"       → 允许所有 npm 命令
    "Bash(npx jest*)"   → 允许 jest 测试

  安全原则:
    只允许已知的、安全的操作
    避免过于宽泛的模式如 "Bash(*)"
```

## 使用示例

### 示例 1：只读开发环境（最小权限）

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep"
    ]
  }
}
```

```
→ Claude 操作:
  Read: src/app.ts            → 自动允许 ✅
  Glob: "**/*.ts"             → 自动允许 ✅
  Grep: "TODO"                → 自动允许 ✅
  Edit: src/app.ts            → 需手动确认 ⚠️
  Write: src/new.ts           → 需手动确认 ⚠️
  Bash: npm test              → 需手动确认 ⚠️

效果:
  → 只读工具全部自动通过
  → 任何修改操作需要手动确认
  → 适合代码审查场景
```

### 示例 2：日常开发（常用命令白名单）

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Write",
      "Glob",
      "Grep",
      "Bash(git *)",
      "Bash(npm test*)",
      "Bash(npm run lint*)",
      "Bash(npm run build*)",
      "Bash(node *)"
    ]
  }
}
```

```
→ Claude 操作:
  Bash: git status            → 自动允许 ✅
  Bash: git log --oneline     → 自动允许 ✅
  Bash: git diff              → 自动允许 ✅
  Bash: npm test              → 自动允许 ✅
  Bash: npm run build         → 自动允许 ✅
  Bash: npm install axios     → 需手动确认 ⚠️（不匹配 npm test/npm run）
  Bash: docker compose up     → 需手动确认 ⚠️
  Bash: rm -rf dist/          → 需手动确认 ⚠️

效果:
  → 日常 git + npm 操作自动通过
  → 安装新包、Docker 等需要确认
  → 平衡效率和安全
```

### 示例 3：CI/CD 自动化环境

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Write",
      "Edit",
      "Bash(*)"
    ],
    "defaultMode": "acceptEdits"
  }
}
```

```
→ Claude 操作:
  任何工具调用都自动允许 ✅
  Bash: npm install           → ✅
  Bash: npm run build         → ✅
  Bash: npm run deploy        → ✅
  Bash: docker push ...       → ✅

效果:
  → 全自动，不弹任何确认
  → 适合 CI/CD 管道、自动化脚本
  → 不适合日常交互使用（太危险）
```

### 示例 4：精确控制特定 Bash 命令

```json
{
  "permissions": {
    "allow": [
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git status*)",
      "Bash(git branch*)",
      "Bash(npm test -- --grep*)",
      "Bash(npx jest src/*)"
    ]
  }
}
```

```
→ Claude 操作:
  Bash: git log --oneline -5  → 匹配 git log* ✅
  Bash: git diff HEAD~1       → 匹配 git diff* ✅
  Bash: git status            → 匹配 git status* ✅
  Bash: git add .             → 不匹配 ❌（没有 git add* 模式）
  Bash: git commit -m "..."   → 不匹配 ❌
  Bash: npm test -- --grep auth → 匹配 ✅
  Bash: npm test              → 不匹配 ❌（需要 --grep 参数）

效果:
  → 精确控制每个允许的命令
  → 只读 git 命令自动通过
  → 写操作 git add/commit 需确认
  → 特定测试命令自动通过
```

### 示例 5：allow + deny 组合使用

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm *)",
      "Bash(npx *)"
    ],
    "deny": [
      "Bash(git push --force*)",
      "Bash(npm publish*)",
      "Bash(npx * --dangerously-skip*)"
    ]
  }
}
```

```
优先级: deny > allow

→ Claude 操作:
  Bash: git status            → allow ✅
  Bash: git push              → allow ✅
  Bash: git push --force      → deny ❌（即使 allow 了 git *）
  Bash: npm install           → allow ✅
  Bash: npm publish           → deny ❌（即使 allow 了 npm *）
  Bash: npm test              → allow ✅
  Bash: npx jest              → allow ✅
  Bash: npx prisma migrate reset --dangerously-skip-seed
                               → deny ❌

效果:
  → allow 设大范围，deny 排除危险操作
  → deny 优先级高于 allow
  → 拦截危险操作: force push、npm publish
```
