# Skill: shell 指定 Shell 环境

> 指南 6.2「Skill 定义」— shell 字段指定执行 Bash 命令时使用的 Shell

## 目录结构

```
my-project/
├── .claude/
│   └── commands/
│       ├── build.sh               ← shell: bash（默认）
│       ├── iis-deploy.md          ← shell: powershell
│       ├── docker-build.md        ← shell: bash
│       └── win-service.md         ← shell: powershell
└── src/
```

## 说明

```
shell 字段:
  可选值: "bash"（默认） 或 "powershell"
  作用: 指定 Skill 中 Bash 工具使用的 Shell 环境

  bash:
    → Linux/macOS 默认 Shell
    → 支持 sh 语法: if/then/fi, for/do/done, $() 等
    → 命令: grep, sed, awk, cat, find, jq

  powershell:
    → Windows 专用 Shell
    → 支持 PS 语法: if(){}, foreach($x in $y){}, $(Get-Content) 等
    → 命令: Get-Content, Select-String, Where-Object, Start-Process

  注意:
    → shell 只影响 Bash 工具执行的命令
    → 不影响 Skill 文件本身的解析
    → Windows 用户可能需要 powershell
    → Linux/macOS 用户使用 bash 即可
```

## 使用示例

### 示例 1：默认 bash 执行标准 Shell 命令

```
build.md:
  ---
  description: Build the project
  shell: bash
  allowed-tools:
    - Bash
    - Read
  ---

→ 执行的命令全部通过 bash:
  Bash: "npm run build"
  Bash: "ls -la dist/"
  Bash: "du -sh dist/"

→ bash 特有的管道操作:
  Bash: "cat package.json | jq '.version'"
  Bash: "find src -name '*.ts' | wc -l"

效果:
  → bash 是默认值，大多数项目使用 bash
  → 标准 Unix 命令语法
```

### 示例 2：Windows 环境使用 PowerShell

```
iis-deploy.md:
  ---
  description: Deploy to IIS on Windows
  shell: powershell
  allowed-tools:
    - Bash
  ---

  Deploy the built application to IIS web server.

→ 执行的命令全部通过 PowerShell:
  Bash: "Get-Website -Name 'MyApp'"
  Bash: "Copy-Item -Path 'dist/*' -Destination 'C:\\inetpub\\wwwroot\\MyApp' -Recurse"
  Bash: "Restart-WebAppPool -Name 'MyAppPool'"

→ PowerShell 特有操作:
  Bash: "(Get-Content web.config) -replace 'debug=true','debug=false' | Set-Content web.config"
  Bash: "Get-EventLog -LogName Application -Newest 10"

效果:
  → Windows IIS 部署需要 PowerShell 命令
  → shell: powershell 确保命令在正确环境执行
  → Windows 特有的管理命令可用
```

### 示例 3：Docker 构建使用 bash（跨平台）

```
docker-build.md:
  ---
  description: Build and push Docker image
  shell: bash
  allowed-tools:
    - Bash
    - Read
  ---

  Build a Docker image and push to registry.

→ bash 命令:
  Bash: "docker build -t myapp:$(git rev-parse --short HEAD) ."
  Bash: "docker tag myapp:latest registry.example.com/myapp:latest"
  Bash: "docker push registry.example.com/myapp:latest"

→ bash 语法（命令替换）:
  $(git rev-parse --short HEAD) → "a3f2b1c"
  → 镜像名: myapp:a3f2b1c

效果:
  → Docker 命令在 bash 中执行
  → bash 的命令替换 $() 用于动态生成标签
  → 跨平台兼容（Docker Desktop 在 Windows 也支持 bash）
```

### 示例 4：Windows 服务管理使用 PowerShell

```
win-service.md:
  ---
  description: Manage Windows services
  shell: powershell
  allowed-tools:
    - Bash
  ---

  Check and manage Windows services related to the application.

→ PowerShell 命令:
  Bash: "Get-Service -Name 'MyApp*' | Format-Table Name,Status,StartType"
  Bash: "Start-Service -Name 'MyAppWorker'"
  Bash: "Get-Process -Name 'MyApp' | Select-Object CPU,WorkingSet"

→ PowerShell 远程管理:
  Bash: "Invoke-Command -ComputerName prod-server -ScriptBlock { Get-Service 'MyApp' }"

效果:
  → Windows 服务管理必须用 PowerShell
  → Get-Service、Start-Service 等 PS 命令
  → 远程管理 Invoke-Command
```

### 示例 5：不指定 shell 时根据平台自动选择

```
run-tests.md:
  ---
  description: Run test suite
  allowed-tools:
    - Bash
  ---

  Run the project's test suite with coverage.

→ 没有 shell 字段

→ 在 Linux/macOS 上:
  默认使用 bash → npm test -- --coverage ✅

→ 在 Windows 上:
  默认使用 bash (Git Bash) → npm test -- --coverage ✅
  （Claude Code 在 Windows 也默认使用 bash）

→ 如果需要 PowerShell 特有命令:
  必须显式指定 shell: powershell

效果:
  → 大多数 npm/node 命令在 bash 和 PowerShell 中都能运行
  → 只有使用平台特有命令时才需要指定 shell
  → 建议: 跨平台命令不指定 shell，平台特有命令明确指定
```
