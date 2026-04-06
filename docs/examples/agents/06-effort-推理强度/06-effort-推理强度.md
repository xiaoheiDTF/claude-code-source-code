# Agent 字段: effort（推理强度）

> 指南 3.2「模型控制」— 控制 Agent 的推理深度

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── fast-linter.md           ← effort: low（简单任务）
│       ├── code-reviewer.md         ← effort: medium（平衡模式）
│       └── security-auditor.md      ← ★ effort: high（复杂任务）
└── src/
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/security-auditor.md

---
description: Security vulnerability scanner for OWASP Top 10 and common attack vectors.
model: sonnet
effort: high                        ← 深度思考，不放过任何细节
---

Security audit checklist:

1. **Injection**: SQL, NoSQL, command, LDAP injection
2. **Auth**: broken auth, session fixation, weak passwords
3. **XSS**: stored, reflected, DOM-based
4. **CSRF**: missing tokens, weak validation
5. **SSRF**: internal network access via user input
6. **Data exposure**: PII in logs, secrets in code, verbose errors

For each vulnerability found:
- OWASP category
- Severity: Critical / High / Medium / Low
- Attack vector
- Proof of concept (safe)
- Remediation
```

## 说明

```
effort 选项:
  - low    → 快速响应，适合简单重复任务（格式检查、导入排序）
  - medium → 平衡模式（默认值）
  - high   → 深度思考，适合复杂分析（安全审计、架构设计）

effort 控制模型的思考深度:
  low:    模型快速给出答案，思考时间短
  medium: 正常思考
  high:   模型花更多时间推理，考虑更多边界情况

建议组合:
  effort: low  + model: haiku   → 极速简单任务
  effort: high + model: opus    → 深度复杂分析
```

---

## 使用示例 1: effort: high — OWASP 安全审计

```
用户: "安全审计 src/api/ 目录"

→ Agent(security-auditor, effort: high)

Agent (高推理强度):
  - 逐文件仔细检查每个输入点
  - 分析 SQL 查询是否参数化
  - 检查每个用户输入是否验证
  - 审查错误处理是否泄露信息

  输出:
    ## Security Audit Report

    ### Critical — SQL Injection
    - **File**: src/api/users.ts:45
    - **OWASP**: A03 Injection
    - **Vector**: `db.query("SELECT * FROM users WHERE id = " + req.params.id)`
    - **Fix**: Use parameterized query

    ### High — Missing CSRF Token
    - **File**: src/api/posts.ts:22
    - **OWASP**: A01 Broken Access Control
    ...

field 效果:
  effort: high 让模型对每个输入点进行深入推理，不只看表面
  SQL 拼接，还会分析二阶注入、存储型 XSS 等隐蔽攻击向量。
  思考时间更长，但漏报率显著降低。
```

---

## 使用示例 2: effort: low — 快速 import 排序

```
用户: "排序 src/ 下所有文件的 import 语句"

→ Agent(fast-linter, effort: low)

Agent (低推理强度):
  Turn 1: Glob("src/**/*.ts")           ← 找到所有文件
  Turn 2: 逐文件读取 import 部分
  Turn 3: 按 eslint import-sort 规则重排
  Turn 4: 报告结果

  输出:
    ✅ 已排序 32 个文件的 import 语句
    排序规则:
    1. react / 框架内置
    2. 第三方库
    3. 本地模块（@/ 别名）
    4. 相对路径导入

field 效果:
  effort: low 让模型快速执行，不做多余分析。
  import 排序是确定性任务，规则固定，不需要深度推理。
  低 effort 意味着模型直接匹配规则并输出，不思考"是否
  有更好的排序方式"等无关问题。
```

---

## 使用示例 3: effort: medium — 日常代码审查

```
用户: "审查新写的 src/hooks/useAuth.ts"

→ Agent(code-reviewer, effort: medium)

Agent (中等推理强度):
  Turn 1: Read("src/hooks/useAuth.ts")
  Turn 2: 分析 Hook 的正确性、性能、边界情况
  Turn 3: 输出审查结果

  输出:
    ## Code Review: useAuth.ts

    ✅ JWT 解码逻辑正确
    ✅ token 过期自动刷新流程合理
    ⚠️ L23: useEffect 缺少 cleanup，可能导致内存泄漏
      建议: 在 return 中添加 abortController.abort()
    ⚠️ L45: concurrent 场景下 refreshToken 可能有竞态条件
      建议: 添加 mutex 或使用 useRef 标记请求状态

    总评: 良好，修复以上 2 个问题后可合并

field 效果:
  effort: medium 是默认值，模型正常思考，不会过度深入
  也不会遗漏明显问题。能发现 useEffect cleanup 缺失、
  竞态条件等常见问题，但不会深入分析 token 加密的
  数学安全性等极端边界。
```

---

## 使用示例 4: effort: high vs low — 同一任务对比

```
用户: "检查 src/utils/validator.ts 有没有问题"

--- effort: low 的输出 ---
  读取文件，快速扫描:
  ✅ 未发现语法错误
  ⚠️ L15: 正则表达式未使用 raw string
  耗时: 2s

--- effort: high 的输出 ---
  深度分析每个验证函数:
  ✅ 语法正确
  ⚠️ L15: email 正则未覆盖 RFC 5322 全部格式
    例: "user+tag@domain.com" 会被错误拒绝
  ⚠️ L28: phone 验证仅支持中国大陆号码，国际号码全失败
  🔴 L42: sanitizeHTML 使用正则过滤 <script> 标签
    攻击者可用 <img src=x onerror=alert(1)> 绕过
    **必须使用 DOMPurify 等成熟库替代正则过滤**
  💡 L55: validatePassword 未检查常见弱密码字典
    建议集成 zxcvbn 或至少检查 top-1000 密码
  耗时: 9s

field 效果:
  low  → 表面扫描，只发现格式类问题，适合"有没有明显错误"
  high → 深入推理每个验证函数的安全边界，能发现正则绕过、
        XSS 注入等安全漏洞。对验证器/过滤器这类安全相关
        代码，high effort 是必要的选择。
```

---

## 使用示例 5: effort: high — 复杂竞态条件分析

```
用户: "分析 src/services/order.ts 的并发安全性"

→ Agent(concurrency-analyzer, effort: high)

Agent (高推理强度):
  Turn 1: Read("src/services/order.ts")
  Turn 2: 梳理所有共享状态和异步操作
  Turn 3: 逐个分析并发场景

  输出:
    ## 并发安全分析

    ### 🔴 Critical: 库存超卖
    - **位置**: L34-L38 createOrder()
    - **场景**: 两个请求同时读到 stock=1，都通过检查
    - **时序**:
      T1: 请求A read stock → 1      T2: 请求B read stock → 1
      T3: 请求A stock-- → 0          T4: 请求B stock-- → -1 ← 超卖!
    - **修复**: 使用数据库乐观锁
      `UPDATE products SET stock = stock - 1
       WHERE id = ? AND stock > 0`
      检查 affected rows === 0 则回滚

    ### ⚠️ Medium: 重复下单
    - **位置**: L22 createOrder()
    - **场景**: 用户双击提交按钮
    - **修复**: 前端防抖 + 后端幂等性（orderId hash 去重）

field 效果:
  effort: high 让模型不仅读代码，还在脑中模拟多个并发执行
  时序，逐步推演 T1/T2/T3/T4 的状态变化。这种"时间线推理"
  在 low effort 下几乎不会发生，是 high effort 的核心价值。
```
