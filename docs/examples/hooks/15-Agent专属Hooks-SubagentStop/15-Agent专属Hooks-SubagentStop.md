# Agent 专属 Hooks: SubagentStop（Agent 完成时）

> 指南 5.3「Agent 可用的 Hook 事件」— SubagentStop 在 Agent 完成工作时触发

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── feature-dev.md          ← 功能开发 + 自动提交
│       ├── refactoring.md          ← 重构 Agent + 生成报告
│       ├── bug-fixer.md            ← Bug 修复 + 自动测试
│       └── doc-writer.md           ← 文档生成 + 格式化
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/feature-dev.md

---
description: Feature implementer with auto-commit
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
hooks:
  SubagentStop:
    - matcher: ""
      hooks:
        - type: command
          command: |
            FILES=$(git diff --name-only)
            if [ -n "$FILES" ]; then
              git add -A
              git commit -m "feat: automated implementation by feature-dev agent"
              echo '{"additionalContext": "Changes auto-committed"}'
            fi
          async: true
---

Implement features based on the specification.
```

## 说明

```
SubagentStop 事件:
  触发时机: Agent 完成所有任务、准备停止时
  matcher: 通常为空 ""（匹配所有）

  注意: 在 frontmatter 中写 "Stop" 也会被自动转换为 "SubagentStop"

  典型用途:
    - 自动提交代码变更
    - 生成工作报告
    - 清理临时资源
    - 发送完成通知
    - 触发后续流程（CI/CD）

  特性:
    - async: true 推荐（不阻塞 Agent 停止流程）
    - 可以访问 git diff 获取 Agent 的所有变更
```

## 使用示例

### 示例 1：Agent 完成后自动 git commit

```
Agent: .claude/agents/feature-dev.md
  SubagentStop → git add -A + git commit

→ 用户: "用 feature-dev 实现用户注册功能"

→ Agent 执行:
  1. Write: src/api/auth/register.ts（注册接口）
  2. Write: src/models/user.ts（用户模型）
  3. Edit: src/routes/index.ts（添加路由）
  4. Bash: npm test → 测试通过

→ Agent 准备停止 → SubagentStop Hook 触发:
  1. git diff --name-only → 3 个文件
  2. git add -A → 暂存所有变更
  3. git commit -m "feat: automated implementation by feature-dev agent"
  4. 返回: {"additionalContext": "Changes auto-committed"}

→ git log:
  a3f2b1c feat: automated implementation by feature-dev agent

效果:
  → Agent 工作成果自动提交
  → 不需要用户手动 commit
  → 提交信息标识是 Agent 自动创建的
```

### 示例 2：重构 Agent 完成后生成变更报告

```
Agent: .claude/agents/refactoring.md
  SubagentStop → 统计变更 + 生成报告

→ 用户: "用 refactoring 把 class 组件转成 hooks"

→ Agent 执行:
  Edit: src/components/Header.tsx
  Edit: src/components/Sidebar.tsx
  Edit: src/components/Dashboard.tsx
  Write: src/hooks/useHeaderState.ts
  Write: src/hooks/useSidebarState.ts
  Bash: npm test → 通过

→ Agent 停止 → SubagentStop Hook 触发:
  1. 统计变更:
     CHANGED=$(git diff --name-only | wc -l)         → 5
     ADDED=$(git diff --diff-filter=A --name-only | wc -l)  → 2
     MODIFIED=$(git diff --diff-filter=M --name-only | wc -l) → 3
     LINES=$(git diff --shortstat)                    → "10 files changed, +156, -89"
  2. 生成报告:
     cat > .claude/reports/refactor-$(date +%Y%m%d).md << 'EOF'
     ## 重构报告
     日期: 2026-04-05
     类型: class → hooks 迁移
     变更: 3 文件修改, 2 文件新增
     统计: +156 / -89
     测试: 全部通过 ✅
     EOF
  3. 返回: {"additionalContext": "重构报告已保存到 .claude/reports/"}

