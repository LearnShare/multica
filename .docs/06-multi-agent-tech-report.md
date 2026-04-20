# Multica 多 Agent 支持及通信机制详细技术报告

## 一、Agent 架构概述

### 1.1 多 Agent 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Multica Server                               │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐             │
│  │   Agent A    │   │   Agent B    │   │   Agent C    │             │
│  │ (Claude)    │   │   (Codex)    │   │ (OpenCode)   │             │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘             │
│         │                  │                  │                     │
│         └──────────────────┼──────────────────┘                     │
│                            │                                      │
│                    ┌───────┴───────┐                              │
│                    │ TaskService  │  任务分发服务                    │
│                    │ (Go)         │                              │
│                    └───────┬───────┘                              │
│                            │                                      │
│              ┌─────────────┼─────────────┐                         │
│              │             │             │                         │
│      ┌───────┴───────┐  ┌──┴───┐  ┌────┴──────┐                │
│      │ DaemonConnection│  │Queue │  │WebSocket  │                │
│      └───────────────┘  └──────┘  └──────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   Local Daemon                                  │
├───┬───────────┬───────────────┬───────────────────────┤
│ A-0│     A-1  │      B-0     │         ...          │ ← 运行中的任务
└──────────────────────────────────────────────────────────────┘
```

### 1.2 核心组件

| 组件 | 位置 | 职责 |
|------|------|------|
| TaskService | server/internal/service/task.go | 任务分发、状态管理 |
| Daemon | server/internal/daemon/daemon.go | 本地运行时 |
| AgentHandler | server/internal/handler/agent.go | API 端点 |
| runtime | server/internal/daemon/runtime.go | Agent CLI 抽象 |

---

## 二、任务队列机制

### 2.1 任务生命周期

```go
// server/internal/service/task.go
// 任务状态流转:
// queued → dispatched → running → completed/failed/cancelled
```

### 2.2 任务入队

#### 2.2.1 Issue 分配触发

```go
func (s *TaskService) EnqueueTaskForIssue(ctx context.Context, issue db.Issue, triggerCommentID ...pgtype.UUID) (db.AgentTaskQueue, error) {
    // 1. 验证 Issue 有分配者
    if !issue.AssigneeID.Valid {
        return db.AgentTaskQueue{}, fmt.Errorf("issue has no assignee")
    }
    
    // 2. 获取 Agent 配置
    agent, err := s.Queries.GetAgent(ctx, issue.AssigneeID)
    if err != nil { ... }
    
    // 3. 检查 Agent 已归档
    if agent.ArchivedAt.Valid { ... }
    
    // 4. 检查 Agent 有运行时
    if !agent.RuntimeID.Valid { ... }
    
    // 5. 创建任务记录
    task, err := s.Queries.CreateAgentTask(ctx, db.CreateAgentTaskParams{
        AgentID:     issue.AssigneeID,
        RuntimeID:  agent.RuntimeID,
        IssueID:    issue.ID,
        Priority:  priorityToInt(issue.Priority),  // Issue 优先级映射到任务优先级
    })
    
    return task, nil
}
```

##### 优先级映射

```
urgent   → 4 (最高)
high    → 3
medium  → 2
low     → 1
none    → 0
```

#### 2.2.2 @ Mention 触发

```go
func (s *TaskService) EnqueueTaskForMention(ctx context.Context, issue db.Issue, agentID pgtype.UUID, triggerCommentID pgtype.UUID) (db.AgentTaskQueue, error) {
    // 与 Issue 分配不同，这里使用显式指定的 agentID
    // 用于在已有 Agent 分配时，额外触发另一个 Agent
}
```

#### 2.2.3 Chat 会话触发

```go
func (s *TaskService) EnqueueChatTask(ctx context.Context, chatSession db.ChatSession) (db.AgentTaskQueue, error) {
    // Chat 任务没有 issue_id，只有 chat_session_id
    // 优先级固定为 2 (medium)
    task, err := s.Queries.CreateChatTask(ctx, db.CreateChatTaskParams{
        AgentID:      chatSession.AgentID,
        RuntimeID:    agent.RuntimeID,
        Priority:    2,
        ChatSessionID: chatSession.ID,
    })
}
```

### 2.3 任务认领 (Claim)

#### 2.3.1 按 Runtime 认领

```go
// server/internal/service/task.go - ClaimTaskForRuntime
func (s *TaskService) ClaimTaskForRuntime(ctx context.Context, runtimeID pgtype.UUID) (*db.AgentTaskQueue, error) {
    // 1. 获取 Runtime 的待处理任务
    tasks, err := s.Queries.ListPendingTasksByRuntime(ctx, runtimeID)
    
    // 2. 遍历尝试认领
    for _, candidate := range tasks {
        // 3. 调用 ClaimTask 检查并发限制
        task, err := s.ClaimTask(ctx, candidate.AgentID)
        if task != nil && task.RuntimeID == runtimeID {
            // 4. 成功认领，返回任务
            return task, nil
        }
    }
    return nil, nil  // 无任务可认领
}
```

#### 2.3.2 并发控制

```go
// server/internal/service/task.go - ClaimTask
func (s *TaskService) ClaimTask(ctx context.Context, agentID pgtype.UUID) (*db.AgentTaskQueue, error) {
    // 1. 获取 Agent 配置
    agent, err := s.Queries.GetAgent(ctx, agentID)
    
    // 2. 检查当前运行中任务数
    running, err := s.Queries.CountRunningTasks(ctx, agentID)
    if err != nil { ... }
    
    // 3. 检查是否超限
    if running >= agent.MaxConcurrentTasks {
        return nil, nil  // 达到并发上限
    }
    
    // 4. 原子更新任务状态 (SELECT FOR UPDATE)
    task, err := s.Queries.ClaimAgentTask(ctx, db.ClaimAgentTaskParams{
        AgentID: agentID,
        Limit:  1,
    })
    
    // 5. 更新 Agent 状态
    s.updateAgentStatus(ctx, agentID, "working")
    
    return &task, nil
}
```

##### Agent 状态

| 状态 | 说明 |
|------|------|
| idle | 空闲 |
| working |工作中 |
| blocked | 被阻塞 |
| error | 错误 |
| offline | 离线 |

### 2.4 任务执行结果

#### 2.4.1 开始任务

```go
// POST /api/daemon/tasks/{taskId}/start
func (s *TaskService) StartTask(ctx context.Context, taskID pgtype.UUID) (*db.AgentTaskQueue, error) {
    task, err := s.Queries.StartAgentTask(ctx, taskID)
    // status: queued → running
}
```

#### 2.4.2 进度报告

```go
// POST /api/daemon/tasks/{taskId}/progress
func (s *TaskService) ReportProgress(ctx context.Context, taskID string, workspaceID string, summary string, step, total int) {
    // step: 当前步骤
    // total: 总步骤数
    // summary: 进度描述
}
```

#### 2.4.3 完成任务

```go
// POST /api/daemon/tasks/{taskId}/complete
func (s *TaskService) CompleteTask(ctx context.Context, taskID pgtype.UUID, result []byte, sessionID, workDir string) (*db.AgentTaskQueue, error) {
    // 1. 更新任务状态为 completed
    task, err := s.Queries.CompleteAgentTask(ctx, taskID, result, sessionID, workDir)
    
    // 2. 重建 Agent 状态 (idle 或 working)
    s.ReconcileAgentStatus(ctx, task.AgentID)
    
    // 3. 广播任务完成事件
    s.broadcastTaskEvent(ctx, protocol.EventTaskCompleted, task)
}
```

#### 2.4.4 失败任务

```go
// POST /api/daemon/tasks/{taskId}/fail
func (s *TaskService) FailTask(ctx context.Context, taskID pgtype.UUID, errMsg, sessionID, workDir string) (*db.AgentTaskQueue, error) {
    // 1. 更新任务状态为 failed
    task, err := s.Queries.FailAgentTask(ctx, taskID, errMsg, sessionID, workDir)
    
    // 2. 重建 Agent 状态
    s.ReconcileAgentStatus(ctx, task.AgentID)
    
    // 3. 更新 Issue 状态为 blocked (可选)
    s.broadcastIssueUpdated(issue)
}
```

---

## 三、Daemon 轮询机制

### 3.1 Polling Loop

```go
// server/internal/daemon/daemon.go - pollLoop
func (d *Daemon) pollLoop(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return nil
        default:
        }
        
        // 检查并发限制
        if d.activeTasks.Load() >= int64(d.cfg.MaxConcurrentTasks) {
            d.logger.Debug("poll: at capacity")
            time.Sleep(d.cfg.PollInterval)
            continue
        }
        
        // 遍历所有注册的 runtime
        for _, runtimeID := range runtimeIDs {
            task := d.claimAndHandleTask(ctx, runtimeID)
            if task != nil {
                go d.handleTask(ctx, task)
            }
        }
    }
}
```

### 3.2 任务认领 (Daemon 侧)

```go
func (d *Daemon) claimAndHandleTask(ctx context.Context, runtimeID string) *Task {
    // 1. 调用 Server API 认领任务
    resp, err := d.client.ClaimTask(ctx, runtimeID)
    if err != nil || resp.Task == nil {
        return nil  // 无任务
    }
    return resp.Task
}
```

### 3.3 轮询间隔配置

```go
// server/internal/daemon/config.go
type Config struct {
    PollInterval: 5 * time.Second,  // 默认 5 秒
}

