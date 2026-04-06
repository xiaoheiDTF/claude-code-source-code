# Agent 字段: permissionMode（权限模式）

> 指南 3.2「权限控制」— 控制工具调用时的权限确认行为

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── code-reviewer.md          ← 不设 permissionMode → 每次确认
│       ├── auto-fixer.md             ← ★ permissionMode: acceptEdits → 自动执行
│       ├── deploy-agent.md           ← permissionMode: acceptEdits
│       └── refactor-planner.md       ← permissionMode: plan
└── src/
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/auto-fixer.md

---
description: Auto-fix lint errors, format code, and apply safe refactoring without asking for confirmation.
model: sonnet
permissionMode: acceptEdits          ← 自动接受编辑和 Bash 命令
maxTurns: 20
tools:
  - Read
  - Edit
  - Bash
  - Grep
  - Glob
---

Auto-fix code issues without asking.

## Tasks
1. Fix ESLint errors (auto-fixable ones)
2. Format code with Prettier
3. Remove unused imports
4. Add missing type annotations (simple cases)

## Rules
- Only fix, don't add new features
- Run linter after each batch of fixes
- Report what was changed
```

## 说明

```
permissionMode 选项:
  - default       → 每次工具调用都需用户确认
  - acceptEdits   → 自动接受 Edit/Write/Bash，不逐个确认
  - dontAsk       → 自动接受所有操作（最激进）
  - plan          → 先出计划，用户确认后再执行

适用场景:
  code-reviewer (只读)    → 不需要 permissionMode（不修改文件）
  auto-fixer (自动修复)   → acceptEdits（频繁 Edit，逐个确认太慢）
  deployer (部署)         → acceptEdits（Bash 命令需要自动执行）
  refactoring (重构)      → plan（先看计划再决定）
```

---

## 使用示例 1: permissionMode: acceptEdits — 批量 lint 自动修复

```
用户: "自动修复所有 lint 错误"

→ Agent(auto-fixer, permissionMode: acceptEdits)

Agent 执行（不需要用户逐个确认）:
  Turn 1: Bash("npx eslint --fix src/")           ✅ 自动执行
  Turn 2: Edit("src/utils.ts") 修复剩余问题       ✅ 自动执行
  Turn 3: Edit("src/api/users.ts") 删除未使用导入  ✅ 自动执行
  Turn 4: Bash("npm test")                        ✅ 自动执行

对比 — 如果不设 permissionMode:
  Turn 1: Bash("npx eslint --fix src/")
  → 弹窗: "允许执行 eslint --fix？" → 用户点确认
  Turn 2: Edit("src/utils.ts")
  → 弹窗: "允许编辑 src/utils.ts？" → 用户点确认
  ...（每个操作都要确认，非常慢）

field 效果:
  acceptEdits 让 Agent 连续执行 Edit/Bash 操作而无需等待确认。
  对于"修复 lint"这类可信任务，每次弹窗确认纯属浪费时间。
  从 4 次确认 → 0 次确认，执行体验从"不断被打断"
  变为"提交后等结果"。
```

---

## 使用示例 2: permissionMode: default — 只读代码审查

```
用户: "审查 src/services/payment.ts 的代码质量"

→ Agent(code-reviewer, permissionMode: default)

Agent 执行（只读操作，无需修改文件）:
  Turn 1: Read("src/services/payment.ts")
  Turn 2: Read("src/utils/currency.ts")       ← 读取依赖
  Turn 3: Read("src/types/payment.ts")         ← 读取类型定义
  Turn 4: 输出审查结果（纯文本，无文件操作）

  输出:
    ## Code Review: payment.ts

    ✅ L12-30: 支付流程逻辑清晰
    ⚠️ L45: 金额用 number 类型有精度风险
    ⚠️ L67: 错误处理只 catch 了网络错误，未处理超时
    💡 L89: 可提取 validateAmount 为独立工具函数

field 效果:
  default 模式下，只读操作（Read/Grep/Glob）本身就不会
  触发确认弹窗，所以 default 和 acceptEdits 在只读任务中
  行为相同。但 Agent 如果尝试 Edit 或 Bash，default 会
  弹窗确认，防止代码审查 Agent 意外修改代码。
```

---

## 使用示例 3: permissionMode: plan — 重构方案确认

```
用户: "重构 src/legacy/ 目录，用现代模式替换回调地狱"

→ Agent(refactor-planner, permissionMode: plan)

