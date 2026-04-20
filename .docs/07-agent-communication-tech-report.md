# Multica Agent 启动与通信机制详细技术报告

## 一、核心问题概述

用户最关心的两个问题：
1. **如何启动一个 Agent？** — 创建会话、分配任务
2. **如何与 Agent 沟通？** — 发送消息、接收回复

---

## 二、启动 Agent 的方式

### 2.1 方式一：创建 Chat 会话 (推荐)

这是最直接的交互方式，创建与 Agent 的一对一对话。

```
POST /api/chat/sessions
{
  "agent_id": "agent-uuid",
  "title": "My Coding Session"
}
```

#### 2.1.1 后端处理流程

```go
// server/internal/handler/chat.go - CreateChatSession
func (h *Handler) CreateChatSession(w http.ResponseWriter, r *http.Request) {
    // 1. 验证用户身份
    userID, ok := requireUserID(w, r)
    if !ok { return }

    // 2. 验证 Agent 存在且未归档
    agent, err := h.Queries.GetAgentInWorkspace(ctx, agentID)
    if agent.ArchivedAt.Valid {
        writeError(w, 400, "agent is archived")
        return
    }

    // 3. 创建 Chat Session 记录
    session, err := h.Queries.CreateChatSession(ctx, db.CreateChatSessionParams{
        WorkspaceID: workspaceID,
        AgentID:    agentID,
        CreatorID:  userID,
        Title:     req.Title,
    })

    // 4. 返回会话
    writeJSON(w, 201, chatSessionToResponse(session))
}
```

#### 2.1.2 数据模型

```sql
CREATE TABLE chat_session (
    id UUID PRIMARY KEY,
    workspace_id UUID REFERENCES workspace,
    agent_id UUID REFERENCES agent,
    creator_id UUID REFERENCES "user",
    title TEXT,
    status TEXT DEFAULT 'active',  -- 'active' | 'archived'
    session_id TEXT,                -- Claude session ID (persist)
    work_dir TEXT,                  -- 工作目录 (persist)
    has_unread BOOLEAN DEFAULT FALSE,
    unread_since TIMESTAMPTZ,
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ
);

CREATE TABLE chat_message (
    id UUID PRIMARY KEY,
    chat_session_id UUID REFERENCES chat_session,
    role TEXT CHECK (role IN ('user', 'assistant')),
    content TEXT NOT NULL,
    task_id UUID REFERENCES agent_task_queue,
    created_at TIMESTAMPTZ
);
```

#### 2.1.3 前端调用

```typescript
// packages/core/chat/mutations.ts
export function useCreateChatSession() {
  return useMutation({
    mutationFn: (data: { agent_id: string; title?: string }) => {
      return api.createChatSession(data);
    },
  });
}

// 使用
const createSession = useCreateChatSession();
createSession.mutate({ agent_id: "agent-uuid", title: "Fix bug #123" });
```

---

### 2.2 方式二：Issue 分配

将 Issue 分配给 Agent，Agent 自动开始处理。

```
POST /api/issues
{
  "title": "Implement login",
  "assignee_id": "agent-uuid"
}
```

#### 2.2.1 自动入队逻辑

```go
// server/internal/handler/issue.go - UpdateIssue
func (h *Handler) UpdateIssue(...) {
    // 当 assignee 变更为 Agent 时
    if newAssigneeType == "agent" {
        // 自动创建任务
        task, err := h.TaskService.EnqueueTaskForIssue(ctx, issue)
    }
}
```

---

### 2.3 方式三：@ Mention 触发

在 Issue 评论中 @ 提及 Agent，触发额外任务。

```
POST /api/issues/{id}/comments
{
  "@agent-name: please review my PR"
}
```

---

## 三、与 Agent 沟通的机制

