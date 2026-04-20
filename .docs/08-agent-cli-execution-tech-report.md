# Multica Agent CLI 执行机制详细技术报告

## 一、核心问题

用户关心的核心问题：**Daemon 如何启动 Agent CLI 并执行任务？**

---

## 二、Agent CLI 执行架构

### 2.1 执行流程概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Daemon                                      │
├─────────────────────────────────────────────────────────────────────┤
│  1. pollLoop()                                         │
│     ↓                                                   │
│  2. claimTask() → GET /daemon/runtimes/{id}/tasks/claim  │
│     ↓                                                   │
│  3. handleTask()                                       │
│     ├─→ Prepare/Prepare() → 创建隔离执行环境             │
│     ├─→ InjectRuntimeConfig() → 注入运行时配置            │
│     ├─→ BuildPrompt() → 构建 Agent prompt             │
│     └─→ agent.New() → 创建 Backend                   │
│             ↓                                         │
│  4. executeAndDrain()                                 │
│         ↓                                             │
│  5. Backend.Execute() → 执行 Agent CLI                │
│         ├─→ claude --output stream-json                │
│         ├─→ codex app-server                        │
│         └─→ opencode run                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| execenv.Prepare | daemon/execenv/execenv.go | 创建隔离执行环境 |
| execenv.InjectRuntimeConfig | daemon/execenv/runtime_config.go | 注入 Agent 运行时配置 |
| BuildPrompt | daemon/prompt.go | 构建任务提示词 |
| agent.New | pkg/agent/agent.go | 创建 Agent 后端 |
| Backend.Execute | pkg/agent/claude.go 等 | 执行具体的 Agent CLI |

---

## 三、执行环境准备

### 3.1 隔离环境创建

```go
// server/internal/daemon/execenv/execenv.go
func Prepare(params PrepareParams, logger *slog.Logger) (*Environment, error) {
    // 1. 创建目录结构
    // {workspacesRoot}/{workspaceID}/{taskID_short}/
    // ├── workdir/    (工作目录)
    // ├── output/    (输出目录)
    // └── logs/     (日志目录)

    envRoot := filepath.Join(params.WorkspacesRoot, params.WorkspaceID, shortID(params.TaskID))
    workDir := filepath.Join(envRoot, "workdir")

    // 2. 写入上下文文件 (skills, agent identity)
    writeContextFiles(workDir, params.Provider, params.Task)

    // 3. 对于 Codex，创建 per-task CODEX_HOME
    if params.Provider == "codex" {
        prepareCodexHomeWithOpts(...)
    }

    return &Environment{
        RootDir:  envRoot,
        WorkDir: workDir,
    }, nil
}
```

### 3.2 环境变量注入

```go
// server/internal/daemon/daemon.go
agentEnv := map[string]string{
    "MULTICA_TOKEN":         d.client.Token(),      // 认证 Token
    "MULTICA_SERVER_URL":    d.cfg.ServerBaseURL, // Server 地址
    "MULTICA_DAEMON_PORT": fmt.Sprintf("%d", d.cfg.HealthPort),
    "MULTICA_WORKSPACE_ID": task.WorkspaceID,
    "MULTICA_AGENT_NAME":   agentName,
    "MULTICA_AGENT_ID":   task.AgentID,
    "MULTICA_TASK_ID":    task.ID,
}

// 添加用户自定义环境变量
for k, v := range task.Agent.CustomEnv {
    agentEnv[k] = v
}

// 添加 multica CLI 到 PATH
binDir := filepath.Dir(selfBin)
agentEnv["PATH"] = binDir + string(os.PathListSeparator) + os.Getenv("PATH")

// Codex 需要 CODEX_HOME
if env.CodexHome != "" {
    agentEnv["CODEX_HOME"] = env.CodexHome
}
```

---

## 四、运行时配置注入

### 4.1 Provider 对应的配置文件

```go
// server/internal/daemon/execenv/runtime_config.go
func InjectRuntimeConfig(workDir, provider string, ctx TaskContextForEnv) error {
    content := buildMetaSkillContent(provider, ctx)

    switch provider {
    case "claude":
        // Claude Code → CLAUDE.md
        os.WriteFile(filepath.Join(workDir, "CLAUDE.md"), content)
    case "codex", "copilot", "opencode", "openclaw", "pi", "cursor":
        // 其他 → AGENTS.md
        os.WriteFile(filepath.Join(workDir, "AGENTS.md"), content)
    case "gemini":
        // Gemini → GEMINI.md
        os.WriteFile(filepath.Join(workDir, "GEMINI.md"), content)
    }
}
```

### 4.2 配置内容结构

