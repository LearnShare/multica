# Multica 核心功能实现分析报告

## 一、Agent 管理功能

### 1.1 数据模型

```go
// server/internal/handler/agent.go
type AgentResponse struct {
    ID                 string            // UUID
    WorkspaceID        string
    RuntimeID           string          // 关联的 runtime
    Name                string
    Description         string
    Instructions        string           // 给 Agent 的指令
    AvatarURL           *string
    RuntimeMode        string           // "local" | "cloud"
    RuntimeConfig      any              // JSON runtime 配置
    CustomEnv          map[string]string // 自定义环境变量
    CustomArgs         []string         // 自定义命令行参数
    McpConfig          json.RawMessage // MCP 服务器配置
    Visibility         string           // "workspace" | "private"
    Status             string           // "idle" | "working" | "blocked" | "error" | "offline"
    MaxConcurrentTasks int32           // 最大并发任务数
    OwnerID             *string        // 所有者用户
    Skills             []SkillResponse // 关联的技能
}
```

### 1.2 支持的 Provider

支持的 AI Agent CLI (从代码中提取):

| Provider | RuntimeMode | 说明 |
|----------|-----------|------|
| claude | local | Claude Code (claude CLI) |
| codex | local | Codex CLI |
| openclaw | local | OpenClaw |
| opencode | local | OpenCode |
| hermes | local | Hermes |
| gemini | local | Gemini CLI |
| pi | local | Pi CLI |
| cursor-agent | local | Cursor Agent |
| cloud | cloud | 云端 API |

### 1.3 Handler 函数

文件: `server/internal/handler/agent.go`

```go
func (h *Handler) ListAgents(...)     // 列出所有 Agent
func (h *Handler) GetAgent(...)    // 获取单个 Agent
func (h *Handler) CreateAgent(...)// 创建 Agent
func (h *Handler) UpdateAgent(...)// 更新 Agent 配置
func (h *Handler) ArchiveAgent(...)// 归档 Agent
func (h *Handler) ClaimTask(...)   // Agent 认领任务
func (h *Handler) ReportTaskResult(...) // Agent 报告任务结果
func (h *Handler) ListAgentRuntimes(...) // 列出 Agent 的运行时
```

### 1.4 MCP 配置支持

Agent 支持 MCP (Model Context Protocol) 配置：

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

---

## 二、Issue 管理功能

### 2.1 数据模型

```go
type IssueResponse struct {
    ID                string
    WorkspaceID       string
    Title             string           // 标题
    Description       *string          // 描述 (Markdown)
    Status            string          // 状态
    Priority          string          // 优先级
    AssigneeType      *string         // "member" | "agent"
    AssigneeID        *string
    Assignee          *ActorResponse  // 分配对象
    CreatorType       string
    CreatorID        string
    Creator           *ActorResponse
    ParentIssueID     *string        // 父 Issue
    ChildIssues       []IssueResponse // 子 Issues
    AcceptanceCriteria []string       // 验收标准
    ContextRefs       []ContextRef  // 上下文引用
    Position          float64        // 排序位置
    DueDate           *string
    Labels           []LabelResponse
    ProjectID        *string
}
```

### 2.2 Status 状态机

```sql
CHECK (status IN (
    'backlog',      -- 待办池
    'todo',        -- 计划中
    'in_progress',  -- 进行中
    'in_review',   -- 审核中
    'done',       -- 已完成
    'blocked',    -- 被阻塞
    'cancelled'   -- 已取消
))
```

### 2.3 Priority 优先级

```
urgent > high > medium > low > none
```

### 2.4 Handler 函数

文件: `server/internal/handler/issue.go`

```go
func (h *Handler) ListIssues(...)         // 分页列表
func (h *Handler) GetIssue(...)           // 获取详情
func (h *Handler) CreateIssue(...)       // 创建 Issue
func (h *Handler) UpdateIssue(...)       // 更新 Issue
func (h *Handler) DeleteIssue(...)       // 删除 Issue
func (h *Handler) SearchIssues(...)       // 搜索
func (h *Handler) ListChildIssues(...)    // 子 Issue 列表
func (h *Handler) ChildIssueProgress(...)// 子 Issue 进度
func (h *Handler) BatchUpdateIssues(...) // 批量操作
```

### 2.5 过滤参数

List Issues 支持的查询参数：

| 参数 | 说明 |
|------|------|
| status | 状态过滤 |
| priority | 优先级过滤 |
| assignee_id | 指定分配者 |
| assignee_ids | 多个分配者 |
| creator_id | 创建者 |
| project_id | 项目 |
| open_only | 仅未完成 |
| limit/offset | 分页 |