### 3.1 发送消息流程

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend                              │
├─────────────────────────────────────────────────────────────┤
│  User types message in chat UI                               │
│         ↓                                                  │
│  POST /api/chat/sessions/{id}/messages                     │
│  { "content": "Help me fix this bug" }                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       Server                                │
├─────────────────────────────────────────────────────────────┤
│  1. 验证会话权限                                           │
│  2. 创建 chat_message (role: user)                         │
│  3. EnqueueChatTask() → 创建 agent_task_queue              │
│  4. 返回 task_id                                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       Daemon                                │
├─────────────────────────────────────────────────────────────┤
│  5. Poll 获取任务                                          │
│  6. BuildPrompt() → 构建 Agent prompt                      │
│  7. Run Agent CLI (claude/codex/etc)                      │
│  8. CompleteTask() with output                            │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       Server                                │
├─────────────────────────────────────────────────────────────┤
│  9. 保存 assistant message (role: assistant)               │
│  10. 广播 chat:done WebSocket 事件                         │
│  11. 更新会话 unread 状态                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      Frontend                              │
├─────────────────────────────────────────────────────────────┤
│  12. WebSocket 接收 chat:done                            │
│  13. 更新 UI 显示 Agent 回复                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 消息发送 API

```
POST /api/chat/sessions/{sessionId}/messages
Content-Type: application/json

Request:
{
  "content": "How do I fix the login bug?"
}

Response:
{
  "message_id": "msg-uuid",
  "task_id": "task-uuid"   # 可用于跟踪任务状态
}
```

### 3.3 后端消息处理

```go
// server/internal/handler/chat.go - SendChatMessage
func (h *Handler) SendChatMessage(w http.ResponseWriter, r *http.Request) {
    sessionID := chi.URLParam(r, "sessionId")

    // 1. 验证会话
    session, err := h.Queries.GetChatSessionInWorkspace(ctx, sessionID)
    if session.Status != "active" {
        writeError(w, 400, "chat session is archived")
        return
    }

    // 2. 创建用户消息 (先创建，确保 daemon 能读取)
    msg, err := h.Queries.CreateChatMessage(ctx, db.CreateChatMessageParams{
        ChatSessionID: session.ID,
        Role:          "user",
        Content:       req.Content,
    })

    // 3. 入队 Chat 任务
    task, err := h.TaskService.EnqueueChatTask(ctx, session)

    // 4. 广播用户消息事件
    h.publish(protocol.EventChatMessage, workspaceID, "member", userID, ChatMessagePayload{
        ChatSessionID: sessionID,
        MessageID:     uuidToString(msg.ID),
        Role:          "user",
        Content:       req.Content,
        TaskID:        uuidToString(task.ID),
    })

    writeJSON(w, 201, SendChatMessageResponse{
        MessageID: uuidToString(msg.ID),
        TaskID:    uuidToString(task.ID),
    })
}
```

### 3.4 Daemon 执行

```go
// server/internal/daemon/prompt.go - buildChatPrompt
func buildChatPrompt(task Task) string {
    var b strings.Builder
    b.WriteString("You are running as a chat assistant for a Multica workspace.\n")
    b.WriteString("A user is chatting with you directly. Respond to their message.\n\n")
    fmt.Fprintf(&b, "User message:\n%s\n", task.ChatMessage)
    return b.String()
}
```

### 3.5 Agent 回复处理

```go
// server/internal/service/task.go - CompleteTask
func (s *TaskService) CompleteTask(ctx context.Context, taskID pgtype.UUID, result []byte, sessionID, workDir string) (*db.AgentTaskQueue, error) {
    // ... 执行任务 ...

    // 保存助手回复
    if task.ChatSessionID.Valid {
        var payload protocol.TaskCompletedPayload
        if err := json.Unmarshal(result, &payload); err == nil && payload.Output != "" {
            // 创建助手消息
            _, err := s.Queries.CreateChatMessage(ctx, db.CreateChatMessageParams{
                ChatSessionID: task.ChatSessionID,
                Role:          "assistant",
                Content:       payload.Output,
            })

            // 设置未读标记
            if err := s.Queries.SetUnreadSinceIfNull(ctx, task.ChatSessionID); err != nil {
                slog.Warn("failed to set unread_since", ...)
            }
        }

        // 广播任务完成事件
        s.broadcastChatDone(ctx, task)
    }

    return &task, nil
}
```

