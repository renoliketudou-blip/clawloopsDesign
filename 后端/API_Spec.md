# CrewClaw 平台 MVP 统一总接口（Authentik 接入版）

供前后端、平台服务与 runtime manager 在“官方 Authentik 首版接入”前提下统一联调使用。

| 文档定位 | 接口与字段基线 |
| --- | --- |
| 适用范围 | 用户侧 / 管理员侧 / 公开 invitation 入口 / 内部服务侧 / runtime manager 内部接口 |
| 修订重点 | 补齐 `auth/options`、invitation 系列接口、post-login 收口接口、管理员 invitation 治理接口，并冻结首版本地密码登录模式 |
| 响应原则 | 所有响应均采用 JSON；字段名直接作为开发基线，不再自行改名 |
| 当前版本 | v0.6-authentik |

---

## 1. 通用响应约定

| 项 | 说明 |
| --- | --- |
| 成功状态码 | 读取接口默认 `200`；创建默认 `201`；异步动作默认 `202` |
| 错误体 | 统一至少包含 `code` 与 `message` |
| 异步任务 | runtime 启停删继续返回 `taskId` |
| 权限约定 | 用户侧接口依赖当前登录用户；管理员侧接口仅 `admin`；internal 接口仅服务间访问 |
| disabled 语义 | 除 `/api/v1/auth/me` 外，disabled 用户访问业务接口统一返回 `403 USER_DISABLED` |
| workspace 访问 | `browserUrl` 仅是入口地址，实际访问仍经 Traefik + Authentik Forward Auth |

---

## 2. 统一错误码

| HTTP | code | 用途 |
| --- | --- | --- |
| 401 | UNAUTHENTICATED | 未登录或会话无效 |
| 403 | ACCESS_DENIED | 权限不足 |
| 403 | USER_DISABLED | 用户已禁用 |
| 404 | USER_NOT_FOUND | 用户不存在 |
| 404 | RUNTIME_NOT_FOUND | runtime 不存在 |
| 404 | INVITATION_NOT_FOUND | invitation 不存在 |
| 404 | MODEL_NOT_FOUND | 模型不存在 |
| 409 | RUNTIME_ACTION_CONFLICT | runtime 正忙或状态冲突 |
| 409 | INVITATION_ALREADY_CONSUMED | invitation 已消费 |
| 409 | INVITATION_REVOKED | invitation 已撤销 |
| 410 | INVITATION_EXPIRED | invitation 已过期 |
| 422 | INVITATION_EMAIL_MISMATCH | 当前接入邮箱与 invitation 目标不匹配 |
| 422 | INVITATION_WORKSPACE_INVALID | invitation 指向的 workspace 无效 |
| 422 | QUOTA_EXCEEDED | 超出 quota |
| 500/502 | *_ERROR | 内部模块或上游错误 |

---

## 3. 接口总览（修订后）

| 方法 | 路径 | 用途 | 权限 |
| --- | --- | --- | --- |
| GET | `/api/v1/auth/me` | 获取当前登录用户 | 用户 |
| GET | `/api/v1/auth/access` | 检查当前用户是否可访问业务 | 用户 |
| GET | `/api/v1/auth/options` | 获取当前登录方式与首版能力开关 | 公开 |
| GET | `/api/v1/auth/post-login` | Authentik 登录后收口入口 | 用户 |
| GET | `/api/v1/public/invitations/{token}` | 查看 invitation 预览信息 | 公开 |
| POST | `/api/v1/public/invitations/{token}/start` | 启动 invitation 接入流程 | 公开 |
| GET | `/api/v1/users/me/quota` | 获取当前用户 quota | 用户 |
| GET | `/api/v1/users/me/runtime` | 获取当前用户 runtime binding（完整对象） | 用户 |
| GET | `/api/v1/users/me/runtime/status` | 获取当前用户 runtime 轻量状态投影 | 用户 |
| POST | `/api/v1/users/me/runtime/start` | 启动或创建 runtime | 用户 |
| POST | `/api/v1/users/me/runtime/stop` | 停止 runtime | 用户 |
| DELETE | `/api/v1/users/me/runtime` | 删除 runtime | 用户 |
| GET | `/api/v1/runtime/tasks/{taskId}` | 查询 runtime 任务状态 | 用户 / admin |
| GET | `/api/v1/models` | 获取当前用户可见模型列表（只读） | 用户 |
| GET | `/api/v1/workspace-entry` | 获取当前用户工作区入口 | 用户 |
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
| POST | `/internal/invitations/{invitationId}/consume` | 消费 invitation 并完成业务绑定 | internal |
| POST | `/internal/invitations/{invitationId}/revoke` | 撤销 invitation | internal |
| GET | `/internal/model-config/users/{userId}` | 获取运行时模型配置 | internal |
| POST | `/internal/usage/records` | 接收 OpenClaw usage 上报 | internal |
| POST | `/internal/runtime-manager/containers/ensure-running` | 确保容器运行 | internal |
| POST | `/internal/runtime-manager/containers/stop` | 停止容器 | internal |
| POST | `/internal/runtime-manager/containers/delete` | 删除容器 | internal |
| GET | `/internal/runtime-manager/containers/{runtimeId}` | 查询容器状态 | internal |

