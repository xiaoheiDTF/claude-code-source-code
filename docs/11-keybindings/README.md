# 11 - 快捷键系统 (Keybindings)

## 对应源码
- `src/keybindings/`（14 个 .ts 文件）

## 关键文件

| 文件 | 作用 |
|------|------|
| defaultBindings.ts | 默认快捷键绑定 |
| useKeybinding.ts | 快捷键 Hook |
| resolver.ts | 快捷键解析器 |

## 架构角色
终端应用的键盘交互系统，支持自定义快捷键映射和 Vim 模式。
