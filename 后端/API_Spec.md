
# ClawLoops 平台 MVP 统一总接口（Authentik 接入版，运行时冻结修订）

供前后端、平台服务、Orchestrator 与 Runtime Manager 在“官方 Authentik 首版接入 + Runtime V1 冻结”前提下统一联调使用。

| 文档定位 | 接口与字段基线 |
| --- | --- |
| 适用范围 | 用户侧 / 管理员侧 / 公开 invitation 入口 / 内部服务侧 / Runtime Manager 内部接口 |
| 修订重点 | 冻结 invitation 生命周期、统一 token 语义、补齐 runtime V1 contract、收紧字段命名与跳转口径 |
| 响应原则 | 所有响应均采用 JSON；字段名直接作为开发基线，不再自行改名 |
| 当前版本 | v0.8-authentik-runtime-frozen |

---

## 1. 通用响应约定

| 项 | 说明 |
| --- | --- |
| 成功状态码 | 读取接口默认 `200`；创建默认 `201`；用户侧 / 管理员侧异步动作默认 `202` |
| 错误体 | 统一至少包含 `code` 与 `message` |
| 异步任务 | **仅 Orchestrator 对外层** runtime 启停删返回 `taskId`；RuntimeManager internal 接口**同步执行**，不返回 `taskId` |
| 权限约定 | 用户侧接口依赖当前登录用户；管理员侧接口仅 `admin`；internal 接口仅服务间访问 |
| internal 鉴权 | internal API 必须通过服务间鉴权（如 mTLS 或 internal token），并且禁止公网访问 |
| disabled 语义 | 除 `/api/v1/auth/me` 外，disabled 用户访问业务接口统一返回 `403 USER_DISABLED`；但 `/api/v1/auth/access` 永远返回 `200`，仅用于状态判断 |
| workspace 访问 | `browserUrl` 仅是受保护入口地址；所有 workspace 子域名必须统一经过 Traefik + Authentik Forward Auth |
| 跳转规则 | `admin` 登录后默认进入 `/admin`；`/admin` 使用独立首页摘要接口；非管理员用户只有在 `ready=true` 时才允许跳转到 `browserUrl`；知道 URL 不代表可访问 |
| 字段命名 | 以本文件“字段冻结清单”为唯一基线，禁止别名漂移 |

---

## 2. 统一错误码

| HTTP | code | 用途 |
| --- | --- | --- |
| 401 | `UNAUTHENTICATED` | 未登录或会话无效 |
| 403 | `ACCESS_DENIED` | 权限不足 |
| 403 | `USER_DISABLED` | 用户已禁用 |
| 404 | `USER_NOT_FOUND` | 用户不存在 |
| 404 | `RUNTIME_NOT_FOUND` | 仅用于业务真相层 runtime 不存在；**不用于 RM 容器事实查询** |
| 404 | `INVITATION_NOT_FOUND` | invitation 不存在 |
| 404 | `MODEL_NOT_FOUND` | 模型不存在 |
| 409 | `RUNTIME_ACTION_CONFLICT` | runtime 正忙、状态冲突，或同一 `runtimeId` 命中多个容器 |
| 409 | `RUNTIME_CONTRACT_DRIFT` | 已有 runtime 容器与冻结 contract 不一致 |
| 409 | `INVITATION_ALREADY_CONSUMED` | invitation 已消费 |
| 409 | `INVITATION_REVOKED` | invitation 已撤销 |
| 410 | `INVITATION_EXPIRED` | invitation 已过期（由 `expiresAt < now` 推导，不单独落库存状态） |
| 422 | `INVITATION_EMAIL_MISMATCH` | 当前认证邮箱与 invitation `targetEmail` 不匹配 |
| 422 | `INVITATION_WORKSPACE_INVALID` | invitation 指向的 workspace 无效 |
| 422 | `QUOTA_EXCEEDED` | 超出 quota |
| 500/502 | `INVITATION_ERROR` | invitation 流程执行失败或上游身份流程失败 |
| 500/502 | `USER_SYNC_ERROR` | 用户同步失败 |
| 500/502 | `RUNTIME_START_FAILED` | runtime 创建、权限初始化或启动探测失败 |
| 500 | `RUNTIME_STOP_FAILED` | runtime 停止失败 |
| 500 | `RUNTIME_DELETE_FAILED` | runtime 删除或目录清理失败 |
| 500/502 | `*_ERROR` | 其他内部模块或上游错误 |

