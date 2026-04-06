# Memory: CLAUDE.md 项目记忆

> 指南 8.1「三种记忆」— 项目 Memory 通过 CLAUDE.md 注入团队规范

## 目录结构

```
my-project/
├── CLAUDE.md                      ← 项目全局指令（提交 VCS）
├── CLAUDE.local.md                ← 项目私有指令（不提交）
├── src/
│   ├── components/
│   │   └── CLAUDE.md              ← 子目录级指令
│   └── api/
│       └── CLAUDE.md
└── .claude/
    └── rules/
        ├── coding-style.md        ← 无条件规则
        └── frontend.md            ← 条件规则（paths 匹配）
```

## 使用示例

### 示例 1：CLAUDE.md 作为项目入职手册

```markdown
# CLAUDE.md

## 技术栈
- Node.js 20 + TypeScript 5.x
- NestJS + Prisma ORM
- PostgreSQL 16 + Redis

## 项目结构
- src/modules/ — 业务模块
- src/common/ — 共享工具
- prisma/ — 数据库 schema

## 开发规范
- 使用依赖注入（NestJS 风格）
- API 端点必须有 DTO 验证
- 测试覆盖率 > 80%
```

→ 新成员 clone → Claude 自动知道项目规范

### 示例 2：CLAUDE.local.md 个人指令

```markdown
# CLAUDE.local.md（不提交 Git）

## 本地环境
- DB: localhost:5432, 用户 dev/dev
- Redis: localhost:6379

## 当前任务
- 正在重构 auth 模块
```

→ .gitignore 添加: CLAUDE.local.md

### 示例 3：子目录 CLAUDE.md 局部指令

```markdown
# src/components/CLAUDE.md

## 组件规范
- 函数式组件 + TypeScript
- 样式用 CSS Modules
- 状态: useState（局部）、Zustand（全局）
```

→ 编辑 src/components/ 下的文件时自动加载

### 示例 4：多层项目记忆叠加

```
编辑 src/components/Button.tsx 时加载:
1. ~/.claude/CLAUDE.md（全局个人）
2. CLAUDE.md（项目团队）
3. CLAUDE.local.md（项目个人）
4. src/components/CLAUDE.md（子目录）
5. .claude/rules/frontend.md（paths 匹配）

→ 5 层记忆提供精确的上下文
```

### 示例 5：CLAUDE.md 持续演进

```
项目初期:
  CLAUDE.md: 基本技术栈和目录结构

项目中期:
  + 添加编码规范（从 code review 中提炼）
  + 添加常见错误列表
  + 添加 API 设计约定

项目成熟:
  CLAUDE.md 成为完整的"项目百科"
  → 新成员读一遍就了解项目全貌
  → Claude 自动遵循所有规范
```
