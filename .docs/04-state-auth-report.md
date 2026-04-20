# Multica 状态管理与认证授权报告

## 一、状态管理架构

### 1.1 状态分层

Multica 使用严格的状态分层：

| 状态类型 | 管理工具 | 位置 | 说明 |
|----------|----------|------|------|
| 服务端状态 | React Query | packages/core/*/queries.ts | API 数据 |
| 客户端状态 | Zustand | packages/core/*/store.ts | UI 状态 |
| 路由状态 | 框架特定 | apps/*/platform/ | 导航 |

### 1.2 核心模块

```
packages/core/
├── api/           # API 客户端 (Axios)
├── auth/          # 认证存储
├── workspace/     # 工作区存储 + hooks
├── issues/       # Issue Query + mutations
├── agents/        # Agent Query + mutations
├── projects/     # Project Query + mutations
├── inbox/        # Inbox Query + mutations
├── realtime/     # WebSocket 客户端
└── storage/      # 持久化存储
```

---

## 二、React Query 使用

### 2.1 Query 模式

```typescript
// packages/core/issues/queries.ts
import { createQueryOptions } from "@tanstack/react-query";

export const issueListOptions = (wsId: string) =>
  createQueryOptions({
    queryKey: ["issues", wsId],
    queryFn: () => api.listIssues(wsId),
  });

export const issueDetailOptions = (wsId: string, issueId: string) =>
  createQueryOptions({
    queryKey: ["issues", wsId, issueId],
    queryFn: () => api.getIssue(wsId, issueId),
  });
```

### 2.2 Mutation 模式

```typescript
// packages/core/issues/mutations.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";

export const useCreateIssue = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: api.createIssue,
    onSuccess: (_, variables) => {
      qc.invalidateQueries({ queryKey: ["issues", variables.workspaceId] });
    },
  });
};
```

### 2.3 失效策略

- **乐观更新**：先更新本地，发送请求，失败则回滚
- **后置失效**：成功后在 `onSettled` 时失效
- **ws 事件失效**：WS 事件触发失效，从不直接写入 store

---

## 三、Zustand 使用

### 3.1 Store 结构

```typescript
// packages/core/workspace/store.ts - 创建
import { create } from "zustand";

interface WorkspaceState {
  currentWorkspaceId: string | null;
  setCurrentWorkspace: (id: string | null) => void;
}

export const useWorkspaceStore = create<WorkspaceState>((set) => ({
  currentWorkspaceId: null,
  setCurrentWorkspace: (id) => set({ currentWorkspaceId: id }),
}));
```

### 3.2 选择器最佳实践

```typescript
// ❌ 错误 - 每次返回新对象
const { issues, filters } = useStore((s) => ({
  issues: s.issues,
  filters: s.filters,
}));

// ✅ 正确 - 分别选择
const issues = useStore((s) => s.issues);
const filters = useStore((s) => s.filters);

// ✅ 正确 - shallow 比较
import { shallow } from "zustand/shallow";
const { issues, filters } = useStore(
  (s) => ({ issues: s.issues, filters: s.filters }),
  shallow
);
```

### 3.3 持久化

```typescript
// packages/core/platform/persist-storage.ts
import { persist, createJSONStorage } from "zustand/middleware";

export const useSomeStore = create(
  persist(
    (set) => ({
      // state
    }),
    {
      name: "some-storage-key", // localStorage key
      storage: createJSONStorage(() => localStorage),
    }
  )
);
```

---

## 四、WebSocket 与状态同步

### 4.1 Realtime 连接

```typescript
// packages/core/realtime/use-realtime-sync.ts
import { useQueryClient } from "@tanstack/react-query";
import { useRealtimeEvent } from "./use-realtime-event";

export function useRealtimeSync(wsId: string) {
  const qc = useQueryClient();

  // 监听工作区事件，自动失效相关 Query
  useRealtimeEvent("issue:*", () => {
    qc.invalidateQueries({ queryKey: ["issues", wsId] });
  });

  useRealtimeEvent("comment:*", () => {
    qc.invalidateQueries({ queryKey: ["comments", wsId] });
  });
}
```

### 4.2 事件类型

```
issue:created   → invalidate "issues" queries
issue:updated → invalidate "issues" queries
issue:deleted → invalidate "issues" queries
comment:added → invalidate "comments" queries
member:joined → invalidate "members" queries
```

---

## 五、认证授权

### 5.1 JWT Token

```
Authorization: Bearer <jwt_token>
```

### 5.2 Token 包含

```go
// server/internal/auth/jwt.go
type Claims struct {
    UserID      string `json:"sub"`
    Email      string `json:"email"`
    Name       string `json:"name"`
    // 自定义声明
    WorkspaceID string `json:"ws_id,omitempty"`
    Role      string `json:"role,omitempty"`
}
```

### 5.3 中间件链

```go
// server/cmd/server/router.go
r.Use(middleware.DaemonAuth(queries))  // Daemon
r.Use(middleware.Auth(queries))       // User
r.Use(middleware.RequireWorkspaceMember(queries))
```

### 5.4 Cookie 刷新

```go
// server/internal/middleware/auth.go
func Auth(queries *db.Queries) func(http.Handler) http.Handler {
    return func(h http.Handler) http.Handler {
        return http.HandlerFunc(func(w, r) {
            // 1. 检查 Authorization header
            // 2. 或检查 cookie
            // 3. 验证 JWT
            // 4. 刷新 CloudFront cookie (如果需要)
        })
    }
}
```

---

## 六、Workspace 隔离

### 6.1 Header 路由

```
X-Workspace-Slug: my-workspace
```

### 6.2 查询过滤

```go
// 所有 workspace-scoped 查询
WHERE workspace_id = $1
```

### 6.3 API 客户端

```typescript
// packages/core/api/client.ts
export const api = createApiClient({
  baseURL: config.apiUrl,
  headers: {
    "X-Workspace-Slug": currentWorkspaceSlug,
  },
});
```

---

## 七、Personal Access Token

### 7.1 PAT 创建

```
POST /api/tokens
{
  "name": "My CLI Token",
  "expires_at": "2024-12-31"
}
```

### 7.2 PAT 使用

```
X-API-Token: <pat>
```

### 7.3 PAT 撤销

```
DELETE /api/tokens/{id}
```

---

## 八、CLI Token

### 8.1 Token 生成

```
POST /api/cli-token
```

返回:

```json
{
  "token": "mc_<uuid>",
  "expires_at": "2024-01-01T00:00:00Z"
}
```

### 8.2 Daemon 认证

```go
// server/internal/middleware/middleware.go
func DaemonAuth(queries *db.Queries) func(http.Handler) http.Handler {
    return func(h http.Handler) http.Handler {
        return http.HandlerFunc(func(w, r) {
            token := r.Header.Get("X-Daemon-Token")
            // 验证 daemon token
        })
    }
}
```

---

## 九、跨域与安全

### 9.1 CORS

```go
// 生产环境配置
ALLOWED_ORIGINS=https://app.multica.ai,https://my.custom.com
```

### 9.2 WebSocket 源检查

```go
// server/internal/realtime/hub.go
func checkOrigin(r *http.Request) bool {
    origin := r.Header.Get("Origin")
    // 检查是否在允许列表
}
```

---

## 十、核心存储模块

### 10.1 认证存储

```typescript
// packages/core/auth/store.ts
interface AuthStore {
  user: User | null;
  isAuthenticated: boolean;
  login: (user: User) => void;
  logout: () => void;
}
```

### 10.2 工作区存储

```typescript
// packages/core/workspace/store.ts
interface WorkspaceStore {
  currentWorkspaceSlug: string | null;
  currentWorkspaceId: string | null;
  setCurrentWorkspace: (slug: string, id: string) => void;
}
```

### 10.3 模态存储

```typescript
// packages/core/modals/store.ts
interface ModalStore {
  createIssueModal: boolean;
  createProjectModal: boolean;
  // ...
  openModal: (modal: string) => void;
  closeModal: (modal: string) => void;
}
```

---

## 十一、错误处理

### 11.1 Query 错误

```typescript
const { isError, error } = useQuery(issueListOptions(wsId));

if (isError) {
  toast.error(error.message);
}
```

### 11.2 Mutation 错误

```typescript
const createIssue = useCreateIssue({
  onError: (error) => {
    toast.error(error.message);
    qc.setQueryData(["issues", wsId], oldData); // 回滚
  },
});
```

---

## 十二、测试模式

### 12.1 Query Mock

```typescript
// vitest setup
vi.mocked(api.listIssues).mockResolvedValue([...]);
```

### 12.2 Store Mock

```typescript
// vitest setup
vi.mocked(useWorkspaceStore.getState).mockReturnValue({
  currentWorkspaceId: "xxx",
});
```