# 01 - 入口层 (Entrypoints)

## 对应源码
- `src/entrypoints/`
- `src/main.tsx`

## 关键文件
| 文件 | 作用 |
|------|------|
| `cli.tsx` | CLI 主入口：版本检查、环境初始化、Feature Flag 处理 |
| `mcp.ts` | MCP 服务器入口 |
| `init.ts` | 初始化逻辑 |
| `main.tsx` | 应用主体：React/Ink 初始化，加载 37+ 模块和服务 |

## 架构角色
应用的启动层，负责环境检测、Feature Flag 解析、以及将控制权转交给主应用循环。
