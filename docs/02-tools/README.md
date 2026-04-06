# 02 - 工具系统 (Tools System)

## 对应源码
- `src/tools/`（37 个工具目录）

## 工具清单

### 文件操作
| 工具 | 作用 |
|------|------|
| FileReadTool | 读取文件（支持 PDF、图片、代码） |
| FileWriteTool | 写入文件 |
| FileEditTool | 编辑文件（sed 风格替换） |
| GlobTool | 文件模式匹配搜索 |
| GrepTool | 文件内容正则搜索 |

### Shell & 执行
| 工具 | 作用 |
|------|------|
| BashTool | Shell 命令执行（含安全沙盒） |
| PowerShellTool | Windows PowerShell 执行 |
| REPLTool | 交互式 REPL 会话 |

### 多智能体
| 工具 | 作用 |
|------|------|
| AgentTool | 子代理调度与管理 |
| SendMessageTool | 代理间消息通信 |

### 任务 & 团队
| 工具 | 作用 |
|------|------|
| TaskCreate/Get/List/Stop/Update | 任务全生命周期 CRUD |
| TeamCreate / TeamDelete | 团队管理 |

### 扩展 & 集成
| 工具 | 作用 |
|------|------|
| MCPTool | Model Context Protocol 第三方集成 |
| SkillTool | 技能调用系统 |
| LSPTool | Language Server Protocol 集成 |
| NotebookEditTool | Jupyter Notebook 编辑 |

### 网络 & 调度
| 工具 | 作用 |
|------|------|
| WebSearchTool | Web 搜索 |
| WebFetchTool | URL 内容获取 |
| CronTool | 定时/周期任务调度 |
| TodoWriteTool | Todo 列表管理 |

## 架构角色
Claude Code 的核心设计——**工具驱动架构**。每个工具是独立目录，自包含 UI、验证逻辑和安全检查。