---

## 3. 首版冻结规则（接口层必须统一）

### 3.1 管理员初始化口径

- 首版官方初始化管理员按默认 **`akadmin`** 处理
- 业务上视为管理员角色账号
- 不再在文档中混用“首个管理员是 `admin`”与“首个管理员是 `akadmin`”两种表达

### 3.2 invitation 双层模型与延迟创建

- **ClawLoops token 是业务入口真相**
- **Authentik `itoken` 是身份执行入口**
- `/invite/{token}`、`/api/v1/public/invitations/{token}`、`/api/v1/public/invitations/{token}/start` 只接受平台 token
- `itoken` 只应出现在 Authentik enrollment URL 中，不对外作为业务主 token
- 首版统一采用 **延迟创建模式**：创建 invitation 时只生成 ClawLoops 业务 token；用户调用 `/start` 时再创建或换取 Authentik invitation / enrollment URL
- 首版禁止混用“提前创建”和“延迟创建”两种模式

### 3.3 invitation 状态真相

- 平台 invitation 状态是最终业务真相
- 首版只存 `pending / consumed / revoked`
- `expired` 不入库，统一通过 `expiresAt < now` 派生
- `revoked / expired / consumed` 必须先由平台校验
- 即使身份侧 token 仍有效，平台也必须以平台状态阻断 `start` 或 `post-login`

### 3.4 邮箱校验与首次接入流程

- 首版统一为 **强邮箱校验**
- 校验发生在 `POST /api/v1/auth/post-login` 阶段，即已经拿到当前 `subjectId` 与认证邮箱之后
- 当前认证邮箱必须匹配 `targetEmail`，否则返回 `INVITATION_EMAIL_MISMATCH`
- 首版首次接入流程固定为：**用户通过 invitation 链接进入 enrollment flow，在 flow 内直接设置密码，并由 Authentik 自动登录**
- “先 magic link 进入、之后再强制改密”不作为首版实现

### 3.5 平台密码禁区

在契约、设计、接口三份文档统一禁止：

- ClawLoops 不保存密码
- ClawLoops 不生成正式临时密码
- ClawLoops 不提供密码落库接口
- ClawLoops 不实现独立改密 API
- 所有密码设置、重置、修改统一走 Authentik Flow

### 3.6 幂等要求

- `POST /api/v1/auth/post-login` 必须幂等
- 同一 `invitationId + userId` 只能成功消费一次
- 重复调用返回已消费 / 已绑定结果
- `consume invitation` 与 `workspace membership binding` 必须原子完成，或定义清晰补偿逻辑
- `POST /api/v1/public/invitations/{token}/start` 也是幂等操作；可重复调用，但同一浏览器会话只保留一个有效 pending invitation 会话
- pending invitation session 必须具备 TTL（建议 10–30 分钟）且仅绑定当前浏览器会话

### 3.7 runtime 与跳转语义

- `task.status` = 操作生命周期
- `observedState` = 资源状态
- `ready` = 最终可访问状态
- `admin` 登录后默认进入 `/admin`
- `/admin` 首屏数据由 `GET /api/v1/admin/home` 提供
- 非管理员用户的工作区跳转只看 `ready`
- `workspace-entry` 是唯一工作区跳转入口接口
- `runtime/status` 仅用于状态展示，不作为最终跳转依据

### 3.8 runtime V1 冻结补充

- `runtimeId` 在平台范围内**全局唯一**
- 用户侧 runtime 启停删是 **Orchestrator 异步任务**
- RM internal 接口是 **同步执行器**，立即返回当前 `observedState`
- V1 runtime 镜像固定为  
  `ghcr.io/openclaw/openclaw@sha256:a5a4c83b773aca85a8ba99cf155f09afa33946c0aa5cc6a9ccb6162738b5da02`
