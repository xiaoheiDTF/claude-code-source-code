# Skill: commands/ 与 skills/ 目录的区别

> 指南 6.1「文件位置」— 两个目录存放 Skill，功能相同但组织方式不同

## 目录结构

```
~/.claude/
├── commands/                      ← 斜杠命令（Slash Commands）
│   ├── commit.md                  → /commit
│   ├── review.md                  → /review
│   └── deploy.md                  → /deploy
└── skills/                        ← 技能（Skills）
    ├── refactor.md                → /refactor
    ├── test-gen.md                → /test-gen
    └── code-explain.md            → /code-explain

.claude/
├── commands/                      ← 项目命令
│   ├── migrate.md                 → /migrate
│   └── seed.md                    → /seed
└── skills/                        ← 项目技能
    ├── api-gen.md                 → /api-gen
    └── db-check.md               → /db-check
```

## 说明

```
commands/ vs skills/:

  相同点:
    - 都是 .md 文件
    - 都通过 /command-name 调用
    - 都支持 frontmatter 字段
    - 都支持 $ARGUMENTS 参数
    - 都有全局和项目两个级别

  不同点（约定上的区别）:

    commands/（斜杠命令）:
    → 偏向"一次性动作"
    → 执行后立即返回结果
    → 例子: /commit（提交）、/deploy（部署）、/test（运行测试）

    skills/（技能）:
    → 偏向"可复用的能力"
    → 可能涉及多轮交互
    → 例子: /refactor（重构）、/code-explain（解释代码）

  实际行为:
    从技术角度，两者完全等价
    区别纯粹是组织约定，方便开发者分类管理
```

## 使用示例

### 示例 1：commands/ 放命令式 Skill（一次性动作）

```
.claude/commands/ 中的文件:

commit.md:
  ---
  description: Generate and execute a git commit
  allowed-tools: [Bash, Read]
  ---
  Based on the current git diff, commit the changes.
  → 执行一次 → 生成提交 → 完成

deploy.md:
  ---
  description: Deploy to staging/production
  allowed-tools: [Bash]
  ---
  Deploy the application. Environment: $ARGUMENTS
  → 执行一次 → 部署 → 完成

migrate.md:
  ---
  description: Run database migrations
  allowed-tools: [Bash, Read]
  ---
  Run pending migrations.
  → 执行一次 → npx prisma migrate deploy → 完成

共同特点:
  → 执行一次就完成
  → 结果是"做了什么"（提交了、部署了、迁移了）
  → 偏向操作性的命令
```

### 示例 2：skills/ 放能力式 Skill（可复用能力）

```
.claude/skills/ 中的文件:

refactor.md:
  ---
  description: Refactor code with analysis
  allowed-tools: [Read, Write, Edit, Bash, Grep, Glob]
  ---
  Analyze and refactor: $ARGUMENTS
  → 可能多轮: 分析 → 提方案 → 修改 → 测试 → 优化

code-explain.md:
  ---
  description: Explain code in detail
  allowed-tools: [Read, Grep, Glob]
  ---
  Explain: $ARGUMENTS
  → 深度分析，可能需要读取多个文件来完整解释

test-gen.md:
  ---
  description: Generate comprehensive tests
  allowed-tools: [Read, Write, Edit, Bash, Grep, Glob]
  ---
  Generate tests for: $ARGUMENTS
  → 读取代码 → 分析逻辑 → 生成测试 → 运行验证

共同特点:
  → 可能涉及多轮操作
  → 结果是"产出了什么"（解释、测试、重构后的代码）
  → 偏向知识性和创造性
```

### 示例 3：同名文件在两个目录中不会冲突（不应该这样做）

```
# 不推荐: 同名文件同时存在于两个目录
.claude/commands/review.md    → /review
.claude/skills/review.md      → /review

→ 查找顺序:
  通常 commands/ 优先于 skills/
  但具体行为取决于实现

→ 建议: 避免在两个目录中放同名文件
  要么放 commands/ 要么放 skills/，不要两边都放

效果:
  → 同名会导致混淆
  → 选择一个目录统一放置
  → 按约定分类即可
```

### 示例 4：团队约定 commands 和 skills 的分类规则

```
团队约定:

commands/ → 操作性命令:
  /commit     → 提交代码
  /deploy     → 部署
  /test       → 运行测试
  /migrate    → 数据库迁移
  /seed       → 填充数据
  /clean      → 清理临时文件

skills/ → 分析/生成类技能:
  /refactor    → 代码重构
  /review      → 代码审查
  /explain     → 代码解释
  /test-gen    → 测试生成
  /api-gen     → API 生成
  /doc-gen     → 文档生成

→ /help 列表按目录分组显示:
  Commands:
    /commit — 提交代码
    /deploy — 部署应用
  Skills:
    /refactor — 代码重构
    /review — 代码审查

效果:
  → 团队有一致的分类标准
  → 新成员容易理解 Skill 体系
  → /help 输出更有组织性
```

### 示例 5：小型项目只用 commands/ 就够了

```
小项目:
.claude/
└── commands/
    ├── commit.md
    ├── review.md
    └── test.md

→ 没有创建 skills/ 目录
→ 所有 Skill 都放在 commands/ 中
→ 完全可以正常工作

大型项目:
.claude/
├── commands/
│   ├── commit.md
│   ├── deploy.md
│   ├── migrate.md
│   ├── seed.md
│   └── rollback.md
└── skills/
    ├── review.md
    ├── refactor.md
    ├── security-audit.md
    ├── perf-optimize.md
    ├── test-gen.md
    ├── api-gen.md
    └── doc-gen.md

→ 分两个目录管理
→ commands 放操作，skills 放分析

效果:
  → 小项目不需要区分，用 commands/ 即可
  → 大项目用两个目录分类更有组织性
  → 这是约定而非强制，根据项目规模选择
```
