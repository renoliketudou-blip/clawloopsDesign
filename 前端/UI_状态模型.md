# ClawLoops 前端 UI 状态模型（冻结版，轻量认证修订）

## 1. 文档目标

本文基于 `后端/API_Spec.md`、`后端/Architecture_Design.md`、`后端/MVP_Contract.md` 提炼前端可直接落地的 UI 状态模型，目标是让前端在不等待后端实现细节反复澄清的前提下，完成页面、状态管理、路由守卫、错误收口与交互冻结。

适用范围：

- 平台主域控制面
- invitation 接入链路
- 用户工作台与 workspace-entry 跳转
- 管理后台

本文只描述前端可见状态真相，不扩展后端未冻结的业务规则。

---

## 2. 全局冻结原则

### 2.1 前端只信任以下接口语义

| 领域 | 唯一可信接口或字段 |
| --- | --- |
| 登录身份 | `GET /api/v1/auth/me` |
| 是否允许访问业务 | `GET /api/v1/auth/access` |
| 登录方式 | `GET /api/v1/auth/options` |
| invitation 预览 | `GET /api/v1/public/invitations/{token}` |
| invitation 接受 | `POST /api/v1/public/invitations/{token}/accept` |
| admin 首页 | `GET /api/v1/admin/home` |
| 公共区域列表 | `GET /api/v1/public-area/files/list` |
| 公共区域写操作 | `POST /api/v1/public-area/files/mkdir`、`POST /api/v1/public-area/files/upload`、`DELETE /api/v1/public-area/files/delete` |
| runtime 最终跳转 | `GET /api/v1/workspace-entry` |
| runtime 展示态 | `GET /api/v1/users/me/runtime/status` |
| runtime 完整真相 | `GET /api/v1/users/me/runtime` |
| runtime 异步动作进度 | `GET /api/v1/runtime/tasks/{taskId}` |

### 2.2 前端禁止做的事

- 不解析 session cookie 作为业务真相
- 不读取、写入或删除 `clawloops_session`
- 不从前端拼装 `userId`、`subjectId` 作为鉴权输入
- 不把 `browserUrl` 直接当可访问条件，必须先看 `workspace-entry.ready`
- 不把 `task.status`、`observedState`、`ready` 混成一个状态
- 不直接调用 `/internal/*`
- 不向 `browserUrl` 拼接 token、ticket、`userId` 或其他鉴权参数
- 不把 `/app` 当成纯中转页
- 不新增通用改密、找回密码、第三方登录入口

### 2.3 前端统一状态切片

前端状态树建议冻结为 7 个一级切片：

| 切片 | 作用 | 主要来源 |
| --- | --- | --- |
| `session` | 当前是否已登录、当前用户信息 | `/auth/me` |
| `access` | 当前登录用户是否允许继续访问业务 | `/auth/access` |
| `invitation` | invitation 预览、接受、完成结果 | `/public/invitations/*` |
| `publicArea` | 公共区域路径、分页、列表与动作态 | `/public-area/files/*` |
| `workspace` | 用户是否已有 workspace、最终跳转状态 | `/workspace-entry` |
| `runtime` | runtime 真相与轻量投影 | `/users/me/runtime`、`/users/me/runtime/status` |
| `admin` | 用户治理、invitation 治理、模型治理、provider 凭据、usage | `/admin/*` |

---

## 3. 冻结领域对象

### 3.1 SessionUser

```json
{
  "userId": "u_001",
  "subjectId": "clawloops:u_001",
  "username": "emp001",
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
```

前端语义：

- `role` 只认 `admin | user`
- `status` 只认 `active | disabled`
- `isAdmin` 用于菜单显隐
- `isDisabled` 只用于展示，业务通行以 `/auth/access` 为准
- `mustChangePassword=true` 时，前端必须优先收口到 `/force-password-change`

### 3.2 AccessGate

```json
{
  "allowed": true,
  "reason": null
}
```

前端语义：

- `allowed=false` 且 `reason=USER_DISABLED` 时，进入禁用拦截态
- `allowed=false` 且 `reason=PASSWORD_CHANGE_REQUIRED` 时，进入强制改密收口态
- `reason` 首版只认 `USER_DISABLED | PASSWORD_CHANGE_REQUIRED | null`
- 即使返回 `200`，也不能继续进入业务页面

