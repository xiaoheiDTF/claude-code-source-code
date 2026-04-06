# settings.json: permissions.additionalDirectories

> 指南 7.2「主要配置项」— additionalDirectories 允许访问项目外的目录

## settings.json 配置

```json
{
  "permissions": {
    "additionalDirectories": [
      "/shared/components",
      "/home/user/common-utils"
    ]
  }
}
```

## 说明

```
additionalDirectories:
  格式: 字符串数组，绝对路径
  作用: 允许 Claude Code 访问项目根目录之外的目录

  默认行为:
    Claude Code 只能访问当前项目目录内的文件
    读写其他目录的文件会被拒绝

  使用场景:
    → 多项目共享公共组件库
    → 引用外部配置文件
    → 跨项目代码搜索
    → 共享工具函数/类型定义
```

## 使用示例

### 示例 1：访问共享组件库

```json
{
  "permissions": {
    "additionalDirectories": [
      "/projects/shared-ui-components"
    ]
  }
}
```

```
→ Claude 操作:
  Read: /projects/shared-ui-components/src/Button.tsx  → 允许 ✅
  Glob: "/projects/shared-ui-components/src/**/*.tsx"  → 允许 ✅
  Edit: /projects/shared-ui-components/src/Button.tsx  → 需确认 ⚠️

→ 没有 additionalDirectories 时:
  Read: /projects/shared-ui-components/...  → 拒绝 ❌（不在项目目录内）

效果:
  → Claude 可以读取和搜索外部共享组件
  → 编辑外部文件仍需确认（安全）
```

### 示例 2：访问多个外部目录

```json
{
  "permissions": {
    "additionalDirectories": [
      "/shared/libs/types",
      "/shared/libs/utils",
      "/home/user/.config/templates"
    ]
  }
}
```

```
→ Claude 可以访问 3 个外部目录:
  /shared/libs/types      → 共享 TypeScript 类型定义
  /shared/libs/utils      → 共享工具函数
  /home/user/.config/templates → 个人模板文件

→ 操作示例:
  Read: /shared/libs/types/api.d.ts       → ✅
  Read: /shared/libs/utils/format.ts      → ✅
  Read: /home/user/.config/templates/api-route.md → ✅
  Grep: "interface" /shared/libs/types/    → ✅
```

### 示例 3：monorepo 访问其他 package

```json
{
  "permissions": {
    "additionalDirectories": [
      "../packages/shared-core",
      "../packages/api-types",
      "../packages/test-utils"
    ]
  }
}
```

```
→ monorepo 中 Claude 可以访问其他 package:
  ../packages/shared-core    → 核心共享代码
  ../packages/api-types      → API 类型定义
  ../packages/test-utils     → 测试工具

→ 操作:
  Read: ../packages/api-types/src/users.ts  → ✅
  Grep: "export interface" ../packages/shared-core/src/ → ✅
```

### 示例 4：不配置时只能访问项目内

```
# 没有 additionalDirectories

my-project/ （项目根目录）
├── src/          → Claude 可以访问 ✅
├── tests/        → Claude 可以访问 ✅
└── package.json  → Claude 可以访问 ✅

/some/other/path/ → Claude 不能访问 ❌
~/other-project/  → Claude 不能访问 ❌

效果:
  → 默认沙箱限制在项目目录内
  → 防止意外访问敏感目录
  → 需要访问外部时显式配置
```

### 示例 5：与 allow/deny 组合保护外部目录

```json
{
  "permissions": {
    "additionalDirectories": [
      "/shared/company-libs"
    ],
    "allow": [
      "Read",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Write(/shared/company-libs/*)",
      "Edit(/shared/company-libs/*)"
    ]
  }
}
```

```
→ 外部目录的访问控制:
  Read: /shared/company-libs/types.ts   → allow ✅（只读）
  Glob: "/shared/company-libs/**/*.ts"  → allow ✅（搜索）
  Grep: "export" /shared/company-libs/  → allow ✅（搜索）
  Edit: /shared/company-libs/types.ts   → deny ❌（禁止修改）
  Write: /shared/company-libs/new.ts    → deny ❌（禁止创建）

效果:
  → 可以读取和搜索外部共享代码
  → 禁止修改外部共享代码
  → 共享库只能由维护者修改
  → allow + deny + additionalDirectories 三重保护
```
