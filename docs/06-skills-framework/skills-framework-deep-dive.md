# Claude Code Skills 框架深度分析

> 基于源码 `src/skills/loadSkillsDir.ts`、`src/skills/bundledSkills.ts`、`src/skills/mcpSkillBuilders.ts`、`src/skills/bundled/index.ts`、`src/tools/SkillTool/SkillTool.ts` 的逐层解读

---

## 全局架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Skill 生命周期                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │ 定义阶段  │ →  │ 发现阶段  │ →  │ 加载阶段  │ →  │ 执行阶段  │  │
│  │          │    │          │    │          │    │          │  │
│  │ SKILL.md │    │ 静态扫描  │    │ 解析FM   │    │ 内联执行  │  │
│  │ frontm.  │    │ 动态发现  │    │ 注册命令  │    │ 分叉执行  │  │
│  │ bundled  │    │ MCP远程  │    │ 去重排序  │    │ 远程执行  │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 一、Skill 定义格式

### Frontmatter 字段全表

一个 Skill 由 `.claude/skills/skill-name/SKILL.md` 文件定义，使用 YAML frontmatter：

```markdown
---
name: my-skill                    # 技能名称（缺省取目录名）
description: 做某件事              # 技能描述
user-invocable: true              # 是否用户可调用 /my-skill（默认 true）
disable-model-invocation: true    # 禁止模型自动调用
allowed-tools: Bash,Read,Write    # 白名单工具（逗号分隔）
model: claude-sonnet-4-6          # 强制使用的模型
effort: high                      # 努力等级：low/medium/high
context: 一些上下文信息             # 附加上下文注入
agent: general-purpose            # 分叉执行时的 agent 类型
paths:                            # 条件激活的路径模式
  - "src/**/*.ts"
  - "*.test.js"
hooks:                            # 技能级 hook 配置
  pre: echo "before"
  post: echo "after"
shell: bash                       # shell 类型
arguments:                        # 参数定义
  - name: target
    description: 目标文件
    required: true
when_to_use: 在需要...时使用       # 模型自调用时的决策依据
---

技能的提示词正文放在这里...
可以包含 $ARGUMENTS 占位符
也可以包含 !`shell command` 动态内容
```

### 字段分类

| 类别 | 字段 | 说明 |
|------|------|------|
| **身份** | `name`, `description` | 技能标识 |
| **可见性** | `user-invocable`, `disable-model-invocation` | 控制谁可以触发 |
| **执行控制** | `model`, `effort`, `agent`, `context` | 运行时参数覆盖 |
| **权限** | `allowed-tools` | 工具白名单 |
| **条件激活** | `paths` | 文件路径模式匹配 |
| **生命周期** | `hooks`, `shell` | Hook 脚本和 Shell 类型 |
| **参数** | `arguments` | 参数定义 |
| **智能路由** | `when_to_use` | 模型决策依据 |

---

## 二、五大加载源与优先级

### 源码位置
`src/skills/loadSkillsDir.ts` — `getSkillDirCommands()`

### 加载流程

```typescript
async function getSkillDirCommands(cwd: string): Promise<{
  commands: Map<string, Command>   // 立即可用的技能
  conditional: Map<string, Command> // 条件技能（需路径匹配激活）
}> {
  // 并行从 5 个源加载
  const [managed, user, project, additional, legacy] = await Promise.all([
    loadManagedSkills(),           // 1. 受管技能（企业部署）
    loadUserSkills(),              // 2. 用户技能（~/.claude/skills/）
    loadProjectSkills(cwd),        // 3. 项目技能（.claude/skills/）
    loadAdditionalSkills(),        // 4. 额外技能（配置指定）
    loadLegacyCommands(cwd),       // 5. 遗留命令（.claude/commands/）
  ])

  // 去重：按 realpath 解析，后加载的覆盖先加载的
  // 分离：有 paths 字段的进入 conditional 组
  return mergeAndSeparate(allSkills)
}
```

### 五个加载源详解

| 优先级 | 源 | 路径 | 说明 |
|--------|-----|------|------|
| 1 (最高) | **Managed** | 系统管理路径 | 企业 IT 管理员部署的技能 |
| 2 | **User** | `~/.claude/skills/` | 用户全局自定义技能 |
| 3 | **Project** | `<project>/.claude/skills/` | 项目级共享技能（可提交到 git） |
| 4 | **Additional** | 配置文件指定 | `settings.json` 中 `additionalSkillsDir` |
| 5 (最低) | **Legacy** | `<project>/.claude/commands/` | 旧版 `.md` 文件格式兼容 |