---

## 三、Skill 系统

### 3.1 数据模型

```go
type SkillResponse struct {
    ID          string
    WorkspaceID string
    Name        string
    Description string
    Content    string          // Skill 定义内容
    FileCount  int32           // 文件数
    Visibility string         // "workspace" | "private"
    UpdatedBy   *string       // 最后更新者
    CreatedAt   string
    UpdatedAt  string
}
```

### 3.2 Skill 结构

Skills 存储为文件集合：

```sql
CREATE TABLE skill (
    id UUID PRIMARY KEY,
    workspace_id UUID REFERENCES workspace,
    name TEXT NOT NULL,
    description TEXT,
    content TEXT,           -- 技能定义
    visibility TEXT DEFAULT 'workspace',
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ
);

CREATE TABLE skill_file (
    id UUID PRIMARY KEY,
    skill_id UUID REFERENCES skill,
    path TEXT NOT NULL,
    content TEXT,
    hash TEXT              -- 文件 hash
);
```

### 3.3 Handler 函数

文件: `server/internal/handler/skill.go`

```go
func (h *Handler) ListSkills(...)        // 列出 skills
func (h *Handler) GetSkill(...)      // 获取 skill
func (h *Handler) CreateSkill(...)  // 创建 skill
func (h *Handler) UpdateSkill(...)// 更新 skill
func (h *Handler) DeleteSkill(...)// 删除 skill
func (h *Handler) ImportSkill(...)// 导入 skill
func (h *Handler) ListSkillFiles(...)// 列出 skill 文件
func (h *Handler) UpsertSkillFile(...) // 上传/更新文件
func (h *Handler) DeleteSkillFile(...)// 删除文件
func (h *Handler) ListAgentSkills(...) // Agent 关联的 skills
func (h *Handler) SetAgentSkills(...) // 绑定 skills 到 Agent
```

### 3.4 Skill 使用

Skills 通过两种方式附加到 Agent：

1. **Agent 配置**：在 Agent 创建/编辑时选择 Skills
2. **触发式**：Agent 在执行过程中根据上下文自动使用

---

## 四、Runtime 运行时

### 4.1 Runtime 模式

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| local | 本地 Daemon 执行 | 开发环境、个人机器 |
| cloud | 云端 API | 生产环境、远程机器 |

### 4.2 Daemon 连接

```go
type DaemonConnection struct {
    ID              UUID
    AgentID         UUID
    DaemonID        string         // Daemon 标识
    Status          string        // "connected" | "disconnected"
    LastHeartbeatAt  *time.Time
    RuntimeInfo     json.RawMessage // 运行时信息
}
```

### 4.3 心跳机制

- Daemon 定期发送心跳
- Server 标记连接状态
- 超时断开检测

### 4.4 Handler 函数

文件: `server/internal/handler/runtime.go`

```go
func (h *Handler) GetRuntimeUsage(...)          // 运行���统计
func (h *Handler) GetRuntimeTaskActivity(...) // 任务活动
func (h *Handler) GetWorkspaceUsageByDay(...) // 按日统计
func (h *Handler) GetWorkspaceUsageSummary(...)// 工作区汇总
func (h *Handler) ListAgentRuntimes(...)       // Agent 运行时列表
func (h *Handler) DeleteAgentRuntime(...)      // 删除运行时
```

---

## 五、多工作区支持

### 5.1 数据模型

```go
type WorkspaceResponse struct {
    ID          string
    Name        string
    Slug        string         // URL 友好标识
    Description *string
    Settings    json.RawMessage// 工作区设置
    Role       string         // 当前用户的角色
    Members    []MemberResponse
    Stats      WorkspaceStats
}
```

### 5.2 Member 角色

```sql
CHECK (role IN ('owner', 'admin', 'member'))
```

### 5.3 权限级别

| 角色 | 权限 |
|------|------|
| owner | 完全控制、删除工作区 |
| admin | 管理成员、设置 |
| member | 基本操作 |

### 5.4 工作区隔离

通过以下方式实现隔离：

1. **数据库级**：`workspace_id` 外键约束
2. **查询级**：所有查询带 `WHERE workspace_id = ?`
3. **API 级**：`X-Workspace-Slug` header 路由
4. **WebSocket 级**：按工作区房间分发事件

---

## 六、任务队列 (Agent Task Queue)

### 6.1 任务生命周期

```
queued → dispatched → running → completed/failed
```

