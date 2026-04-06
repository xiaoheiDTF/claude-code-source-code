# settings.json 示例集合

> 对应指南「七、settings.json 配置」

## 文件夹列表

| 文件夹 | 对应指南知识点 | 演示内容 |
|--------|--------------|---------|
| [01-配置文件位置与优先级](./01-配置文件位置与优先级/) | 7.1 位置优先级 | 4 个配置位置、合并规则、项目覆盖全局 |
| [02-permissions-allow白名单](./02-permissions-allow白名单/) | 7.2 permissions | 自动允许的操作模式匹配 |
| [03-permissions-deny黑名单](./03-permissions-deny黑名单/) | 7.2 permissions | 不可覆盖的拒绝规则 |
| [04-permissions-defaultMode权限模式](./04-permissions-defaultMode权限模式/) | 7.2 permissions | default/acceptEdits/plan/auto 模式选择 |
| [05-permissions-additionalDirectories](./05-permissions-additionalDirectories/) | 7.2 permissions | 允许访问项目外的目录 |
| [06-hooks配置](./06-hooks配置/) | 7.2 hooks | settings.json 中 hooks 字段的写法 |
| [07-mcpServers-MCP服务器](./07-mcpServers-MCP服务器/) | 7.2 mcpServers | PostgreSQL、GitHub 等外部 MCP 配置 |
| [08-agentModel与theme](./08-agentModel与theme/) | 7.2 agentModel | 模型选择和主题设置 |
| [09-verbose与调试](./09-verbose与调试/) | 7.2 verbose | 详细日志输出和调试 |
| [10-完整settings.json实战](./10-完整settings.json实战/) | 7.2 综合 | 前端团队/后端/安全审计/CI/新开发者 5 套完整配置 |

## 每个文件夹的结构

```
每个文件夹内包含:
1. 完整的 settings.json 配置示例
2. 字段说明和可选值
3. 使用示例 — 5 个不同场景的配置效果
```

## 快速开始

1. 选择一个配置场景
2. 复制 settings.json 内容到 `.claude/settings.json`
3. 根据项目需求调整 allow/deny 列表
4. 提交到 VCS 共享给团队（settings.json）
5. 个人偏好放 settings.local.json（不提交）
