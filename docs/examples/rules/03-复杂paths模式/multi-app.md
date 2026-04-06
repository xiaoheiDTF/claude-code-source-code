---
# 逗号分隔：与 YAML 列表写法等效
paths: "packages/ui/**, packages/design-system/**, packages/components/**"
---

# 复杂 paths — 逗号分隔多路径

> 逗号分隔与 YAML 列表写法等效，适合路径较短时一行写完

## 目录结构

```
my-monorepo/
├── .claude/
│   └── rules/
│       └── ui-lib.md                ← ★ 本规则（逗号分隔三个 packages）
├── packages/
│   ├── ui/                          ← 匹配 ✓
│   │   └── src/
│   │       └── Button.tsx           ← 触发
│   ├── design-system/               ← 匹配 ✓
│   │   └── src/
│   │       └── tokens.ts            ← 触发
│   ├── components/                  ← 匹配 ✓
│   │   └── src/
│   │       └── Modal.tsx            ← 触发
│   ├── core/                        ← 不匹配 ✗
│   └── shared/                      ← 不匹配 ✗
└── apps/
    └── web/
        └── src/                     ← 不匹配 ✗
```

## 逗号分隔 vs YAML 列表（两种写法完全等效）

```yaml
# 写法 1：逗号分隔（一行）
paths: "packages/ui/**, packages/design-system/**, packages/components/**"

# 写法 2：YAML 列表（多行）
paths:
  - "packages/ui/**"
  - "packages/design-system/**"
  - "packages/components/**"
```

## 触发示例

```
场景 1: 用户让 Claude 在 ui 包加新组件
  → Write("packages/ui/src/Select.tsx")
  → 匹配 "packages/ui/**" ✓ → 注入

场景 2: 用户让 Claude 修改设计 token
  → Edit("packages/design-system/src/tokens.ts")
  → 匹配 "packages/design-system/**" ✓ → 注入

场景 3: 用户让 Claude 改 web 应用代码
  → Edit("apps/web/src/App.tsx")
  → 不匹配 ✗ → 不注入（由 frontend.md 负责）
```