---

## 四、实时通信 (WebSocket)

### 4.1 连接建立

```typescript
// packages/core/realtime/use-realtime.ts
import { useRealtimeConnection } from "./use-realtime-connection";

function ChatPage() {
  const ws = useRealtimeConnection(wsId);
  // 自动连接工作区的 WebSocket
}
```

### 4.2 事件监听

| 事件名 | 说明 | Payload |
|--------|------|--------|
| `chat:message` | 新消息 | chat_session_id, message_id, role, content |
| `chat:done` | Agent 完成 | chat_session_id, task_id |
| `chat:session_read` | 会话已读 | chat_session_id |
| `task:progress` | 任务进度 | task_id, summary, step, total |

### 4.3 前端处理

```typescript
// packages/core/realtime/use-realtime-sync.ts
export function useRealtimeChatEvents(wsId: string) {
  const qc = useQueryClient();

  // 监听 Chat 消息
  useRealtimeEvent("chat:message", (payload) => {
    const { chat_session_id, ...message } = payload;

    // 更新消息列表缓存
    qc.setQueryData(chatKeys.messages(wsId, chat_session_id), (old) => [
      ...(old || []),
      message,
    ]);
  });

  // 监听 Agent 完成
  useRealtimeEvent("chat:done", (payload) => {
    const { chat_session_id } = payload;

    // 标记会话为最新
    qc.invalidateQueries({
      queryKey: chatKeys.session(wsId, chat_session_id),
    });
  });
}
```

---

## 五、任务状态追踪

### 5.1 任务查询 API

```
GET /api/chat/sessions/{id}/tasks/pending

Response:
{
  "task": {
    "id": "task-uuid",
    "status": "running"  // queued | running
  }
}
```

### 5.2 WebSocket 事件

```go
// 任务开始
case protocol.EventTaskStarted:
    // 更新 UI: 显示 "Agent is thinking..."

// 任务进度
case protocol.EventTaskProgress:
    // 更新 UI: 显示进度条 / 步骤

// 任务完成
case protocol.EventTaskCompleted:
    // 1. 保存助手消息
    // 2. 广播 chat:done
    // 3. 更新 UI: 显示回复
```

---

## 六、通信协议详解

### 6.1 请求/响应格式

#### 6.1.1 创建会话

```http
POST /api/chat/sessions
Authorization: Bearer <jwt>
X-Workspace-Slug: my-workspace

{
  "agent_id": "agent-uuid-xxxx",
  "title": "Debug session"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "session-uuid-xxxx",
  "workspace_id": "ws-uuid-xxxx",
  "agent_id": "agent-uuid-xxxx",
  "creator_id": "user-uuid-xxxx",
  "title": "Debug session",
  "status": "active",
  "has_unread": false,
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z"
}
```

#### 6.1.2 发送消息

```http
POST /api/chat/sessions/{sessionId}/messages
Authorization: Bearer <jwt>

{
  "content": "How do I fix the login bug?"
}
```

```http
HTTP/1.1 201 Created

{
  "message_id": "msg-uuid-xxxx",
  "task_id": "task-uuid-xxxx"
}
```

#### 6.1.3 获取消息

```http
GET /api/chat/sessions/{sessionId}/messages
```

```http
HTTP/1.1 200 OK

[
  {
    "id": "msg-uuid-1",
    "role": "user",
    "content": "How do I fix the login bug?",
    "created_at": "2024-01-01T00:00:00Z"
  },
  {
    "id": "msg-uuid-2",
    "role": "assistant",
    "content": "Looking at the code, the login issue is caused by...",
    "created_at": "2024-01-01T00:00:05Z"
  }
]
```