### 3.3 AuthOptions

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

前端语义：

- `methods` 首版只渲染一个入口
- `passwordPolicy` 是登录页、邀请设密页、强制改密页展示密码规则提示的唯一真相
- `forcedPasswordChange=true` 只表示存在受限强制改密流，不表示前端开放通用改密入口
- `passwordRecovery=false` 时，不显示找回密码入口

### 3.4 InvitationPreview

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

前端语义：

- `status` 只认 `pending | consumed | revoked`
- `expired` 不在字段中单独出现，来源于接口错误码
- 预览页只展示，不做业务消费
- 若存在 `loginUsername`，页面必须优先把它作为“推荐登录账号”展示
- 若 `targetEmail` 是代理邮箱，普通用户界面不应把它作为主说明文案

### 3.5 InvitationAcceptResult

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

前端语义：

- 成功后不再进入中转页
- `redirectTo` 是唯一跳转依据
- `workspaceBinding` 用于 `/app` 首屏确认“已加入哪个 workspace”
- `replayed=true` 表示命中幂等重放成功，前端仍按普通成功态处理，不额外提示重复提交错误

### 3.6 WorkspaceEntry

```json
{
  "workspaceId": "ws_001",
  "runtimeId": "rt_001",
  "observedState": "running",
  "ready": true,
  "browserUrl": "https://ws-001.clawloops.app"
}
```

前端语义：

- `ready=true` 才允许跳转
- `browserUrl` 不可在别处直接复用作最终跳转依据
- 跳转时不附加额外鉴权参数，workspace 子域访问权限依赖浏览器当前已持有的平台 session

### 3.7 PasswordChangeResult

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

前端语义：

- 该结果只用于首登强制改密成功后的跳转
- `redirectTo` 是唯一后续落点依据

### 3.8 PublicAreaListResult

```json
{
  "entries": [
    {
      "name": "docs",
      "type": "directory",
      "size": 0,
      "updatedAt": "2026-04-10T09:30:00Z"
    },
    {
      "name": "readme.pdf",
      "type": "file",
      "size": 102400,
      "updatedAt": "2026-04-10T09:35:00Z"
    }
  ],
  "page": 1,
  "pageSize": 10,
  "total": 21,
  "totalPages": 3
}
```

前端语义：

- `entries` 为当前目录分页结果
- 排序信任后端（目录优先、名称升序）
- `pageSize` 首版固定为 10
- 路径状态由路由 query `path/page` 承载
- 路径栏展示完整绝对路径，但权限仍由后端判定

---

## 4. 页面级状态机

## 4.1 `/login`

状态：

- `idle`
- `submitting`
- `successRedirecting`
- `invalidCredentials`
- `disabled`
- `systemError`

切换规则：

- 点击登录后进入 `submitting`
- 成功后进入 `successRedirecting`
- `INVALID_CREDENTIALS` 进入 `invalidCredentials`
- `USER_DISABLED` 进入 `disabled`
- 若 `mustChangePassword=true` 或 `redirectTo=/force-password-change`，必须立刻跳转强制改密页

## 4.2 `/force-password-change`

状态：

- `idle`
- `submitting`
- `successRedirecting`
- `invalidCurrentPassword`
- `passwordInvalid`
- `systemError`

切换规则：

- 页面仅对已登录且 `mustChangePassword=true` 的用户可见
- 提交成功后进入 `successRedirecting` 并跳转 `/admin`
- `CURRENT_PASSWORD_INCORRECT` 进入 `invalidCurrentPassword`
- `PASSWORD_CHANGE_INVALID` 留在当前页并展示表单级错误

## 4.3 `/disabled`

状态：

- `shown`

切换规则：

- 当 `/auth/access.reason=USER_DISABLED` 时进入该页
- 页面不再继续请求业务数据

## 4.4 `/403`

状态：

- `shown`

切换规则：

- 当访问后台或其他受限页命中 `ACCESS_DENIED` 时进入该页

## 4.5 `/invite/:token`