- RM internal 请求体**删除 `imageRef`**
- `compat.openclawConfigDir / compat.openclawWorkspaceDir` 是 `ensure-running` **必填**
- 统一共享网络为 `clawloops_shared`
- `internalEndpoint` 固定为 `http://rt-<runtimeId>:18789`
- `18789` 是唯一必检端口；`18790` 仅兼容保留
- `routeHost` 在 RM 中只作 label 追踪，不参与关键 drift 判定

---

## 4. 接口总览（冻结后）

| 方法 | 路径 | 用途 | 权限 |
| --- | --- | --- | --- |
| GET | `/api/v1/auth/me` | 获取当前登录用户 | 用户 |
| GET | `/api/v1/auth/access` | 检查当前用户是否可访问业务（永远返回 200） | 用户 |
| GET | `/api/v1/auth/options` | 获取当前登录方式与首版能力开关 | 公开 |
| POST | `/api/v1/auth/post-login` | Authentik 登录后收口入口（幂等） | 用户 |
| GET | `/api/v1/public/invitations/{token}` | 查看 invitation 预览信息 | 公开 |
| POST | `/api/v1/public/invitations/{token}/start` | 启动 invitation 接入流程（幂等） | 公开 |
| GET | `/api/v1/users/me/quota` | 获取当前用户 quota | 用户 |
| GET | `/api/v1/users/me/runtime` | 获取当前用户 runtime binding（完整对象） | 用户 |
| GET | `/api/v1/users/me/runtime/status` | 获取当前用户 runtime 轻量状态投影 | 用户 |
| POST | `/api/v1/users/me/runtime/start` | 启动或创建 runtime | 用户 |
| POST | `/api/v1/users/me/runtime/stop` | 停止 runtime | 用户 |
| POST | `/api/v1/users/me/runtime/delete` | 删除 runtime | 用户 |
| GET | `/api/v1/runtime/tasks/{taskId}` | 查询 runtime 任务状态 | 用户 / admin |
| GET | `/api/v1/models` | 获取当前用户可见模型列表（只读） | 用户 |
| GET | `/api/v1/workspace-entry` | 获取当前用户工作区入口（唯一工作区跳转入口） | 用户 |
| GET | `/api/v1/admin/home` | 获取管理后台首页摘要与待处理事项 | admin |
| GET | `/api/v1/admin/users` | 获取用户列表 | admin |
| GET | `/api/v1/admin/users/{userId}` | 获取用户详情 | admin |
| PATCH | `/api/v1/admin/users/{userId}/status` | 启用 / 禁用用户 | admin |
| GET | `/api/v1/admin/users/{userId}/runtime` | 获取指定用户 runtime 详情 | admin |
| GET | `/api/v1/admin/invitations` | 获取 invitation 列表 | admin |
| POST | `/api/v1/admin/invitations` | 创建 invitation | admin |
| GET | `/api/v1/admin/invitations/{invitationId}` | 获取 invitation 详情 | admin |
| POST | `/api/v1/admin/invitations/{invitationId}/revoke` | 撤销 invitation | admin |
| POST | `/api/v1/admin/invitations/{invitationId}/resend` | 重发 invitation | admin |
| GET | `/api/v1/admin/models` | 获取全局模型列表与策略 | admin |
| PUT | `/api/v1/admin/models/{modelId}` | 更新模型开关、可见性与默认策略 | admin |
| GET | `/api/v1/admin/provider-credentials` | 获取平台 provider 凭据列表与状态 | admin |
| POST | `/api/v1/admin/provider-credentials` | 新增平台 provider 凭据 | admin |
| POST | `/api/v1/admin/provider-credentials/{credentialId}/verify` | 校验平台 provider 凭据 | admin |
| DELETE | `/api/v1/admin/provider-credentials/{credentialId}` | 删除平台 provider 凭据 | admin |
| GET | `/api/v1/admin/usage/summary` | 获取全局 usage 汇总 | admin |
| POST | `/internal/users/sync` | 首次登录同步 / 创建用户 | internal |
| POST | `/internal/users/{userId}/runtime-binding/ensure` | 确保 runtime binding 存在 | internal |
| PUT | `/internal/users/{userId}/runtime-binding` | 创建或更新 runtime binding | internal |
| PATCH | `/internal/users/{userId}/runtime-binding/state` | 更新 runtime 状态 | internal |
| POST | `/internal/invitations` | 创建 invitation 真相对象 | internal |
| POST | `/internal/invitations/{invitationId}/consume` | 消费 invitation 并完成业务绑定（幂等） | internal |
| POST | `/internal/invitations/{invitationId}/revoke` | 撤销 invitation | internal |
| GET | `/internal/model-config/users/{userId}` | 获取运行时模型配置 | internal |
| POST | `/internal/usage/records` | 接收 OpenClaw usage 上报 | internal |
| POST | `/internal/runtime-manager/containers/ensure-running` | 确保容器运行（RM 同步接口） | internal |
| POST | `/internal/runtime-manager/containers/stop` | 停止容器（RM 同步接口） | internal |
| POST | `/internal/runtime-manager/containers/delete` | 删除容器（RM 同步接口） | internal |
| GET | `/internal/runtime-manager/containers/{runtimeId}` | 查询容器状态（RM 同步接口） | internal |

