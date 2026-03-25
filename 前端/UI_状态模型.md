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
| runtime 最终跳转 | `GET /api/v1/workspace-entry` |
| runtime 展示态 | `GET /api/v1/users/me/runtime/status` |
| runtime 完整真相 | `GET /api/v1/users/me/runtime` |
| runtime 异步动作进度 | `GET /api/v1/runtime/tasks/{taskId}` |

### 2.2 前端禁止做的事

- 不解析 session cookie 作为业务真相
- 不从前端拼装 `userId`、`subjectId` 作为鉴权输入
- 不把 `browserUrl` 直接当可访问条件，必须先看 `workspace-entry.ready`
- 不把 `task.status`、`observedState`、`ready` 混成一个状态
- 不直接调用 `/internal/*`
- 不把 `/app` 当成纯中转页
- 不新增首版改密、找回密码、第三方登录入口

### 2.3 前端统一状态切片

前端状态树建议冻结为 6 个一级切片：

| 切片 | 作用 | 主要来源 |
| --- | --- | --- |
| `session` | 当前是否已登录、当前用户信息 | `/auth/me` |
| `access` | 当前登录用户是否允许继续访问业务 | `/auth/access` |
| `invitation` | invitation 预览、接受、完成结果 | `/public/invitations/*` |
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
  "isDisabled": false
}
```

前端语义：

- `role` 只认 `admin | user`
- `status` 只认 `active | disabled`
- `isAdmin` 用于菜单显隐
- `isDisabled` 只用于展示，业务通行以 `/auth/access` 为准

### 3.2 AccessGate

```json
{
  "allowed": true,
  "reason": null
}
```

前端语义：

- `allowed=false` 且 `reason=USER_DISABLED` 时，进入禁用拦截态
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
  "features": {
    "passwordChange": false,
    "passwordRecovery": false,
    "thirdPartyLogin": false
  }
}
```

前端语义：

- `methods` 首版只渲染一个入口
- `passwordChange=false` 和 `passwordRecovery=false` 时，不显示相关入口

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

## 4.2 `/invite/:token`

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
- `accepted=true` 后进入 `accepted` 并跳转 `/app`

## 4.3 `/app`

状态：

- `bootstrapping`
- `accessDenied`
- `firstJoinConfirmed`
- `normalDashboard`

前端要求：

- 首次接入成功后的确认信息在 `/app` 内承接
- 不额外做 `/post-login`

## 4.4 `/workspace-entry`

状态：

- `checking`
- `notReady`
- `readyToRedirect`
- `error`

---

## 5. 路由守卫冻结

### 5.1 公开路由

- `/login`
- `/invite/:token`

### 5.2 登录用户路由

- `/app`
- `/workspace-entry`

### 5.3 管理员路由

- `/admin`
- `/admin/*`

守卫顺序建议：

1. 判断是否需要登录
2. 读取 `/auth/me`
3. 读取 `/auth/access`
4. 若需 admin，再判断 `isAdmin`

---

## 6. 错误码到 UI 的映射

| 错误码 | 页面 | UI 动作 |
| --- | --- | --- |
| `UNAUTHENTICATED` | 已登录页 | 回 `/login` |
| `INVALID_CREDENTIALS` | `/login` | 表单级报错 |
| `USER_DISABLED` | 登录页/已登录页 | 显示禁用说明 |
| `ACCESS_DENIED` | `/admin/*` | 403 页面 |
| `INVITATION_NOT_FOUND` | `/invite/:token` | 不存在页 |
| `INVITATION_EXPIRED` | `/invite/:token` | 已过期页 |
| `INVITATION_REVOKED` | `/invite/:token` | 已撤销页 |
| `INVITATION_ALREADY_CONSUMED` | `/invite/:token` | 已消费页 |
| `INVITATION_USERNAME_MISMATCH` | `/invite/:token` | 保持表单并提示正确用户名 |
| `INVITATION_PASSWORD_INVALID` | `/invite/:token` | 密码规则错误态 |

---

## 7. 首版不做

- 改密页面
- 找回密码页面
- 第三方登录按钮
- 登录后收口页
- 前端自行管理 token 生命周期

---

## 8. 最终状态模型结论

前端只需要记住：

1. `session` 和 `access` 是登录后唯一真相
2. invitation 接入在 `/invite/:token` 内完成闭环
3. `/app` 承接首次接入成功确认
4. `/workspace-entry` 只做最终跳转
5. 首版不做改密与找回密码

---

v0.5-轻量认证修订  
2026-03-25
