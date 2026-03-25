# ClawLoops 平台 MVP 统一总接口（轻量认证版，运行时冻结修订）

供前后端、平台服务、Orchestrator 与 Runtime Manager 在“业务内轻量认证 + Runtime V1 冻结”前提下统一联调使用。

| 文档定位 | 接口与字段基线 |
| --- | --- |
| 适用范围 | 用户侧 / 管理员侧 / 公开 invitation 入口 / 内部服务侧 / Runtime Manager 内部接口 |
| 修订重点 | 冻结登录与 invitation 接受接口、统一 session 语义、删除外部 IAM 依赖，并明确首版不做改密与找回密码 |
| 响应原则 | 所有响应均采用 JSON；字段名直接作为开发基线，不再自行改名 |
| 当前版本 | v0.12-lightweight-auth |

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
| workspace 访问 | `browserUrl` 仅是受保护入口地址；所有 workspace 子域名必须统一经过平台 session 鉴权 |
| 跳转规则 | `admin` 登录后默认进入 `/admin`；普通用户登录后默认进入 `/app`；非管理员用户只有在 `ready=true` 时才允许跳转到 `browserUrl` |
| 字段命名 | 以本文件“字段冻结清单”为唯一基线，禁止别名漂移 |

---

## 2. 统一错误码

| HTTP | code | 用途 |
| --- | --- | --- |
| 401 | `UNAUTHENTICATED` | 未登录或会话无效 |
| 401 | `INVALID_CREDENTIALS` | 用户名或密码错误 |
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
| 422 | `INVITATION_USERNAME_MISMATCH` | 当前接入用户名与 invitation 指定用户名不匹配 |
| 422 | `INVITATION_PASSWORD_INVALID` | 首次设密不符合平台密码规则 |
| 422 | `INVITATION_WORKSPACE_INVALID` | invitation 指向的 workspace 无效 |
| 422 | `QUOTA_EXCEEDED` | 超出 quota |
| 500 | `SESSION_ERROR` | session 建立、撤销或校验失败 |
| 500 | `INVITATION_ERROR` | invitation 接受流程失败 |
| 500/502 | `RUNTIME_START_FAILED` | runtime 创建、权限初始化或启动探测失败 |
| 500 | `RUNTIME_STOP_FAILED` | runtime 停止失败 |
| 500 | `RUNTIME_DELETE_FAILED` | runtime 删除或目录清理失败 |
| 500/502 | `*_ERROR` | 其他内部模块错误 |

---

## 3. 首版冻结规则（接口层必须统一）

### 3.1 管理员初始化口径

- 首版通过平台初始化脚本或种子数据创建管理员账号
- 文档统一使用“种子管理员账号”口径
- 不再混用任何外部 IAM bootstrap 管理员概念

### 3.2 invitation 单层模型

- **ClawLoops token 是唯一 invitation 入口真相**
- `/invite/{token}`、`GET /api/v1/public/invitations/{token}`、`POST /api/v1/public/invitations/{token}/accept` 只接受平台 token
- `accept` 会直接完成首次设密、membership 绑定、invitation 消费与 session 建立
- 首版禁止再引入额外 enrollment token

### 3.3 invitation 状态真相

- 平台 invitation 状态是最终业务真相
- 首版只存 `pending / consumed / revoked`
- `expired` 不入库，统一通过 `expiresAt < now` 派生
- `revoked / expired / consumed` 必须由平台优先校验

### 3.4 登录与首次接入口径

- 首版普通登录固定为 **用户名优先登录 + 密码**
- 首次接入固定为：**用户通过 invitation 链接进入站内接入页，提交初始密码，并由平台自动建立登录 session**
- 普通用户登录成功后的默认落点应为 `/app`
- 首版普通用户只会绑定 0 或 1 个 workspace，不存在前端 workspace 选择分支

### 3.5 平台密码边界

在契约、设计、接口三份文档统一禁止：

- 平台保存密码明文
- 平台暴露密码明文给前端或日志
- 平台在首版新增改密 API
- 平台在首版新增找回密码 API

### 3.6 幂等要求

- `POST /api/v1/public/invitations/{token}/accept` 在重复提交时必须给出稳定结果
- 同一 `invitationId + userId` 只能成功消费一次
- `consume invitation` 与 `workspace membership binding` 必须原子完成，或定义清晰补偿逻辑