---

## 4. 认证相关接口

### 4.1 获取当前登录用户

GET `/api/v1/auth/me`

返回当前 CrewClaw 视角下的登录用户信息。

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

### 4.2 检查当前用户是否可访问业务

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

### 4.3 获取登录方式选项

GET `/api/v1/auth/options`

**用途**：告诉前端首版当前启用了哪些登录方式。

**首版固定响应**：

```json
{
  "provider": "authentik",
  "methods": [
    {
      "type": "local_password",
      "enabled": true,
      "label": "账号密码"
    }
  ],
  "futureMethods": [
    "google",
    "github",
    "enterprise_sso",
    "wechat",
    "dingtalk",
    "feishu"
  ]
}
```

### 4.4 登录完成收口入口

GET `/api/v1/auth/post-login`

**用途边界**：

- Authentik 完成认证或 enrollment 之后，浏览器回到该入口；
- 模块 1 在此触发 `/internal/users/sync`；
- 若当前存在待消费 invitation 上下文，则完成 workspace / role 绑定；
- 然后重定向到工作台或 `/workspace-entry`。

**成功响应示例**：

```json
{
  "status": "ok",
  "userId": "u_001",
  "invitationApplied": true,
  "redirectTo": "/workspace"
}
```

---

## 5. 公开 invitation 接口

### 5.1 查看 invitation 预览

GET `/api/v1/public/invitations/{token}`

**用途**：用户点击邮件或分享链接后，前端先调用本接口确认邀请有效性并展示接入页。

**成功示例**：

```json
{
  "valid": true,
  "invitation": {
    "targetEmail": "user@example.com",
    "workspaceId": "ws_001",
    "workspaceName": "Design Team",
    "role": "workspace_member",
    "expiresAt": "2026-03-31T23:59:59Z"
  }
}
```

**失败示例**：

```json
{
  "valid": false,
  "code": "INVITATION_EXPIRED",
  "message": "This invitation has expired."
}
```

### 5.2 启动 invitation 接入流程

POST `/api/v1/public/invitations/{token}/start`

**职责边界**：

- 校验 platform token；
- 写入短期 pending invitation 会话；
- 生成或读取 Authentik enrollment invitation；
- 返回可跳转到 Authentik 的 URL。

**成功示例**：

```json
{
  "status": "accepted",
  "redirectUrl": "https://auth.example.com/if/flow/crewclaw-invitation-enrollment/?itoken=xxxx"
}
```

**说明**：

- 本接口不直接消费 invitation；
- 真正消费发生在 Authentik 登录成功并回到 `/api/v1/auth/post-login` 后。

---

## 6. 用户侧接口

### 6.1 获取当前用户 runtime binding（完整对象）

GET `/api/v1/users/me/runtime`

**用途边界**：

用于详情页、删除确认页、排障场景；不建议高频轮询。

**示例**：

```json
{
  "userId": "u_001",
  "runtime": {
    "runtimeId": "rt_001",
    "volumeId": "vol_001",
    "imageRef": "crewclaw-runtime-wrapper:openclaw-1.0.0",
    "desiredState": "running",
    "observedState": "running",
    "browserUrl": "https://u-001.crewclaw.example.com",
    "retentionPolicy": "preserve_workspace",
    "lastError": null
  }
}
```

### 6.2 获取当前用户 runtime 轻量状态投影

GET `/api/v1/users/me/runtime/status`

**示例**：

```json
{
  "runtimeId": "rt_001",
  "desiredState": "running",
  "observedState": "running",
  "ready": true,
  "browserUrl": "https://u-001.crewclaw.example.com",
  "reason": null,
  "lastError": null
}
```

### 6.3 启动或创建 runtime

POST `/api/v1/users/me/runtime/start`

**响应示例**：

```json
{
  "taskId": "rtask_001",
  "action": "ensure_running",
  "status": "accepted"
}
```

### 6.4 停止 runtime

POST `/api/v1/users/me/runtime/stop`

**响应示例**：

