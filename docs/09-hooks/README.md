# 09 - Hooks 层 (React Hooks)

## 对应源码
- `src/hooks/`（83 个 .ts/.tsx 文件）

## 代表性 Hooks

| Hook | 作用 |
|------|------|
| useApiKeyVerification | API Key 验证 |
| useCommandKeybindings | 命令快捷键绑定 |
| useDiffData | Diff 数据处理 |
| useBuddyNotification | Buddy 通知 |

## 架构角色
UI 组件的交互逻辑层，遵循 React Hooks 模式，将数据和副作用逻辑从组件中解耦。