// 环境变量: MULTICA_DAEMON_POLL_INTERVAL
```

---

## 四、Agent 通信协议

### 4.1 API 端点

#### 4.1.1 Daemon 注册

```
POST /api/daemon/register
X-Daemon-Token: <token>

Response:
{
    "daemon_id": "...",
    "workspace_id": "...",
    "runtimes": [
        {
            "id": "...",
            "name": "agent-claude",
            "provider": "claude",
            "status": "idle"
        }
    ]
}
```

#### 4.1.2 任务认领

```
GET /api/daemon/runtimes/{runtimeId}/tasks/claim

Response:
{
    "task": {
        "id": "...",
        "agent_id": "...",
        "issue_id": "...",
        "agent": {
            "id": "...",
            "name": "My Agent",
            "instructions": "You are a software engineer...",
            "skills": [...],
            "custom_env": {...},
            "custom_args": [...],
            "mcp_config": {...}
        },
        "repos": [...],
        "prior_session_id": "...",
        "prior_work_dir": "..."
    }
}
```

#### 4.1.3 任务状态更新

```
POST /api/daemon/tasks/{taskId}/start
POST /api/daemon/tasks/{taskId}/progress
POST /api/daemon/tasks/{taskId}/complete
POST /api/daemon/tasks/{taskId}/fail