```json
{
  "taskId": "rtask_002",
  "action": "stop",
  "status": "accepted"
}
```

### 6.5 删除 runtime

DELETE `/api/v1/users/me/runtime`

**请求体**：

```json
{
  "retentionPolicy": "preserve_workspace"
}
```

### 6.6 查询 runtime 任务状态

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

### 6.7 获取模型列表（只读）

GET `/api/v1/models`

**修订点**：

- 普通用户侧不开放模型绑定管理；
- 不开放 provider 凭据管理；
- 只展示当前用户可见模型。

### 6.8 获取工作区入口

GET `/api/v1/workspace-entry`

**响应示例**：

```json
{
  "ready": true,
  "runtimeId": "rt_001",
  "browserUrl": "https://u-001.crewclaw.example.com"
}
```

---

## 7. 管理员侧接口

### 7.1 获取用户列表

GET `/api/v1/admin/users`

建议最小字段：

- `userId`
- `subjectId`
- `role`
- `status`
- `authMethod`
- `runtimeObservedState`
- `lastLoginAt`

### 7.2 获取用户详情

GET `/api/v1/admin/users/{userId}`

建议最小字段：

- `userId`
- `subjectId`
- `tenantId`
- `role`
- `status`
- `createdAt`
- `updatedAt`
- `auth.provider`
- `auth.method`

### 7.3 修改用户状态

PATCH `/api/v1/admin/users/{userId}/status`

**请求示例**：

```json
{
  "status": "disabled"
}
```

### 7.4 获取指定用户 runtime 详情

GET `/api/v1/admin/users/{userId}/runtime`

### 7.5 Invitation 列表

GET `/api/v1/admin/invitations`

**建议过滤参数**：

- `status`
- `workspaceId`
- `targetEmail`

**响应示例**：

```json
{
  "items": [
    {
      "invitationId": "inv_001",
      "targetEmail": "user@example.com",
      "workspaceId": "ws_001",
      "role": "workspace_member",
      "status": "pending",
      "expiresAt": "2026-03-31T23:59:59Z"
    }
  ]
}
```

### 7.6 创建 invitation

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

**响应示例**：

```json
{
  "invitationId": "inv_001",
  "status": "pending",
  "inviteUrl": "https://crewclaw.example.com/invite/eyJ...",
  "targetEmail": "user@example.com",
  "workspaceId": "ws_001",
  "role": "workspace_member",
  "expiresAt": "2026-03-31T23:59:59Z"
}
```

### 7.7 获取 invitation 详情

GET `/api/v1/admin/invitations/{invitationId}`

**响应示例**：

```json
{
  "invitationId": "inv_001",
  "targetEmail": "user@example.com",
  "workspaceId": "ws_001",
  "role": "workspace_member",
  "status": "pending",
  "authentikInvitationRef": "ak_inv_xxx",
  "expiresAt": "2026-03-31T23:59:59Z",
  "consumedAt": null,
  "consumedByUserId": null,
  "lastError": null
}
```

### 7.8 撤销 invitation

POST `/api/v1/admin/invitations/{invitationId}/revoke`

**响应示例**：

```json
{
  "invitationId": "inv_001",
  "status": "revoked"
}
```

### 7.9 重发 invitation

POST `/api/v1/admin/invitations/{invitationId}/resend`

**响应示例**：

```json
{
  "invitationId": "inv_001",
  "status": "pending",
  "inviteUrl": "https://crewclaw.example.com/invite/eyJ..."
}
```

### 7.10 全局模型与 provider 凭据治理

保留原接口：

- GET `/api/v1/admin/models`
- PUT `/api/v1/admin/models/{modelId}`
- GET `/api/v1/admin/provider-credentials`
- POST `/api/v1/admin/provider-credentials`
- POST `/api/v1/admin/provider-credentials/{credentialId}/verify`
- DELETE `/api/v1/admin/provider-credentials/{credentialId}`
- GET `/api/v1/admin/usage/summary`

---

## 8. 内部服务接口

### 8.1 同步 / 创建用户

POST `/internal/users/sync`

**职责边界**：

- 仍以 `subjectId` 幂等；
- 默认 `tenantId=t_default`；
- 默认 `auth.provider=authentik`；
- 默认 `auth.method=local_password`（首版）。

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

### 8.2 创建 invitation 真相对象

POST `/internal/invitations`

**请求示例**：

```json
{
  "targetEmail": "user@example.com",
  "workspaceId": "ws_001",
  "role": "workspace_member",
  "expiresAt": "2026-03-31T23:59:59Z",
  "authentikInvitationRef": "ak_inv_xxx"
}
```

