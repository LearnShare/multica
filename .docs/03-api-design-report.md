# Multica API 设计报告

## 一、API 路由概览

### 1.1 路由分组

Multica API 使用 Chi router 的嵌套路由模式，分为以下主要组：

| 路由组 | 认证方式 | 说明 |
|--------|----------|------|
| `/api/daemon` | Daemon Token | Agent 运行时 API |
| `/api/*` | Auth (JWT) | 用户 API |
| Workspace 路由 | RequireWorkspaceMember | 需要工作区成员身份 |

### 1.2 认证层级

```
1. Daemon Token → /api/daemon/* (仅 daemon)
2. JWT Token → /api/* (用户)
3. Workspace Member → /api/workspaces/{id}/*
```

---

## 二、Daemon API (/api/daemon)

### 2.1 注册与心跳

```
POST /api/daemon/register        # 注册 daemon
POST /api/daemon/deregister      # 注销 daemon
POST /api/daemon/heartbeat      # 心跳
GET  /api/daemon/workspaces/{workspaceId}/repos # 获取工作区仓库
```

### 2.2 任务管理

```
POST /api/daemon/runtimes/{runtimeId}/tasks/claim    # 认领任务
GET  /api/daemon/runtimes/{runtimeId}/tasks/pending # 待处理任务
POST /api/daemon/runtimes/{runtimeId}/ping/{pingId}/result   # 报告 Ping 结果
POST /api/daemon/runtimes/{runtimeId}/update/{updateId}/result # 报告更新结果
```

### 2.3 任务状态

```
GET  /api/daemon/tasks/{taskId}/status       # 获取任务状态
POST /api/daemon/tasks/{taskId}/start       # 开始任务
POST /api/daemon/tasks/{taskId}/progress   # 报告进度
POST /api/daemon/tasks/{taskId}/complete   # 完成任务
POST /api/daemon/tasks/{taskId}/fail      # 任务失败
POST /api/daemon/tasks/{taskId}/usage      # 报告资源使用
POST /api/daemon/tasks/{taskId}/messages   # 上报消息
GET  /api/daemon/tasks/{taskId}/messages   # 获取消息历史
```

---

## 三、用户 API (/api/*)

### 3.1 用户级路由

```
GET  /api/config      # 获取配置
GET  /api/me         # 当前用户
PATCH /api/me       # 更新用户
POST /api/cli-token # 生成 CLI Token
POST /api/upload-file # 上传文件
```

---

## 四、工作区 API (/api/workspaces)

### 4.1 工作区列表与创建

```
GET  /api/workspaces/                   # 列表 (用户所有工作区)
POST /api/workspaces/                  # 创建新工作区
```

### 4.2 工作区详情

```
GET /api/workspaces/{id}               # 获取工作区
PUT /api/workspaces/{id}               # 更新工作区 (admin+)
PATCH /api/workspaces/{id}            # 部分更新 (admin+)
DELETE /api/workspaces/{id}           # 删除工作区 (owner only)
```

### 4.3 成员管理

```
GET  /api/workspaces/{id}/members     # 列出成员
GET  /api/workspaces/{id}/members/{memberId} # 获取成员
POST /api/workspaces/{id}/members  # 创建邀请 (admin+)
PATCH /api/workspaces/{id}/members/{memberId} # 更新成员角色
DELETE /api/workspaces/{id}/members/{memberId} # 删除成员
POST /api/workspaces/{id}/leave    # 离开工作区
```

### 4.4 邀请管理

```
GET  /api/workspaces/{id}/invitations    # 列出邀请 (admin+)
POST /api/workspaces/{id}/invitations   # 创建邀请 (admin+)
DELETE /api/workspaces/{id}/invitations/{invitationId} # 撤销邀请
```

---

## 五、Issue API

### 5.1 Issue CRUD

```
GET    /api/issues/                    # 列表
GET    /api/issues/search              # 搜索
GET    /api/issues/child-progress       # 子 Issue 进度
POST   /api/issues/                    # 创建
POST   /api/issues/batch-update       # 批量更新
POST   /api/issues/batch-delete       # 批量删除
GET    /api/issues/{id}                # 详情
PUT    /api/issues/{id}                # 更新
DELETE /api/issues/{id}                # 删除
```

### 5.2 Issue 评论

```
GET  /api/issues/{id}/comments        # 列表
POST /api/issues/{id}/comments       # 创建
GET  /api/issues/{id}/comments/{commentId} # 详情
DELETE /api/issues/{id}/comments/{commentId} # 删除
```

### 5.3 Issue 关系

```
GET  /api/issues/{id}/relations      # 关系
POST /api/issues/{id}/relations      # 添加关系
DELETE /api/issues/{id}/relations/{relationId} # 删除关系
```

---

## 六、Agent API

### 6.1 Agent CRUD

