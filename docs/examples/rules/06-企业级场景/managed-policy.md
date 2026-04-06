# 企业级场景 — 托管策略（Managed Policy）

> 由 IT 安全团队管理，放在系统级目录，所有开发者强制加载，不可被覆盖
> 最高优先级：全局 < 项目 < 子目录 < 托管策略

## 目录结构

```
/etc/claude-code/                               ← ★ 系统级托管目录（Linux/macOS）
│                                               ← Windows: %PROGRAMDATA%\claude-code\
├── rules/
│   ├── security-policy.md                      ← 强制安全策略
│   ├── enterprise-coding.md                    ← 强制编码标准
│   └── compliance.md                           ← 合规要求
│
└── settings.json                               ← 强制权限配置
    （deny: rm -rf, curl|*, wget|*, sudo*, nc|*）

~/.claude/                                      ← 用户全局（优先级低于托管）
├── rules/
│   └── personal-preferences.md                 ← 个人偏好
└── settings.json

my-project/                                     ← 项目级（优先级低于托管和全局）
├── .claude/
│   ├── rules/
│   │   └── project-specific.md                 ← 项目规则
│   └── settings.json
└── src/
```

## 各文件内容

### /etc/claude-code/rules/security-policy.md

```markdown
# 企业安全策略（托管，不可修改）

## 绝对禁止
- 禁止将代码推送到非企业 Git 仓库（GitHub Enterprise Only）
- 禁止在代码中包含任何凭证信息
- 禁止使用 eval()、new Function()、vm.runInContext
- 禁止使用 md5/sha1 作为密码哈希（使用 bcrypt/argon2）
- 禁止使用 http（必须 https）
- 禁止在生产代码中留 TODO/FIXME/HACK

## 代码审查
- 所有代码必须通过 SonarQube 扫描
- Critical/Blocker 漏洞必须修复后合并
- 依赖通过 Snyk/License 审查
- 禁止引入 GPL 许可证依赖

## 合规
- 金融数据 → SOX
- 用户数据 → GDPR
- 支付数据 → PCI-DSS
- 所有变更必须有审计追踪
```

### /etc/claude-code/settings.json

```json
{
  "permissions": {
    "deny": [
      "Bash(curl*|*)",
      "Bash(wget*|*)",
      "Bash(nc*|*)",
      "Bash(rm -rf*)",
      "Bash(*sudo*)",
      "Bash(*chmod 777*)",
      "Bash(*--no-verify*)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "audit-log-changes",
            "async": true
          }
        ]
      }
    ]
  }
}
```

## 部署目录结构（全平台）

```
Linux/macOS:
  /etc/claude-code/
  ├── rules/                  ← 托管规则目录
  │   ├── security-policy.md
  │   ├── enterprise-coding.md
  │   └── compliance.md
  └── settings.json           ← 托管配置

Windows:
  C:\ProgramData\claude-code\
  ├── rules\
  │   ├── security-policy.md
  │   ├── enterprise-coding.md
  │   └── compliance.md
  └── settings.json
```

## 优先级叠加（所有开发者统一）

```
任何项目中，Claude 看到的规则（从低到高）:

优先级 0 — 托管策略（/etc/claude-code/rules/）     ← ★ 最高，不可覆盖
  security-policy.md     → "禁止 eval, md5, http, 硬编码凭证"
  enterprise-coding.md   → "ESLint @company/config, Prettier, Conventional Commits"
  compliance.md          → "SOX, GDPR, PCI-DSS 审计"

优先级 1 — 用户全局（~/.claude/rules/）            ← 个人偏好
  personal-preferences.md → "代码风格偏好"
  （不能与托管策略冲突，冲突时托管策略优先）

优先级 2 — 项目级（.claude/rules/）                ← 项目特定
  project-specific.md     → "本项目技术栈规范"
  （不能与托管策略冲突）

优先级 3 — settings.json deny 规则                  ← 托管配置的禁止命令
  Bash(rm -rf*)           → 永远被禁止
  Bash(curl*|*)           → 永远被禁止（防止数据外泄）
  Bash(*sudo*)            → 永远被禁止
```

## 触发示例

```
场景 1: 开发者让 Claude 使用 curl 请求外部 API

Claude 尝试: Bash("curl https://api.example.com/data")
  → 托管 settings.json 的 deny 规则匹配 "Bash(curl*|*)"
  → 权限被拒绝，不执行
  → Claude 只能用项目内的 HTTP 库（fetch/axios）编写代码

场景 2: 开发者让 Claude 用 eval 解析 JSON

Claude 想写: const data = eval('(' + input + ')')
  → 托管 security-policy.md 明确禁止 eval()
  → Claude 改用: const data = JSON.parse(input)

场景 3: 开发者让 Claude 写支付代码

操作: Write("src/payment/checkout.ts")
  → 托管 security-policy.md     ← 强制（禁止硬编码凭证）
  → 托管 compliance.md          ← 强制（PCI-DSS 合规）
  → 项目规则                    ← 补充（tokenization 细节）
  → 所有层级叠加，托管优先级最高

场景 4: 开发者在项目 rules 中尝试覆盖托管策略

项目规则写: "允许在生产代码中使用 TODO 注释"
  → 但托管策略写: "禁止在生产代码中留 TODO/FIXME/HACK"
  → 托管策略优先级更高 → 项目规则被忽略
  → Claude 仍然不会写 TODO 注释
```
