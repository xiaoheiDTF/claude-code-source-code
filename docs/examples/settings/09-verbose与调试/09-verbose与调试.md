# settings.json: verbose 与调试配置

> 指南 7.2「主要配置项」— verbose 启用详细日志输出

## settings.json 配置

```json
{
  "verbose": true
}
```

## 说明

```
verbose 字段:
  类型: boolean
  默认: false
  作用: 启用详细日志输出，显示更多内部信息

  启用后 (true):
    → 显示工具调用的详细参数
    → 显示模型选择的推理过程
    → 显示 Hook 触发和执行日志
    → 显示 MCP 服务器通信详情
    → 显示 token 使用统计

  关闭时 (false):
    → 只显示关键信息
    → 输出更简洁
    → 日常使用推荐

  适用场景:
    → 调试 Hook 行为
    → 排查权限问题
    → 理解 Claude 的决策过程
    → 优化 token 使用
```

## 使用示例

### 示例 1：启用 verbose 查看 Hook 触发详情

```json
{
  "verbose": true,
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/check.sh" }]
      }
    ]
  }
}
```

```
→ verbose: false 时:
  Claude: "我来修改 app.ts..."
  [Edit] src/app.ts
  完成。

→ verbose: true 时:
  Claude: "我来修改 app.ts..."
  [Hook] PreToolUse → matcher: "Edit" → 匹配 ✅
  [Hook] 执行: bash .claude/hooks/check.sh
  [Hook] stdin: {"tool_name":"Edit","tool_input":{"file_path":"src/app.ts",...}}
  [Hook] stdout: {"permissionDecision":"allow"}
  [Hook] 结果: 允许执行
  [Edit] src/app.ts → old: "const x = 1" → new: "const x = 2"
  [耗时] 1.2s
  [Token] 输入: 245, 输出: 128

效果:
  → 可以看到 Hook 是否触发、参数、返回值
  → 调试 Hook 时必须开启 verbose
```

### 示例 2：排查权限问题

```json
{
  "verbose": true,
  "permissions": {
    "allow": ["Bash(npm test*)"],
    "deny": ["Bash(rm -rf*)"],
    "defaultMode": "default"
  }
}
```

```
→ 用户: "运行测试并清理 dist 目录"

→ verbose: true 输出:
  [Perm] Bash: npm test -- --coverage
  [Perm] 检查 deny 列表: 不匹配
  [Perm] 检查 allow 列表: 匹配 "Bash(npm test*)" ✅
  [Perm] 结果: 允许执行

  [Perm] Bash: rm -rf dist/
  [Perm] 检查 deny 列表: 匹配 "Bash(rm -rf*)" ❌
  [Perm] 结果: 拒绝执行

  [Perm] Bash: npm run clean
  [Perm] 检查 deny 列表: 不匹配
  [Perm] 检查 allow 列表: 不匹配
  [Perm] defaultMode: "default" → 需要手动确认
  [Perm] 弹窗确认: "执行 npm run clean？ [Y/n]"

效果:
  → 每个命令的权限判定过程清晰可见
  → 可以看到为什么被 allow/deny/defaultMode
  → 排查 "为什么这个命令被拒绝" 的问题
```

### 示例 3：查看 MCP 服务器通信

```json
{
  "verbose": true,
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

```
→ verbose: true 输出:
  [MCP] 启动服务器: postgres
  [MCP] 命令: npx -y @modelcontextprotocol/server-postgres postgresql://localhost/mydb
  [MCP] 连接成功 ✅
  [MCP] 可用工具: query, describeTable, listTables

  [MCP] 调用: postgres.query("SELECT * FROM users LIMIT 5")
  [MCP] 请求: {"method": "tools/call", "params": {"name": "query", "arguments": {"sql": "..."}}}
  [MCP] 响应: {"result": [{"id": 1, "name": "Alice"}, ...]}
  [MCP] 耗时: 230ms

效果:
  → MCP 服务器启动、连接、调用全链路可见
  → 排查 MCP 连接问题
  → 优化 MCP 查询性能
```

### 示例 4：查看 Token 使用统计

```json
{
  "verbose": true
}
```

```
→ 每次工具调用后显示:
  [Token] 本轮: 输入 1,245 / 输出 512
  [Token] 累计: 输入 15,678 / 输出 3,456
  [Token] 预估成本: $0.12

→ 会话结束时显示:
  [Token] 会话总计:
    输入: 45,230 tokens
    输出: 12,456 tokens
    工具调用: 23 次
    预估成本: $0.34

效果:
  → 实时了解 token 消耗
  → 优化长会话的成本
  → 识别高 token 消耗的操作
```

### 示例 5：生产环境关闭 verbose

```json
# 开发环境 .claude/settings.json
{
  "verbose": true,
  "permissions": { "defaultMode": "acceptEdits" }
}

# 生产环境 .claude/settings.json
{
  "verbose": false,
  "permissions": { "defaultMode": "default" }
}
```

```
→ 开发环境:
  verbose: true → 详细日志帮助调试
  acceptEdits → 快速迭代

→ 生产环境:
  verbose: false → 简洁输出
  default → 安全确认

效果:
  → 开发时开启 verbose 帮助理解行为
  → 生产/日常关闭 verbose 减少噪音
  → 按环境切换配置
```