Request:
{
    "comment": "Task description or error message",
    "session_id": "claude:...",     // Claude session ID
    "work_dir": "/path/to/work",  // 工作目录
    "usage": [...]             // Token 使用统计
}
```

### 4.2 任务响应结构

```go
// server/internal/daemon/types.go
type Task struct {
    ID                    string     `json:"id"`
    AgentID               string     `json:"agent_id"`
    RuntimeID             string     `json:"runtime_id"`
    IssueID               string     `json:"issue_id"`
    WorkspaceID           string     `json:"workspace_id"`
    Agent                 *AgentData `json:"agent,omitempty"`
    Repos                 []RepoData `json:"repos,omitempty"`
    PriorSessionID        string     `json:"prior_session_id,omitempty"`
    PriorWorkDir          string     `json:"prior_work_dir,omitempty"`
    TriggerCommentID      string     `json:"trigger_comment_id,omitempty"`
    TriggerCommentContent string     `json:"trigger_comment_content,omitempty"`
    ChatSessionID       string     `json:"chat_session_id,omitempty"`
    ChatMessage         string     `json:"chat_message,omitempty"`
}

type AgentData struct {
    ID           string            `json:"id"`
    Name         string            `json:"name"`
    Instructions string            `json:"instructions"`
    Skills       []SkillData       `json:"skills"`
    CustomEnv    map[string]string `json:"custom_env,omitempty"`
    CustomArgs   []string        `json:"custom_args,omitempty"`
    McpConfig    json.RawMessage  `json:"mcp_config,omitempty"`
}
```

---

## 五、多 Agent 协调机制

### 5.1 Runtime 概念

```go
type Runtime struct {
    ID       string  // UUID
    Name     string  // 显示名称
    Provider string  // "claude", "codex", "opencode", etc.
    Status   string  // "idle", "working"
}
```

### 5.2 多 Runtime 支持

```go
// 一个 Daemon 可以注册多个 Runtime
// 每个 Runtime 可以分配给不同的 Agent

