# Skill: description 与 argument-hint

> 指南 6.2「Skill 定义」— description 描述命令，argument-hint 引导参数输入

## 目录结构

```
my-project/
├── .claude/
│   └── commands/
│       ├── commit.md              ← description + argument-hint
│       ├── review.md              ← 只有 description
│       ├── search.md              ← 详细的 argument-hint
│       └── fix.md                 ← 多行 description
└── src/
```

## Skill 文件内容

```markdown
# 文件: .claude/commands/commit.md

---
description: Generate a commit with conventional commit message
argument-hint: "[scope] description"
allowed-tools:
  - Bash
  - Read
---

Based on the current git diff, generate a conventional commit message
and commit the changes.

$ARGUMENTS
```

## 说明

```
description 字段:
  作用: 在 /help 列表和命令提示中显示
  格式: 一句话描述命令的功能
  最佳实践: 动词开头，简明扼要

argument-hint 字段:
  作用: 提示用户应该输入什么参数
  格式: 用方括号 [] 标记可选参数
  显示位置: 在命令名后面作为参数提示

$ARGUMENTS 变量:
  在 Skill 正文中的占位符
  被替换为用户实际输入的参数文本
  如果用户没输入参数 → 替换为空字符串
```

## 使用示例

### 示例 1：带 argument-hint 的 commit 命令

```
commit.md:
  description: "Generate a commit with conventional commit message"
  argument-hint: "[scope] description"

→ 用户输入: /help
  显示列表:
    /commit [scope] description — Generate a commit with conventional commit message

→ 用户输入: /commit fix login bug
  argument-hint 不匹配问题: 用户输入了 "fix login bug"
  $ARGUMENTS = "fix login bug"

  Skill 正文展开:
    "Based on the current git diff, generate a conventional commit message...
     fix login bug"

  → Claude 理解: scope=fix, description=login bug
  → 生成: fix(auth): resolve login validation bug

效果:
  → argument-hint 帮助用户知道该输入什么
  → $ARGUMENTS 携带用户输入传递给 Skill
  → Claude 结合 hint 和实际参数理解意图
```

### 示例 2：没有 argument-hint 的 review 命令

```
review.md:
  description: "Review code changes for quality and security issues"
  （无 argument-hint）

→ 用户输入: /help
  显示列表:
    /review — Review code changes for quality and security issues

→ 用户输入: /review
  $ARGUMENTS = ""（空）

  Skill 正文: "Review all uncommitted changes..."
  → Claude 自动审查所有未提交的变更

→ 用户输入: /review src/auth.ts
  $ARGUMENTS = "src/auth.ts"

  Skill 正文: "Review all uncommitted changes... src/auth.ts"
  → Claude 重点审查 src/auth.ts

效果:
  → 没有 argument-hint 也可以工作
  → 参数是可选的
  → 用户可以自由输入或不输入
```

### 示例 3：详细的 argument-hint 引导复杂参数

```
search.md:
  description: "Search codebase with advanced patterns"
  argument-hint: "<pattern> [--type ts|py|js] [--dir path] [--case sensitive]"

→ 用户输入: /help
  显示: /search <pattern> [--type ts|py|js] [--dir path] [--case sensitive]

→ 用户输入: /search "TODO|FIXME" --type ts --dir src/
  $ARGUMENTS = '"TODO|FIXME" --type ts --dir src/'

  → Claude 解析参数:
     pattern = "TODO|FIXME"
     type = ts
     dir = src/

→ 用户输入: /search deprecated
  $ARGUMENTS = 'deprecated'
  → 使用默认设置搜索

效果:
  → 复杂命令用 argument-hint 说明所有选项
  → [] 标记可选参数, <> 标记必选参数
  → 用户可以只输入核心参数
```

### 示例 4：多行 description 在帮助列表中显示

```
fix.md:
  description: "Fix a bug by analyzing the error and implementing a solution"

→ /help 列表:
  /fix — Fix a bug by analyzing the error and implementing a solution

对比简短 description:
fix.md:
  description: "Fix bugs"

→ /help 列表:
  /fix — Fix bugs

效果:
  → description 应该足够描述清楚功能
  → 但不要太长（建议 10-30 个英文单词）
  → 一句话说明"这个命令做什么"
```

### 示例 5：description 为空或不填

```
# 没有 frontmatter 的 Skill
task.md:
  （直接写正文，没有 --- frontmatter ---）

  Complete the following task: ...

→ /help 列表:
  /task — （无描述）

→ 用户仍然可以使用: /task 实现用户注册
  只是帮助列表中看不到描述

# description 为空
task.md:
  ---
  description: ""
  ---

  Complete the task...

→ 同样在帮助列表中无描述

效果:
  → description 不是必填字段
  → 但建议填写，方便用户了解命令用途
  → /help 列表的可读性取决于 description 质量
```
