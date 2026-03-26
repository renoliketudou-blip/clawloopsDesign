# ClawLoops 平台 MVP 统一总接口（轻量认证版，运行时冻结修订）

供前后端、平台服务、Orchestrator 与 Runtime Manager 在“业务内轻量认证 + Runtime V1 冻结”前提下统一联调使用。

| 文档定位 | 接口与字段基线 |
| --- | --- |
| 适用范围 | 用户侧 / 管理员侧 / 公开 invitation 入口 / 内部服务侧 / Runtime Manager 内部接口 |
| 修订重点 | 冻结登录与 invitation 接受接口、统一 session 语义、删除外部 IAM 依赖，并补充种子管理员首登强制改密流程 |
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
| 跳转规则 | 种子管理员若 `mustChangePassword=true` 则登录后优先进入 `/force-password-change`；其他 `admin` 默认进入 `/admin`；普通用户默认进入 `/app`；非管理员用户只有在 `ready=true` 时才允许跳转到 `browserUrl` |
| 字段命名 | 以本文件“字段冻结清单”为唯一基线，禁止别名漂移 |

---

## 2. 统一错误码

| HTTP | code | 用途 |
| --- | --- | --- |
| 401 | `UNAUTHENTICATED` | 未登录或会话无效 |
| 401 | `INVALID_CREDENTIALS` | 用户名或密码错误 |
| 403 | `ACCESS_DENIED` | 权限不足 |
| 403 | `USER_DISABLED` | 用户已禁用 |
| 403 | `PASSWORD_CHANGE_REQUIRED` | 当前会话必须先完成强制改密 |
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
| 422 | `CURRENT_PASSWORD_INCORRECT` | 强制改密时当前密码校验失败 |
| 422 | `PASSWORD_CHANGE_INVALID` | 新密码不符合平台规则，或与当前密码不允许相同 |
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
- 种子管理员默认初始密码固定为 `admin`
- 种子管理员首次登录前必须带 `mustChangePassword=true`
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
- 种子管理员首次使用默认密码 `admin` 登录成功后，必须立刻跳转 `/force-password-change`
- 在 `mustChangePassword=true` 期间，除 `/force-password-change` 与登出外，不允许继续访问其他业务页
- 普通用户登录成功后的默认落点应为 `/app`
- 首版普通用户只会绑定 0 或 1 个 workspace，不存在前端 workspace 选择分支

### 3.5 平台密码边界

在契约、设计、接口三份文档统一禁止：

- 平台保存密码明文
- 平台暴露密码明文给前端或日志
- 平台开放任意可见的通用自助改密入口
- 平台在首版新增找回密码 API

首版唯一允许新增的改密能力：

- `POST /api/v1/auth/password/change`
- 仅允许已登录用户修改自己的当前密码
- 首版前端只把它用于种子管理员首次登录后的强制改密

首版统一密码规则：

- `minLength = 8`
- `maxLength = 64`
- 必须至少包含 1 个字母与 1 个数字
- 不允许与 `username` 相同
- 不允许继续使用默认管理员密码 `admin`
- invitation 首次设密与首登强制改密共用同一套规则

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
| POST | `/api/v1/auth/password/change` | 修改当前登录用户密码（首版仅用于强制改密） | 用户 |
| GET | `/api/v1/auth/me` | 获取当前登录用户 | 用户 |
| GET | `/api/v1/auth/access` | 检查当前用户是否可访问业务（永远返回 200） | 用户 |
| GET | `/internal/auth/workspace-access` | workspace 子域统一网关鉴权入口 | internal |
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
| GET | `/api/v1/admin/usage/summary` | 获取管理侧 usage 汇总与趋势 | admin |

---

## 5. 认证相关接口

### 5.1 `GET /api/v1/auth/options`

用途：

