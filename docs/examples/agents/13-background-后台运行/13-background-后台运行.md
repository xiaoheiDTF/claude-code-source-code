# Agent 字段: background（后台运行）

> 指南 3.2「运行模式」— `background: true` 让 Agent 始终在后台运行

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       └── test-runner.md           ← ★ background: true
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/test-runner.md

---
description: Run tests in the background. Results are reported when complete.
background: true                     ← 始终后台运行，不阻塞主对话
tools:
  - Read
  - Bash
  - Grep
  - Glob
maxTurns: 10
---

Run the specified tests and report results.

## Output
- Test suite name
- Pass/Fail count
- Failed test details
- Coverage summary (if applicable)
```

## 说明

```
background 选项:
  background: true  → 始终后台运行
  background: false → 前台运行（默认）
  省略              → 前台运行（默认）

后台运行效果:
  → 主对话不等待 Agent 完成
  → 用户可以继续其他工作
  → Agent 完成后发送通知
  → 工具池受限：只保留 ASYNC_AGENT_ALLOWED_TOOLS

前台 vs 后台:
  前台: 用户: "跑测试" → 等待 → 测试结果 → 继续对话
  后台: 用户: "跑测试" → 立即返回 → 用户继续其他工作 → 测试完成通知
```

## 使用示例

### 示例 1：后台运行完整测试套件，同时继续开发

```
用户: "跑一下完整的测试套件"

→ Claude: "测试已在后台启动，你可以继续其他工作。"

主对话继续:
  用户: "同时帮我看看这个 bug"       ← 不用等测试完成
  → Claude 继续处理 bug

后台 test-runner 执行:
  Turn 1: Bash("npm test -- --coverage")    ← 后台运行
  Turn 2-3: 读取失败的测试文件
  Turn 4: 输出结果

  → 通知: "测试完成: 142 passed, 3 failed, 覆盖率 87%"

background 效果:
  → Agent 在后台执行，主对话立即恢复
  → 用户不需要等待漫长的测试过程
  → 测试完成后通过通知告知结果
  → 用户可以在等待期间处理其他任务
```

### 示例 2：后台执行代码质量扫描

```
用户: "扫描一下整个项目的代码质量"

→ Claude: "代码质量扫描已在后台启动，你可以继续其他工作。"

后台 Agent 执行:
  Turn 1: Bash("npx eslint src/ --format json")          ← ESLint 扫描
  Turn 2: Bash("npx tsc --noEmit 2>&1")                  ← TypeScript 类型检查
  Turn 3: Bash("npx prettier --check src/")               ← 格式检查
  Turn 4: Bash("npm run complexity 2>/dev/null || echo 'N/A'")  ← 复杂度分析

主对话继续:
  用户: "帮我重构一下 utils 目录"     ← 同时进行重构工作
  → Claude 在前台帮用户重构

  → 通知: "代码质量扫描完成: ESLint 23 warnings, TypeScript 2 errors, Prettier 5 files need formatting"

background 效果:
  → 代码质量扫描可能耗时几分钟，后台运行不阻塞
  → 前台主对话可以做完全不同的工作
  → 扫描结果通过通知送达，用户决定何时处理
```

### 示例 3：后台监控构建过程

```
用户: "触发生产构建，同时帮我准备 release notes"

→ Claude: "构建已在后台启动。"

后台 Agent 执行:
  Turn 1: Bash("npm run build:production 2>&1 | tee build.log")
  Turn 2: 如果构建失败 → 读取 build.log 分析错误
  Turn 3: 输出构建结果

主对话同时进行:
  用户: "列出这个版本的所有改动"
  → Claude: "基于 git log，这个版本的改动包括..."
  → Claude 帮用户编写 release notes

  → 通知: "生产构建完成: bundle size 234KB (gzip 78KB), 构建耗时 45s"

background 效果:
  → 构建过程通常很长，后台运行节省等待时间
  → 用户在等待构建时可以并行准备发布材料
  → Agent 报告构建产物的大小和耗时
  → 如果构建失败，Agent 会分析日志并报告原因
```

### 示例 4：后台运行依赖安全审计

```
用户: "检查一下项目的依赖安全性"

→ Claude: "安全审计已在后台启动。"

后台 Agent 执行:
  Turn 1: Bash("npm audit --json")                        ← npm 安全审计
  Turn 2: Bash("npx better-npm-audit audit")              ← 增强审计
  Turn 3: Bash("npx snyk test 2>/dev/null || echo 'snyk not configured'")
  Turn 4: 汇总安全报告

主对话继续:
  用户: "帮我实现一个新接口 POST /api/feedback"
  → Claude 正常帮用户写代码

  → 通知: "安全审计完成: 2 high severity, 5 moderate. 详情: lodash@4.17.19 有原型污染风险，建议升级到 4.17.21"

background 效果:
  → 安全审计涉及网络请求和数据库查询，耗时较长
  → 后台运行不影响正常开发节奏
  → Agent 能运行多种审计工具并汇总结果
  → 发现高危漏洞时通过通知提醒用户
```

### 示例 5：后台生成 API 文档

```
用户: "重新生成 API 文档，同时帮我修复 #567"

→ Claude: "文档生成已在后台启动。"

后台 Agent 执行:
  Turn 1: Bash("npx typedoc --out docs/api src/index.ts")  ← 生成 TypeDoc
  Turn 2: Bash("npx swagger-jsdoc -d src/routes/ -o docs/swagger.json")  ← OpenAPI
  Turn 3: Read("docs/api/index.html") → 检查生成结果
  Turn 4: 输出生成报告

主对话同时进行:
  用户: "修复 #567：分页返回结果不正确"
  → Claude 在前台定位 bug 并修复

  → 通知: "API 文档生成完成: 32 endpoints documented, 15 type definitions,
           输出目录: docs/api/ 和 docs/swagger.json"

background 效果:
  → 文档生成可能需要扫描整个代码库
  → 后台运行允许用户同步进行其他工作
  → Agent 验证生成结果的完整性
  → 生成完成后用户可以选择在浏览器中查看
```
