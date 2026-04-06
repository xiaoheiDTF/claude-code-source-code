# Skill: 文件位置与查找机制

> 指南 6.1「文件位置」— Skills 可放在全局或项目级目录中

## 目录结构

```
# 全局 Skills
~/.claude/
├── commands/                      ← 全局斜杠命令
│   ├── commit.md                  → /commit
│   ├── review.md                  → /review
│   └── deploy.md                  → /deploy
└── skills/                        ← 全局技能
    ├── refactor.md                → /refactor
    └── test-gen.md                → /test-gen

# 项目 Skills
my-project/
├── .claude/
│   ├── commands/                  ← 项目斜杠命令
│   │   ├── migrate.md             → /migrate
│   │   └── seed.md                → /seed
│   └── skills/                    ← 项目技能
│       ├── api-gen.md             → /api-gen
│       └── db-check.md            → /db-check
└── src/
```

## 说明

```
Skill 文件位置（4 个目录）:

  全局命令:  ~/.claude/commands/*.md
  全局技能:  ~/.claude/skills/*.md
  项目命令:  <project>/.claude/commands/*.md
  项目技能:  <project>/.claude/skills/*.md

文件名 → 斜杠命令名:
  commit.md → /commit
  test-gen.md → /test-gen
  my-custom.md → /my-custom

查找优先级:
  项目级 > 全局级
  同名时项目级覆盖全局级

commands vs skills:
  commands/ → 传统斜杠命令
  skills/   → 技能（功能类似，组织方式不同）
  两者都会注册为可用的 /斜杠命令
```

## 使用示例

### 示例 1：全局 Skill 跨项目使用

```
~/.claude/commands/commit.md 存在

→ 在项目 A 中:
  用户输入: /commit
  → 查找: .claude/commands/commit.md → 不存在
  → 查找: ~/.claude/commands/commit.md → 存在 ✅
  → 执行全局 commit 命令

→ 在项目 B 中:
  用户输入: /commit
  → 同样找到全局 ~/.claude/commands/commit.md
  → 执行全局 commit 命令

→ 在项目 C 中:
  用户输入: /commit
  → 同样生效

效果:
  → 全局 Skill 在所有项目中可用
  → 适合通用工具: commit、review、deploy
```

### 示例 2：项目级 Skill 覆盖全局

```
全局: ~/.claude/commands/commit.md
  内容: 使用 Conventional Commits 格式

项目: my-project/.claude/commands/commit.md
  内容: 使用项目自定义的 Gitmoji + Jira ID 格式

→ 在 my-project 中:
  用户输入: /commit
  → 查找: .claude/commands/commit.md → 存在 ✅（项目级优先）
  → 执行项目级 commit 命令（Gitmoji + Jira ID）

→ 在 other-project 中:
  用户输入: /commit
  → 查找: .claude/commands/commit.md → 不存在
  → 查找: ~/.claude/commands/commit.md → 存在 ✅
  → 执行全局 commit 命令（Conventional Commits）

效果:
  → 项目级覆盖全局级
  → 团队可以有项目专属的 Skill
  → 不影响其他项目的全局 Skill
```

### 示例 3：多个 Skill 的命名规则

```
文件名 → 命令名的映射:

~/.claude/commands/
├── commit.md          → /commit
├── code-review.md     → /code-review
├── test-gen.md        → /test-gen
├── deploy-prod.md     → /deploy-prod
├── db_migrate.md      → /db_migrate（下划线也支持）

→ 用户输入:
  /commit              → 匹配 commit.md ✅
  /code-review         → 匹配 code-review.md ✅
  /test-gen            → 匹配 test-gen.md ✅
  /deploy              → 无匹配 ❌（需要完整名称 deploy-prod）

效果:
  → 文件名（去掉 .md）就是命令名
  → 支持连字符和下划线
  → 需要完整输入命令名
```

### 示例 4：commands/ 和 skills/ 目录同时存在

```
~/.claude/
├── commands/
│   └── commit.md          → /commit
└── skills/
    └── refactor.md        → /refactor

.claude/
├── commands/
│   └── migrate.md         → /migrate
└── skills/
    ├── api-gen.md          → /api-gen
    └── test-gen.md         → /test-gen

→ 可用命令列表:
  /commit      (全局 commands)
  /refactor    (全局 skills)
  /migrate     (项目 commands)
  /api-gen     (项目 skills)
  /test-gen    (项目 skills)

→ 用户输入 /commit  → 执行全局 commands/commit.md
→ 用户输入 /refactor → 执行全局 skills/refactor.md
→ 用户输入 /migrate → 执行项目 commands/migrate.md

效果:
  → commands/ 和 skills/ 的文件都会注册为 /命令
  → 从用户角度，两者调用方式相同
  → 从组织角度，commands 偏命令式，skills 偏技能式
```

### 示例 5：子目录中的 Skill 不被识别

```
~/.claude/commands/
├── commit.md              → /commit ✅
├── utils/
│   └── helper.md          → 不识别 ❌
└── nested/
    └── deep-command.md    → 不识别 ❌

.claude/skills/
├── api-gen.md             → /api-gen ✅
└── sub/
    └── sub-skill.md       → 不识别 ❌

→ 用户输入:
  /commit        → 匹配 ✅
  /helper        → 不存在 ❌
  /deep-command  → 不存在 ❌
  /sub-skill     → 不存在 ❌

效果:
  → 只识别 commands/ 和 skills/ 顶层的 *.md 文件
  → 子目录中的文件不会注册为命令
  → 如果 Skill 很多，用前缀命名: api-gen.md, db-migrate.md
```
