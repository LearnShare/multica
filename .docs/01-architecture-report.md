# Multica 架构分析报告

## 一、项目概述

Multica 是一个 AI-native 任务管理平台，被称为"像 Linear 一样的工具，但 AI agents 是第一等公民"。支持将多种 AI 编程 agent（Claude Code、Codex、OpenClaw、OpenCode、Hermes、Gemini、Pi、Cursor Agent）转化为团队成员进行任务管理。

## 二、技术栈总览

| 层级 | 技术 |
|------|------|
| 前端 Web | Next.js 16 (App Router) |
| 前端 Desktop | Electron + electron-vite |
| 后端 | Go 1.26+ with Chi router |
| 数据库 | PostgreSQL 17 + pgvector |
| ORM | sqlc |
| 状态管理 | React Query + Zustand |
| 包管理 | pnpm workspaces + Turborepo |
| Real-time | gorilla/websocket |

## 三、目录结构

```
multica/
├── apps/
│   ├── web/                 # Next.js Web 应用
│   │   ├── app/            # App Router 页面
│   │   ├── platform/       # Web 平台特定代码
│   │   └── package.json
│   └── desktop/            # Electron 桌面应用
│       ├── src/main/        # 主进程
│       ├── src/renderer/   # 渲染进程
│       └── package.json
├── packages/
│   ├── core/              # 核心业务逻辑 (零 react-dom)
│   │   ├── api/           # API 客户端
│   │   ├── stores/        # Zustand 状态存储
│   │   ├── hooks/         # React Query hooks
│   │   └── realtime/      # WebSocket 客户端
│   ├── ui/               # 原子 UI 组件
│   ├── views/            # 共享业务视图
│   └── tsconfig/         # 共享 TypeScript 配置
├── server/
│   ├── cmd/
│   │   ├── server/       # Go HTTP 服务器
│   │   ├── multica/      # CLI 工具
│   │   └── migrate/     # 数据库迁移工具
│   ├── internal/
│   │   ├── handler/      # HTTP handlers (40+ 文件)
│   │   ├── service/     # 业务服务
│   │   ├── auth/       # 认证授权
│   │   ├── realtime/   # WebSocket Hub
│   │   ├── events/    # 事件总线
│   │   ├── middleware/# HTTP 中间件
│   │   └── storage/    # 存储层
│   ├── migrations/     # SQL 迁移文件 (50+)
│   └── pkg/db/       # sqlc 生成的代码
├── e2e/               # Playwright E2E 测试
├── docs/               # 项目文档
└── scripts/           # 安装脚本
```

## 四、后端架构 (Go)

### 4.1 核心模块

| 模块 | 职责 |
|------|------|
| handler/ | HTTP 请求处理，40+ 个 handler 文件 |
| service/ | 业务逻辑服务层 |
| auth/ | JWT 认证、OAuth |
| realtime/ | WebSocket 连接管理 |
| events/ | 内存事件总线 |
| middleware/ | HTTP 中间件 |
| storage/ | 文件存储抽象 |

### 4.2 HTTP 路由设计

使用 Chi router 的嵌套路由模式：

```go
r.Route("/api/workspaces", func(r chi.Router) {
    r.Route("/{id}", func(r chi.Router) {
        r.With(middleware.RequireWorkspaceRole...).Get("/", h.GetWorkspace)
        r.Route("/members/{memberId}", ...)
        r.Route("/issues", ...)
        r.Route("/agents", ...)
        r.Route("/skills", ...)
    })
})
```

### 4.3 Handler 响应模式

每个 handler 文件对应一个资源域：

- `agent.go` - Agent CRUD、任务队列
- `issue.go` - Issue 管理、搜索、批量操作
- `workspace.go` - 工作区、成员管理
- `skill.go` - 技能管理
- `runtime.go` - 运行时统计
- `comment.go` - 评论管理
- `project.go` - 项目管理
- `inbox.go` - 通知收件箱
- `pin.go` - 固定功能
- `reaction.go` - 反应/表情

