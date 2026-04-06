# settings.json: agentModel 与 theme

> 指南 7.2「主要配置项」— agentModel 模型选择 与 theme 主题设置

## settings.json 配置

```json
{
  "agentModel": "inherit",
  "theme": "dark"
}
```

## 说明

```
agentModel 字段:
  可选值:
    "inherit"    → 继承父级模型（默认）
    "sonnet"     → 使用 Sonnet 模型
    "opus"       → 使用 Opus 模型
    "haiku"      → 使用 Haiku 模型

  作用: 设置默认使用的 AI 模型
  优先级: 命令行参数 > settings.json > 默认

theme 字段:
  可选值:
    "dark"       → 深色主题
    "light"      → 浅色主题
    "system"     → 跟随系统设置

  作用: 设置 Claude Code 的显示主题
  仅影响终端显示颜色，不影响 AI 行为
```

## 使用示例

### 示例 1：inherit 继承父级模型

```json
{
  "agentModel": "inherit"
}
```

```
→ 默认行为:
  → 使用 Claude Code 当前激活的模型
  → 用户可以通过 /model 命令临时切换
  → 子 Agent 默认继承主 Agent 的模型

→ 主 Agent 使用 opus:
  → 子 Agent 也使用 opus

→ 主 Agent 切换到 sonnet:
  → 子 Agent 也使用 sonnet

效果:
  → 最灵活的选择
  → 用户控制模型切换
  → 子 Agent 跟随主 Agent
```

### 示例 2：强制使用 Sonnet（平衡成本和能力）

```json
{
  "agentModel": "sonnet"
}
```

```
→ 所有操作使用 Sonnet 模型:
  代码编辑     → Sonnet ✅
  代码审查     → Sonnet ✅
  Bug 修复     → Sonnet ✅
  文档生成     → Sonnet ✅
  复杂架构设计 → Sonnet ⚠️（可能不够强）

→ 成本对比:
  Opus:   $15/百万输入 token, $75/百万输出 token
  Sonnet: $3/百万输入 token,  $15/百万输出 token
  Haiku:  $0.25/百万输入 token, $1.25/百万输出 token

  Sonnet 是 Opus 成本的 1/5
  能力约为 Opus 的 80-90%

效果:
  → 日常开发首选 Sonnet
  → 成本显著低于 Opus
  → 能力足够大多数场景
```

### 示例 3：Haiku 模型用于简单任务

```json
{
  "agentModel": "haiku"
}
```

```
→ 使用 Haiku 模型（最快最便宜）:
  文件搜索     → Haiku ✅（足够）
  简单格式化   → Haiku ✅（足够）
  代码注释     → Haiku ✅（足够）
  复杂重构     → Haiku ❌（不够）
  安全审计     → Haiku ❌（不够）

→ 速度对比:
  Haiku:  响应最快
  Sonnet: 中等速度
  Opus:   响应最慢但最准确

效果:
  → 适合大量简单任务
  → 最低成本
  → 不适合需要深度推理的任务
```

### 示例 4：深色/浅色主题切换

```json
{
  "theme": "dark"
}
```

```
→ dark 主题:
  背景色深，文字浅色
  适合夜间开发
  减少眼睛疲劳
  Claude Code 默认主题

→ light 主题:
  背景色浅，文字深色
  适合白天/明亮环境
  截图更清晰（打印友好）

→ system 主题:
  跟随操作系统设置
  macOS/Windows 自动切换
  白天浅色，夜间深色

效果:
  → 纯视觉设置，不影响 AI 行为
  → 个人偏好
  → 可随时通过 /config 切换
```

### 示例 5：不同环境使用不同模型

```json
# ~/.claude/settings.json（全局 — 默认 Sonnet）
{
  "agentModel": "sonnet",
  "theme": "dark"
}

# 工作项目 .claude/settings.json（用 Opus 更严谨）
{
  "agentModel": "opus"
}

# 个人项目 .claude/settings.json（用 Haiku 省钱）
{
  "agentModel": "haiku"
}

# 临时覆盖（命令行）
claude --model opus
→ 这次会话使用 Opus，不影响 settings.json
```

```
→ 优先级:
  命令行 --model  >  项目 settings.json  >  全局 settings.json

→ 用户 A 的配置:
  全局: sonnet
  工作项目: opus（覆盖全局）
  个人项目: haiku（覆盖全局）

→ 用户 B 的配置:
  全局: opus
  所有项目: inherit（使用全局 opus）

效果:
  → 按项目安全级别选择模型
  → 工作项目用更强的模型
  → 个人实验项目用便宜的模型
  → 命令行临时覆盖灵活切换
```