### 3.7 runtime 与跳转语义

- `task.status` = 操作生命周期
- `observedState` = 资源状态
- `ready` = 最终可访问状态
- `admin` 登录后默认进入 `/admin`
- `/admin` 首屏数据由 `GET /api/v1/admin/home` 提供
- 非管理员用户的工作区跳转只看 `ready`
- `workspace-entry` 是唯一工作区跳转入口接口

---

## 4. 接口总览（冻结后）

| 方法 | 路径 | 用途 | 权限 |
| --- | --- | --- | --- |
| GET | `/api/v1/auth/options` | 获取当前登录方式与首版能力开关 | 公开 |
| POST | `/api/v1/auth/login` | 用户名密码登录 | 公开 |
| POST | `/api/v1/auth/logout` | 退出当前会话 | 用户 |
| GET | `/api/v1/auth/me` | 获取当前登录用户 | 用户 |
| GET | `/api/v1/auth/access` | 检查当前用户是否可访问业务（永远返回 200） | 用户 |
| GET | `/api/v1/public/invitations/{token}` | 查看 invitation 预览信息 | 公开 |
| POST | `/api/v1/public/invitations/{token}/accept` | 启动接入并完成首次设密（幂等） | 公开 |
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

---

## 5. 认证相关接口

### 5.1 `GET /api/v1/auth/options`

用途：

- 告诉前端当前首版只支持哪种登录方式
- 明确首版不支持改密、找回密码、第三方登录

示例响应：

```json
{
  "provider": "clawloops",
  "methods": [
    {
      "type": "local_password",
      "label": "用户名优先登录"
    }
  ],
  "features": {
    "passwordChange": false,
    "passwordRecovery": false,
    "thirdPartyLogin": false
  }
}
```

### 5.2 `POST /api/v1/auth/login`

请求体：

```json
{
  "username": "emp001",
  "password": "secret"
}
```

成功响应：

```json
{
  "redirectTo": "/app",
  "user": {
    "userId": "u_001",
    "subjectId": "clawloops:u_001",
    "username": "emp001",
    "tenantId": "t_default",
    "role": "user",
    "status": "active",
    "auth": {
      "provider": "clawloops",
      "method": "local_password"
    },
    "isAdmin": false,
    "isDisabled": false
  }
}
```

规则：

- 成功时由服务端设置 session cookie
- `admin` 返回 `redirectTo=/admin`
- disabled 用户返回 `403 USER_DISABLED`

### 5.3 `POST /api/v1/auth/logout`

成功响应：

```json
{
  "ok": true
}
```

规则：

- 撤销当前 session
- 清理浏览器侧 session cookie

### 5.4 `GET /api/v1/auth/me`

示例响应：

```json
{
  "userId": "u_001",
  "subjectId": "clawloops:u_001",
  "username": "emp001",
  "tenantId": "t_default",
  "role": "user",
  "status": "active",
  "auth": {
    "provider": "clawloops",
    "method": "local_password"
  },
  "isAdmin": false,
  "isDisabled": false
}
```

### 5.5 `GET /api/v1/auth/access`

示例响应：

```json
{
  "allowed": true,
  "reason": null
}
```

说明：

- 该接口永远返回 `200`
- `allowed=false` 时由前端做禁用态或无权限态渲染

---

## 6. invitation 相关接口

### 6.1 `GET /api/v1/public/invitations/{token}`

示例响应：

```json
{
  "valid": true,
  "invitation": {
    "invitationId": "inv_001",
    "targetEmail": "emp001@noemail.local",
    "loginUsername": "emp001",
    "workspaceId": "ws_001",
    "workspaceName": "Design Team",
    "role": "workspace_member",
    "status": "pending",
    "expiresAt": "2026-03-31T23:59:59Z"
  }
}
```

规则：

- 预览接口只展示，不消费 invitation
- 若为无真实邮箱场景，前端应优先展示 `loginUsername`

### 6.2 `POST /api/v1/public/invitations/{token}/accept`

请求体：

```json
{
  "username": "emp001",
  "password": "MyInitPassword123",
  "passwordConfirm": "MyInitPassword123"
}
```

成功响应：