### 8.3 消费 invitation 并完成业务绑定

POST `/internal/invitations/{invitationId}/consume`

**职责边界**：

- 校验 invitation 当前仍为 `pending`；
- 校验 `subjectId` / `userId` 合法；
- 创建或更新 workspace membership；
- 把 invitation 标记为 `consumed`。

**请求示例**：

```json
{
  "userId": "u_001",
  "subjectId": "authentik:12345"
}
```

**成功示例**：

```json
{
  "invitationId": "inv_001",
  "status": "consumed",
  "userId": "u_001",
  "workspaceId": "ws_001",
  "role": "workspace_member"
}
```

### 8.4 撤销 invitation

POST `/internal/invitations/{invitationId}/revoke`

### 8.5 确保 runtime binding 存在

POST `/internal/users/{userId}/runtime-binding/ensure`

保持原冻结行为不变。

### 8.6 创建或更新 runtime binding

PUT `/internal/users/{userId}/runtime-binding`

### 8.7 更新 runtime binding 状态

PATCH `/internal/users/{userId}/runtime-binding/state`

### 8.8 获取运行时模型配置

GET `/internal/model-config/users/{userId}`

### 8.9 接收 OpenClaw usage 上报

POST `/internal/usage/records`

---

## 9. Runtime Manager 内部接口

### 9.1 确保容器运行

POST `/internal/runtime-manager/containers/ensure-running`

**请求示例**：

```json
{
  "userId": "u_001",
  "runtimeId": "rt_001",
  "imageRef": "crewclaw-runtime-wrapper:openclaw-1.0.0",
  "volumeId": "vol_001",
  "routeHost": "u-001.crewclaw.example.com",
  "configMount": {
    "configFilePath": "/var/lib/crewclaw/runtime-configs/u_001/model-gateway.json",
    "secretFilePath": "/var/lib/crewclaw/runtime-secrets/u_001/gateway.token"
  },
  "retentionPolicy": "preserve_workspace"
}
```

### 9.2 停止、删除、查询容器

- POST `/internal/runtime-manager/containers/stop`
- POST `/internal/runtime-manager/containers/delete`
- GET `/internal/runtime-manager/containers/{runtimeId}`

---

## 10. 前端联调建议

### 10.1 用户工作台初始化

顺序建议：

1. `/api/v1/auth/me`
2. `/api/v1/users/me/runtime/status`
3. `/api/v1/models`

### 10.2 invitation 接入页

顺序建议：

1. 打开 `/invite/:token`
2. 前端调 `/api/v1/public/invitations/{token}`
3. 用户点击继续
4. 前端调 `/api/v1/public/invitations/{token}/start`
5. 浏览器跳转 `redirectUrl`
6. 登录完成后由 `/api/v1/auth/post-login` 收口

### 10.3 runtime 启动

1. `POST /api/v1/users/me/runtime/start`
2. 轮询 `/api/v1/runtime/tasks/{taskId}`
3. 刷新 `/api/v1/users/me/runtime/status`

### 10.4 工作区跳转

1. 先调 `/api/v1/workspace-entry`
2. 只有 `ready=true` 才跳转到 `browserUrl`

---

## 11. 字段冻结清单

| 禁止行为 | 冻结要求 |
| --- | --- |
| 命名漂移 | 不要把 `subjectId` 改成 `externalUid`，不要把 `invitationId` 改成 `inviteId` 后又混用 |
| 状态混用 | 不要把 `invitation.status` 与 runtime `observedState` 混成同一字段 |
| token 误用 | 平台 `inviteToken` 与 Authentik `itoken` 不是同一个概念，不得混用 |
| 密码越权 | 平台不得新增用户密码落库接口 |
| 登录方式漂移 | 首版不向前端开放 Google / GitHub / 企业 SSO 等入口 |
| binding 责任漂移 | `workspaceId / role` 的最终绑定由平台完成，不交给前端直写 |
| 地址混用 | 不要用一个 `endpoint` 同时表示 `browserUrl` 和 `internalEndpoint` |

---

## 12. 最终接口基线结论

接口层面的关键变化只有四件事：

1. 新增 **`/api/v1/auth/options`**，把首版登录方式固定下来；
2. 新增 **公开 invitation 入口**，用于接入完成页；
3. 新增 **管理员 invitation 管理接口**；
4. 新增 **post-login 收口接口**，把 Authentik 登录成功与 CrewClaw 业务绑定连接起来。

这样一来，首版就可以在不改上游源码的情况下，把 Authentik 真正接成你的统一身份层，而不是停留在“文档里提到过 Authentik”这一层。

