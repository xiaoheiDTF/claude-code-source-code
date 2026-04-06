# 最小无条件规则 — 全局禁止 any

> 无 `paths` → 启动时始终加载，所有项目生效
> 放在 `~/.claude/rules/` 全局目录

## 目录结构

```
~/.claude/                           ← 用户全局目录
├── CLAUDE.md                        ← 全局指令
├── settings.json                    ← 全局配置
├── rules/
│   └── no-any.md                    ← ★ 本规则（全局，所有项目生效）
├── agents/
└── commands/

project-a/                           ← 项目 A（自动加载 no-any.md）
├── .claude/rules/
│   └── ...

project-b/                           ← 项目 B（也自动加载 no-any.md）
├── .claude/rules/
│   └── ...
```

## 规则文件内容

```markdown
# 文件: ~/.claude/rules/no-any.md

禁止使用 `any` 类型。如果类型不确定：
1. 使用 `unknown` 并通过类型守卫收窄
2. 使用泛型 `<T>` 保留类型信息
3. 使用 eslint-disable 注释临时绕过（必须附带原因说明）
```

## 触发示例

```
用户: "帮我处理这个 API 响应的数据"

Claude 看到 no-any.md 后:
✅ const data: unknown = await response.json()
   if (isUserResponse(data)) { ... }              ← unknown + 类型守卫

✅ function parse<T>(input: string): T { ... }    ← 泛型保留类型

❌ const data: any = await response.json()         ← any，被全局规则拒绝
❌ function process(input: any) { ... }            ← any 参数
```