---

## 5. 认证相关接口

### 5.1 获取当前登录用户

GET `/api/v1/auth/me`

**示例**：

```json
{
  "authenticated": true,
  "user": {
    "userId": "u_001",
    "subjectId": "authentik:12345",
    "tenantId": "t_default",
    "role": "admin",
    "status": "active",
    "auth": {
      "provider": "authentik",
      "method": "local_password"
    },
    "isAdmin": true,
    "isDisabled": false
  }
}
```

### 5.2 检查当前用户是否可访问业务

GET `/api/v1/auth/access`

**示例**：

```json
{
  "allowed": true,
  "reason": null
}
```

若用户已禁用：

```json
{
  "allowed": false,
  "reason": "USER_DISABLED"
}
```

### 5.3 获取登录方式选项

GET `/api/v1/auth/options`

接口约定（当前实现）：

- 公开接口，不要求用户登录态
- 固定返回 `200`
- 首版仅返回一种登录方式：`local_password`

**首版固定响应（与实现对齐）**：

```json
{
  "provider": "authentik",
  "methods": [
    {
      "type": "local_password",
      "enabled": true,
      "label": "账号密码登录"
    }
  ]
}
```

### 5.4 登录完成收口入口

POST `/api/v1/auth/post-login`

**请求示例**：

```json
{
  "pendingInvitationSessionId": "pis_001"
}
```

**成功响应示例**：

```json
{
  "status": "ok",
  "userId": "u_001",
  "invitationApplied": true,
  "entryType": "workspace",
  "redirectTo": "/workspace-entry",
  "result": "already_bound_or_consumed"
}
```

跳转规则：

- `appRole=admin` 时，`redirectTo` 返回 `/admin`
- 非管理员用户时，`redirectTo` 返回 `/workspace-entry`
- `entryType` 至少支持 `admin_console | workspace`

---

## 6. 公开 invitation 接口

### 6.1 查看 invitation 预览

GET `/api/v1/public/invitations/{token}`

**成功示例**：

```json
{
  "valid": true,
  "invitation": {
    "targetEmail": "user@example.com",
    "workspaceId": "ws_001",
    "workspaceName": "Design Team",
    "role": "workspace_member",
    "status": "pending",
    "expiresAt": "2026-03-31T23:59:59Z"
  }
}
```

### 6.2 启动 invitation 接入流程

POST `/api/v1/public/invitations/{token}/start`

**成功示例**：

```json
{
  "status": "accepted",
  "pendingInvitationSession": {
    "ttlSeconds": 1200
  },
  "redirectUrl": "https://auth.example.com/if/flow/clawloops-invitation-enrollment/?itoken=xxxx"
}
```

---

## 7. 用户侧接口

### 7.1 获取当前用户 runtime binding（完整对象）

GET `/api/v1/users/me/runtime`

**示例**：

```json
{
  "userId": "u_001",
  "runtime": {
    "runtimeId": "rt_001",
    "volumeId": "vol_001",
    "imageRef": "ghcr.io/openclaw/openclaw@sha256:a5a4c83b773aca85a8ba99cf155f09afa33946c0aa5cc6a9ccb6162738b5da02",
    "desiredState": "running",
    "observedState": "running",
    "browserUrl": "https://u-001.clawloops.example.com",
    "internalEndpoint": "http://rt-rt_001:18789",
    "retentionPolicy": "preserve_workspace",
    "lastError": null
  }
}
```

