# 环境变量: CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD

> 指南 九「环境变量」— 允许 --add-dir 目录加载 CLAUDE.md

## 说明

```
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD:
  作用: 允许通过 --add-dir 添加的目录加载 CLAUDE.md
  格式: 布尔值

  行为:
    默认 --add-dir 添加的外部目录不加载其 CLAUDE.md
    设置为 true 后，额外目录中的 CLAUDE.md 也会被加载

  适用场景:
    → 跨项目工作时加载外部项目的规范
    → 共享组件库附带项目规范
```

## 使用示例

### 示例 1：启用 CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD

```bash
export CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=true
claude
```

### 示例 2：禁用（默认行为）

```bash
# 不设置，使用默认值
claude
```

### 示例 3：临时设置单次会话

```bash
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=true claude
```

### 示例 4：在 .env 或 shell 配置中永久设置

```bash
# ~/.bashrc 或 ~/.zshrc
export CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=true
```

### 示例 5：CI 环境设置

```yaml
env:
  CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD: "true"
```
