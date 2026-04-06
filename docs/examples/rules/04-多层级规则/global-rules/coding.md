# 多层级规则 — 全局层（优先级最低）

> 放在 `~/.claude/rules/`，所有项目生效，可被项目级和子目录级覆盖

## 目录结构

```
~/.claude/                                    ← 用户 HOME 目录
├── CLAUDE.md                                 ← 全局个人指令
├── settings.json                             ← 全局配置
├── rules/
│   └── coding.md                             ← ★ 本规则（全局，所有项目生效）
├── agents/
└── commands/

~/projects/
├── project-a/                                ← 项目 A（加载全局 + 项目级）
│   ├── .claude/rules/
│   │   └── coding.md                        ← 项目级覆盖全局同名
│   └── ...
└── project-b/                                ← 项目 B（只加载全局，无项目级覆盖）
    └── ...
```

## 规则文件内容

```markdown
# 文件: ~/.claude/rules/coding.md

- TypeScript strict mode，禁止 any
- 函数必须有返回类型
- 使用 ESLint + Prettier
- Conventional Commits
- 变量: camelCase | 类/接口: PascalCase | 常量: UPPER_SNAKE_CASE
- 文件名: kebab-case
- 不硬编码密钥，SQL 参数化，验证用户输入
```

## 触发示例

```
场景 A: 在 project-b 中（无项目级 coding.md 覆盖）

用户: "帮我写一个 formatCurrency 函数"

Claude 上下文:
  ✅ ~/.claude/rules/coding.md           ← 全局规则生效

生成结果:
  function formatCurrency(amount: number, currency: string): string { ... }
  ✅ camelCase 命名  ✅ 有返回类型  ✅ 无 any


场景 B: 在 project-a 中（有项目级 coding.md 覆盖全局）

用户: "帮我写一个 formatCurrency 函数"

Claude 上下文:
  ❌ ~/.claude/rules/coding.md           ← 被项目级同名覆盖，不生效
  ✅ project-a/.claude/rules/coding.md   ← 项目级规则生效
     （项目级可能允许 Next.js metadata 中用 any，覆盖全局的禁止 any）

  → 按项目级规范生成
```