状态：

- `loadingPreview`
- `previewReady`
- `submitting`
- `accepted`
- `notFound`
- `expired`
- `revoked`
- `consumed`
- `usernameMismatch`
- `passwordInvalid`
- `systemError`

切换规则：

- 页面先进入 `loadingPreview`
- 预览成功后进入 `previewReady`
- 提交后进入 `submitting`
- `accepted=true` 后进入 `accepted` 并按 `redirectTo` 跳转
- `replayed=true` 不单独建错误态，仍归入 `accepted`

## 4.6 `/app`

状态：

- `bootstrapping`
- `accessDenied`
- `firstJoinConfirmed`
- `normalDashboard`

前端要求：

- 首次接入成功后的确认信息在 `/app` 内承接
- 不额外做 `/post-login`

## 4.7 `/workspace-entry`

状态：

- `checking`
- `notReady`
- `readyToRedirect`
- `error`

## 4.8 `/public-area` 与 `/admin/public-area`

状态：

- `loadingList`
- `listReady`
- `submittingMkdir`
- `submittingUpload`
- `submittingDelete`
- `conflict`
- `forbidden`
- `systemError`

切换规则：

- 首屏进入 `loadingList`，列表成功后进入 `listReady`
- `overwrite=false` 同名冲突进入 `conflict`
- 非 admin 触发覆盖上传/删除进入 `forbidden`
- 回退越级由前端阻断，保持在 `listReady`

---

## 5. 路由守卫冻结

### 5.1 公开路由

- `/login`
- `/disabled`
- `/403`
- `/invite/:token`

### 5.2 强制改密路由

- `/force-password-change`

### 5.3 登录用户路由

- `/app`
- `/workspace-entry`

### 5.4 管理员路由

- `/admin`
- `/admin/*`

守卫顺序建议：

1. 判断是否需要登录
2. 读取 `/auth/me`
3. 若 `mustChangePassword=true`，除 `/force-password-change` 外全部重定向
4. 对其他业务页读取 `/auth/access`
5. 若需 admin，再判断 `isAdmin`

---

## 6. 错误码到 UI 的映射

| 错误码 | 页面 | UI 动作 |
| --- | --- | --- |
| `UNAUTHENTICATED` | 已登录页 | 回 `/login` |
| `INVALID_CREDENTIALS` | `/login` | 表单级报错 |
| `USER_DISABLED` | 登录页/已登录页 | 跳 `/disabled` 或显示禁用说明 |
| `PASSWORD_CHANGE_REQUIRED` | 已登录页 | 立即跳 `/force-password-change` |
| `CURRENT_PASSWORD_INCORRECT` | `/force-password-change` | 高亮当前密码输入并报错 |
| `PASSWORD_CHANGE_INVALID` | `/force-password-change` | 表单级报错 |
| `ACCESS_DENIED` | `/admin/*` | 跳 `/403` |
| `INVITATION_NOT_FOUND` | `/invite/:token` | 不存在页 |
| `INVITATION_EXPIRED` | `/invite/:token` | 已过期页 |
| `INVITATION_REVOKED` | `/invite/:token` | 已撤销页 |
| `INVITATION_ALREADY_CONSUMED` | `/invite/:token` | 已消费页 |
| `INVITATION_USERNAME_MISMATCH` | `/invite/:token` | 保持表单并提示正确用户名 |
| `INVITATION_PASSWORD_INVALID` | `/invite/:token` | 密码规则错误态 |

---

## 7. 首版不做

- 通用改密页面
- 找回密码页面
- 第三方登录按钮
- 登录后收口页
- 前端自行管理 token 生命周期

---

## 8. 最终状态模型结论

前端只需要记住：

1. `session` 和 `access` 是登录后唯一真相
2. 若 `mustChangePassword=true`，必须先进入 `/force-password-change`
3. invitation 接入在 `/invite/:token` 内完成闭环
4. `/app` 承接首次接入成功确认
5. `/workspace-entry` 只做最终跳转
6. 公共区域状态由 `publicArea` 切片统一管理
7. 首版不做通用改密与找回密码

---

v0.5-轻量认证修订  
2026-03-25