```
GET    /api/agents/                    # 列表
POST   /api/agents/                   # 创建
GET    /api/agents/{id}               # 详情
PUT    /api/agents/{id}               # 更新
DELETE /api/agents/{id}               # 删除
POST   /api/agents/{id}/archive       # 归档
POST   /api/agents/{id}/unarchive    # 取消归档
GET    /api/agents/{id}/tasks        # Agent 任务历史
```

---

## 七、Skill API

### 7.1 Skill 管理

```
GET    /api/skills/                   # 列表
POST   /api/skills/                   # 创建
GET    /api/skills/{id}                # 详情
PUT    /api/skills/{id}               # 更新
DELETE /api/skills/{id}               # 删除
POST   /api/skills/import             # 导入
GET    /api/skills/{id}/files         # 文件列表
POST   /api/skills/{id}/files         # 上传文件
DELETE /api/skills/{id}/files/{fileId} # 删除文件
```

### 7.2 Agent 关联

```
GET  /api/agents/{id}/skills         # 获取 Agent 的 skills
PUT  /api/agents/{id}/skills        # 设置 Agent 的 skills
```

---

## 八、Project API

```
GET    /api/projects/                # 列表
POST   /api/projects/               # 创建
GET    /api/projects/{id}           # 详情
PUT    /api/projects/{id}           # 更新
DELETE /api/projects/{id}           # 删除
GET    /api/projects/search         # 搜索
```

---

## 九、其他 API

### 9.1 Runtime 使用统计

```
GET  /api/usage          # 使用统计
GET  /api/runtimes      # 运行时列表
GET  /api/runtimes/{runtimeId}/usage # 运行时使用详情
```

### 9.2 Inbox 通知

```
GET    /api/inbox/                   # 列表
GET    /api/inbox/unread-count      # 未读计数
POST   /api/inbox/{id}/read         # 标记已读
POST   /api/inbox/{id}/archive     # 归档
POST   /api/inbox/read-all         # 全部已读
POST   /api/inbox/archive-all     # 全部归档
```

### 9.3 Pin 固定

```
GET    /api/pins/                   # 列表
POST   /api/pins/                   # 添加
DELETE /api/pins/{id}               # 删除
PUT    /api/pins/reorder            # 重新排序
```

### 9.4 Reactions 表情

```
POST   /api/comments/{id}/reactions       # 添加
DELETE /api/comments/{id}/reactions/{id} # 删除
POST   /api/issues/{id}/reactions        # 添加
DELETE /api/issues/{id}/reactions/{id} # 删除
```

### 9.5 Autopilot

```
GET    /api/autopilots/              # 列表
POST   /api/autopilots/             # 创建
GET    /api/autopilots/{id}         # 详情
PUT    /api/autopilots/{id}         # 更新
DELETE /api/autopilots/{id}         # 删除
GET    /api/autopilots/{id}/triggers # 触发器
POST   /api/autopilots/{id}/test    # 测试
```

### 9.6 Chat

```
GET  /api/chat/sessions/             # 列表
POST /api/chat/sessions/            # 创建
GET  /api/chat/sessions/{id}        # 详情
DELETE /api/chat/sessions/{id}      # 删除
GET  /api/chat/sessions/{id}/messages # 消息
POST /api/chat/sessions/{id}/messages # 发送消息
```

---

## 十、查询参数规范

### 10.1 分页参数

| 参数 | 默认值 | 最大值 | 说明 |
|------|--------|--------|------|
| limit | 100 | 1000 | 每页数量 |
| offset | 0 | - | 偏移量 |

### 10.2 过滤参数

Issue 列表支持：

```
?status=in_progress
?priority=high,urgent
?assignee_id=xxx
?assignee_ids=xxx,yyy
?creator_id=xxx
?project_id=xxx
?open_only=true
```

### 10.3 其他参数

```
?include_archived=true   # 包含已归档
?include_comments=true   # 包含评论
```

---

## 十一、响应格式

### 11.1 列表响应

```json
{
  "issues": [...],
  "total": 100
}
```

### 11.2 详情响应

```json
{
  "id": "xxx",
  "name": "xxx",
  ...
}
```

### 11.3 错误响应

```json
{
  "error": "Error message",
  "code": "ERROR_CODE"
}
```

### 11.4 状态码

| 状态码 | 说明 |
|--------|------|
| 200 | 成功 |
| 201 | 创建成功 |
| 400 | 请求错误 |
| 401 | 未认证 |
| 403 | 无权限 |
| 404 | 未找到 |
| 500 | 服务器错误 |

---

## 中间件链

### 12.1 Auth 中间件

```go
middleware.DaemonAuth(queries)    // Daemon Token
middleware.Auth(queries)         // JWT + 刷新 Cookie
middleware.RequireWorkspaceMember(queries) // 工作区成员
middleware.RequireWorkspaceRole(queries, "owner", "admin") // 角色检查
```

### 12.2 请求流程

```
1. 请求进入
2. DaemonAuth OR Auth 验证身份
3. RequireWorkspaceMember 检查成员
4. RequireWorkspaceRole (admin+ 路由)
5. Handler 处理
6. 事件发布 → WebSocket 广播
```