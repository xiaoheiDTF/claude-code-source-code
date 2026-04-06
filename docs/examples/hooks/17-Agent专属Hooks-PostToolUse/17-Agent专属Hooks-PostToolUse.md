# Agent 专属 Hooks: PostToolUse（Agent 内工具调用后）

> 指南 5.3「Agent 可用的 Hook 事件」— Agent 内部的 PostToolUse 钩子

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── ts-dev.md               ← TypeScript 开发 + 自动 lint
│       ├── api-dev.md              ← API 开发 + 自动测试
│       ├── frontend-dev.md         ← 前端开发 + 自动格式化
│       └── infra-dev.md            ← 基础设施 + 自动验证
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/ts-dev.md

---
description: TypeScript developer with auto-lint
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
            if [[ "$FILE" == *.ts || "$FILE" == *.tsx ]]; then
              npx eslint --fix "$FILE" 2>/dev/null
              echo "{\"additionalContext\": \"Linted: $FILE\"}"
            fi
          async: true
---

You write TypeScript code following strict standards.
```

## 说明

```
Agent 内 PostToolUse:
  触发时机: Agent 内工具执行成功后
  matcher: 匹配工具名

  与全局 PostToolUse 的关系:
    1. 全局 PostToolUse 先执行
    2. Agent 内 PostToolUse 后执行
    3. 两者都返回 additionalContext → 合并注入

  典型用途:
    - 编辑后自动 lint/format
    - 写入后自动测试
    - 变更后自动暂存
    - 操作后注入分析结果

  推荐: async: true
    Agent 的 PostToolUse 通常用于辅助操作
    async 避免阻塞 Agent 的主流程
```

## 使用示例

### 示例 1：TypeScript Agent 编辑后自动 lint + 类型检查

```
Agent: .claude/agents/ts-dev.md
  PostToolUse → matcher: "Edit|Write" → eslint + tsc

→ 用户: "用 ts-dev 实现用户服务"

→ Agent 连续编辑:
  Edit: src/services/user.service.ts → 成功
  → PostToolUse Hook 触发（async）:
    1. npx eslint --fix "src/services/user.service.ts" → 修复 2 个问题
    2. npx tsc --noEmit "src/services/user.service.ts" → 类型检查通过
    3. 返回: {"additionalContext": "Linted + 类型检查通过: user.service.ts"}

  Edit: src/controllers/user.controller.ts → 成功
  → PostToolUse Hook 触发（async）:
    1. eslint --fix → 发现缺少返回类型
    2. 自动添加返回类型注解
    3. tsc → 类型错误: 参数类型不匹配
    4. 返回: {"additionalContext": "类型错误: controller 第 23 行参数类型不匹配"}

→ Agent 看到 lint/类型结果，自动修复类型错误

效果:
  → 每次 Edit 自动 lint + 类型检查
  → 类型错误立即被发现并修复
  → 不需要在最后统一检查
```

### 示例 2：API 开发 Agent 写入后自动运行相关测试

```
Agent: .claude/agents/api-dev.md
  PostToolUse → matcher: "Write" → 自动运行对应的测试文件

hooks:
  PostToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
            # 检查是否有对应的测试文件
            TEST_FILE=$(echo "$FILE" | sed 's|src/|tests/|' | sed 's|\.ts$|.test.ts|')
            if [ -f "$TEST_FILE" ]; then
              RESULT=$(npx jest "$TEST_FILE" --no-coverage 2>&1 | tail -5)
              echo "{\"additionalContext\": \"测试结果: $RESULT\"}"
            fi
          async: true

→ Agent 运行:
  Write: src/api/users.ts → 成功
  → PostToolUse Hook:
    1. 查找 tests/api/users.test.ts → 存在
    2. 运行: npx jest tests/api/users.test.ts
    3. 结果: "Tests: 3 passed, 1 failed"
    4. 返回: {"additionalContext": "测试结果: 3 passed, 1 failed"}

→ Agent 看到有 1 个测试失败，立即修复:
  "API 写入后有 1 个测试失败，检查测试用例..."
  → 修复失败测试

效果:
  → 写入新文件后自动运行对应测试
  → 即时反馈，不等全部写完再测
  → 测试驱动开发自动化