- 告诉前端当前首版只支持哪种登录方式
- 明确首版只开放强制改密、不开放找回密码与第三方登录

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
  "passwordPolicy": {
    "minLength": 8,
    "maxLength": 64,
    "requireLetter": true,
    "requireNumber": true,
    "disallowUsernameAsPassword": true,
    "disallowDefaultAdminPassword": true
  },
  "features": {
    "forcedPasswordChange": true,
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
    "isDisabled": false,
    "mustChangePassword": false,
    "passwordChangeReason": null
  }
}
```

规则：

- 成功时由服务端设置 session cookie
- cookie 名称固定为 `clawloops_session`
- 当 `mustChangePassword=true` 时返回 `redirectTo=/force-password-change`
- 非强制改密状态下，`admin` 返回 `redirectTo=/admin`
- disabled 用户返回 `403 USER_DISABLED`

### 5.3 `POST /api/v1/auth/password/change`

请求体：

```json
{
  "currentPassword": "admin",
  "newPassword": "admin#2026!new",
  "newPasswordConfirm": "admin#2026!new"
}
```

成功响应：

```json
{
  "changed": true,
  "redirectTo": "/admin",
  "user": {
    "userId": "u_admin",
    "subjectId": "clawloops:u_admin",
    "username": "admin",
    "tenantId": "t_default",
    "role": "admin",
    "status": "active",
    "auth": {
      "provider": "clawloops",
      "method": "local_password"
    },
    "isAdmin": true,
    "isDisabled": false,
    "mustChangePassword": false,
    "passwordChangeReason": null
  }
}
```

规则：

- 仅允许当前已登录用户修改自己的密码
- 首版前端只在 `mustChangePassword=true` 时暴露该能力
- `currentPassword` 校验失败时返回 `422 CURRENT_PASSWORD_INCORRECT`
- `newPassword` 不得与当前密码相同
- `newPassword` 必须满足 `/auth/options.passwordPolicy`
- 成功后必须更新密码哈希，并清除 `mustChangePassword`
- 成功后必须轮换当前 session，并用同名 cookie 覆盖旧值，避免继续使用旧认证上下文

### 5.4 `POST /api/v1/auth/logout`

成功响应：

```json
{
  "ok": true
}
```

规则：

- 撤销当前 session
- 清理浏览器侧 session cookie，且必须使用与签发时相同的 `Domain / Path / SameSite / Secure`

### 5.5 `GET /api/v1/auth/me`

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
  "isDisabled": false,
  "mustChangePassword": false,
  "passwordChangeReason": null
}
```

说明：

- `mustChangePassword=true` 时，前端必须把该用户收口到 `/force-password-change`
- 首版该字段通常只会出现在种子管理员首次登录场景

### 5.6 `GET /api/v1/auth/access`

示例响应：

```json
{
  "allowed": false,
  "reason": "PASSWORD_CHANGE_REQUIRED"
}
```

说明：

- 该接口永远返回 `200`
- `allowed=false` 时由前端做禁用态或无权限态渲染
- `reason` 首版只认 `USER_DISABLED | PASSWORD_CHANGE_REQUIRED | null`
- 当 `reason=PASSWORD_CHANGE_REQUIRED` 时，前端必须立刻跳转 `/force-password-change`
- 当 `reason=USER_DISABLED` 时，前端应进入账号禁用说明页或禁用拦截页

### 5.7 session cookie 冻结规则

- cookie 名称统一为 `clawloops_session`
- `HttpOnly=true`
- 生产环境 `Secure=true`
- `SameSite=Lax`
- `Path=/`
- 生产环境 `Domain` 必须覆盖主域与 workspace 子域，例如 `.clawloops.example.com`
- `POST /api/v1/auth/login`、`POST /api/v1/auth/password/change`、`POST /api/v1/public/invitations/{token}/accept` 成功后，若创建或轮换 session，必须写入同一套 cookie 属性
- `POST /api/v1/auth/logout` 清 cookie 时必须复用完全一致的 `Domain / Path`

### 5.8 `GET /internal/auth/workspace-access`

用途：

- 作为 Traefik `ForwardAuth` 或等价轻量代理的唯一放行判断入口
- 保护所有 workspace 子域，防止用户绕过控制面直接访问 `browserUrl`

请求约定：

- 网关必须透传浏览器原始 cookie
- 网关必须透传 `Host`
- 网关必须透传 `X-Forwarded-Proto`、`X-Forwarded-Host`、`X-Forwarded-Uri`、`X-Forwarded-Method`
- 平台依据 host 路由规则自行解析目标 `workspaceId`，不信任前端自填参数

成功响应：

- 返回 `200`
- 允许附带只读透传头：`X-Clawloops-User-Id`、`X-Clawloops-Subject-Id`、`X-Clawloops-Workspace-Id`

拒绝规则：

- 无 session、session 无效或已撤销：`401 UNAUTHENTICATED`
- 已登录但用户 `disabled`：`403 USER_DISABLED`
- 已登录但 `mustChangePassword=true`：`403 PASSWORD_CHANGE_REQUIRED`
- 已登录但不具备目标 workspace membership：`403 ACCESS_DENIED`
- 已登录但目标 runtime 当前不可进入：`403 ACCESS_DENIED`
- 网关对所有非 `2xx` 响应一律不得继续转发到 runtime

