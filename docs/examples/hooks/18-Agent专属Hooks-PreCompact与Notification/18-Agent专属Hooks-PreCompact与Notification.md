# Agent 专属 Hooks: PreCompact 与 Notification

> 指南 5.3「Agent 可用的 Hook 事件」— Agent 上下文压缩前 + 通知事件

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── long-task.md            ← 长任务 Agent（PreCompact 保存进度）
│       ├── monitor.md              ← 监控 Agent（Notification 响应）
│       ├── research.md             ← 研究Agent（PreCompact 摘要）
│       └── background-worker.md    ← 后台工作 Agent（通知 + 压缩）
├── src/
└── package.json
```

## 说明

```
PreCompact 事件:
  触发时机: Agent 的上下文即将被压缩前（超出 token 限制时）
  matcher: 通常为空 ""

  典型用途:
    - 保存关键信息（防止压缩丢失重要上下文）
    - 生成当前进度的摘要
    - 将重要数据写入文件持久化

  为什么需要 PreCompact:
    上下文压缩会丢弃早期对话内容
    Agent 可能丢失关键的需求、约束、中间结果
    PreCompact 提供最后的机会保存重要信息

Notification 事件:
  触发时机: Agent 收到通知时（如被 SendMessage、task 完成通知等）
  matcher: 通常为空 ""

  典型用途:
    - 响应外部消息
    - 根据通知调整行为
    - 记录收到的通知
```

## 使用示例

### 示例 1：长任务 Agent 压缩前保存进度到文件

```
Agent: .claude/agents/long-task.md
  PreCompact → 保存当前进度到进度文件

hooks:
  PreCompact:
    - matcher: ""
      hooks:
        - type: command
          command: |
            # 保存当前 git 状态
            BRANCH=$(git branch --show-current)
            CHANGED=$(git diff --name-only 2>/dev/null | tr '\n' ',')
            LAST_COMMIT=$(git log -1 --oneline)

            cat > .claude/progress/long-task-$(date +%Y%m%d-%H%M%S).md << EOF
            ## Agent 进度快照
            时间: $(date)
            分支: $BRANCH
            最近提交: $LAST_COMMIT
            已修改文件: $CHANGED
            EOF

            echo '{"additionalContext": "进度已保存到 .claude/progress/，压缩后可读取恢复"}'

→ Agent 执行大规模重构（已运行 30 分钟，上下文快满了）

→ PreCompact Hook 触发:
  1. 记录当前分支、最近提交、已修改文件
  2. 保存到 .claude/progress/long-task-20260405-143022.md
  3. 返回: "进度已保存，压缩后可读取恢复"

→ 上下文压缩后:
  Agent 读取 .claude/progress/ 文件 → 恢复之前的工作进度
  "我看到之前已经修改了 5 个文件，继续处理剩余的 3 个..."

效果:
  → 压缩前自动保存进度
  → 压缩后 Agent 可以恢复上下文
  → 防止长任务因压缩而丢失进度
```

### 示例 2：研究 Agent 压缩前生成摘要注入

```
Agent: .claude/agents/research.md
  PreCompact → 生成研究成果摘要

hooks:
  PreCompact:
    - matcher: ""
      hooks:
        - type: command
          command: |
            # 读取研究笔记文件
            if [ -f .claude/research-notes.md ]; then
              NOTES=$(cat .claude/research-notes.md)
              # 只取前 500 字符作为摘要
              SUMMARY=$(echo "$NOTES" | head -20)
              echo "{\"additionalContext\": \"研究摘要（压缩保留）: $SUMMARY\"}"
            fi

→ Agent 执行深度代码研究（已读取 50+ 文件，上下文快满）

→ PreCompact Hook 触发:
  1. 读取 .claude/research-notes.md（Agent 之前写入的笔记）
  2. 提取前 20 行作为摘要
  3. 返回: {"additionalContext": "研究摘要: 已分析 50 个文件..."}

→ 上下文压缩后:
  Agent 保留了研究摘要
  "根据之前的研究，项目使用了 3 种状态管理方案: Redux(15 files),
   Context(8 files), Zustand(3 files)。继续分析..."

效果:
  → 压缩前将关键发现注入为摘要
  → Agent 压缩后仍保留核心信息
  → 研究成果不会因压缩完全丢失
