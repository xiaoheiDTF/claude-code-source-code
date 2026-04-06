# 环境变量: CLAUDE_CODE_DISABLE_BACKGROUND_TASKS

> 指南 九「环境变量」— 禁用后台任务

## 说明

```
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS:
  作用: 禁用所有后台任务
  格式: 布尔值（true/false 或 1/0）

  行为:
    设置为 true 后，所有 background: true 的 Agent 不会在后台运行
    Agent 只能在前台同步执行
    防止后台任务消耗额外资源

  适用场景:
    → 资源受限环境（CI、容器）
    → 需要完全同步执行的场景
    → 调试时排除后台任务干扰
```

## 使用示例

### 示例 1：禁用后台任务确保同步执行

```bash
export CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=true
claude
```

```
→ 所有 Agent 在前台同步执行
→ background: true 的配置被忽略
→ 用户可以实时看到每个操作
```

### 示例 2：CI 环境禁用后台

```yaml
env:
  CLAUDE_CODE_DISABLE_BACKGROUND_TASKS: "true"
```

```
→ CI 管道中不会有后台进程
→ 避免后台任务在 CI 结束后继续运行
→ 确保可预测的执行顺序
```

### 示例 3：调试时排除后台干扰

```bash
export CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1
claude
```

```
→ 调试模式下所有操作可见
→ 不会有后台 Agent 的意外行为
→ 方便排查问题
```

### 示例 4：不禁用时后台 Agent 正常运行

```bash
# 不设置（默认）
claude
```

```
→ background: true 的 Agent 在后台运行
→ 主流程不阻塞
→ 完成后自动通知
```

### 示例 5：临时禁用单个会话

```bash
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=true claude --chat
```

```
→ 只影响当前会话
→ 其他会话不受影响
```