### 6.2 数据模型

```go
type AgentTaskResponse struct {
    ID               string
    AgentID          string
    RuntimeID        string
    IssueID          string
    WorkspaceID     string
    Status          string         // 任务状态
    Priority        int32        // 优先级
    DispatchedAt     *string    // 分发时间
    StartedAt        *string    // 开始时间
    CompletedAt     *string    // 完成时间
    Result          any          // 执行结果
    Error           *string     // 错误信息
    PriorSessionID   string      // 同 Issue 的前一个 session
    PriorWorkDir    string      // 同 Issue 的前一个工作目录
    TriggerCommentID *string    // 触发任务的评论
    ChatSessionID   string      // 聊天会话 (非空表示聊天任务)
    ChatMessage    string      // 用户消息
}
```

### 6.3 任务触发

任务可以通过以下方式触发：

1. **Issue 分配**：Agent 被分配到 Issue
2. **评论触发**：在 Issue 下 @agent
3. **手动调度**：手动将 Issue 加入队列

### 6.4 队列管理

```go
func (h *Handler) DispatchTask(...)     // 分发任务到 Agent
func (h *Handler) ClaimTask(...)       // Agent 认领任务
func (h *Handler) ReportTaskResult(...)// Agent 报告结果
func (h *Handler) CancelTask(...)      // 取消任务
```

---

## 七、实时通信

### 7.1 WebSocket 连接

文件: `server/internal/realtime/hub.go`

```go
// 心跳配置
writeWait = 10 * time.Second   // 写入超时
pongWait = 60 * time.Second   // pong 等待
pingPeriod = 54 * time.Second // ping 间隔
```

### 7.2 连接认证

支持的认证方式：

1. **JWT Token**: Authorization header
2. **Workspace Slug**: X-Workspace-Slug header
3. **PAT**: X-API-Token header

### 7.3 工作区房间

每建立一个 WebSocket 连接：
1. 解析工作区 ID
2. 加入该工作区的"房间"
3. 接收该工作区的所有事件

### 7.4 事件类型

常见工作区事件：

| 事件 | 说明 |
|------|------|
| issue:created | Issue 创建 |
| issue:updated | Issue 更新 |
| issue:deleted | Issue 删除 |
| issue:moved | Issue 移动 |
| comment:added | 评论添加 |
| comment:deleted | 评论删除 |
| agent:status | Agent 状态变化 |
| task:dispatched | 任务分发 |
| member:joined | 成员加入 |
| member:left | 成员离开 |

---

## 八、Inbox 通知

### 8.1 收件箱项

```go
type InboxItemResponse struct {
    ID          string
    RecipientType string       // "member" | "agent"
    RecipientID   string
    Type         string        // 通知类型
    Severity    string        // "action_required" | "attention" | "info"
    IssueID     *string
    Title       string
    Body        *string
    Read        bool
    Archived   bool
}
```

### 8.2 Handler 函数

```go
func (h *Handler) ListInbox(...)          // 列表
func (h *Handler) MarkInboxRead(...)     // 标记已读
func (h *Handler) ArchiveInboxItem(...) // 归档
func (h *Handler) CountUnreadInbox(...) // 未读计数
func (h *Handler) MarkAllInboxRead(...) // 全部已读
func (h *Handler) ArchiveAllInbox(...)  // 全部归档
```

---

## 九、Projects 项目分组

### 9.1 数据模型

```go
type ProjectResponse struct {
    ID          string
    WorkspaceID string
    Name       string
    Description *string
    Color      string         // 颜色
    Icon       *string     // 图标
    Priority   *int32       // 排序优先级
    IssueStats ProjectIssueStats
}
```

### 9.2 功能

- Issue 按项目分组
- 项目级统计
- 项目级权限

---

## 十、API 设计原则

### 10.1 RESTful 模式

```
GET    /api/workspaces/{id}/issues       # 列表
GET    /api/workspaces/{id}/issues/{id} # 详情
POST   /api/workspaces/{id}/issues     # 创建
PATCH  /api/workspaces/{id}/issues/{id}# 更新
DELETE /api/workspaces/{id}/issues/{id}# 删除
```

### 10.2 查询参数

- 分页: `limit`, `offset`
- 过滤: `status`, `priority`, `assignee_id`
- 展开: `include_comments`, `include_relations`

### 10.3 响应格式

```json
{
  "issues": [...],
  "total": 100
}
```

### 10.4 错误格式

```json
{
  "error": "error message",
  "code": "ERROR_CODE"
}
```