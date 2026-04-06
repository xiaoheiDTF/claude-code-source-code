# Skill: paths 条件匹配

> 指南 6.2「Skill 定义」— paths 字段限制 Skill 只在访问匹配文件时生效

## 目录结构

```
my-project/
├── .claude/
│   ├── commands/
│   │   ├── ts-review.md           ← paths: ["**/*.ts", "**/*.tsx"]
│   │   ├── db-check.md            ← paths: ["prisma/**", "**/*.sql"]
│   │   ├── api-doc.md             ← paths: ["src/api/**"]
│   │   ├── frontend-fix.md        ← paths: ["src/components/**", "src/styles/**"]
│   │   └── full-review.md         ← 无 paths（始终可用）
├── src/
│   ├── api/
│   ├── components/
│   └── styles/
├── prisma/
└── package.json
```

## 说明

```
paths 字段:
  格式: YAML 列表，gitignore 风格的路径模式
  作用: 限制 Skill 只在访问匹配路径的文件时可用
  语法: 同 Rules 的 paths（支持 **、*、{a,b}、!）

  行为:
    - 指定 paths → Skill 只在相关文件被访问时出现在推荐中
    - 不指定 paths → Skill 始终可用

  与 Rules paths 的区别:
    Rules paths → 控制规则何时加载
    Skill paths → 控制命令何时推荐/可用

  实际效果:
    paths 不阻止手动调用 /command
    但影响 Skill 的推荐和自动加载
```

## 使用示例

### 示例 1：TypeScript 审查 Skill 只在 .ts/.tsx 文件相关时推荐

```
ts-review.md:
  ---
  description: Review TypeScript code for type safety
  paths:
    - "**/*.ts"
    - "**/*.tsx"
  allowed-tools:
    - Read
    - Grep
    - Glob
  ---

  Review TypeScript files for type safety issues.

→ 用户正在编辑 src/auth/login.tsx:
  推荐的 Skill 列表中包含 /ts-review ✅

→ 用户正在编辑 styles/global.css:
  推荐的 Skill 列表中不包含 /ts-review ❌

→ 用户正在查看 prisma/schema.prisma:
  推荐的 Skill 列表中不包含 /ts-review ❌

→ 用户手动输入: /ts-review
  无论 paths 匹配与否，手动调用始终有效 ✅

效果:
  → paths 控制 Skill 的推荐时机
  → 减少不相关 Skill 的干扰
  → 手动调用不受 paths 限制
```

### 示例 2：数据库检查 Skill 只在 prisma 目录相关时推荐

```
db-check.md:
  ---
  description: Check database schema and migrations
  paths:
    - "prisma/**"
    - "**/*.sql"
  ---

→ 用户访问 prisma/schema.prisma:
  /db-check 出现在推荐中 ✅

→ 用户访问 prisma/migrations/20260405-add-user-email/migration.sql:
  /db-check 出现在推荐中 ✅

→ 用户访问 src/services/user.service.ts:
  /db-check 不推荐（虽然文件可能用数据库）
  → 但用户仍可手动 /db-check ✅

效果:
  → 数据库相关文件触发推荐
  → 非 SQL/Prisma 文件不会推荐
  → 聚焦相关上下文
```

### 示例 3：API 文档 Skill 匹配 API 目录

```
api-doc.md:
  ---
  description: Generate API documentation
  paths:
    - "src/api/**"
  ---

→ 用户修改 src/api/users.ts:
  /api-doc 推荐出现 ✅
  "检测到 API 文件变更，是否需要更新文档？"

→ 用户修改 src/utils/helpers.ts:
  /api-doc 不推荐 ❌

→ 用户修改 src/components/UserList.tsx:
  /api-doc 不推荐 ❌

效果:
  → 只在 API 目录下的文件变更时推荐
  → 前端代码和工具函数不触发 API 文档推荐
```

### 示例 4：多路径匹配前端文件

```
frontend-fix.md:
  ---
  description: Auto-fix frontend code issues
  paths:
    - "src/components/**"
    - "src/styles/**"
    - "src/hooks/**"
  ---

→ 用户修改 src/components/Button.tsx: 匹配 components/** ✅
→ 用户修改 src/styles/theme.css: 匹配 styles/** ✅
→ 用户修改 src/hooks/useAuth.ts: 匹配 hooks/** ✅
→ 用户修改 src/api/auth.ts: 不匹配 ❌
→ 用户修改 src/utils/format.ts: 不匹配 ❌

效果:
  → 多个路径模式用列表指定
  → 匹配任一模式即可
  → 前端三件套: components + styles + hooks
```

### 示例 5：无 paths 的 Skill 始终推荐

```
full-review.md:
  ---
  description: Full project review
  allowed-tools:
    - Read
    - Grep
    - Glob
    - Bash
  ---

  Review the entire project structure and code quality.

→ 没有 paths 字段

→ 在任何文件、任何目录下:
  /full-review 始终出现在推荐中 ✅

→ 无论用户在编辑什么:
  都能看到 /full-review 命令

效果:
  → 不指定 paths = 始终可用
  → 适合通用型 Skill（项目级审查、全局搜索等）
  → 建议: 通用命令不设 paths，特定命令设 paths
```
