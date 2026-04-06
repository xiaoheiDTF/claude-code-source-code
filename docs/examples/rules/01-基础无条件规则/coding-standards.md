# 基础无条件规则 — TypeScript 编码规范

> 无 `paths` frontmatter → 启动时始终加载，对所有文件生效

## 目录结构

```
my-project/                          ← 项目根目录
├── CLAUDE.md                        ← 项目总指令
├── .claude/
│   ├── settings.json
│   └── rules/
│       ├── coding-standards.md      ← ★ 本规则（无条件，始终加载）
│       ├── commit-convention.md
│       └── ...
├── src/
│   ├── app.ts
│   └── utils/
│       └── helpers.ts
└── tests/
    └── app.test.ts
```

## 规则文件内容

```markdown
# 文件: .claude/rules/coding-standards.md

- TypeScript strict mode，禁止 `any` 类型
- 所有函数必须有显式返回类型注解
- 错误必须处理，不允许空 `catch` 块
- 使用 `interface` 定义对象类型，`type` 用于联合/交叉类型
- 优先使用 `const`，仅在需要重新赋值时使用 `let`，禁止 `var`
- 所有导出的函数和类必须有 JSDoc 注释
- 文件命名：kebab-case（如 `user-service.ts`）
- 导入排序：node 内置 → 外部包 → 内部模块，每组之间空行分隔
```

## 触发示例

```
用户: "帮我写一个 getUserById 函数"

Claude 看到 coding-standards.md 后会遵循:
✅ function getUserById(id: string): Promise<User | null> { ... }   ← 有返回类型
✅ const result = await db.user.findUnique(...)                      ← 用 const
✅ } catch (error) { logger.error(error); throw error; }            ← 有错误处理
❌ function getUserById(id) { ... }         ← 无返回类型，被规则拒绝
❌ let result = await ...                    ← 不需要重赋值却用了 let
❌ } catch (e) { }                          ← 空 catch，被规则拒绝
```