```

### 示例 3：前端 Agent 编辑后自动格式化 + 截图对比

```
Agent: .claude/agents/frontend-dev.md
  PostToolUse → matcher: "Edit" → prettier + 组件截图

hooks:
  PostToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
            if [[ "$FILE" == *.tsx || "$FILE" == *.css ]]; then
              # 格式化
              npx prettier --write "$FILE" 2>/dev/null
              # 检查样式文件变更
              if [[ "$FILE" == *.css ]]; then
                echo "{\"additionalContext\": \"CSS 已格式化。建议手动检查视觉效果。\"}"
              else
                echo "{\"additionalContext\": \"组件已格式化: $FILE\"}"
              fi
            fi
          async: true

→ Agent 运行:
  Edit: src/components/Button.tsx → 成功
  → PostToolUse:
    1. prettier --write → 格式化
    2. {"additionalContext": "组件已格式化: src/components/Button.tsx"}

  Edit: src/styles/global.css → 成功
  → PostToolUse:
    1. prettier --write → 格式化
    2. {"additionalContext": "CSS 已格式化。建议手动检查视觉效果。"}

效果:
  → 编辑后自动格式化代码
  → CSS 变更额外提醒视觉检查
  → 保持前端代码风格一致
```

### 示例 4：基础设施 Agent 修改配置后自动验证

```
Agent: .claude/agents/infra-dev.md
  PostToolUse → matcher: "Write|Edit" → 验证配置文件语法

hooks:
  PostToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
            # 根据文件类型验证
            case "$FILE" in
              *.yaml|*.yml)
                yamllint "$FILE" 2>&1 || echo "{\"additionalContext\": \"⚠️ YAML 语法错误: $FILE\"}"
                ;;
              *.json)
                jq empty "$FILE" 2>&1 || echo "{\"additionalContext\": \"⚠️ JSON 语法错误: $FILE\"}"
                ;;
              Dockerfile)
                hadolint "$FILE" 2>&1 || true
                ;;
              *.tf)
                terraform validate 2>&1 || echo "{\"additionalContext\": \"⚠️ Terraform 验证失败\"}"
                ;;
            esac
          async: true

→ Agent 运行:
  Edit: docker-compose.yml → 成功
  → PostToolUse: yamllint → 通过 ✅

  Write: kubernetes/deployment.yaml → 成功
  → PostToolUse: yamllint → 发现缩进错误 ⚠️
  → {"additionalContext": "⚠️ YAML 语法错误: kubernetes/deployment.yaml"}
  → Agent 自动修复缩进

效果:
  → 配置文件写入后立即验证语法
  → 不同文件类型使用不同验证工具
  → 防止无效配置进入系统
```

### 示例 5：Agent 内 PostToolUse 注入代码质量分析

```
Agent: .claude/agents/dev.md
  PostToolUse → matcher: "Edit|Write" → 代码复杂度分析

hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
            if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then exit 0; fi

            # 代码行数
            LINES=$(wc -l < "$FILE")
            # 函数数量
            FUNCS=$(grep -cE 'function |const .* = \(|=> {' "$FILE" 2>/dev/null || echo 0)
            # TODO 数量
            TODOS=$(grep -ciE 'TODO|FIXME|HACK' "$FILE" 2>/dev/null || echo 0)
            # 复杂度（简单估算）
            IFS=$(grep -cE 'if|else|for|while|switch|try' "$FILE" 2>/dev/null || echo 0)

            echo "{\"additionalContext\": \"代码质量快照 → $FILE:
              行数: $LINES | 函数: $FUNCS | 分支: $IFS | TODO: $TODOS\"}"
          async: true

→ Agent 编辑文件后收到:
  "代码质量快照 → src/utils/validator.ts:
   行数: 89 | 函数: 4 | 分支: 12 | TODO: 2"

→ Agent 判断:
  "分支复杂度较高（12 个分支），建议拆分为更小的函数。
   还有 2 个 TODO 需要处理。"

效果:
  → 每次编辑后自动分析代码质量
  → Agent 能感知代码复杂度变化
  → 主动提出优化建议
```
