# Memory: Auto Memory 自动记忆

> 指南 8.1「三种记忆」— Auto Memory 自动积累用户偏好

## 目录结构

```
~/.claude/
└── memory/
    └── MEMORY.md                  ← 自动积累的用户偏好
```

## 说明

```
Auto Memory 特点:
  路径: ~/.claude/memory/MEMORY.md
  作用: 自动积累用户的偏好和习惯
  范围: 当前用户，所有项目共享
  生命周期: 持久化，跨会话保留

写入方式:
  1. 用户命令: /remember <内容> → 直接写入 MEMORY.md
  2. Claude 自动记忆 → 对话中识别到用户偏好

读取方式:
  → 每次 Claude 会话启动时自动加载
  → 作为系统上下文注入给 Claude
```

## 使用示例

### 示例 1：使用 /remember 保存偏好

```
→ 用户: /remember 我的项目使用 pnpm，不要用 npm

→ Claude 写入 ~/.claude/memory/MEMORY.md:
  - 我的项目使用 pnpm，不要用 npm

→ 下次会话:
  用户: "安装 axios"
  Claude: "pnpm add axios" ← 自动用 pnpm
```

### 示例 2：积累技术栈偏好

```
/remember 我主要用 TypeScript 开发
/remember 前端使用 React + Next.js
/remember 数据库用 Prisma ORM
/remember 测试框架用 Vitest

→ MEMORY.md 逐渐丰富:
  - 我主要用 TypeScript 开发
  - 前端使用 React + Next.js
  - 数据库用 Prisma ORM
  - 测试框架用 Vitest

→ 下次新项目 Claude 自动使用这些技术栈
```

### 示例 3：代码风格偏好

```
/remember 代码风格: 函数式组件，不用 class；使用 arrow function；2 空格缩进

→ 生成代码时自动遵循:
  ✅ const Button = () => { ... }
  ❌ class Button extends Component
  ✅ 2 空格缩进
```

### 示例 4：记住项目约定

```
/remember 提交信息格式: <type>(<scope>): <description>，必须包含 Jira ID

→ /commit 时自动按约定格式生成:
  "feat(auth): add SSO login [JIRA-123]"
```

### 示例 5：跨项目通用的完整偏好文件

```
~/.claude/memory/MEMORY.md:

## 开发环境
- 操作系统: Windows 10
- Shell: Git Bash
- 编辑器: VS Code
- 包管理器: pnpm

## 编码偏好
- 语言: TypeScript strict mode
- 前端: React 函数式组件 + hooks
- 状态管理: Zustand
- 样式: Tailwind CSS
- 测试: Vitest + Testing Library

## 交流偏好
- 使用中文交流
- 代码注释用英文
- 解释时附带代码示例

## 注意事项
- 不要使用 any 类型
- 不要用 class 组件
- 提交信息用 Conventional Commits
```