冻结说明：

- `imageRef` 是平台记录的**实际生效镜像真相**
- 但在 V1 中，调用方**不能**通过 RM internal API 覆盖它
- `internalEndpoint` 必须统一使用 `18789`，且基于固定 alias，不得再出现 `3000`

### 7.2 获取当前用户 runtime 轻量状态投影

GET `/api/v1/users/me/runtime/status`

**示例**：

```json
{
  "runtimeId": "rt_001",
  "desiredState": "running",
  "observedState": "running",
  "task": {
    "status": "running"
  },
  "ready": true,
  "browserUrl": "https://u-001.clawloops.example.com",
  "reason": null,
  "lastError": null
}
```

### 7.3 启动或创建 runtime

POST `/api/v1/users/me/runtime/start`

**响应示例**：

```json
{
  "taskId": "rtask_001",
  "action": "ensure_running",
  "status": "accepted"
}
```

### 7.4 停止 runtime

POST `/api/v1/users/me/runtime/stop`

**响应示例**：

```json
{
  "taskId": "rtask_002",
  "action": "stop",
  "status": "accepted"
}
```

### 7.5 删除 runtime

POST `/api/v1/users/me/runtime/delete`

**请求体**：

```json
{
  "retentionPolicy": "preserve_workspace"
}
```

**响应示例**：

```json
{
  "taskId": "rtask_003",
  "action": "delete",
  "status": "accepted"
}
```

### 7.6 查询 runtime 任务状态

GET `/api/v1/runtime/tasks/{taskId}`

**响应示例**：

```json
{
  "taskId": "rtask_001",
  "userId": "u_001",
  "runtimeId": "rt_001",
  "action": "ensure_running",
  "status": "running",
  "message": "starting runtime"
}
```

### 7.7 获取模型列表（只读）

GET `/api/v1/models`

普通用户侧不开放模型绑定管理；只展示当前用户可见模型。

### 7.8 获取工作区入口

GET `/api/v1/workspace-entry`

说明：

- 该接口只服务非管理员用户的工作区跳转
- `admin` 登录后的默认首页是 `/admin`，不依赖此接口

**响应示例**：

```json
{
  "ready": true,
  "runtimeId": "rt_001",
  "browserUrl": "https://u-001.clawloops.example.com"
}
```

---

## 8. 管理员侧接口

### 8.1 获取管理后台首页摘要

GET `/api/v1/admin/home`

用途：

- 作为 `/admin` 默认首页的唯一首屏聚合接口
- 返回平台治理摘要与待处理事项
- 不承担写操作

**响应示例**：

```json
{
  "summary": {
    "totalUsers": 128,
    "activeUsers": 120,
    "disabledUsers": 8,
    "pendingInvitations": 12,
    "expiringInvitations24h": 3,
    "runningRuntimes": 47,
    "runtimeErrors": 2
  },
  "attention": {
    "pendingInvitations": [
      {
        "invitationId": "inv_001",
        "targetEmail": "user@example.com",
        "workspaceId": "ws_001",
        "role": "workspace_member",
        "status": "pending",
        "expiresAt": "2026-03-31T23:59:59Z"
      }
    ],
    "runtimeAlerts": [
      {
        "userId": "u_002",
        "runtimeId": "rt_002",
        "observedState": "error",
        "lastError": "container start failed",
        "updatedAt": "2026-03-25T10:00:00Z"
      }
    ]
  }
}
```

冻结要求：

- `summary.*` 字段必须一次性返回，前端不自行拼装计数
- `attention.pendingInvitations[]` 用于首页快捷进入邀请治理
- `attention.runtimeAlerts[]` 用于首页快捷进入用户详情排障
- 首页接口为只读接口，写操作仍在各自管理页完成

### 8.2 获取用户列表

GET `/api/v1/admin/users`

建议最小字段：

- `userId`
- `subjectId`
- `role`
- `status`
- `authMethod`
- `runtimeObservedState`
- `lastLoginAt`

### 8.3 获取用户详情

GET `/api/v1/admin/users/{userId}`

### 8.4 修改用户状态

PATCH `/api/v1/admin/users/{userId}/status`

