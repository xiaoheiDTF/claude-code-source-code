# 04 - 状态管理 (State Management)

## 对应源码
- `src/state/`

## 关键文件

| 文件 | 作用 |
|------|------|
| AppState.tsx | 主应用状态 React Context |
| AppStateStore.ts | 不可变状态存储实现 |
| selectors.ts | 计算属性 / 派生状态 |
| onChangeAppState.ts | 状态变更通知 |
| teammateViewHelpers.ts | 代理视图状态辅助 |
| store.ts | 通用 Store 工具 |

## 架构角色
采用 **Immutable Store + React Context** 模式，为整个应用提供集中式、不可变的状态管理。