效果:
  → Agent 完成后自动生成变更报告
  → 包含详细统计（修改/新增/行数）
  → 报告持久化保存，方便回顾
```

### 示例 3：Bug 修复 Agent 完成后自动运行测试套件

```
Agent: .claude/agents/bug-fixer.md
  SubagentStop → 运行完整测试 + 生成修复摘要

→ 用户: "用 bug-fixer 修复登录超时的问题"

→ Agent 执行:
  1. 读取 src/auth/login.ts → 分析超时原因
  2. Edit: src/auth/login.ts → 添加超时处理
  3. Edit: src/auth/config.ts → 调整超时参数

→ Agent 停止 → SubagentStop Hook 触发:
  1. 运行相关测试:
     npx jest src/auth/ --verbose → 8/8 通过
  2. 运行完整测试:
     npm test → 48/48 通过
  3. 生成修复摘要:
     BUG_DESC=$(git diff --name-only)
     echo "{\"additionalContext\": \"Bug 修复完成并验证:
       修改文件: src/auth/login.ts, src/auth/config.ts
       相关测试: 8/8 通过 ✅
       完整测试: 48/48 通过 ✅
       建议手动测试: 在慢网络环境下验证登录流程\"}"

效果:
  → 修复完成后自动运行全部测试
  → 确认修复没有引入新问题
  → 提供手动验证建议
```

### 示例 4：文档 Agent 完成后自动格式化

```
Agent: .claude/agents/doc-writer.md
  SubagentStop → 格式化文档 + 检查链接

→ 用户: "用 doc-writer 更新 API 文档"

→ Agent 执行:
  Write: docs/api/users.md
  Edit: docs/api/auth.md
  Edit: docs/README.md

→ Agent 停止 → SubagentStop Hook 触发:
  1. 格式化 Markdown:
     for f in $(git diff --name-only | grep '\.md$'); do
       npx prettier --write "$f" 2>/dev/null
     done
  2. 检查断链:
     npx markdown-link-check docs/api/users.md
     → 3 links checked, 0 dead
  3. 生成目录:
     npx doctoc docs/README.md
  4. 返回: {"additionalContext": "文档已格式化, 链接检查通过, 目录已更新"}

效果:
  → 文档写完后自动格式化（prettier）
  → 检查 Markdown 链接是否有效
  → 自动更新目录（TOC）
  → 文档质量保证
```

### 示例 5：Agent 完成后发送通知并触发 CI

```
Agent: .claude/agents/feature-dev.md
  SubagentStop → Slack 通知 + GitHub Actions 触发

→ Agent 完成功能开发并停止

→ SubagentStop Hook 触发:
  1. 获取变更信息:
     BRANCH=$(git branch --show-current)
     COMMIT=$(git log -1 --oneline)
     FILES=$(git diff --name-only HEAD~1 | wc -l)
  2. 发送 Slack 通知:
     curl -s -X POST "$SLACK_WEBHOOK" -d "{
       \"text\": \"✅ feature-dev Agent 完成任务\n分支: $BRANCH\n提交: $COMMIT\n变更: $FILES 个文件\"
     }"
  3. 触发 CI 流水线:
     curl -s -X POST \
       -H "Authorization: token $GITHUB_TOKEN" \
       -H "Accept: application/vnd.github.v3+json" \
       https://api.github.com/repos/myorg/myrepo/dispatches \
       -d "{\"event_type\": \"agent_completed\", \"client_payload\": {\"branch\": \"$BRANCH\"}}"
  4. 返回: {"additionalContext": "已发送通知并触发 CI 流水线"}

→ Slack 收到:
  ✅ feature-dev Agent 完成任务
  分支: feature/user-search
  提交: a3f2b1c feat: add user search
  变更: 5 个文件

→ GitHub Actions 自动开始:
  → npm install → npm test → npm run build → npm run e2e

效果:
  → Agent 完成后自动通知团队
  → 自动触发 CI 验证变更
  → 完整的自动化流水线: Agent 开发 → 提交 → 通知 → 验证
```
