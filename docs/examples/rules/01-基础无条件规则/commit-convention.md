# 基础无条件规则 — Git 提交信息规范

> 无 `paths` frontmatter → 启动时始终加载

## 目录结构

```
my-project/
├── .claude/
│   └── rules/
│       ├── coding-standards.md
│       ├── commit-convention.md      ← ★ 本规则（无条件，始终加载）
│       └── ...
├── src/
└── package.json
```

## 规则文件内容

```markdown
# 文件: .claude/rules/commit-convention.md

使用 Conventional Commits 格式：

type(scope): description

### 类型
- feat: 新功能
- fix: Bug 修复
- refactor: 重构
- docs: 文档
- test: 测试
- chore: 构建/工具

### 规则
- description 用祈使语气、小写、不加句号
- 破坏性变更加 BREAKING CHANGE footer
- 每次提交只做一件事
```

## 触发示例

```
用户: "提交当前的登录 bug 修复"

Claude 看到 commit-convention.md 后会遵循:
✅ git commit -m "fix(auth): resolve login timeout on slow network"
✅ git commit -m "feat(api): add rate limiting middleware"
❌ git commit -m "Fixed bug"                 ← 不是 conventional commit
❌ git commit -m "fix: Fixed the bug."       ← 大写+句号，不符合规范
❌ git commit -m "fix + feat: login + rate"  ← 混合多件事
```