说明：

- 该接口只返回“允许 / 拒绝”结论，不承担浏览器跳转编排
- 下游 runtime 不得把透传头重新作为新的信任边界

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
  "replayed": false,
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

幂等重放成功响应示例：

```json
{
  "accepted": true,
  "replayed": true,
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
- 首次成功消费返回 `200`
- 若 invitation 已被同一 `loginUsername` 对应用户成功消费，再次提交返回 `200`，且响应体语义必须保持稳定；可通过 `replayed=true` 表示命中幂等重放
- 幂等重放不得再次创建用户、再次写入 membership、再次生成第二个业务副作用
- 若 invitation 已被其他用户消费，返回 `409 INVITATION_ALREADY_CONSUMED`
- 不允许前端自己补消费逻辑
- `password` 必须满足 `/auth/options.passwordPolicy`
- 成功时由服务端设置或轮换 `clawloops_session` cookie，并使用冻结的 cookie 属性

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

管理侧列表接口统一约定：

- 默认 `page=1`
- 默认 `pageSize=20`
- `pageSize` 最大 `100`
- 若无额外说明，所有时间字段统一使用 UTC ISO8601 字符串
- `/admin` 首页、列表页与详情页都必须直接信任管理侧接口返回，不允许前端自行拼摘要真相

### 8.1 `GET /api/v1/admin/home`

用途：

- 作为 `/admin` 默认首页唯一数据源
- 返回摘要卡片、待办事项和快捷入口

示例响应：

```json
{
  "summary": {
    "totalUsers": 24,
    "activeUsers": 22,
    "disabledUsers": 2,
    "pendingInvitations": 5,
    "runningRuntimes": 18,
    "runtimeErrors": 1,
    "enabledModels": 6,
    "providerCredentialIssues": 1
  },
  "todo": [
    {
      "id": "todo_inv_expiring",
      "type": "INVITATION_EXPIRING",
      "severity": "warning",
      "title": "3 条 invitation 即将过期",
      "count": 3,
      "action": {
        "label": "查看 invitation",
        "href": "/admin/invitations"
      }
    },
    {
      "id": "todo_runtime_error",
      "type": "RUNTIME_ERROR",
      "severity": "critical",
      "title": "1 个 runtime 处于 error",
      "count": 1,
      "action": {
        "label": "查看用户 runtime",
        "href": "/admin/users"
      }
    }
  ],
  "quickLinks": [
    {
      "label": "用户管理",
      "href": "/admin/users"
    },
    {
      "label": "邀请管理",
      "href": "/admin/invitations"
    },
    {
      "label": "模型治理",
      "href": "/admin/models"
    },
    {
      "label": "Provider 凭据",
      "href": "/admin/provider-credentials"
    },
    {
      "label": "Usage 汇总",
      "href": "/admin/usage"
    }
  ],
  "generatedAt": "2026-03-25T09:30:00Z"
}
```

字段语义：

- `summary` 只用于首页摘要卡片，不由前端再拼其他接口
- `todo[]` 为管理员待处理事项，按 `severity` 从高到低排序
- `severity` 只认 `info / warning / critical`
- `quickLinks[]` 为首页快捷入口，顺序应与后台左侧导航一致

### 8.2 `GET /api/v1/admin/users`

查询参数：

- `page`
- `pageSize`
- `keyword`：按 `username` 模糊查询
- `status`：`active / disabled`
- `role`：`admin / user`

示例响应：

```json
{
  "items": [
    {
      "userId": "u_001",
      "username": "emp001",
      "role": "user",
      "status": "active",
      "workspaceId": "ws_001",
      "workspaceName": "Design Team",
      "hasRuntime": true,
      "runtimeObservedState": "running",
      "lastLoginAt": "2026-03-25T08:40:00Z",
      "createdAt": "2026-03-01T10:00:00Z"
    },
    {
      "userId": "u_002",
      "username": "admin01",
      "role": "admin",
      "status": "active",
      "workspaceId": null,
      "workspaceName": null,
      "hasRuntime": false,
      "runtimeObservedState": null,
      "lastLoginAt": "2026-03-25T09:00:00Z",
      "createdAt": "2026-02-28T09:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 2
}
```

字段语义：

- `items[]` 为用户列表页唯一真相
- `workspaceId / workspaceName` 为空表示当前用户尚未绑定 workspace
- `hasRuntime` 表示是否存在 runtime 绑定，不等同于 runtime 当前正在运行
- `runtimeObservedState` 只做列表轻量展示，不作为最终可进入判断

### 8.3 `GET /api/v1/admin/users/{userId}`

示例响应：

```json
{
  "user": {
    "userId": "u_001",
    "subjectId": "clawloops:u_001",
    "username": "emp001",
    "role": "user",
    "status": "active",
    "tenantId": "t_default",
    "workspaceId": "ws_001",
    "workspaceName": "Design Team",
    "workspaceRole": "workspace_member",
    "lastLoginAt": "2026-03-25T08:40:00Z",
    "createdAt": "2026-03-01T10:00:00Z"
  }
}
```

字段语义：

- `user` 为用户详情页基础信息真相
- `workspaceRole` 表示该用户在目标 workspace 的业务角色
- 用户详情页的 runtime 展示由 `GET /api/v1/admin/users/{userId}/runtime` 单独提供，前端不要把两个接口混成一个对象

### 8.4 `PATCH /api/v1/admin/users/{userId}/status`

请求体：

```json
{
  "status": "disabled"
}
```

成功响应：

```json
{
  "userId": "u_001",
  "status": "disabled",
  "effectiveAt": "2026-03-25T09:45:00Z"
}
```

规则：

- `status` 只允许 `active / disabled`
- `disabled` 后，该用户已有 session 应被主动失效或在最短时间内失效
- `disabled` 后，该用户业务接口应统一收口到 `403 USER_DISABLED`

### 8.5 `GET /api/v1/admin/users/{userId}/runtime`

示例响应：

```json
{
  "workspaceId": "ws_001",
  "workspaceName": "Design Team",
  "runtime": {
    "runtimeId": "rt_001",
    "desiredState": "running",
    "observedState": "running",
    "ready": true,
    "browserUrl": "https://ws-001.clawloops.app",
    "internalEndpoint": "http://rt-rt_001:18789",
    "retentionPolicy": "preserve_workspace",
    "volumeId": "vol_001",
    "imageRef": "ghcr.io/clawloops/runtime:v1"
  }
}
```

字段语义：

- `runtime = null` 表示该用户尚未建立 runtime 绑定
- `desiredState` 是业务目标态，`observedState` 是资源事实态，`ready` 是最终可访问态
- `browserUrl` 仅用于展示入口，不代表匿名可访问
- `internalEndpoint` 只用于平台内部诊断，不允许前端拿它做浏览器跳转

### 8.6 `GET /api/v1/admin/invitations`

查询参数：

- `page`
- `pageSize`
- `status`：`pending / consumed / revoked`
- `keyword`：按 `loginUsername` 或 `targetEmail` 模糊查询

示例响应：

```json
{
  "items": [
    {
      "invitationId": "inv_001",
      "loginUsername": "emp001",
      "targetEmail": "emp001@noemail.local",
      "workspaceId": "ws_001",
      "workspaceName": "Design Team",
      "role": "workspace_member",
      "status": "pending",
      "expiresAt": "2026-03-31T23:59:59Z",
      "inviteUrl": "https://clawloops.example.com/invite/ptok_xxx",
      "consumedByUserId": null,
      "consumedByUsername": null
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

字段语义：

- 列表页高频操作直接使用 `items[]`
- `inviteUrl` 是创建成功后与列表页复制动作使用的标准字段
- `consumedByUserId / consumedByUsername` 为空表示 invitation 尚未被消费

### 8.7 `POST /api/v1/admin/invitations`

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
  "loginUsername": "emp001",
  "workspaceId": "ws_001",
  "workspaceName": "Design Team",
  "role": "workspace_member",
  "status": "pending",
  "expiresAt": "2026-03-31T23:59:59Z",
  "inviteUrl": "https://clawloops.example.com/invite/ptok_xxx"
}
```

字段语义：

- `loginUsername` 是用户首次接入与后续登录优先使用的用户名
- `targetEmail` 可为空或为代理邮箱，不要求前端把它作为主提示文案
- `inviteUrl` 是管理员复制和发送 invitation 的唯一标准地址

### 8.8 `GET /api/v1/admin/invitations/{invitationId}`

示例响应：

```json
{
  "invitation": {
    "invitationId": "inv_001",
    "loginUsername": "emp001",
    "targetEmail": "emp001@noemail.local",
    "workspaceId": "ws_001",
    "workspaceName": "Design Team",
    "role": "workspace_member",
    "status": "pending",
    "expiresAt": "2026-03-31T23:59:59Z",
    "inviteUrl": "https://clawloops.example.com/invite/ptok_xxx",
    "consumedByUserId": null,
    "consumedByUsername": null,
    "createdAt": "2026-03-25T09:20:00Z"
  }
}
```

字段语义：

- `invitation` 为 invitation 详情页唯一真相
- `status` 只认 `pending / consumed / revoked`
- `expired` 不单独存储，仍由 `expiresAt < now` 派生

### 8.9 `POST /api/v1/admin/invitations/{invitationId}/revoke`

成功响应：

```json
{
  "invitationId": "inv_001",
  "status": "revoked",
  "effectiveAt": "2026-03-25T10:00:00Z"
}
```

规则：

- 仅 `pending` invitation 可撤销
- 撤销成功后，对应 `inviteUrl` 必须立即失效

### 8.10 `POST /api/v1/admin/invitations/{invitationId}/resend`

成功响应：

```json
{
  "invitationId": "inv_001",
  "status": "pending",
  "expiresAt": "2026-04-07T23:59:59Z",
  "inviteUrl": "https://clawloops.example.com/invite/ptok_xxx",
  "resentAt": "2026-03-25T10:05:00Z"
}
```

规则：

- 首版 `resend` 不创建第二条 invitation，沿用同一 `invitationId`
- 若需要延长有效期，应在该动作中返回最新 `expiresAt`
- 已 `consumed` 或已 `revoked` 的 invitation 不允许 `resend`

### 8.11 `GET /api/v1/admin/models`

示例响应：

```json
{
  "items": [
    {
      "modelId": "gpt-4.1-mini",
      "displayName": "GPT-4.1 Mini",
      "provider": "openai",
      "enabled": true,
      "visibleToUsers": true,
      "isDefault": true,
      "updatedAt": "2026-03-25T09:00:00Z"
    },
    {
      "modelId": "claude-3-7-sonnet",
      "displayName": "Claude 3.7 Sonnet",
      "provider": "anthropic",
      "enabled": true,
      "visibleToUsers": false,
      "isDefault": false,
      "updatedAt": "2026-03-24T16:00:00Z"
    }
  ]
}
```

字段语义：

- `enabled=false` 表示平台不再允许该模型被路由使用
- `visibleToUsers=false` 表示该模型不出现在用户侧 `/api/v1/models` 列表
- `isDefault=true` 表示该模型是当前默认推荐项，不等同于唯一可用项

### 8.12 `PUT /api/v1/admin/models/{modelId}`

请求体：

```json
{
  "enabled": true,
  "visibleToUsers": true,
  "isDefault": false
}
```

成功响应：

```json
{
  "modelId": "gpt-4.1-mini",
  "enabled": true,
  "visibleToUsers": true,
  "isDefault": false,
  "updatedAt": "2026-03-25T10:10:00Z"
}
```

规则：

- `enabled / visibleToUsers / isDefault` 为首版模型治理最小可编辑字段
- 若某模型被设为新的 `isDefault=true`，平台应保证同一时刻只有一个默认模型

### 8.13 `GET /api/v1/admin/provider-credentials`

示例响应：

```json
{
  "items": [
    {
      "credentialId": "pc_001",
      "provider": "openai",
      "label": "OpenAI Prod",
      "maskedKey": "sk-****9abc",
      "status": "active",
      "lastVerifiedAt": "2026-03-25T08:50:00Z",
      "createdAt": "2026-03-01T09:00:00Z"
    }
  ]
}
```

字段语义：

- `maskedKey` 只用于列表展示，永远不返回明文密钥
- `status` 只认 `unverified / active / verification_failed`
- `lastVerifiedAt` 为空表示该凭据尚未完成验证

### 8.14 `POST /api/v1/admin/provider-credentials`

请求体：

```json
{
  "provider": "openai",
  "label": "OpenAI Prod",
  "apiKey": "sk-live-secret",
  "baseUrl": "https://api.openai.com/v1"
}
```

成功响应：

```json
{
  "credentialId": "pc_001",
  "provider": "openai",
  "label": "OpenAI Prod",
  "maskedKey": "sk-****9abc",
  "status": "unverified",
  "lastVerifiedAt": null,
  "createdAt": "2026-03-25T10:15:00Z"
}
```

规则：

- 请求体允许提交明文 `apiKey`，但响应体与日志都不得回传明文
- `baseUrl` 为空时表示使用 provider 默认地址

### 8.15 `POST /api/v1/admin/provider-credentials/{credentialId}/verify`

成功响应：

```json
{
  "credentialId": "pc_001",
  "verified": true,
  "status": "active",
  "lastVerifiedAt": "2026-03-25T10:16:00Z"
}
```

规则：

- 验证动作只改变凭据健康状态，不改变 `credentialId`
- 验证失败时返回明确错误，并将 `status` 收口为 `verification_failed`

### 8.16 `DELETE /api/v1/admin/provider-credentials/{credentialId}`

成功响应：

```json
{
  "ok": true
}
```

规则：

- 删除后该凭据不再参与模型路由
- 若被删除凭据当前仍被默认模型使用，后端应拒绝删除或先完成策略切换

### 8.17 `GET /api/v1/admin/usage/summary`

用途：

- 作为 `/admin/usage` 页面唯一数据源
- 返回全局 usage 摘要、时间序列趋势和排名信息

查询参数：

- `from`
- `to`
- `granularity`：`day / week`

示例响应：

```json
{
  "period": {
    "from": "2026-03-01T00:00:00Z",
    "to": "2026-03-25T23:59:59Z",
    "granularity": "day"
  },
  "summary": {
    "requestCount": 12840,
    "promptTokens": 5320000,
    "completionTokens": 2140000,
    "totalTokens": 7460000,
    "estimatedCostUsd": "132.40",
    "activeUsers": 18,
    "activeWorkspaces": 18
  },
  "series": [
    {
      "bucketStartAt": "2026-03-24T00:00:00Z",
      "requestCount": 420,
      "totalTokens": 251000,
      "estimatedCostUsd": "4.90"
    },
    {
      "bucketStartAt": "2026-03-25T00:00:00Z",
      "requestCount": 460,
      "totalTokens": 278000,
      "estimatedCostUsd": "5.20"
    }
  ],
  "topUsers": [
    {
      "userId": "u_001",
      "username": "emp001",
      "requestCount": 880,
      "totalTokens": 520000,
      "estimatedCostUsd": "10.40"
    }
  ],
  "topModels": [
    {
      "modelId": "gpt-4.1-mini",
      "requestCount": 6240,
      "totalTokens": 4010000,
      "estimatedCostUsd": "71.20"
    }
  ],
  "generatedAt": "2026-03-25T10:20:00Z"
}
```

字段语义：

- `period` 是当前统计窗口回显，前端筛选栏应直接信任它
- `summary` 用于顶部统计卡片
- `series[]` 用于趋势图，不要求前端自己按明细重算
- `topUsers[]` 与 `topModels[]` 为只读排行，不承担权限判定
- `estimatedCostUsd` 是平台估算值，用于治理观察，不等同于最终财务结算

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
| 密码越权 | 首版不得新增通用改密 API 或找回密码 API；仅允许受限的首登强制改密接口 |
| 登录方式漂移 | 首版不向前端开放 Google / GitHub / 企业 SSO 等入口 |
| 地址混用 | 不要用一个 `endpoint` 同时表示 `browserUrl` 和 `internalEndpoint` |
| RM 参数漂移 | 不要在 V1 RM 请求体重新加入 `imageRef / networkName / gatewayPort` |

---

## 11. 最终接口基线结论

接口层面的关键冻结如下：

1. **登录统一走 `POST /api/v1/auth/login`**
2. **Invitation 采用单层模型，`POST /api/v1/public/invitations/{token}/accept` 作为幂等接入入口**
3. **`/auth/access` 永远返回 `200`，仅用于状态判断**
4. **非首登强制改密场景下，`admin` 登录后默认进入 `/admin`；普通用户登录后默认进入 `/app`；`workspace-entry` 是唯一工作区跳转入口**
5. **runtime 删除改为 `POST /api/v1/users/me/runtime/delete`，不再依赖 DELETE body**
6. **所有 workspace 子域名必须统一经过平台 session 鉴权**
7. **RuntimeManager internal 接口同步执行，`taskId` 只存在于 Orchestrator 对外层**
8. **V1 runtime 统一使用 `clawloops_shared`、`18789`、固定 alias 与固定镜像**
9. **种子管理员默认密码为 `admin`，首次登录必须先进入 `/force-password-change` 完成改密**
10. **首版不做通用改密与找回密码**

---

v0.12-轻量认证修订
reno  
2026-03-25