// 架构:
// Daemon
// ├── Runtime-A (claude)
// │   ├── Agent-A1 (Primary Coder)
// │   └── Agent-A2 (Reviewer)
// └── Runtime-B (codex)
//     └── Agent-B1 (Security Scanner)
```

### 5.3 任务分发策略

```go
// server/internal/service/task.go - 任务分发逻辑
func (s *TaskService) EnqueueTaskForIssue(ctx context.Context, issue db.Issue, ...) (db.AgentTaskQueue, error) {
    // 1. Issue 必须有 assignee (Agent)
    // 2. Agent 必须有 runtime_id
    // 3. 任务绑定到 Agent.RuntimeID
    
    // 意义:
// 不同的 Agent 可以共享同一个 Runtime (串行执行)
// 同一个 Agent 可以有独立 Runtime (并行执行，受 max_concurrent_tasks 限制)
}
```

### 5.4 会话保持 (Session Persistence)

```go
// 支持同一个 Issue 的多轮对话
PriorSessionID   // 之前任务的 Claude session ID
PriorWorkDir     // 工作目录，用于resume

// 在 ClaimTask 响应中返回
// Daemon 使用这些信息继续之前的会话
task, err := d.runTask(runCtx, task, provider, taskLog)
    - 使用 --continue 或 resume flag
```

---

## 六、MCP 配置支持

### 6.1 MCP 服务器配置

```go
type AgentData struct {
    // ...
    McpConfig json.RawMessage `json:"mcp_config,omitempty"`
}
```

### 6.2 MCP 配置格式

```json
{
  "servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
      "env": {
        "KEY": "value"
      }
    },
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git", "--repository", "/repo/path"]
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "xxx"
      }
    }
  }
}
```

### 6.3 MCP 配置管理

```go
// server/internal/handler/agent.go
// API 端点
GET  /api/agents/{id}           // 获取 (含 MCP)
PUT  /api/agents/{id}           // 更新 MCP

// 敏感信息脱敏
func redactMcpConfig(resp *AgentResponse) {
    if resp.McpConfig != nil {
        resp.McpConfig = nil
        resp.McpConfigRedacted = true  // 标记为已脱敏
    }
}

