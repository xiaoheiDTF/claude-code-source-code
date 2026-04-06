# 环境变量: CLAUDE_CODE_SUBAGENT_MODEL

> 指南 九「环境变量」— 强制所有子 Agent 使用指定模型

## 说明

```
CLAUDE_CODE_SUBAGENT_MODEL:
  作用: 强制所有子 Agent（subagent）使用指定的 AI 模型
  格式: 模型名称字符串
  可选值: opus, sonnet, haiku, inherit

  行为:
    设置后，所有通过 Agent 工具启动的子 Agent 都使用该模型
    覆盖 Agent frontmatter 中的 model 字段
    不影响主 Agent 的模型选择

  适用场景:
    → 成本控制: 所有子 Agent 用 haiku 省钱
    → 质量保证: 所有子 Agent 用 opus 确保质量
    → 统一管理: 不用逐个 Agent 配置模型
```

## 使用示例

### 示例 1：所有子 Agent 用 haiku 节省成本

```bash
export CLAUDE_CODE_SUBAGENT_MODEL=haiku
claude
```

```
→ 主 Agent: 使用 opus（默认或用户选择）
→ 子 Agent（code-reviewer）: 使用 haiku ✅
→ 子 Agent（test-runner）: 使用 haiku ✅
→ 子 Agent（security-scanner）: 使用 haiku ✅

→ 即使 Agent 定义了 model: opus，也被覆盖为 haiku
→ 成本降低约 60 倍（opus → haiku）
```

### 示例 2：所有子 Agent 用 sonnet 平衡质量和成本

```bash
export CLAUDE_CODE_SUBAGENT_MODEL=sonnet
claude
```

```
→ 所有子 Agent 使用 sonnet
→ sonnet 是成本和能力的最佳平衡点
→ 适合日常开发
```

### 示例 3：不设置时各 Agent 使用各自定义的模型

```bash
# 不设置环境变量
claude
```

```
→ code-reviewer Agent (model: opus) → 使用 opus
→ test-runner Agent (model: haiku) → 使用 haiku
→ dev Agent (model: sonnet) → 使用 sonnet
→ 每个 Agent 按自己的配置运行
```

### 示例 4：CI/CD 环境统一用 haiku

```yaml
# .github/workflows/claude-ci.yml
env:
  CLAUDE_CODE_SUBAGENT_MODEL: haiku
```

```
→ CI 环境中所有子 Agent 用 haiku（最低成本）
→ 不影响功能，只是推理深度较浅
→ 适合批量自动化任务
```

### 示例 5：临时切换模型测试

```bash
# 临时设置，只影响当前会话
CLAUDE_CODE_SUBAGENT_MODEL=opus claude
```

```
→ 当前会话所有子 Agent 用 opus
→ 关闭终端后环境变量失效
→ 适合临时需要高质量输出的场景
```