### 去重机制

```typescript
// 使用 realpath 解析解决符号链接
const resolvedPath = await fs.realpath(nestedDir)

// 同名技能：后加载的覆盖先加载的
// 所以 project 级别可以覆盖 user 级别
if (!seenPaths.has(resolvedPath)) {
  seenPaths.add(resolvedPath)
  skills.set(skillName, skill)
}
```

---

## 三、技能发现机制

### 3.1 静态加载（启动时）

```
启动时 getSkillDirCommands(cwd)
    │
    ├─ 扫描 ~/.claude/skills/*/SKILL.md
    ├─ 扫描 .claude/skills/*/SKILL.md
    ├─ 扫描 managed 路径
    ├─ 扫描 additional 路径
    └─ 扫描 .claude/commands/*.md（遗留）
        │
        ├─ 无 paths 字段 → 直接注册为可用技能
        └─ 有 paths 字段 → 放入 conditional 组，等待激活
```

### 3.2 动态发现（运行时）

```typescript
// src/skills/loadSkillsDir.ts — discoverSkillDirsForPaths()

// 当工具操作涉及文件时，从文件路径向上遍历到 cwd
// 在每一层检查 .claude/skills/ 目录
async function discoverSkillDirsForPaths(
  filePaths: string[],
  cwd: string,
): Promise<string[]> {
  const skillDirs = new Set<string>()

  for (const filePath of filePaths) {
    let dir = path.dirname(filePath)

    while (dir !== cwd && dir !== path.dirname(dir)) {
      const skillsDir = path.join(dir, '.claude', 'skills')
      if (await exists(skillsDir)) {
        skillDirs.add(skillsDir)
      }
      dir = path.dirname(dir)
    }
  }

  return [...skillDirs]
}
```

**设计意义**：当 Claude 读取 `/home/user/project/submodule/src/file.ts` 时，会自动发现 `/home/user/project/submodule/.claude/skills/` 中的技能，即使当前 cwd 在 `/home/user/project/`。

### 3.3 条件技能激活

```typescript
// src/skills/loadSkillsDir.ts — activateConditionalSkillsForPaths()

// 使用 gitignore 风格的路径匹配
// paths: ["src/**/*.ts", "*.test.js"]
// 当操作文件匹配任一模式时，技能被激活
function activateConditionalSkillsForPaths(
  conditionalSkills: Map<string, Command>,
  filePaths: string[],
): Map<string, Command> {
  const activated = new Map<string, Command>()

  for (const [name, skill] of conditionalSkills) {
    const patterns = skill.frontmatter.paths
    if (matchesAnyPattern(filePaths, patterns)) {
      activated.set(name, skill)
    }
  }

  return activated
}
```

### 3.4 技能合并与优先级

```typescript
// src/skills/loadSkillsDir.ts — addSkillDirectories()

// 深层路径优先于浅层路径
// /home/user/project/submodule/.claude/skills/review
// 覆盖
// /home/user/project/.claude/skills/review
function addSkillDirectories(
  existing: Map<string, Command>,
  discovered: Map<string, Command>,
  discoveredDirs: string[],
): Map<string, Command> {
  // 按路径深度降序排列（深者优先）
  discoveredDirs.sort((a, b) => b.split('/').length - a.split('/').length)

  for (const dir of discoveredDirs) {
    for (const [name, skill] of discovered) {
      if (skill.dir === dir) {
        existing.set(name, skill)  // 覆盖浅层同名技能
      }
    }
  }
  return existing
}
```

---

## 四、Frontmatter 解析

### 源码位置
`src/skills/loadSkillsDir.ts` — `parseSkillFrontmatterFields()`

```typescript
interface ParsedFrontmatterFields {
  name?: string
  description?: string
  allowedTools?: string[]          // "Bash,Read" → ["Bash", "Read"]
  model?: string
  effort?: 'low' | 'medium' | 'high'
  context?: string
  agent?: string
  paths?: string[]                 // 条件激活路径
  hooks?: {
    pre?: string
    post?: string
  }
  shell?: string
  arguments?: Array<{
    name: string
    description: string
    required?: boolean
  }>
  userInvocable?: boolean          // 默认 true
  disableModelInvocation?: boolean // 默认 false
  whenToUse?: string
}
```

