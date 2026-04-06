# Skill: $ARGUMENTS 参数传递

> 指南 6.2「Skill 定义」— $ARGUMENTS 占位符被替换为用户输入的参数

## 目录结构

```
my-project/
├── .claude/
│   └── commands/
│       ├── commit.md              ← $ARGUMENTS 作为提交信息提示
│       ├── find.md                ← $ARGUMENTS 作为搜索关键词
│       ├── refactor.md            ← $ARGUMENTS 作为重构目标
│       ├── test.md                ← $ARGUMENTS 作为测试范围
│       └── explain.md             ← $ARGUMENTS 多处使用
└── src/
```

## 说明

```
$ARGUMENTS 变量:
  格式: $ARGUMENTS（大写，精确匹配）
  作用: 在 Skill 正文中的占位符
  替换: 执行时被替换为用户输入的参数文本

  行为:
    - 用户输入: /command hello world → $ARGUMENTS = "hello world"
    - 用户输入: /command → $ARGUMENTS = ""（空字符串）
    - 可以在正文中多处使用 $ARGUMENTS
    - 替换是纯文本替换（不做解析）

  与 argument-hint 的关系:
    argument-hint → 告诉用户该输入什么（提示）
    $ARGUMENTS → 接收用户实际输入的值（变量）
```

## 使用示例

### 示例 1：commit 命令的参数作为提交提示

```
commit.md 正文:
  "Based on the current git diff, generate a commit message.
   Additional context: $ARGUMENTS"

→ 用户输入: /commit fix login timeout
  $ARGUMENTS = "fix login timeout"

  展开后:
  "Based on the current git diff, generate a commit message.
   Additional context: fix login timeout"

  → Claude 理解: 用户强调修复登录超时
  → 生成: fix(auth): resolve login session timeout

→ 用户输入: /commit
  $ARGUMENTS = ""

  展开后:
  "Based on the current git diff, generate a commit message.
   Additional context: "

  → Claude 自己分析 diff 生成合适的提交信息

效果:
  → 参数是可选的
  → 有参数时引导 Claude 的理解方向
  → 无参数时 Claude 自主分析
```

### 示例 2：find 命令的参数作为搜索关键词

```
find.md 正文:
  "Search the entire codebase for: $ARGUMENTS
   Show file paths, line numbers, and surrounding context."

→ 用户输入: /find deprecated API calls
  $ARGUMENTS = "deprecated API calls"

  展开后:
  "Search the entire codebase for: deprecated API calls
   Show file paths, line numbers, and surrounding context."

  → Claude 执行 Grep: "deprecated" 相关模式
  → 列出所有使用已废弃 API 的位置

→ 用户输入: /find TODO|FIXME|HACK
  $ARGUMENTS = "TODO|FIXME|HACK"

  → Claude 使用正则搜索所有待办标记

效果:
  → $ARGUMENTS 直接传递搜索关键词
  → Claude 可以灵活处理各种搜索模式
```

### 示例 3：refactor 命令的参数指定重构目标

```
refactor.md 正文:
  "Refactor the following: $ARGUMENTS
   Maintain all existing functionality and tests.
   Follow the project's coding standards."

→ 用户输入: /refactor convert class components to hooks
  $ARGUMENTS = "convert class components to hooks"

  → Claude:
  1. Glob: 找到所有 class 组件
  2. 逐个转换为 hooks 函数组件
  3. 运行测试验证

→ 用户输入: /refactor extract shared logic from utils into separate modules
  $ARGUMENTS = "extract shared logic from utils into separate modules"

  → Claude:
  1. Read: src/utils.ts（一个巨大的工具文件）
  2. 识别共享逻辑分组
  3. 拆分为独立模块: string.ts, date.ts, validation.ts

效果:
  → 参数指定重构的具体目标
  → 同一个 Skill 可以执行不同类型的重构
  → 灵活性来自 $ARGUMENTS 的自由文本输入
```

### 示例 4：test 命令的参数控制测试范围

```
test.md 正文:
  "Run tests for: $ARGUMENTS
   If no specific target, run all tests.
   Generate a summary report."

→ 用户输入: /test src/auth/
  $ARGUMENTS = "src/auth/"

  → Claude: npx jest src/auth/ --verbose

→ 用户输入: /test user service
  $ARGUMENTS = "user service"

  → Claude: 搜索 user 相关测试文件 → npx jest --testPathPattern="user"

→ 用户输入: /test
  $ARGUMENTS = ""

  → Claude: npm test -- --coverage（运行全部测试）

效果:
  → 参数控制测试范围
  → 空参数执行完整测试套件
  → Claude 理解自然语言描述并匹配文件
```

### 示例 5：$ARGUMENTS 在正文中多处使用

```
explain.md 正文:
  "Explain the code in: $ARGUMENTS

   Focus areas:
   1. What does $ARGUMENTS do?
   2. How is $ARGUMENTS implemented?
   3. What are the dependencies of $ARGUMENTS?

   Provide a clear explanation suitable for code review."

→ 用户输入: /explain src/auth/jwt.ts

  展开后:
  "Explain the code in: src/auth/jwt.ts

   Focus areas:
   1. What does src/auth/jwt.ts do?
   2. How is src/auth/jwt.ts implemented?
   3. What are the dependencies of src/auth/jwt.ts?

   Provide a clear explanation suitable for code review."

  → Claude 生成结构化解释:
     1. 功能: JWT token 的生成、验证、刷新
     2. 实现: 使用 jsonwebtoken 库，HS256 算法
     3. 依赖: config.ts (密钥), User model (用户数据)

效果:
  → $ARGUMENTS 可以在正文中多次出现
  → 每次出现都被替换为相同的参数值
  → 构建结构化的提示词模板
  → 同一参数驱动多个分析角度
```