```

### 示例 3：监控 Agent 收到通知时记录并响应

```
Agent: .claude/agents/monitor.md
  Notification → 记录通知 + 注入上下文

hooks:
  Notification:
    - matcher: ""
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            # 记录通知
            echo "$(date '+%H:%M:%S') | $INPUT" >> .claude/notifications.log
            echo '{"additionalContext": "收到通知，请检查是否需要采取行动"}'

→ 后台监控 Agent 运行中:
  收到通知: "测试套件完成: 45 passed, 3 failed"
  → Notification Hook:
    1. 记录到 .claude/notifications.log
    2. 返回: {"additionalContext": "收到通知，请检查是否需要采取行动"}
  → Agent 检查通知内容:
    "测试有 3 个失败，我来分析失败原因..."

效果:
  → Agent 收到外部通知时自动记录
  → 注入提示让 Agent 关注通知内容
  → 监控 Agent 可以响应异步事件
```

### 示例 4：后台工作 Agent 的通知 + 压缩组合

```
Agent: .claude/agents/background-worker.md
  PreCompact → 保存任务队列进度
  Notification → 响应新任务分配

hooks:
  PreCompact:
    - matcher: ""
      hooks:
        - type: command
          command: |
            # 保存任务队列状态
            TOTAL=$(cat .claude/task-queue/total 2>/dev/null || echo "0")
            DONE=$(cat .claude/task-queue/done 2>/dev/null || echo "0")
            CURRENT=$(cat .claude/task-queue/current 2>/dev/null || echo "none")
            echo "{\"additionalContext\": \"任务进度: $DONE/$TOTAL 完成, 当前: $CURRENT\"}"

  Notification:
    - matcher: ""
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            echo "$INPUT" >> .claude/worker-notifications.log
            echo '{"additionalContext": "收到新通知，检查是否有新任务需要处理"}'

→ Agent 运行流程:

  1. 处理任务 1-5（正常）
  2. 收到通知: "新任务已添加到队列"
     → Notification Hook → Agent 读取新任务
  3. 处理任务 6-10（上下文增长）
  4. 上下文接近限制 → PreCompact Hook:
     → 保存: "任务进度: 10/25 完成, 当前: task-11"
  5. 压缩后恢复:
     → Agent 知道已完成了 10/25，从 task-11 继续
  6. 收到新通知 → Notification Hook 处理

效果:
  → PreCompact 保证长任务不丢失进度
  → Notification 让 Agent 能响应外部事件
  → 两个 Hook 配合实现可靠的后台工作流
```

### 示例 5：PreCompact 将重要约束持久化到 CLAUDE.md

```
Agent: .claude/agents/dev.md
  PreCompact → 将用户的关键约束写入文件

hooks:
  PreCompact:
    - matcher: ""
      hooks:
        - type: command
          command: |
            # 检查是否有活跃的约束文件
            if [ -f .claude/active-constraints.md ]; then
              CONSTRAINTS=$(cat .claude/active-constraints.md)
              # 将约束追加到 Agent 专属记忆文件
              echo "$CONSTRAINTS" >> .claude/agent-memory/dev/CONSTRAINTS.md
              echo "{\"additionalContext\": \"关键约束已持久化到记忆文件，压缩后可通过 memory 恢复\"}"
            fi

→ 场景:
  用户: "这次修改绝对不能改动 public API 的签名，
        必须保持向后兼容，测试覆盖率不能低于 80%"

→ Agent 工作一段时间后（上下文快满）:

→ PreCompact Hook 触发:
  1. 读取 .claude/active-constraints.md:
     "- 不修改 public API 签名
      - 保持向后兼容
      - 测试覆盖率 ≥ 80%"
  2. 追加到 .claude/agent-memory/dev/CONSTRAINTS.md
  3. 返回: "关键约束已持久化"

→ 压缩后:
  Agent 读取约束文件 → 记住用户的核心要求
  "我记得约束: 不改 API 签名、保持兼容、覆盖率 ≥ 80%"

效果:
  → 用户的硬性要求不会因压缩丢失
  → PreCompact 是保存关键约束的最后机会
  → 持久化到文件比 additionalContext 更可靠
  → 压缩后通过 memory 系统恢复
```