// 只有 owner/admin 可以查看原始 MCP 配置
```

### 6.4 Daemon 端 MCP 注入

```go
// server/internal/daemon/daemon.go
func (d *Daemon) handleTask(ctx context.Context, task Task) {
    // ...
    
    // 构建执行环境
    env := execenv.TaskContextForEnv{
        // ...
        McpConfig: task.Agent.McpConfig,  // 传递给执行环境
    }
    
    result, err := d.runTask(ctx, task, provider, taskLog)
}
```

---

## 七、Agent 执行流程

### 7.1 完整执行流程图

```
┌─────────────────────────────────────────────────────────────┐
│                   Daemon Side                              │
├─────────────────────────────────────────────────────────────┤
│  1. pollLoop()                                        │
│     ↓                                               │
│  2. claimTask() → GET /daemon/runtimes/{id}/tasks/claim │
│     ↓                                               │
│  3. StartTask() → POST /daemon/tasks/{id}/start         │
│     ↓                                               │
│  4. 构建执行环境                                    │
│     - 创建工作目录                                   │
│     - 设置环境变量                                  │
│     - 注入 Skills                                   │
│     - 配置 MCP 服务器                               │
│     ↓                                               │
│  5. 运行 Agent CLI                                 │
│     - claude [options] --print < prompt                 │
│     - codex run                                    │
│     ↓                                               │
│  6. 处理执行结果                                   │
│     - 成功: completeTask()                         │
│     - 失败: failTask()                            │
│     - 阻塞: failTask() (status=blocked)           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Server Side                              │
├─────────────────────────────────────────────────────────────┤
│  7. 更新任务状态                                    │
│  8. 重建 Agent 状态 (Check running tasks)              │
│  9. 广播 WebSocket 事件                           │
│  10. 更新 Issue 状态 (可选)                         │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 执行环境构建

```go
// server/internal/daemon/execenv/
type TaskContextForEnv struct {
    IssueID             string
    TriggerCommentID   string
    AgentID            string
    AgentName         string
    AgentInstructions string
    AgentSkills      []SkillForEnv
    Repos            []RepoForEnv
    McpConfig        json.RawMessage
    Session          string  // prior session ID
}
```

---

## 八、实时事件

### 8.1 任务事件

| 事件 | 说明 | Payload |
|------|------|----------|
| task:dispatched | 任务已分发 | task |
| task:started | 任务开始 | task |
| task:completed | 任务完成 | task |
| task:failed | 任务失败 | task |
| task:cancelled | 任务取消 | task |
| task:progress | 任务进度 | progress |

### 8.2 Agent 状态事件

| 事件 | 说明 |
|------|------|
| agent:status | Agent 状态变化 |
| agent:updated | Agent 配置更新 |

---

## 九、关键配置

### 9.1 Agent 配置

```go
type Agent struct {
    ID                  UUID
    WorkspaceID         UUID
    Name                string
    RuntimeMode         string  // "local" | "cloud"
    RuntimeID           UUID   // 关联的 runtime
    Visibility         string  // "workspace" | "private"
    MaxConcurrentTasks  int32   // 并发任务数
    CustomEnv          map[string]string
    CustomArgs         []string
    McpConfig          json.RawMessage
}
```

### 9.2 Daemon 配置

```go
type Config struct {
    ServerBaseURL      string
    DaemonToken       string
    Agents           map[string]AgentEntry  // provider -> AgentEntry
    MaxConcurrentTasks int
    PollInterval     time.Duration
}
```

### 9.3 环境变量

```
MULTICA_SERVER_URL        # Server 地址
MULTICA_DAEMON_TOKEN    # Daemon 认证 Token
MULTICA_DAEMON_POLL_INTERVAL  # 轮询间隔
```

---

## 十、最佳实践

### 10.1 多 Agent 协作模式

1. **主从模式**
   - Agent A: Primary Coder (分配大部分 Issue)
   - Agent B: Reviewer (@mention 触发)

2. **并行模式**
   - 多个 Agent 分配不同 Runtime
   - 同一 Issue 可以分配给多个 Agent

3. **专业分工**
   - Agent A: Frontend (React)
   - Agent B: Backend (Go)
   - Agent C: DevOps

### 10.2 任务分配策略

1. **自动分配**：Issue 分配给 Agent 时自动入队
2. **手动触发**：@agent mention 触发新任务
3. **优先级**：Issue priority → task priority

### 10.3 会话管理

1. 使用 `prior_session_id` 保持对话连续性
2. 使用 `prior_work_dir` 复用工作目录
3. Chat 会话自动持久化 session_id