# Multica 多工作区支持研究报告

## 一、工作区模型

### 1.1 数据结构

```go
type WorkspaceResponse struct {
    ID          string      // UUID
    Name        string      // 工作区名称
    Slug        string      // URL 友好标识
    Description *string    // 描述
    Context     *string    // 上下文描述
    Settings    any         // JSON 配置
    Repos       any         // 关联仓库
    IssuePrefix string      // Issue 编号前缀
    CreatedAt   string
    UpdatedAt   string
}
```

### 1.2 Slug 规则

```go
var workspaceSlugPattern = regexp.MustCompile(`^[a-z0-9]+(?:-[a-z0-9]+)*$`)
```

规则：
- 仅小写字母、数字、连字符
- 不能以连字符开头或结尾
- 示例：`my-workspace`, `team-alpha`

---

## 二、成员管理

### 2.1 角色系统

```sql
CHECK (role IN ('owner', 'admin', 'member'))
```

| 角色 | 创建工作区 | 管理成员 | 管理设置 | 删除工作区 |
|------|-----------|----------|----------|------------|
| owner | ✅ | ✅ | ✅ | ✅ |
| admin | ❌ | ✅ | ✅ | ❌ |
| member | ❌ | ❌ | ❌ | ❌ |

### 2.2 成员关系

```sql
CREATE TABLE member (
    id UUID PRIMARY KEY,
    workspace_id UUID REFERENCES workspace(id) ON DELETE CASCADE,
    user_id UUID REFERENCES "user"(id) ON DELETE CASCADE,
    role TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'member')),
    created_at TIMESTAMPTZ,
    UNIQUE(workspace_id, user_id)
);
```

---

## 三、权限控制

### 3.1 中间件层级

```
Auth (验证用户)
    ↓
RequireWorkspaceMember (验证成员身份)
    ↓
RequireWorkspaceRole (验证角色权限)
```

### 3.2 成员检查

```go
// server/internal/handler/handler.go
func (h *Handler) requireWorkspaceMember(
    w http.ResponseWriter,
    r *http.Request,
    workspaceID,
    notFoundMsg string,
) (db.Member, bool) {
    // 查询 member 表
    // 不存在则返回 403
}
```

### 3.3 角色检查

```go
func (h *Handler) requireWorkspaceRole(
    w http.ResponseWriter,
    r *http.Request,
    workspaceID,
    notFoundMsg string,
    roles ...string,
) (db.Member, bool) {
    // 检查成员角色是否在 roles 中
}
```

---

## 四、邀请系统

### 4.1 邀请流程

```sql
CREATE TABLE workspace_invitation (
    id UUID PRIMARY KEY,
    workspace_id UUID,
    email TEXT NOT NULL,
    role TEXT NOT NULL,
    invited_by UUID,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ,
    accepted_at TIMESTAMPTZ
);
```

### 4.2 API 端点

```
POST   /api/workspaces/{id}/invitations     # 创建邀请 (admin+)
GET    /api/workspaces/{id}/invitations     # 列表 (admin+)
DELETE /api/workspaces/{id}/invitations/{id} # 撤销 (admin+)

GET    /api/invitations                     # 我的邀请
GET    /api/invitations/{id}               # 详情
POST   /api/invitations/{id}/accept        # 接受
POST   /api/invitations/{id}/decline       # 拒绝
```

---

## 五、工作区隔离机制

### 5.1 URL 路由隔离

```
/:slug/issues         # slug = workspace slug
/:slug/settings
/:slug/agents
```

### 5.2 Header 隔离

```
X-Workspace-Slug: my-workspace
```

### 5.3 数据库隔离

```sql
-- 所有 workspace-scoped 查询
SELECT * FROM issue WHERE workspace_id = $1;
SELECT * FROM agent WHERE workspace_id = $1;
```

### 5.4 WebSocket 隔离

```go
// 连接时指定工作区
ws := hub.Connect(workspaceID)
// 事件仅广播到对应工作区的客户端
hub.BroadcastToWorkspace(workspaceID, event)
```

---

## 六、Issue 前缀

### 6.1 自动生成

```go
func generateIssuePrefix(name string) string {
    // 从名称提取字母，转大写，取前3个
    // "My Team" → "MYT"
    // "Team Alpha" → "TEA"
}
```

### 6.2 唯一性

- Issue 编号格式：`{PREFIX}-{序号}`
- 例如：`MYT-1`, `MYT-2`, `TEA-1`

---

## 七、工作区设置

### 7.1 Settings JSON

```json
{
  "default_assignee": null,
  "auto_close_completed": true,
  "features": {
    "chat": true,
    "autopilot": true
  }
}
```

### 7.2 Repos 配置

```json
{
  "repos": [
    {
      "url": "https://github.com/org/repo",
      "name": "repo",
      "branch": "main"
    }
  ]
}
```

---

## 八、前端实现

### 8.1 工作区切换

```typescript
// packages/core/workspace/store.ts
interface WorkspaceStore {
  currentWorkspaceSlug: string | null;
  currentWorkspaceId: string | null;
  setCurrentWorkspace: (slug: string, id: string) => void;
}
```

### 8.2 API 客户端

```typescript
// 每个请求自动携带工作区
export const api = createApiClient({
  headers: {
    "X-Workspace-Slug": workspaceSlug,
  },
});
```

### 8.3 路由参数

```tsx
// Web: apps/web/app/[slug]/issues/page.tsx
const { slug } = useParams();
const wsId = useWorkspaceIdFromSlug(slug);

// Desktop: 路由自动从 store 获取
```

---

## 九、离开/删除工作区

### 9.1 离开工作区

```
POST /api/workspaces/{id}/leave
```

规则：
- Owner 不能离开，只能删除
- 离开后自动从所有 Issue 取消分配

### 9.2 删除工作区

```
DELETE /api/workspaces/{id}
```

规则：
- 仅 Owner 可删除
- 级联删除所有关联数据

---

## 十、Reserved Slugs

### 10.1 保留 Slug

项目保留一些 slug 防止与路由冲突：

```sql
CREATE TABLE reserved_slug (
    slug TEXT PRIMARY KEY,
    reason TEXT
);

-- 预置保留
INSERT INTO reserved_slug VALUES ('new', '冲突路由');
INSERT INTO reserved_slug VALUES ('settings', '内置页面');
```

### 10.2 创建时检查

```go
func (h *Handler) CreateWorkspace(...) {
    // 检查 slug 是否已保留
    if isReservedSlug(slug) {
        writeError(w, 400, "slug is reserved")
        return
    }
}
```

---

## 十一、最佳实践

### 11.1 工作区创建流程

1. 用户提交 name
2. 自动生成 slug（可自定义）
3. 检查 slug 可用性
4. 创建工作区 + 成员记录（owner）

### 11.2 工作区切换

1. 获取 slug 列表
2. 切换时：
   - 更新 URL
   - 更新 store
   - 清空/失效相关 Query
   - 重连 WebSocket