```markdown
# Multica Agent Runtime

## Agent Identity
**You are: MyAgent** (ID: `agent-uuid`)
[Agent Instructions...]

## Available Commands
### Read
- `multica issue get <id> --output json`
- `multica issue list [--status X] --output json`
...

### Write
- `multica issue create --title "..."`
...

## Repositories
| URL | Description |
|-----|-------------|
| https://github.com/org/repo | Main repo |

## Workflow
1. Run `multica issue get <id> --output json`
2. Run `multica issue status <id> in_progress`
3. [Do work...]
4. `multica issue comment add <id> --content "..."`
5. `multica issue status <id> in_review`

## Skills
- **SkillName1**
- **SkillName2**

## Mentions
- **Issue**: `[MUL-123](mention://issue/<issue-id>)`
- **Member**: `[@Name](mention://member/<user-id>)`

## Important: Always Use the `multica` CLI
...
```

---

## 五、Prompt 构建

### 5.1 Prompt 类型

```go
// server/internal/daemon/prompt.go
func BuildPrompt(task Task) string {
    if task.ChatSessionID != "" {
        return buildChatPrompt(task)           // Chat 任务
    }
    if task.TriggerCommentID != "" {
        return buildCommentPrompt(task)         // @Mention 触发
    }
    return buildIssuePrompt(task)              // Issue 分配
}
```

### 5.2 Chat Prompt

```go
func buildChatPrompt(task Task) string {
    return `You are running as a chat assistant for a Multica workspace.
A user is chatting with you directly. Respond to their message.

User message:
{task.ChatMessage}`
}
```

### 5.3 Issue Prompt

```go
func buildIssuePrompt(task Task) string {
    return `You are running as a local coding agent for a Multica workspace.

Your assigned issue ID is: {task.IssueID}

Start by running `multica issue get {task.IssueID} --output json` to understand your task, then complete it.`
}
```

---

## 六、Agent CLI 启动

### 6.1 Backend 接口

```go
// server/pkg/agent/agent.go
type Backend interface {
    Execute(ctx context.Context, prompt string, opts ExecOptions) (*Session, error)
}

type ExecOptions struct {
    Cwd             string          // 工作目录
    Model           string          // 模型覆盖
    SystemPrompt    string          // 系统提示词
    MaxTurns        int            // 最大轮次
    Timeout         time.Duration   // 超时时间
    ResumeSessionID string         // 恢复会话 ID
    CustomArgs      []string       // 自定义参数
    McpConfig       json.RawMessage // MCP 配置
}
```

### 6.2 支持的 Agent 类型

```go
// server/pkg/agent/agent.go
var launchHeaders = map[string]string{
    "claude":   "claude (stream-json)",    // Claude Code
    "codex":    "codex app-server",     // Codex
    "copilot":  "copilot (json)",      // Copilot
    "cursor":   "cursor-agent",        // Cursor Agent
    "gemini":   "gemini (stream-json)",// Gemini CLI
    "hermes":   "hermes acp",          // Hermes
    "openclaw": "openclaw agent",      // OpenClaw
    "opencode": "opencode run",        // OpenCode
    "pi":       "pi (json mode)",      // Pi CLI
}
```

---

## 七、Claude Code 执行详解

### 7.1 CLI 参数

```go
// server/pkg/agent/claude.go
func buildClaudeArgs(opts ExecOptions, logger *slog.Logger) []string {
    args := []string{
        "--output-format", "stream-json",  // 流式 JSON 输出
    }

    if opts.Model != "" {
        args = append(args, "--model", opts.Model)
    }

    if opts.MaxTurns > 0 {
        args = append(args, "--max-turns", strconv.Itoa(opts.MaxTurns))
    }

    // 会话恢复
    if opts.ResumeSessionID != "" {
        args = append(args, "--resume", opts.ResumeSessionID)
    }

    return args
}
```

### 7.2 执行命令

```bash
claude --output-format stream-json [--model <model>] [--max-turns N] [--resume <session-id>] [--mcp-config <path>]
```

### 7.3 流式输出处理

```go
// server/pkg/agent/claude.go
scanner := bufio.NewScanner(stdout)

for scanner.Scan() {
    var msg claudeSDKMessage
    json.Unmarshal([]byte(line), &msg)

    switch msg.Type {
    case "assistant":
        // Agent 输出消息
    case "user":
        // 用户输入消息
    case "system":
        // 系统消息 (session_id, usage)
    case "error":
        // 错误消息
    }
}
```

### 7.4 JSON 输出格式

```json
{
  "type": "assistant",
  "message": {
    "role": "assistant",
    "content": [
      {"type": "text", "text": "Thinking..."}
    ]
  }
}
{
  "type": "tool_use",
  "name": "Bash",
  "input": {"command": "ls -la"}
}
{
  "type": "tool_result",
  "tool_use_id": "toolu_xxx",
  "content": "..."
}
{
  "type": "system",
  "session_id": "xxx",
  "usage": {"input": 1000, "output": 500}
}
```

---

## 八、执行选项

### 8.1 超时控制

```go
timeout := opts.Timeout
if timeout == 0 {
    timeout = 20 * time.Minute  // 默认 20 分钟
}
runCtx, cancel := context.WithTimeout(ctx, timeout)
```

### 8.2 会话恢复

```go
// 如果存在 prior_session_id，恢复之前会话
if task.PriorSessionID != "" {
    execOpts.ResumeSessionID = task.PriorSessionID
}

// 如果恢复失败，重试新会话
if result.Status == "failed" && task.PriorSessionID != "" && result.SessionID == "" {
    execOpts.ResumeSessionID = ""
    retryResult, retryTools, retryErr := d.executeAndDrain(...)
}
```

### 8.3 MCP 配置

```go
// 写入临时文件
if len(opts.McpConfig) > 0 {
    path, err := writeMcpConfigToTemp(opts.McpConfig)
    args = append(args, "--mcp-config", mcpConfigPath)
}

// 执行后清理
defer os.Remove(mcpConfigPath)
```

---

## 九、执行结果处理

### 9.1 结果类型

```go
type Result struct {
    Status     string            // "completed", "failed", "blocked", "timeout"
    Output     string           // Agent 输出文���
    Error      string            // 错误信息
    SessionID  string           // 会话 ID (用于恢复)
    WorkDir    string           // 工作目录 (复用)
    Usage      map[string]TokenUsage // Token 使用统计
}
```

### 9.2 状态转换

```
queued → dispatched → running → completed/failed
                                 ↓
                              "blocked" (超时或输出为空)
```

### 9.3 结果处理

```go
switch result.Status {
case "completed":
    // 保存 assistant message
    // 广播 chat:done
case "failed":
    // 记录失败原因
    // 更新 Issue 状态为 blocked (可选)
case "timeout":
    // 超时处理，保留 session_id 用于恢复
}
```

---

## 十、各 Agent CLI 详细参数

### 10.1 Claude Code (`claude`)

```bash
claude -p <prompt> \
    --output-format stream-json \
    --input-format stream-json \
    --verbose \
    --strict-mcp-config \
    --permission-mode bypassPermissions \
    [--model <model>] \
    [--max-turns <N>] \
    [--append-system-prompt <text>] \
    [--resume <session-id>] \
    [--mcp-config <path>]
```

**Blocked Args** (无法被 custom_args 覆盖):
- `--output-format`
- `--input-format`
- `--verbose`
- `--mcp-config`

**特点**:
- `-p`: 提示词作为标志参数
- `stream-json`: 流式 JSON 输出
- `--resume`: 恢复之前会话

---

### 10.2 Codex (`codex`)

```bash
codex app-server \
    --listen stdio:// \
    [<custom_args>]
```

**Blocked Args**:
- `--listen` (必须为 stdio://)

**特点**:
- JSON-RPC 2.0 协议 over stdin/stdout
- 需要自定义协议实现 tool 调用

---

### 10.3 OpenCode (`opencode`)

```bash
opencode run \
    --format json \
    [--model <model>] \
    [--prompt <system_prompt>] \
    [--session <session-id>] \
    [<custom_args>] \
    <prompt>
```

**Blocked Args**:
- `--format` (必须为 json)

**特点**:
- `run` 子命令
- `--session`: 会话恢复
- 从环境变量 `OPENCODE_PERMISSION` 自动批准工具

---

### 10.4 OpenClaw (`openclaw`)

```bash
openclaw agent \
    --local \
    --json \
    --yes \
    --session-id <id> \
    [<custom_args>]
```

**Blocked Args**:
- `--local` (本地模式)
- `--json` (JSON 输出)
- `--session-id` (daemon 管理)
- `--message` (prompt 由 daemon 设置)
- `--model` (模型在注册时指定)
- `--system-prompt` (在 --message 中注入)

**特点**:
- JSON 输出到 stderr (非 stdout)
- 自动生成 session-id: `multica-<timestamp>`

---

### 10.5 Hermes (`hermes`)

```bash
hermes acp \
    [<custom_args>]
```

**Blocked Args**:
- `acp` (必须使用 ACP 协议)

**特点**:
- ACP (Agent Communication Protocol) JSON-RPC
- 环境变量 `HERMES_YOLO_MODE=1` 自动批准工具

---

### 10.6 Gemini CLI (`gemini`)

```bash
gemini -p <prompt> \
    --yolo \
    -o stream-json \
    [-m <model>] \
    [-r <session-id>] \
    [<custom_args>]
```

**特点**:
- `-p`: 提示词标志
- `--yolo`: 自动批准工具
- `-o stream-json`: 流式输出
- `-r`: 会话恢复

---

### 10.7 Pi CLI (`pi`)

```bash
pi -p <prompt> \
    --mode json \
    [--session <path>] \
    [--provider <provider>] \
    [--model <model>] \
    --tools read,bash,edit,write,grep,find,ls \
    [--append-system-prompt <text>] \
    [<custom_args>] \
    <prompt>
```

**特点**:
- `--mode json`: JSON 模式
- `--tools`: 启用工具列表
- `--provider/model`: 指定模型

---

### 10.8 Cursor Agent (`cursor-agent`)

```bash
cursor-agent \
    --output-format stream-json \
    [--model <model>] \
    [--max-turns <N>] \
    [--resume <session-id>] \
    [<custom_args>]
```

**特点**:
- 类似 Claude Code 的流式输出
- `stream-json` 格式

---

### 10.9 GitHub Copilot (`copilot`)

```bash
copilot -p <prompt> \
    --output-format json \
    --allow-all \
    --no-ask-user \
    [--model <model>] \
    [--resume <session-id>] \
    [<custom_args>]
```

**特点**:
- `--allow-all`: 允许所有工具和 URL
- `--no-ask-user`: 无需用户交互
- 单次执行模式 (非交互)

---

### 10.10 参数对比表

| Agent | 通信协议 | 输出格式 | 会话恢复 | 阻塞参数 |
|-------|----------|----------|----------|----------|
| Claude | stdin/stdout | stream-json | --resume | --output-format |
| Codex | JSON-RPC | stdio:// | N/A | --listen |
| OpenCode | stdin/stdout | json | --session | --format |
| OpenClaw | stderr | json | --session-id | --local, --json |
| Hermes | ACP | JSON-RPC | N/A | acp |
| Gemini | stdin/stdout | stream-json | -r | None |
| Pi | stdin/stdout | json | --session | None |
| Cursor | stdin/stdout | stream-json | --resume | None |
| Copilot | stdin/stdout | json | --resume | None |

---

### 10.11 自定义参数示例

```json
// Agent 配置
{
  "custom_args": [
    "--max-turns", "50",
    "--no-diff"
  ]
}
```

**注意**: 部分参数被阻塞，无法通过 `custom_args` 覆盖。

---

## 十一、关键代码位置

### 11.1 执行入口

| 文件 | 行号 | 说明 |
|------|------|------|
| daemon/daemon.go | 773 | handleTask 入口 |
| daemon/daemon.go | 826 | runTask 执行 |
| daemon/daemon.go | 1039 | executeAndDrain 调用 |

### 11.2 Agent CLI

| 文件 | 说明 |
|------|------|
| pkg/agent/agent.go | Backend 接口 |
| pkg/agent/claude.go | Claude Code 后端 |
| pkg/agent/codex.go | Codex 后端 |
| pkg/agent/opencode.go | OpenCode 后端 |

### 11.3 执行环境

| 文件 | 说明 |
|------|------|
| execenv/execenv.go | 环境准备 |
| execenv/runtime_config.go | 配置注入 |
| execenv/context.go | 上下文构建 |
| daemon/prompt.go | Prompt 构建 |

---

## 十二、环境变量汇总

### 12.1 Daemon 设置

| 环境变量 | 说明 |
|----------|------|
| MULTICA_TOKEN | 认证 Token |
| MULTICA_SERVER_URL | Server 地址 |
| MULTICA_DAEMON_PORT | Daemon 端口 |
| MULTICA_WORKSPACE_ID | 工作区 ID |
| MULTICA_AGENT_NAME | Agent 名称 |
| MULTICA_AGENT_ID | Agent ID |
| MULTICA_TASK_ID | 任务 ID |

### 12.2 Agent 设置

| 环境变量 | 说明 |
|----------|------|
| PATH | 包含 multica CLI 路径 |
| CODEX_HOME | Codex per-task 主目录 |
| ANTHROPIC_API_KEY | (用户配置) |
| ANTHROPIC_BASE_URL | (用户配置) |

---

## 十三、最佳实践

### 12.1 自定义 Agent 参数

```json
// Agent 配置
{
  "custom_args": ["--max-turns", "50", "--no-diff"],
  "custom_env": {
    "ANTHROPIC_API_KEY": "sk-..."
  }
}
```

### 12.2 MCP 配置

```json
{
  "mcp_config": {
    "servers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
      }
    }
  }
}
```

### 12.3 工作目录复用

```go
// 任务完成时保存 work_dir
task, err := s.Queries.CompleteAgentTask(ctx, taskID, result, sessionID, workDir)

// 下次任务带 prior_work_dir
task.PriorWorkDir // → execenv.Reuse() 恢复
```