Agent 先出计划（不执行任何修改）:
  Turn 1-3: 分析 legacy 目录，识别所有回调模式
  Turn 4:   输出重构计划

  输出:
    ## 重构计划

    ### 概要
    - 文件数: 12 个需要重构
    - 变更类型: callback → async/await
    - 预计修改行数: ~340 行

    ### 分步计划
    1. **Phase 1**: 重构 src/legacy/db.js
       - callback → async/await（3 个函数）
       - 风险: 低（纯内部实现）

    2. **Phase 2**: 重构 src/legacy/api.js
       - callback → Promise chain → async/await
       - 风险: 中（对外接口不变，内部实现改）

    3. **Phase 3**: 重构 src/legacy/file-ops.js
       - fs callback → fs.promises
       - 风险: 低（标准 API 替换）

    是否按此计划执行？

→ 用户确认后，Agent 才开始执行 Edit/Bash 操作

field 效果:
  plan 模式强制 Agent 先展示完整计划，等用户确认后再动手。
  对于重构这种高风险操作，用户可以在 Agent 执行前：
  - 了解影响范围
  - 调整优先级
  - 取消不想要的步骤
  避免盲目执行后发现方向错误。
```

---

## 使用示例 4: permissionMode: acceptEdits — CI/CD 自动部署

```
用户: "部署 v2.3.1 到 staging 环境"

→ Agent(deploy-agent, permissionMode: acceptEdits)

Agent 连续执行部署流水线（无需确认）:
  Turn 1: Bash("git tag v2.3.1 && git push origin v2.3.1")
          ✅ 自动执行 → 创建并推送 tag
  Turn 2: Bash("docker build -t app:v2.3.1 .")
          ✅ 自动执行 → 构建镜像（耗时 2 分钟）
  Turn 3: Bash("docker push registry.example.com/app:v2.3.1")
          ✅ 自动执行 → 推送镜像仓库
  Turn 4: Bash("kubectl set image deployment/app app=registry.example.com/app:v2.3.1 -n staging")
          ✅ 自动执行 → 更新 K8s 部署
  Turn 5: Bash("kubectl rollout status deployment/app -n staging --timeout=120s")
          ✅ 自动执行 → 等待滚动更新完成
  Turn 6: Bash("curl -f https://staging.example.com/health")
          ✅ 自动执行 → 健康检查

  输出:
    ✅ 部署成功
    - 镜像: registry.example.com/app:v2.3.1
    - 环境: staging
    - Pod: 3/3 Running
    - 健康检查: 通过

对比 — 如果用 default 模式:
  每条 Bash 命令都需确认，6 次弹窗打断部署流程，
  中间等待时间浪费，部署从 3 分钟变成 15 分钟。

field 效果:
  部署流水线包含多条 Bash 命令，且命令之间有依赖关系
  （tag → build → push → deploy → verify）。
  acceptEdits 让整个链路自动执行，部署体验从
  "点 6 次确认"变成"一条命令搞定"。
```

---

## 使用示例 5: permissionMode: dontAsk — 最激进的完全自动模式

```
用户: "清理项目中所有临时文件和未使用的依赖"

→ Agent(cleanup-agent, permissionMode: dontAsk)

Agent 全自动执行（无任何确认）:
  Turn 1: Bash("find . -name '*.tmp' -o -name '*.bak' -o -name '.DS_Store'")
          ✅ 自动执行 → 找到 23 个临时文件
  Turn 2: Bash("find . -name '*.tmp' -delete && find . -name '*.bak' -delete")
          ✅ 自动执行 → 删除临时文件（不确认！）
  Turn 3: Bash("npx depcheck --json")
          ✅ 自动执行 → 分析未使用依赖
  Turn 4: Bash("npm uninstall moment lodash-old temp-lib")
          ✅ 自动执行 → 卸载未使用依赖（不确认！）
  Turn 5: Bash("rm -rf node_modules && npm install")
          ✅ 自动执行 → 重新安装（不确认！）
  Turn 6: Bash("npm test")
          ✅ 自动执行 → 验证项目仍正常

  输出:
    ✅ 清理完成
    - 删除临时文件: 23 个
    - 卸载未使用依赖: 3 个（moment, lodash-old, temp-lib）
    - 磁盘节省: 47MB
    - 测试: 全部通过

⚠️ 风险警告:
  dontAsk 意味着 Agent 可以执行任何操作而不需确认，包括:
  - 删除文件（rm）
  - 卸载依赖（npm uninstall）
  - 修改配置
  如果 Agent 判断错误，用户无法在执行前拦截。

field 效果:
  dontAsk 是最激进的权限模式，所有工具调用自动通过。
  与 acceptEdits 的区别:
    acceptEdits → 自动接受 Edit/Write/Bash，但某些敏感操作仍会提示
    dontAsk     → 完全不弹窗，一切操作自动执行
  仅用于: 可逆的低风险任务（清理临时文件、格式化）。
  禁止用于: 生产部署、数据库操作、不可逆删除。
```