**请求示例**：

```json
{
  "status": "disabled"
}
```

### 8.5 获取指定用户 runtime 详情

GET `/api/v1/admin/users/{userId}/runtime`

### 8.6 invitation 列表

GET `/api/v1/admin/invitations`

### 8.7 创建 invitation

POST `/api/v1/admin/invitations`

**请求示例**：

```json
{
  "targetEmail": "user@example.com",
  "workspaceId": "ws_001",
  "role": "workspace_member",
  "expiresInHours": 72
}
```

### 8.8 获取 invitation 详情

GET `/api/v1/admin/invitations/{invitationId}`

### 8.9 撤销 invitation

POST `/api/v1/admin/invitations/{invitationId}/revoke`

### 8.10 重发 invitation

POST `/api/v1/admin/invitations/{invitationId}/resend`

### 8.11 全局模型与 provider 凭据治理

保留原接口：

- GET `/api/v1/admin/models`
- PUT `/api/v1/admin/models/{modelId}`
- GET `/api/v1/admin/provider-credentials`
- POST `/api/v1/admin/provider-credentials`
- POST `/api/v1/admin/provider-credentials/{credentialId}/verify`
- DELETE `/api/v1/admin/provider-credentials/{credentialId}`
- GET `/api/v1/admin/usage/summary`

---

## 9. 内部服务接口

### 9.1 同步 / 创建用户

POST `/internal/users/sync`

**请求示例**：

```json
{
  "subjectId": "authentik:12345",
  "email": "user@example.com",
  "username": "alice",
  "name": "Alice",
  "auth": {
    "provider": "authentik",
    "method": "local_password"
  }
}
```

### 9.2 创建 invitation 真相对象

POST `/internal/invitations`

### 9.3 消费 invitation 并完成业务绑定

POST `/internal/invitations/{invitationId}/consume`

### 9.4 撤销 invitation

POST `/internal/invitations/{invitationId}/revoke`

### 9.5 确保 runtime binding 存在

POST `/internal/users/{userId}/runtime-binding/ensure`

### 9.6 创建或更新 runtime binding

PUT `/internal/users/{userId}/runtime-binding`

### 9.7 更新 runtime binding 状态

PATCH `/internal/users/{userId}/runtime-binding/state`

### 9.8 获取运行时模型配置

GET `/internal/model-config/users/{userId}`

### 9.9 接收 OpenClaw usage 上报

POST `/internal/usage/records`

### 9.10 Authentik → ClawLoops 错误映射策略

| Authentik / 上游场景 | ClawLoops 错误码 |
| --- | --- |
| enrollment flow 执行失败 / invitation 无法生成 | `INVITATION_ERROR` |
| 用户创建成功但同步失败 | `USER_SYNC_ERROR` |
| 用户中断 flow / 会话丢失 | `INVITATION_ERROR` |

---

## 10. Runtime Manager 内部接口（冻结细化）

### 10.1 确保容器运行

POST `/internal/runtime-manager/containers/ensure-running`

冻结边界：

- RM internal 接口是**同步接口**
- `imageRef` 已从请求体删除
- `compat.openclawConfigDir / openclawWorkspaceDir` 必填
- `routeHost` 仅用于 label 追踪
- 关键 contract drift 返回 `409 RUNTIME_CONTRACT_DRIFT`
- `internalEndpoint` 固定为 `http://rt-<runtimeId>:18789`

**请求示例**：

```json
{
  "userId": "u_001",
  "runtimeId": "rt_001",
  "volumeId": "vol_001",
  "routeHost": "u-001.clawloops.example.com",
  "configMount": {
    "configFilePath": "/var/lib/clawloops/runtime-configs/u_001/openclaw.json",
    "secretFilePath": "/var/lib/clawloops/runtime-secrets/u_001/gateway.token"
  },
  "retentionPolicy": "preserve_workspace",
  "compat": {
    "openclawConfigDir": "/var/lib/clawloops/users/u_001/config",
    "openclawWorkspaceDir": "/var/lib/clawloops/users/u_001/workspace",
  },
  "env": {
    "OPENCLAW_GATEWAY_TOKEN": "<redacted>"
  },
  "envOverrides": {
    "OPENCLAW_ALLOW_INSECURE_PRIVATE_WS": "true"
  }
}
```