### 解析流程

```
SKILL.md 文件内容
    │
    ▼
提取 --- ... --- 之间的 YAML
    │
    ▼
YAML.safeLoad() 解析为对象
    │
    ▼
字段映射（kebab-case → camelCase）
    │  allowed-tools → allowedTools
    │  user-invocable → userInvocable
    │  when_to_use → whenToUse
    │
    ▼
allowedTools 字符串按逗号分隔为数组
    │
    ▼
返回 ParsedFrontmatterFields
```

---

## 五、命令注册与提示词构造

### 源码位置
`src/skills/loadSkillsDir.ts` — `createSkillCommand()`

```typescript
function createSkillCommand(
  name: string,
  frontmatter: ParsedFrontmatterFields,
  content: string,       // frontmatter 之后的正文
  skillDir: string,      // 技能目录绝对路径
): Command {

  return {
    name,
    description: frontmatter.description ?? '',
    // ...

    getPromptForCommand(args: string): string {
      let prompt = content

      // 1. $ARGUMENTS 替换
      prompt = prompt.replace(/\$ARGUMENTS/g, args)

      // 2. ${CLAUDE_SKILL_DIR} 替换
      prompt = prompt.replace(
        /\$\{CLAUDE_SKILL_DIR\}/g,
        skillDir
      )

      // 3. ${CLAUDE_SESSION_ID} 替换
      prompt = prompt.replace(
        /\$\{CLAUDE_SESSION_ID\}/g,
        currentSessionId
      )

      // 4. !`shell command` 动态注入
      //    匹配 !`...` 模式，执行 shell 命令并替换为输出
      prompt = await processShellInjections(prompt)

      return prompt
    }
  }
}
```

### Shell 命令注入

