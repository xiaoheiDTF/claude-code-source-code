# Skill: 项目级与全局级 Skill

> 指南 6.1「文件位置」— Skill 的全局和项目两个级别及其优先级

## 目录结构

```
# 全局
~/.claude/
├── commands/
│   ├── commit.md                  → 所有项目共享
│   ├── review.md                  → 所有项目共享
│   └── deploy.md                  → 所有项目共享
└── skills/
    └── refactor.md                → 所有项目共享

# 项目 A
project-a/
├── .claude/
│   ├── commands/
│   │   ├── commit.md              → 覆盖全局 commit
│   │   └── migrate.md             → 项目专属
│   └── skills/
│       └── api-gen.md             → 项目专属
└── src/

# 项目 B
project-b/
├── .claude/
│   └── commands/
│       └── seed.md                → 项目专属
└── src/
```

## 说明

```
两级 Skill:

  全局: ~/.claude/commands/*.md 和 ~/.claude/skills/*.md
    → 所有项目都可用
    → 个人通用工具

  项目: <project>/.claude/commands/*.md 和 <project>/.claude/skills/*.md
    → 仅当前项目可用
    → 可提交 VCS，团队共享

  优先级:
    项目级 > 全局级（同名时项目级覆盖全局级）

  合并行为:
    不同名的 Skill → 全局和项目级合并（都可用）
    同名的 Skill → 项目级覆盖全局级
```

## 使用示例

### 示例 1：全局 Skill 在所有项目中可用

```
~/.claude/commands/commit.md 存在

→ 在 project-a 中: /commit → 全局 commit ✅
→ 在 project-b 中: /commit → 全局 commit ✅
→ 在 project-c 中: /commit → 全局 commit ✅

→ 全局 commit.md 内容:
  "Generate a conventional commit message from git diff"

→ 三个项目都使用 Conventional Commits 格式

效果:
  → 一次定义，所有项目通用
  → 适合个人习惯: commit 格式、review 方式等
  → 在任何项目中输入 /commit 都能使用
```

### 示例 2：项目级 Skill 覆盖全局

```
全局 ~/.claude/commands/commit.md:
  "Generate a conventional commit message"

项目 project-a/.claude/commands/commit.md:
  "Generate a commit with Gitmoji and Jira ticket ID.
   Format: <gitmoji> <type>(<scope>): <description> [JIRA-XXX]
   Example: ✨ feat(auth): add SSO login [PROJ-123]"

→ 在 project-a 中:
  /commit → 使用项目级 commit（Gitmoji + Jira ID）✅

→ 在 project-b 中:
  /commit → 使用全局 commit（Conventional Commits）✅

→ 在 project-c 中:
  /commit → 使用全局 commit（Conventional Commits）✅

效果:
  → project-a 有自己的提交规范
  → 其他项目不受影响，继续用全局规范
  → 项目级覆盖实现了"项目特殊需求"的定制
```

### 示例 3：全局 + 项目 Skill 合并（不同名）

```
全局:
  ~/.claude/commands/
  ├── commit.md        → /commit ✅
  ├── review.md        → /review ✅
  └── deploy.md        → /deploy ✅

项目:
  project-a/.claude/commands/
  ├── migrate.md       → /migrate ✅
  └── seed.md          → /seed ✅

→ 在 project-a 中可用的命令:
  /commit    （来自全局）
  /review    （来自全局）
  /deploy    （来自全局）
  /migrate   （来自项目）
  /seed      （来自项目）
  → 5 个命令全部可用！

→ 在 project-b 中（没有项目级 commands）:
  /commit    （来自全局）
  /review    （来自全局）
  /deploy    （来自全局）
  → 只有 3 个全局命令

效果:
  → 不同名的 Skill 合并（全部可用）
  → 项目可以在全局基础上增加项目专属 Skill
  → 不需要为每个项目重复定义通用 Skill
```

### 示例 4：项目级 Skill 提交 VCS 团队共享

```
project-a/.claude/commands/migrate.md:
  ---
  description: Run Prisma migrations with safety checks
  allowed-tools: [Bash, Read]
  ---
  Run database migrations with pre-checks:
  1. Check pending migrations
  2. Create backup
  3. Run migrations
  4. Verify schema

→ git add .claude/commands/migrate.md
→ git commit -m "feat: add migrate command"
→ git push

→ 团队成员 A 拉取代码:
  → 自动获得 /migrate 命令 ✅

→ 团队成员 B 拉取代码:
  → 自动获得 /migrate 命令 ✅

→ 新成员加入项目:
  → git clone → 自动获得 /migrate 命令 ✅

效果:
  → 项目级 Skill 通过 Git 共享给整个团队
  → 所有成员使用统一的迁移流程
  → 新人无需手动配置
```

### 示例 5：三层覆盖 — 全局 + 项目 + 个人理解

```
场景: 团队有项目级 commit 规范，但个人偏好不同

全局（个人）~/.claude/commands/commit.md:
  "Use conventional commits with emoji prefix"

项目 project-a/.claude/commands/commit.md:
  "Use Jira ID + conventional commits format"

→ 开发者 Alice（喜欢项目规范）:
  在 project-a 中: /commit → 项目级（Jira ID）✅

→ 开发者 Bob（喜欢自己的全局规范）:
  删除了项目级 commit.md（只在本地）:
  rm .claude/commands/commit.md
  /commit → 全局级（emoji prefix）✅

→ 开发者 Carol:
  创建了自己的本地覆盖 .claude/commands.local/commit.md:
  （注意: commands.local/ 不一定被支持，取决于实现）
  → 仍使用项目级或全局级

最佳实践:
  → 项目级 Skill 提交 VCS（团队共享）
  → 全局 Skill 放 ~/.claude/（个人偏好）
  → 不要为了个人偏好修改项目级 Skill
  → 个人定制放全局级

效果:
  → 三层: 全局 → 项目 → 个人
  → 项目级覆盖全局，实现团队统一
  → 个人可以通过全局级定制自己的通用工具
```
