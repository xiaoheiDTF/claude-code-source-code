# 10 - 工具函数 (Utils)

## 对应源码
- `src/utils/`（298 个 .ts 文件，最大的源码目录）

## 关键模块

| 模块 | 作用 |
|------|------|
| api.ts | API 通信封装 |
| auth.ts | 认证逻辑 |
| git.ts | Git 操作封装 |
| bash.ts | Shell/Bash 工具 |
| permissions.ts | 权限检查 |
| background/ | 后台任务管理 |

## 架构角色
全局共享的工具函数库，为工具系统、服务层、UI 层提供通用能力。
