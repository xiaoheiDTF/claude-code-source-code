# 03 - 服务层 (Services)

## 对应源码
- `src/services/`

## 服务清单

| 服务 | 作用 |
|------|------|
| analytics | 事件日志和遥测指标收集 |
| tokenEstimation | Token 计数，控制 API 用量 |
| mcp | MCP 服务器生命周期管理 |
| SessionMemory | 会话级记忆持久化 |
| compact | 上下文压缩（突破上下文窗口限制） |
| lsp | 语言服务协议集成 |
| voice | 语音交互能力 |
| settingsSync | 配置同步 |
| autoDream | 自动会话管理 |
| diagnosticTracking | 错误和性能追踪 |
| AgentSummary | 代理对话摘要 |

## 架构角色
横切关注点的抽象层，为工具系统和主应用提供可复用的后端服务。
