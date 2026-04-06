# 07 - 远程桥接 (Bridge / Remote)

## 对应源码
- `src/bridge/`（31 个文件）

## 关键文件

| 文件 | 作用 |
|------|------|
| bridgeMain.ts | 桥接主逻辑 |
| bridgeApi.ts | 桥接 API 接口 |
| bridgeMessaging.ts | 消息处理 |
| sessionRunner.ts | 会话运行管理 |

## 架构角色
通过 WebSocket 实现**远程会话**，使 Claude Code 可以作为远程服务运行，支持 IDE 集成、远程开发等场景。