---

## 七、关键代码位置

### 7.1 Server (Go)

| 文件 | 职责 |
|------|------|
| `handler/chat.go` | Chat 会话 API |
| `service/task.go` | 任务入队、执行 |
| `daemon/daemon.go` | 任务轮询、执行 |
| `daemon/prompt.go` | Agent Prompt 构建 |

### 7.2 Frontend (TypeScript)

| 文件 | 职责 |
|------|------|
| `packages/core/chat/mutations.ts` | Chat mutations |
| `packages/core/chat/queries.ts` | Chat queries |
| `packages/core/realtime/*` | WebSocket 连接 |

---

## 八、完整交互流程图

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  User    │     │ Frontend │     │  Server  │     │  Daemon  │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │ 1.点击开始      │               │               │
     │───────────────>>│               │               │
     │               │ 2.POST       │               │
     │               │──────────────>>│               │
     │               │               │ 3.创建会话     │
     │               │               │──────────────>>│
     │               │               │ 4.返回会话    │
     │               │               │<<─────────────│
     │ 5.显示会话     │               │               │
     │<<─────────────│               │               │
     │               │               │               │
     │ 6.输入消息     │               │               │
     │───────────────>>│               │               │
     │               │ 7.POST消息    │               │
     │               │──────────────>>│               │
     │               │               │ 8.创建消息+入队│
     │               │               │──────────────>>│
     │               │               │ 9.返回task_id │
     │               │               │<<─────────────│
     │               │ 10.WS事件    │               │
     │               │<<─────────────│               │
     │               │               │               │
     │               │               │ 11.Poll       │
     │               │               │<<─────────────│
     │               │               │ 12.返回任务   │
     │               │               │───────────────>>│
     │               │               │               │ 13.执行Agent │
     │               │               │               │──────────────>>
     │               │               │               │ 14.Run CLI
     │               │               │               │──────────────>>
     │               │               │               │ 15.Output
     │               │               │               │<<─────────────
     │               │ 16.WebSocket  │               │
     │               │ chat:done    │               │
     │               │<<─────────────│               │
     │ 17.显示回复   │               │               │
     │<<─────────────│               │               │
```

---

## 九、API 端点汇总

### 9.1 会话管理

```
POST   /api/chat/sessions              # 创建会话
GET    /api/chat/sessions              # 列表
GET    /api/chat/sessions/{id}        # 详情
DELETE /api/chat/sessions/{id}        # 归档
POST   /api/chat/sessions/{id}/read   # 标记已读
```

### 9.2 消息管理

```
GET    /api/chat/sessions/{id}/messages  # 消息列表
POST   /api/chat/sessions/{id}/messages # 发送消息
```

### 9.3 任务管理

```
GET  /api/chat/sessions/{id}/tasks/pending  # 待处理任务
```

---

## 十、最佳实践

### 10.1 启动 Agent 流程

1. **创建会话**
   ```typescript
   const session = await api.createChatSession({
     agent_id: agent.id,
     title: "Help me with login"
   });
   ```

2. **发送消息**
   ```typescript
   await api.sendChatMessage(session.id, {
     content: "Fix the login bug on line 42"
   });
   ```

3. **监听回复** (WebSocket)
   ```typescript
   ws.subscribe("chat:done", (payload) => {
     if (payload.chat_session_id === session.id) {
       refreshMessages();
     }
   });
   ```

### 10.2 跟踪任务状态

```typescript
// 轮询任务状态
const taskStatus = await api.getTaskStatus(taskId);

// 或监听 WebSocket
ws.subscribe("task:progress", (payload) => {
  updateProgressBar(payload.step, payload.total);
});
```

### 10.3 会话持久化

- `session_id`: Claude session ID，用于恢复对话上下文
- `work_dir`: 工作目录，复用之前的代码变更
- 下次消息自动传递这些信息给 Agent