### 4.4 事件驱动

使用内存事件总线 (`events/bus.go`)：

```go
type Event struct {
    Type    string
    Payload any
}

func (b *Bus) Publish(e Event)
func (b *Bus) Subscribe(typePrefix string, handler Handler)
```

## 五、前端架构

### 5.1 Monorepo 包结构

```
@multica/core     # 纯业务逻辑 (stores, hooks, API client)
@multica/ui      # 无业务 UI 组件 (shadcn/Base UI)
@multica/views   # 共享业务视图 (页面级组件)
@multica/web    # Next.js 应用
@multica/desktop# Electron 应用
```

### 5.2 状态管理原则

- **React Query**: 负责所有服务端状态（issues, members, agents, inbox）
- **Zustand**: 负责所有客户端状态（UI 选择、过滤器、草稿、模态状态）
- **WebSocket 事件**: 仅用于失效 Query 缓存，从不直接写入 store

### 5.3 平台桥接

`packages/core/platform/` 提供 `CoreProvider`，初始化：
- API 客户端
- Auth/Workspace stores
- WebSocket 连接
- QueryClient

每个应用提供自己的 `NavigationAdapter` 用于路由。

## 六、数据库设计

### 6.1 核心表

| 表名 | 职责 |
|------|------|
| user | 用户账号 |
| workspace | 工作区 (多租户) |
| member | 用户-工作区关联 (角色) |
| agent | AI Agent 配置 |
| issue | 任务/问题 |
| comment | 评论 |
| issue_label | 标签 |
| project | 项目分组 |
| skill | 可复用技能 |
| inbox_item | 通知 |
| agent_task_queue | Agent 任务队列 |
| daemon_connection | Daemon 连接管理 |

### 6.2 关键关系

- Issue 分配给 Member 或 Agent（多态关联）
- Comment 作者可以是 Member 或 Agent（多态关联）
- Agent 支持 local/cloud 两种 runtime_mode
- Task Queue 管理 Agent 任务生命周期

### 6.3 迁移管理

50+ 个 SQL 迁移文件，按序号排列：
- `001_init.up.sql` - 初始 schema
- `00X_*.up.sql` - 功能迭代迁移

## 七、WebSocket 实时通信

### 7.1 Hub 管理

`server/internal/realtime/hub.go`:
- 管理活跃 WebSocket 连接
- 心跳检测 (60s pong timeout)
- 工作区房间隔离

### 7.2 连接认证

支持多种认证方式：
- JWT Token
- X-Workspace-Slug header
- Personal Access Token

### 7.3 消息类型

- 工作区事件广播 (issue:created, comment:added, etc)
- 客户端订阅特定工作区

## 八、关键架构决策

### 8.1 Internal Packages 模式

所有共享包导出原始 `.ts`/`.tsx` 文件（无预编译），消费应用的 bundler 直接编译。实现零配置 HMR 和即时 go-to-definition。

### 8.2 包边界规则

- `packages/core/`: 零 react-dom、零 localStorage、零 process.env
- `packages/ui/`: 零 @multica/core 导入
- `packages/views/`: 零 next/* 导入、零 react-router 导入

### 8.3 工作区隔离

- 所有查询通过 workspace_id 过滤
- X-Workspace-Slug header 路由请求
- 成员检查门控访问

### 8.4 任务队列模式

Agent 任务的生命周期：
```
queued → dispatched → running → completed/failed
```

## 九、测试策略

| 层级 | 工具 | 位置 |
|------|------|------|
| 单元测试 (TS) | Vitest | 各 package 内 |
| 单元测试 (Go) | go test | server/internal/ |
| E2E | Playwright | e2e/ |

## 十、CI/CD

- Node 22 + Go 1.26.1
- PostgreSQL pgvector/pgvector:pg17
- GitHub Actions
- GoReleaser 多平台构建

## 十一、参考资源

- README.md
- CLAUDE.md (开发者指南)
- CONTRIBUTING.md
- CLI_AND_DAEMON.md