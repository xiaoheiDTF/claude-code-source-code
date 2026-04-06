# settings.json: permissions.defaultMode 权限模式

> 指南 7.2「主要配置项」— defaultMode 控制未匹配 allow/deny 的操作默认行为

## settings.json 配置

```json
{
  "permissions": {
    "defaultMode": "default"
  }
}
```

## 说明

```
defaultMode 选项:

  "default"        → 每次操作弹出确认（最安全）
  "acceptEdits"    → 自动接受文件编辑，Bash 仍需确认
  "plan"           → 所有操作需要先制定计划再执行
  "auto"           → 全部自动执行（最危险，慎用）
  "bypassPermissions" → 跳过所有权限检查
  "dontAsk"        → 自动执行，使用最合适的权限

  优先级:
    deny > allow > defaultMode
    在 allow/deny 之外的命令由 defaultMode 决定

  不同模式适合不同场景:
    新项目/不确定 → default（手动确认）
    熟悉项目开发 → acceptEdits（自动编辑）
    重要操作审核 → plan（计划模式）
    CI/CD 自动化 → auto 或 dontAsk
```

## 使用示例

### 示例 1：default — 每次操作手动确认

```json
{
  "permissions": {
    "defaultMode": "default"
  }
}
```

```
→ Claude 执行:
  Edit: src/app.ts → 弹窗: "允许编辑 src/app.ts？ [Y/n]" ⚠️
  Write: src/new.ts → 弹窗: "允许创建 src/new.ts？ [Y/n]" ⚠️
  Bash: npm test → 弹窗: "允许执行 npm test？ [Y/n]" ⚠️
  Read: src/app.ts → 弹窗: "允许读取 src/app.ts？ [Y/n]" ⚠️

→ 每次 Claude 使用任何工具都需要用户手动确认

适合:
  → 新项目，不了解 Claude 行为
  → 安全敏感项目
  → 想完全掌控 Claude 的每一步

缺点:
  → 频繁弹窗，效率低
  → 适合初期了解，不适合日常开发
```

### 示例 2：acceptEdits — 自动接受文件编辑

```json
{
  "permissions": {
    "defaultMode": "acceptEdits",
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)"
    ]
  }
}
```

```
→ Claude 执行:
  Read: src/app.ts       → 自动允许 ✅
  Edit: src/app.ts       → 自动允许 ✅
  Write: src/new.ts      → 自动允许 ✅
  Glob: "**/*.ts"        → 自动允许 ✅
  Grep: "TODO"           → 自动允许 ✅
  Bash: npm test         → 弹窗确认 ⚠️
  Bash: npm run build    → 弹窗确认 ⚠️
  Bash: rm -rf dist/     → deny ❌（被黑名单拦截）

→ 文件读写自动通过，命令执行需确认

适合:
  → 日常开发（最推荐的模式）
  → 文件编辑频繁，命令执行需把关
  → 平衡效率和安全
```

### 示例 3：plan — 先制定计划再执行

```json
{
  "permissions": {
    "defaultMode": "plan"
  }
}
```

```
→ 用户: "添加用户注册功能"

→ Claude 行为:
  1. 首先制定计划（不执行任何操作）:
     "## 实现计划
     1. 创建用户模型 src/models/user.ts
     2. 创建注册 API src/api/auth/register.ts
     3. 添加输入验证 middleware
     4. 编写测试 tests/auth/register.test.ts
     5. 注册路由 src/routes/index.ts"

  2. 等待用户确认计划:
     "是否执行以上计划？ [Y/n]"

  3. 用户确认后，逐个执行并确认:
     Write: src/models/user.ts → 弹窗 ⚠️
     Write: src/api/auth/register.ts → 弹窗 ⚠️

适合:
  → 复杂任务需要先审核方案
  → 重要功能开发
  → 团队协作中需要 review 方案

特点:
  → Claude 先想清楚再动手
  → 用户可以在计划阶段提出修改意见
  → 避免做了一半发现方向不对
```

### 示例 4：auto — 全自动执行

```json
{
  "permissions": {
    "defaultMode": "auto",
    "deny": [
      "Bash(rm -rf*)",
      "Bash(npm publish*)"
    ]
  }
}
```

```
→ Claude 执行:
  Edit: src/app.ts       → 自动允许 ✅
  Write: src/new.ts      → 自动允许 ✅
  Bash: npm test         → 自动允许 ✅
  Bash: npm install      → 自动允许 ✅
  Bash: git add .        → 自动允许 ✅
  Bash: git commit       → 自动允许 ✅
  Bash: rm -rf dist/     → deny ❌（被黑名单拦截）
  Bash: npm publish      → deny ❌（被黑名单拦截）

→ 所有操作自动执行，只有 deny 列表拦截

适合:
  → CI/CD 自动化
  → 高度信任的场景
  → 配合 deny 保护危险操作

注意:
  → 日常使用需谨慎
  → 必须配合 deny 保护关键操作
  → Claude 可能执行你未预期的命令
```

### 示例 5：不同项目使用不同 defaultMode

```json
# ~/.claude/settings.json（全局 — 安全默认）
{
  "permissions": {
    "defaultMode": "default"
  }
}

# 个人项目: .claude/settings.json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
→ 个人项目自动编辑，效率高

# 工作项目: .claude/settings.json
{
  "permissions": {
    "defaultMode": "plan"
  }
}
→ 工作项目先审计划再执行

# 开源贡献项目: .claude/settings.json
{
  "permissions": {
    "defaultMode": "default"
  }
}
→ 陌生项目全部手动确认

# CI 环境: 环境变量覆盖
{
  "permissions": {
    "defaultMode": "auto"
  }
}
→ 自动化环境全自动

效果:
  → 根据项目信任度选择不同模式
  → 全局设安全默认，项目按需覆盖
  → 个人/工作/开源/CI 各有合适的策略
```