```json
{
  "accepted": true,
  "redirectTo": "/app",
  "user": {
    "userId": "u_001",
    "subjectId": "clawloops:u_001",
    "username": "emp001",
    "tenantId": "t_default",
    "role": "user",
    "status": "active",
    "auth": {
      "provider": "clawloops",
      "method": "local_password"
    },
    "isAdmin": false,
    "isDisabled": false
  },
  "workspaceBinding": {
    "workspaceId": "ws_001",
    "workspaceName": "Design Team",
    "role": "workspace_member"
  }
}
```

冻结规则：

- 成功时一次性完成用户激活、密码哈希写入、membership 绑定、invitation 消费与 session 建立
- 若 invitation 已被同一用户成功消费，再次提交返回稳定结果
- 不允许前端自己补消费逻辑

---

## 7. runtime 与工作区跳转接口

### 7.1 `GET /api/v1/workspace-entry`

示例响应：

```json
{
  "workspaceId": "ws_001",
  "runtimeId": "rt_001",
  "observedState": "running",
  "ready": true,
  "browserUrl": "https://ws-001.clawloops.app"
}
```

规则：

- 非管理员用户只通过该接口获得最终跳转依据
- 只有 `ready=true` 才允许前端整页跳转

### 7.2 `GET /api/v1/users/me/runtime/status`

示例响应：

```json
{
  "runtimeId": "rt_001",
  "observedState": "running",
  "ready": true,
  "browserUrl": "https://ws-001.clawloops.app"
}
```

说明：

- 该接口只用于展示状态，不作为最终跳转依据

---

## 8. 管理侧接口补充

### 8.1 `POST /api/v1/admin/invitations`

请求体：

```json
{
  "targetEmail": "emp001@noemail.local",
  "loginUsername": "emp001",
  "workspaceId": "ws_001",
  "role": "workspace_member",
  "expiresAt": "2026-03-31T23:59:59Z"
}
```

成功响应：

```json
{
  "invitationId": "inv_001",
  "status": "pending",
  "inviteUrl": "https://clawloops.example.com/invite/ptok_xxx"
}
```

### 8.2 `PATCH /api/v1/admin/users/{userId}/status`

请求体：

```json
{
  "status": "disabled"
}
```

规则：

- `disabled` 后，该用户已有 session 应被主动失效或在最短时间内失效

---

## 9. 首版不提供的接口

以下接口在首版禁止出现：

- `POST /api/v1/auth/post-login`
- `POST /api/v1/auth/change-password`
- `POST /api/v1/auth/forgot-password`
- `POST /api/v1/auth/reset-password`
- 任何第三方登录回调接口

---

## 10. 字段冻结清单

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
| token 误用 | 平台 token 就是唯一 invitation 入口，不要再引入第二套 enrollment token |
| 状态混用 | 不要把 `task.status`、`observedState`、`ready` 混成同一语义 |
| 密码越权 | 首版不得新增改密 API 或找回密码 API |
| 登录方式漂移 | 首版不向前端开放 Google / GitHub / 企业 SSO 等入口 |
| 地址混用 | 不要用一个 `endpoint` 同时表示 `browserUrl` 和 `internalEndpoint` |
| RM 参数漂移 | 不要在 V1 RM 请求体重新加入 `imageRef / networkName / gatewayPort` |

---

## 11. 最终接口基线结论

接口层面的关键冻结如下：

1. **登录统一走 `POST /api/v1/auth/login`**
2. **Invitation 采用单层模型，`POST /api/v1/public/invitations/{token}/accept` 作为幂等接入入口**
3. **`/auth/access` 永远返回 `200`，仅用于状态判断**
4. **`admin` 登录后默认进入 `/admin`；普通用户登录后默认进入 `/app`；`workspace-entry` 是唯一工作区跳转入口**
5. **runtime 删除改为 `POST /api/v1/users/me/runtime/delete`，不再依赖 DELETE body**
6. **所有 workspace 子域名必须统一经过平台 session 鉴权**
7. **RuntimeManager internal 接口同步执行，`taskId` 只存在于 Orchestrator 对外层**
8. **V1 runtime 统一使用 `clawloops_shared`、`18789`、固定 alias 与固定镜像**
9. **首版不做改密与找回密码**

---

v0.12-轻量认证修订
reno  
2026-03-25
