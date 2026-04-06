# 08 - UI 组件层 (UI Components)

## 对应源码
- `src/components/`（113 个 .tsx 文件）
- `src/ink/`（43 个文件 — Ink 终端 UI 框架适配）
- `src/context/`（9 个 React Context）

## 关键组件

| 组件 | 作用 |
|------|------|
| App.tsx | 主应用组件 |
| AgentProgressLine.tsx | 代理进度显示 |
| ConsoleOAuthFlow.tsx | OAuth 认证流程 UI |
| CompanionSprite.tsx | Buddy/助手精灵 UI |

## UI 技术栈
- **React + Ink** — 终端 UI 渲染
- **React Context** — 状态注入
- **自定义 Hooks** — 交互逻辑

## 架构角色
终端用户界面的全部实现，基于 React 组件化模式，通过 Ink 适配器在终端中渲染。