```typescript
// 匹配 !`command` 模式
const SHELL_INJECTION_REGEX = /!`([^`]+)`/g

async function processShellInjections(prompt: string): Promise<string> {
  return prompt.replace(SHELL_INJECTION_REGEX, async (match, command) => {
    // MCP 技能不允许 shell 注入（安全限制）
    if (isMcpSkill) {
      return match  // 保持原样，不执行
    }

    const result = await execCommand(command)
    return result.stdout
  })
}
```

### Legacy 命令格式兼容

```
.claude/commands/my-command.md       → /my-command
.claude/commands/sub/my-command.md   → /sub:my-command

目录分隔符 → 冒号，支持子目录组织
```

---

## 六、执行引擎

### 源码位置
`src/tools/SkillTool/SkillTool.ts` — `call()`

### 执行架构

```
用户输入 /skill-name 或模型调用 SkillTool
        │
        ▼
┌──────────────────────────────────┐
│  validateInput()                 │
│  ├─ 检查技能是否存在              │
│  ├─ 检查是否是 prompt-based       │
│  └─ 检查是否被禁用                │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  checkPermissions()              │
│  ├─ 精确匹配 deny/allow 规则      │
│  ├─ 前缀匹配 skill-name:*        │
│  ├─ 安全属性检查（自动允许）       │
│  └─ 默认 → ask                   │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  判断执行路径                     │
│  ├─ agent 字段存在 → 分叉执行     │
│  ├─ 远程技能     → 远程执行       │
│  └─ 否则         → 内联执行       │
└──────┬───────────┬───────────────┘
       │           │
  ┌────▼────┐ ┌────▼────┐
  │ 内联执行 │ │ 分叉执行 │
  │         │ │         │
  │ 注入提示 │ │ 子Agent │
  │ 词到当前 │ │ 隔离运行 │
  │ 对话上下 │ │ 独立token│
  │ 文      │ │ 预算    │
  └─────────┘ └─────────┘
```

### 6.1 内联执行

```typescript
// SkillTool.ts — call() 内联路径

async call(input, context, ...) {
  const command = findCommand(input.skill_name)

  // 获取技能提示词（含模板替换）
  const prompt = command.getPromptForCommand(input.arguments ?? '')

  // 追加到当前对话的消息列表
  return {
    data: { type: 'skill_result', content: prompt },
    newMessages: [
      createUserMessage({ content: prompt })
    ],
    // contextModifier 修改后续上下文
    contextModifier: (ctx) => {
      // 1. 注入 allowedTools 到 alwaysAllowRules
      if (frontmatter.allowedTools) {
        ctx.alwaysAllowRules = [
          ...ctx.alwaysAllowRules,
          ...frontmatter.allowedTools.map(tool => ({
            tool: tool.trim(),
            rule: '*',
          }))
        ]
      }

      // 2. 覆盖模型
      if (frontmatter.model) {
        ctx.model = frontmatter.model
      }

      // 3. 覆盖努力等级
      if (frontmatter.effort) {
        ctx.thinkingBudget = effortToBudget(frontmatter.effort)
      }

      return ctx
    }
  }
}
```

**关键设计**：内联执行不创建新的对话轮次，而是将技能提示词注入当前上下文，通过 `contextModifier` 修改后续行为（工具白名单、模型、努力等级）。

### 6.2 分叉执行（Forked）

```typescript
// SkillTool.ts — executeForkedSkill()

async executeForkedSkill(
  skillName: string,
  prompt: string,
  agentType: string,
  context: ToolUseContext,
): Promise<ToolResult> {
  // 创建隔离的子 agent
  const subAgent = createAgent({
    type: agentType,           // 如 "general-purpose"
    model: frontmatter.model,  // 可自定义模型
    tools: getAllowedTools(frontmatter.allowedTools),
    tokenBudget: calculateBudget(frontmatter.effort),
    systemPrompt: buildSkillSystemPrompt(prompt),
  })

  // 运行子 agent
  const result = await runAgent(subAgent, {
    messages: [createUserMessage({ content: prompt })],
    abortSignal: context.abortController.signal,
  })

  return {
    data: result.output,
    newMessages: result.messages,
  }
}
```

**与内联的区别**：

| 维度 | 内联执行 | 分叉执行 |
|------|----------|----------|
| 对话上下文 | 共享主对话 | 隔离子对话 |
| Token 预算 | 受主对话限制 | 独立预算 |
| 工具访问 | 继承主对话工具 | 白名单控制 |
| 结果集成 | 直接注入消息流 | 结果作为工具输出返回 |
| 适用场景 | 简单提示词注入 | 复杂多步骤任务 |

### 6.3 远程技能执行

```typescript
// SkillTool.ts — executeRemoteSkill()

async executeRemoteSkill(
  skillUri: string,    // AKI/GCS URI
  context: ToolUseContext,
): Promise<ToolResult> {
  // 1. 检查本地缓存
  const cached = await checkLocalCache(skillUri)
  if (cached && !isExpired(cached)) {
    return executeFromCache(cached)
  }

  // 2. 从远程存储下载 SKILL.md
  const skillContent = await fetchRemoteSkill(skillUri)

  // 3. 缓存到本地
  await cacheSkillLocally(skillUri, skillContent)

  // 4. 解析并执行
  const { frontmatter, content } = parseSkillFile(skillContent)
  return executeInline(frontmatter, content, context)
}
```

---

## 七、权限模型

### 源码位置
`src/tools/SkillTool/SkillTool.ts` — `checkPermissions()`

### 三层权限检查

```
checkPermissions(input)
    │
    ├─ 第 1 层：精确匹配规则
    │   settings.json 中:
    │   { "permissions": { "allow": ["skill-name"], "deny": ["skill-name"] } }
    │
    ├─ 第 2 层：前缀匹配
    │   "skill-name:*" → 匹配技能的所有参数变体
    │
    ├─ 第 3 层：安全属性自动允许
    │   skillHasOnlySafeProperties(skill)
    │   ├─ 无 allowed-tools
    │   ├─ 无 model 覆盖
    │   ├─ 无 agent 类型
    │   ├─ 无 hooks
    │   ├─ 无 shell 命令注入
    │   └─ 无 context 修改
    │   → 自动 allow
    │
    └─ 默认：ask（弹出权限对话框）
```

### 安全属性白名单

```typescript
function skillHasOnlySafeProperties(skill: ParsedSkill): boolean {
  // 以下任一存在则不安全
  if (skill.frontmatter.allowedTools) return false
  if (skill.frontmatter.model) return false
  if (skill.frontmatter.agent) return false
  if (skill.frontmatter.hooks) return false
  if (skill.content.includes('!`')) return false  // shell 注入
  if (skill.frontmatter.context) return false

  return true  // 纯文本提示词，安全
}
```

### Deny/Allow 规则优先级

```
deny 精确匹配 > allow 精确匹配 > deny 前缀匹配 > allow 前缀匹配 > 安全自动允许 > ask
```

---

## 八、Bundled Skills（内置技能）

### 源码位置
`src/skills/bundled/index.ts` + `src/skills/bundledSkills.ts`

### 注册机制

```typescript
// src/skills/bundled/index.ts
export function initBundledSkills(): void {
  registerBundledSkill({
    name: 'updateConfig',
    skillDir: 'updateConfig',
    files: {
      'SKILL.md': () => import('./updateConfig/SKILL.md'),
    },
  })

  registerBundledSkill({
    name: 'simplify',
    skillDir: 'simplify',
    files: {
      'SKILL.md': () => import('./simplify/SKILL.md'),
      'reviewer.ts': () => import('./simplify/reviewer.ts'),
    },
  })

  // ... 15+ 个内置技能
}
```

### 懒加载文件提取

```typescript
// src/skills/bundledSkills.ts

interface BundledSkillDefinition {
  name: string
  skillDir: string                          // 目标子目录名
  files: Record<string, () => Promise<string>>  // 文件名 → 懒加载器
}

const bundledSkills: BundledSkillDefinition[] = []

export function registerBundledSkill(def: BundledSkillDefinition): void {
  bundledSkills.push(def)
}

// 懒提取到磁盘
export async function ensureBundledSkillExtracted(
  skillName: string,
  targetBaseDir: string,  // 通常是 ~/.claude/skills/
): Promise<string> {
  const skill = bundledSkills.find(s => s.name === skillName)
  if (!skill) throw new Error(`Unknown bundled skill: ${skillName}`)

  const skillDir = path.join(targetBaseDir, skill.skillDir)

  for (const [fileName, loader] of Object.entries(skill.files)) {
    const filePath = path.join(skillDir, fileName)

    // 安全写入：O_NOFOLLOW | O_EXCL
    // - O_NOFOLLOW: 不跟随符号链接（防止路径遍历）
    // - O_EXCL: 排他创建（防止竞态条件）
    const content = await loader()  // 懒加载内容
    await safeWriteFile(filePath, content)
  }

  return skillDir
}
```

### 安全写入防护

```typescript
async function safeWriteFile(filePath: string, content: string): Promise<void> {
  // 1. 路径遍历保护
  const resolved = path.resolve(filePath)
  if (!resolved.startsWith(expectedBaseDir)) {
    throw new Error('Path traversal detected')
  }

  // 2. 使用 O_NOFOLLOW | O_EXCL 标志
  //    确保不会覆盖已有文件，不会跟随符号链接
  const fd = await fs.open(resolved, O_NOFOLLOW | O_EXCL | O_CREAT | O_WRONLY)
  await fs.write(fd, content)
  await fs.close(fd)
}
```

### 已注册的 Bundled Skills

| 技能名 | 说明 |
|--------|------|
| `updateConfig` | 更新 Claude Code 配置 |
| `keybindings` | 键盘快捷键管理 |
| `verify` | 验证/检查 |
| `debug` | 调试辅助 |
| `loremIpsum` | 生成占位文本 |
| `skillify` | 将提示词转为技能 |
| `remember` | 记忆管理 |
| `simplify` | 代码简化审查（启动 3 个并行 Agent） |
| `batch` | 批量执行 |
| `stuck` | 卡住时的恢复策略 |
| `dream` | 创意/头脑风暴 |
| `hunter` | 代码搜索/查找 |
| `loop` | 循环任务（CronCreate 封装） |
| `scheduleRemoteAgents` | 调度远程 Agent |
| `claudeApi` | Claude API 使用指南 |
| `claudeInChrome` | Chrome 集成 |
| `runSkillGenerator` | 技能生成器 |

### simplify 技能示例（多 Agent 并行）

```typescript
// src/skills/bundled/simplify.ts
// 该技能在执行时启动 3 个并行的 AgentTool agent：

// Agent 1: 代码复用审查
// 搜索代码库中的重复模式，建议抽象

// Agent 2: 代码质量审查
// 检查命名、结构、错误处理

// Agent 3: 效率审查
// 检查性能、算法复杂度、不必要的计算

// 每个代理独立运行，结果汇总后给出综合建议
```

### loop 技能示例（CronCreate 封装）

```typescript
// src/skills/bundled/loop.ts
// 解析用户输入为循环任务：
// "/loop 每天早上9点 检查测试" → CronCreate

// 输入格式：[interval] <prompt>
// 支持的自然语言间隔：
// - "every N minutes/hours"
// - "every day at HH:MM"
// - "every weekday at HH:MM"
// - 中文："每天"、"每小时"、"工作日"

// 内部转换为 cron 表达式，调用 CronCreateTool
```

---

## 九、MCP 技能集成

### 源码位置
`src/skills/mcpSkillBuilders.ts`

### 写入一次注册（Write-Once Registry）

```typescript
// mcpSkillBuilders.ts 解决循环依赖：
// loadSkillsDir.ts ←→ MCP skill loading

type SkillBuilder = (skillDef: McpSkillDefinition) => Command

let skillBuilder: SkillBuilder | null = null

export function registerSkillBuilder(builder: SkillBuilder): void {
  if (skillBuilder) {
    throw new Error('Skill builder already registered')  // 写入一次
  }
  skillBuilder = builder
}

export function getSkillBuilder(): SkillBuilder {
  if (!skillBuilder) {
    throw new Error('Skill builder not yet registered')
  }
  return skillBuilder
}
```

**设计目的**：`loadSkillsDir.ts` 需要创建 Command 对象，但 Command 的某些功能依赖 MCP 模块，而 MCP 模块又依赖 loadSkillsDir。写入一次注册打破了这个循环。

### MCP 技能的特殊限制

| 限制 | 原因 |
|------|------|
| 禁止 `!`shell`` 注入 | 远程技能不可信 |
| 禁止 `${CLAUDE_SKILL_DIR}` | 无本地目录 |
| 提示词内容不可包含本地路径 | 安全隔离 |

---

## 十、技能与工具管线集成

### 在 Tool Pipeline 中的位置

```
SkillTool 作为标准 Tool 注册到工具列表
        │
        ▼
┌─────────────────────────────────┐
│  标准 10 步管线                  │
│  findTool → parse → validate    │
│  → backfill → preHooks          │
│  → resolvePermission → canUse   │
│  → call() → postHooks → result  │
└─────────────────────────────────┘
        │
        │  SkillTool.call() 内部
        ▼
┌─────────────────────────────────┐
│  技能特有逻辑                    │
│  ├─ 查找命令                    │
│  ├─ 获取提示词（模板替换）        │
│  ├─ 选择执行路径                │
│  └─ 构造 contextModifier       │
└─────────────────────────────────┘
```

### contextModifier 的传播

```typescript
// 技能执行后，contextModifier 修改后续对话的行为：

contextModifier: (ctx) => {
  // 1. 工具白名单注入
  ctx.alwaysAllowRules = [
    ...ctx.alwaysAllowRules,
    ...frontmatter.allowedTools.map(tool => ({
      tool: tool.trim(),
      rule: '*',  // 允许该工具的所有操作
    }))
  ]

  // 2. 模型覆盖
  if (frontmatter.model) {
    ctx.model = frontmatter.model
  }

  // 3. 努力等级覆盖
  if (frontmatter.effort) {
    ctx.thinkingBudget = mapEffortToBudget(frontmatter.effort)
  }

  return ctx
}

// contextModifier 在 toolExecution.ts 的 Step 10 中应用：
resultingMessages.push({
  message: createUserMessage({ content: contentBlocks }),
  contextModifier: toolContextModifier
    ? { toolUseID, modifyContext: toolContextModifier }
    : undefined,
})
```

---

## 十一、设计优缺点

### 优点

| 优点 | 说明 |
|------|------|
| **声明式定义** | 用 Markdown + YAML frontmatter，非代码，降低门槛 |
| **5 源加载** | 企业/用户/项目/额外/遗留，覆盖所有使用场景 |
| **动态发现** | 从文件操作路径自动向上遍历发现技能 |
| **条件激活** | `paths` 模式匹配，只在相关场景出现 |
| **安全分层** | 安全属性自动允许、deny 规则不可绕过、MCP 隔离 |
| **双执行模式** | 内联（轻量）和分叉（隔离）满足不同复杂度 |
| **Bundled 懒加载** | 内置技能按需提取，不增加启动时间 |
| **模板系统** | `$ARGUMENTS`、`${CLAUDE_SKILL_DIR}`、Shell 注入提供灵活性 |

### 缺点

| 缺点 | 说明 |
|------|------|
| **loadSkillsDir.ts 过大** | 1087 行，混合了解析/加载/发现/激活/合并逻辑 |
| **两种加载格式并存** | `SKILL.md` 目录格式和 `.md` 文件格式增加维护成本 |
| **Shell 注入安全面** | `!`command`` 在非 MCP 技能中允许任意命令执行 |
| **去重依赖 realpath** | 符号链接密集环境可能有意外行为 |
| **frontmatter 解析无 schema 验证** | 错误的 YAML 静默忽略，难以调试 |
| **条件技能匹配性能** | 每次文件操作都需要对所有条件技能做模式匹配 |

---

## 十二、完整数据流图

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Skill 完整生命周期                           │
│                                                                      │
│  定义                                                                 │
│  ┌───────────────────┐  ┌───────────────────┐  ┌──────────────────┐ │
│  │ .claude/skills/   │  │ ~/.claude/skills/ │  │ Bundled Skills   │ │
│  │ my-skill/         │  │ global-skill/     │  │ (registerBundled │ │
│  │ └─ SKILL.md       │  │ └─ SKILL.md       │  │  Skill())        │ │
│  └────────┬──────────┘  └────────┬──────────┘  └────────┬─────────┘ │
│           │                      │                       │           │
│  发现      │                      │                       │           │
│           ▼                      ▼                       ▼           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │         getSkillDirCommands(cwd)                            │    │
│  │    ┌──────────────────────────────────────────────┐         │    │
│  │    │ 5 个源并行加载 → realpath 去重 → 分离条件技能  │         │    │
│  │    └──────────────────────────────────────────────┘         │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                              │                                      │
│  加载                         │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │         parseSkillFrontmatterFields()                       │    │
│  │    ┌──────────────────────────────────────────────┐         │    │
│  │    │ YAML 解析 → 字段映射 → 类型转换 → Command 创建 │         │    │
│  │    └──────────────────────────────────────────────┘         │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                              │                                      │
│  注册                         │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │         SkillTool 注册到工具列表                              │    │
│  │    ┌──────────────────────────────────────────────┐         │    │
│  │    │ commands Map → SkillTool.inputSchema          │         │    │
│  │    │ 条件技能 → 等待 activateConditionalSkills     │         │    │
│  │    └──────────────────────────────────────────────┘         │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                              │                                      │
│  执行                         │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │         SkillTool.call()                                    │    │
│  │    ┌──────────┐  ┌──────────────┐  ┌─────────────────────┐ │    │
│  │    │ 内联执行  │  │ 分叉执行      │  │ 远程执行             │ │    │
│  │    │ prompt→  │  │ agent→run    │  │ fetch→cache→        │ │    │
│  │    │ context  │  │ Agent()      │  │ executeInline()     │ │    │
│  │    │ Modifier │  │              │  │                     │ │    │
│  │    └──────────┘  └──────────────┘  └─────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 十三、与 Tool Pipeline 的对照

| 维度 | Tool Pipeline | Skills Framework |
|------|---------------|------------------|
| 本质 | 工具执行的通用安全管线 | 基于 Tool Pipeline 的上层扩展 |
| 入口 | `runToolUse()` | `SkillTool.call()`（走 Tool Pipeline） |
| 定义方式 | TypeScript 代码 | Markdown + YAML |
| 权限 | 10 层纵深防御 | 复用 Tool Pipeline + 安全属性白名单 |
| 可扩展性 | 需要代码开发 | 用户通过文件系统自定义 |
| 执行隔离 | 工具在主进程执行 | 可选子 Agent 隔离 |
| 输出 | `ToolResult` | `ToolResult` + `contextModifier` |

Skills 框架是 Claude Code 扩展性的核心——通过声明式 Markdown 文件定义能力，利用 Tool Pipeline 的安全机制执行，同时提供从简单提示词注入到完全隔离子 Agent 的灵活执行模式。