**成功示例**：

```json
{
  "runtimeId": "rt_001",
  "observedState": "creating",
  "internalEndpoint": "http://rt-rt_001:18789",
  "message": "creating"
}
```

### 10.2 停止、删除、查询容器

- POST `/internal/runtime-manager/containers/stop`
- POST `/internal/runtime-manager/containers/delete`
- GET `/internal/runtime-manager/containers/{runtimeId}`

冻结补充：

- `stop(nonexistent)=stopped`
- `delete(nonexistent)=deleted`
- `GET` 找不到容器事实时返回 `200 + observedState=deleted`
- 若同一 `runtimeId` 匹配到多个受管容器，返回 `409 RUNTIME_ACTION_CONFLICT`

---

## 11. 前端联调建议

### 11.1 用户工作台初始化

1. `/api/v1/auth/me`
2. `/api/v1/auth/access`
3. `/api/v1/users/me/runtime/status`
4. `/api/v1/models`

### 11.2 invitation 接入页

1. 打开 `/invite/:token`
2. 前端调 `/api/v1/public/invitations/{token}`
3. 用户点击继续
4. 前端调 `/api/v1/public/invitations/{token}/start`
5. 浏览器跳转 `redirectUrl`
6. 登录完成后由 `POST /api/v1/auth/post-login` 收口

### 11.3 runtime 启动

1. `POST /api/v1/users/me/runtime/start`
2. 轮询 `/api/v1/runtime/tasks/{taskId}`
3. 刷新 `/api/v1/users/me/runtime/status`

### 11.4 工作区跳转

1. 若当前用户为 `admin`，直接进入 `/admin`
2. `/admin` 首屏请求 `GET /api/v1/admin/home`
3. 若当前用户不是 `admin`，先调 `/api/v1/workspace-entry`
4. 只有 `ready=true` 才跳转到 `browserUrl`

---

## 12. 字段冻结清单

以下字段为固定命名，禁止别名漂移：

- `subjectId`
- `invitationId`
- `inviteTokenHash`
- `workspaceId`
- `role`
- `desiredState`
- `observedState`
- `browserUrl`
- `internalEndpoint`
- `invitation.status`

**禁止行为**：

| 禁止行为 | 冻结要求 |
| --- | --- |
| 命名漂移 | 不要把 `subjectId` 改成 `externalUid`；不要把 `invitationId` 改成 `inviteId` |
| token 误用 | 平台 token 与 Authentik `itoken` 绝不是同一概念 |
| 状态混用 | 不要把 `task.status`、`observedState`、`ready` 混成同一语义 |
| 密码越权 | 平台不得新增用户密码落库接口或独立改密 API |
| 登录方式漂移 | 首版不向前端开放 Google / GitHub / 企业 SSO 等入口 |
| 地址混用 | 不要用一个 `endpoint` 同时表示 `browserUrl` 和 `internalEndpoint` |
| RM 参数漂移 | 不要在 V1 RM 请求体重新加入 `imageRef / networkName / gatewayPort` |

---

## 13. 最终接口基线结论

接口层面的关键冻结如下：

1. **管理员初始化统一按 `akadmin` 口径处理**
2. **Invitation 采用双层模型，但首版一律延迟创建 Authentik invitation**
3. **`POST /api/v1/auth/post-login` 作为幂等收口入口**
4. **`/auth/access` 永远返回 `200`，仅用于状态判断**
5. **`admin` 登录后默认进入 `/admin`，并通过 `GET /api/v1/admin/home` 加载首页摘要；`workspace-entry` 是唯一工作区跳转入口，前端只在 `ready=true` 时跳转**
6. **runtime 删除改为 `POST /api/v1/users/me/runtime/delete`，不再依赖 DELETE body**
7. **所有 workspace 子域名必须统一经过 Traefik + Authentik Forward Auth**
8. **RuntimeManager internal 接口同步执行，`taskId` 只存在于 Orchestrator 对外层**
9. **V1 runtime 统一使用 `clawloops_shared`、`18789`、固定 alias 与固定镜像**

---

v0.8-authentik-runtime-frozen  
reno  
2026-03-23
