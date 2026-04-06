# Agent 字段: isolation（隔离运行）

> 指南 3.2「运行模式」— `isolation: worktree` 在隔离的 git worktree 中运行

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       └── experimenter.md          ← ★ isolation: worktree
├── src/
│   └── app.ts
└── package.json

# Agent 运行时自动创建临时 worktree:
.claude/
└── worktrees/
    └── experimenter-abc123/         ← 隔离的 git worktree
        ├── src/
        │   └── app.ts              ← 独立副本，不影响主目录
        └── ...
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/experimenter.md

---
description: Experimental agent that tries bold approaches in an isolated worktree. Safe to experiment without affecting main code.
isolation: worktree                  ← 在隔离的 git worktree 中运行
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
model: sonnet
permissionMode: acceptEdits
maxTurns: 30
---

You are an experimental developer. Try bold, unconventional approaches.

## Rules
- You are in an isolated worktree — safe to break things
- Try the most creative solution first
- Document what worked and what didn't
- If the experiment succeeds, describe how to apply to main branch
```

## 说明

```
isolation 选项:
  isolation: worktree → 在独立 git worktree 中运行
  isolation: remote   → 在远程环境中运行
  省略               → 在当前目录运行（默认）

worktree 隔离效果:
  1. 自动创建临时 git worktree
  2. Agent 在 worktree 中操作（不影响主目录）
  3. Agent 完成后:
     - 成果可以合并回主分支
     - 或丢弃（不影响主目录）

适用场景:
  - 实验性重构（不确定效果）
  - 大规模代码生成（可能需要大量修改）
  - A/B 对比方案（同时试两种方案）
```

## 使用示例

### 示例 1：实验性重写模块

```
用户: "试试把整个 auth 模块重写成函数式风格"

→ Agent(experimenter, isolation: worktree)

1. 自动创建 worktree:
   git worktree add .claude/worktrees/experimenter-abc123 HEAD

2. Agent 在 worktree 中操作:
   Edit("src/auth/jwt.ts")          ← 修改的是 worktree 副本
   Edit("src/auth/oauth.ts")        ← 修改的是 worktree 副本
   Write("src/auth/new-module.ts")  ← 新文件在 worktree 中
   Bash("npm test")                 ← 在 worktree 中运行测试

3. 主目录完全不受影响:
   src/auth/jwt.ts                  ← 原文件不变 ✅
   src/auth/oauth.ts                ← 原文件不变 ✅

4. Agent 输出:
   "实验完成。函数式重写成功，所有测试通过。
    应用到主分支: git merge experimenter-abc123
    或查看 diff: git diff HEAD..experimenter-abc123 -- src/auth/"

isolation 效果:
  → 所有修改发生在独立的 worktree 中
  → 主目录代码完全不受影响，即使实验搞砸了也安全
  → 实验成功后可以选择性合并，失败直接丢弃
```

### 示例 2：A/B 对比两种实现方案

```
用户: "不确定这个列表用虚拟滚动还是分页加载好，两种都试试"

→ 同时启动两个隔离 Agent:

  Agent A(experimenter-a, isolation: worktree):
    → 创建 worktree: .claude/worktrees/virtual-scroll-abc/
    → 实现虚拟滚动方案:
      Write("src/components/ProductList.virtual.tsx")
      Write("src/hooks/useVirtualScroll.ts")
      Bash("npm test")
    → 结果: 首屏渲染 200ms，内存占用低，滚动流畅

  Agent B(experimenter-b, isolation: worktree):
    → 创建 worktree: .claude/worktrees/pagination-def456/
    → 实现分页加载方案:
      Write("src/components/ProductList.paginated.tsx")
      Write("src/hooks/usePagination.ts")
      Bash("npm test")
    → 结果: 首屏渲染 80ms，实现简单，但翻页有白屏

isolation 效果:
  → 两个 Agent 在各自隔离的 worktree 中独立工作
  → 互不干扰，主目录代码保持不变
  → 用户对比两个结果后选择更好的方案合并
  → 丢弃的方案 worktree 可以直接删除
```

### 示例 3：大规模自动重构的安全试验

```
用户: "把项目中所有 class 组件迁移到 hooks 函数组件"

→ Agent(experimenter, isolation: worktree)

Agent 在隔离 worktree 中执行大规模修改:
  1. Grep("class.*extends React.Component")     ← 找到 45 个类组件
  2. 逐个转换:
     Edit("src/components/Header.tsx")          ← class → function
     Edit("src/components/Sidebar.tsx")         ← class → function
     Edit("src/components/Dashboard.tsx")       ← class → function
     ... （45 个文件）
  3. Write("src/hooks/useHeaderState.ts")       ← 提取自定义 hooks
  4. Bash("npm test")                           ← 验证所有测试
  5. Bash("npm run build")                      ← 验证构建

isolation 效果:
  → 涉及 45 个文件的大规模修改在隔离环境中进行
  → 如果中途发现某个组件无法转换，不会影响主目录
  → Agent 可以放心地做激进修改
  → 转换完成后用户检查 diff，确认无误再合并
  → 如果发现问题，直接丢弃 worktree 即可
```

### 示例 4：验证依赖升级兼容性

```
用户: "把 React 从 18 升级到 19，看看会不会有 breaking changes"

→ Agent(experimenter, isolation: worktree)

Agent 在隔离 worktree 中执行:
  1. Bash("npm install react@19 react-dom@19")   ← 升级依赖
  2. Bash("npm run build 2>&1")                   ← 检查构建
     → 发现 3 个类型错误
  3. Read 报错文件，分析 breaking changes
  4. Edit 修复兼容性问题
  5. Bash("npm test")                             ← 跑测试
     → 2 个测试失败（API 变更导致）
  6. 记录所有 breaking changes

isolation 效果:
  → 升级依赖有风险，隔离环境确保安全
  → 即使 npm install 导致 node_modules 大变化也不影响主目录
  → Agent 记录了所有需要修改的地方
  → 用户可以参考报告决定是否在主分支进行升级
  → 临时升级环境直接丢弃，无残留
```

### 示例 5：试验新技术栈并评估可行性

```
用户: "试试把状态管理从 Redux 换成 Zustand，看看需要改多少"

→ Agent(experimenter, isolation: worktree)

Agent 在隔离 worktree 中执行:
  1. Grep("useSelector\|useDispatch\|createSlice")  ← 找到 Redux 使用点
     → 发现 38 个文件使用 Redux
  2. Bash("npm install zustand")                     ← 安装 Zustand
  3. 先做一个文件的 POC:
     Write("src/store/userStore.ts")                 ← 创建 Zustand store
     Edit("src/components/UserProfile.tsx")           ← 迁移组件
     Bash("npm test -- --grep UserProfile")           ← 验证单个组件
  4. POC 成功后批量迁移关键模块:
     Edit("src/store/cartStore.ts")
     Edit("src/store/productStore.ts")
  5. 输出评估报告:
     "迁移可行性: 高。预计需要修改 38 个文件，工作量约 2 天。
      主要变更: store 定义方式、组件中的调用方式。
      已在 worktree 中完成 5 个模块的 POC 验证。"

isolation 效果:
  → 试验新技术栈不需要创建新分支或 fork
  → worktree 自动创建，试验结束自动清理
  → Agent 可以大胆安装新依赖、删除旧依赖
  → POC 代码保留在 worktree 中供参考
  → 评估报告帮助用户做决策，主项目零风险
```
