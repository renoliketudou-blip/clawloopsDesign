# ClawLoops 前端 UI 状态模型（冻结版）

## 1. 文档目标

本文基于 `后端/API_Spec.md`、`后端/Architecture_Design.md`、`后端/MVP_Contract.md` 提炼前端可直接落地的 UI 状态模型，目标是让前端在不等待后端实现细节反复澄清的前提下，完成页面、状态管理、路由守卫、错误收口与交互冻结。

适用范围：

- 平台主域控制面
- invitation 接入链路
- post-login 收口页
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
| invitation 预览 | `GET /api/v1/public/invitations/{token}` |
| invitation 启动 | `POST /api/v1/public/invitations/{token}/start` |
| 登录后收口 | `POST /api/v1/auth/post-login` |
| runtime 最终跳转 | `GET /api/v1/workspace-entry` |
| runtime 展示态 | `GET /api/v1/users/me/runtime/status` |
| runtime 完整真相 | `GET /api/v1/users/me/runtime` |
| runtime 异步动作进度 | `GET /api/v1/runtime/tasks/{taskId}` |

### 2.2 前端禁止做的事

- 不解析 Authentik token / JWT 作为业务真相
- 不从前端拼装 `userId`、`subjectId` 作为鉴权输入
- 不把 `browserUrl` 直接当可访问条件，必须先看 `workspace-entry.ready`
- 不把 `task.status`、`observedState`、`ready` 混成一个状态
- 不把 invitation 邮箱校验前移到前端判断
- 不直接调用 `/internal/*`
- 不直接请求 Authentik API 获取业务状态

### 2.3 前端统一状态切片

前端状态树建议冻结为 7 个一级切片：

| 切片 | 作用 | 主要来源 |
| --- | --- | --- |
| `session` | 当前是否已登录、当前用户信息 | `/auth/me` |
| `access` | 当前登录用户是否允许继续访问业务 | `/auth/access` |
| `invitation` | invitation 预览、启动、完成结果 | `/public/invitations/*` + `/auth/post-login` |
| `workspace` | 用户是否已有 workspace、是否需要选择 | `/auth/post-login`、`/workspace-entry` |
| `runtime` | runtime 真相与轻量投影 | `/users/me/runtime`、`/users/me/runtime/status` |
| `runtimeTask` | 启停删任务进度 | `/runtime/tasks/{taskId}` |
| `admin` | 用户治理、invitation 治理、模型治理、provider 凭据、usage | `/admin/*` |

---

## 3. 冻结领域对象

### 3.1 SessionUser

```json
{
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

### 3.3 InvitationPreview

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

前端语义：

- `status` 只认 `pending | consumed | revoked`
- `expired` 不在字段中单独出现，来源于接口错误码
- 预览页只展示，不做业务消费

### 3.4 PostLoginResult

冻结采用附录 A3 返回格式：

```json
{
  "hasWorkspace": true,
  "workspaceId": "ws_xxx",
  "needsWorkspaceSelection": false
}
```

兼容接口示例中的历史字段时，前端可额外容忍：

```json
{
  "status": "ok",
  "userId": "u_001",
  "invitationApplied": true,
  "redirectTo": "/workspace-entry",
  "result": "already_bound_or_consumed"
}
```

冻结建议：

- BFF 适配层对外统一映射为：
  - `hasWorkspace`
  - `workspaceId | null`
  - `needsWorkspaceSelection`
  - `invitationApplied | false`
  - `result | null`

### 3.5 RuntimeStatusProjection

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

前端语义冻结：

- `task.status` 表示动作生命周期
- `observedState` 表示 runtime 资源观测态
- `ready` 表示是否允许跳转
- `browserUrl` 只能在 `ready=true` 且通过 `/workspace-entry` 确认后使用

### 3.6 RuntimeTask

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

前端语义：

- `action` 至少支持 `ensure_running | stop | delete`
- `status` 至少支持 `pending | running | succeeded | failed | canceled`
- 任务页或弹层只能展示进度，不直接决定是否跳转 workspace

### 3.7 WorkspaceEntry

```json
{
  "ready": true,
  "runtimeId": "rt_001",
  "browserUrl": "https://u-001.clawloops.example.com"
}
```

前端语义：

- 这是唯一跳转工作区的依据
- `ready=false` 时必须停留在控制面，不可跳转

---

## 4. 路由级状态机

## 4.1 应用启动状态机

适用入口：

- `/`
- `/app`
- `/admin/*`
- `/workspace-entry`

状态机：

1. `booting`
2. `checkingSession`
3. `unauthenticated`
4. `checkingAccess`
5. `disabledBlocked`
6. `authenticatedReady`
7. `bootstrapFailed`

转移规则：

| 当前状态 | 事件 | 下一状态 | UI 行为 |
| --- | --- | --- | --- |
| `booting` | 应用加载 | `checkingSession` | 显示全局骨架屏 |
| `checkingSession` | `/auth/me` 返回未登录或 401 | `unauthenticated` | 跳登录页或统一登录入口 |
| `checkingSession` | `/auth/me` 返回已登录 | `checkingAccess` | 并发拉取业务准入 |
| `checkingAccess` | `allowed=true` | `authenticatedReady` | 继续页面初始化 |
| `checkingAccess` | `allowed=false && reason=USER_DISABLED` | `disabledBlocked` | 显示账号已禁用页 |
| `checkingSession` 或 `checkingAccess` | 网络或 5xx | `bootstrapFailed` | 显示重试页 |

冻结要求：

- 除公开页外，页面必须先过这套状态机
- `admin` 页面通过 `session.user.role === admin` 额外判定

## 4.2 invitation 页状态机

路由：`/invite/:token`

状态机：

1. `loadingPreview`
2. `previewValid`
3. `previewInvalid`
4. `starting`
5. `redirectingToAuthentik`
6. `startFailed`

转移规则：

| 当前状态 | 事件 | 下一状态 | UI 行为 |
| --- | --- | --- | --- |
| `loadingPreview` | 预览成功且 `valid=true` | `previewValid` | 展示邮箱、workspace、角色、有效期 |
| `loadingPreview` | 404/409/410/422 | `previewInvalid` | 渲染不可继续页面 |
| `previewValid` | 点击继续接入 | `starting` | 按钮禁用，显示提交中 |
| `starting` | 成功返回 `redirectUrl` | `redirectingToAuthentik` | 浏览器整页跳转 |
| `starting` | 失败 | `startFailed` | 显示可重试错误 |

错误映射冻结：

| code | 页面态 |
| --- | --- |
| `INVITATION_NOT_FOUND` | 链接不存在 |
| `INVITATION_ALREADY_CONSUMED` | 链接已使用 |
| `INVITATION_REVOKED` | 邀请已撤销 |
| `INVITATION_EXPIRED` | 邀请已过期 |
| `INVITATION_WORKSPACE_INVALID` | 邀请对应工作区无效 |
| `INVITATION_ERROR` | 系统暂时无法发起接入 |

注意：

- `start` 可重试
- 前端不持有 `pendingInvitationSessionId` 作为必需参数
- pending invitation 上下文优先在 cookie / server session

## 4.3 post-login 收口页状态机

路由：`/post-login` 或登录后前端回跳入口页

状态机：

1. `initializing`
2. `callingPostLogin`
3. `postLoginSucceeded`
4. `workspaceMissing`
5. `needsWorkspaceSelection`
6. `postLoginFailed`

转移规则：

| 当前状态 | 事件 | 下一状态 | UI 行为 |
| --- | --- | --- | --- |
| `initializing` | 拿到登录态 | `callingPostLogin` | 显示“正在完成登录” |
| `callingPostLogin` | `hasWorkspace=true && needsWorkspaceSelection=false` | `postLoginSucceeded` | 跳到工作台入口页 |
| `callingPostLogin` | `hasWorkspace=true && needsWorkspaceSelection=true` | `needsWorkspaceSelection` | 跳 `workspace-entry` |
| `callingPostLogin` | `hasWorkspace=false` | `workspaceMissing` | 显示无可用工作区 |
| `callingPostLogin` | 4xx/5xx | `postLoginFailed` | 显示明确错误和重试 |

错误映射冻结：

| code | 页面态 |
| --- | --- |
| `INVITATION_EMAIL_MISMATCH` | 当前登录邮箱与邀请邮箱不匹配 |
| `INVITATION_REVOKED` | 邀请已失效 |
| `INVITATION_ALREADY_CONSUMED` | 邀请已被消费，但可继续登录收口 |
| `INVITATION_EXPIRED` | 邀请已过期 |
| `INVITATION_WORKSPACE_INVALID` | 目标工作区不可用 |
| `USER_SYNC_ERROR` | 登录成功但用户同步失败 |
| `USER_DISABLED` | 登录成功但账号被禁用 |
| `INVITATION_ERROR` | 接入流程执行失败 |

冻结规则：

- `post-login` 由前端主动触发
- 请求来源是真实登录会话，不能依赖前端伪造 `userId`
- 即使页面刷新也允许重复调用，不应产生重复副作用

## 4.4 workspace-entry 状态机

路由：`/workspace-entry`

状态机：

1. `loadingEntry`
2. `readyToRedirect`
3. `runtimeNotReady`
4. `noWorkspace`
5. `entryFailed`

转移规则：

| 当前状态 | 事件 | 下一状态 | UI 行为 |
| --- | --- | --- | --- |
| `loadingEntry` | `ready=true` | `readyToRedirect` | 立即跳转 `browserUrl` |
| `loadingEntry` | `ready=false` 且有 runtime | `runtimeNotReady` | 展示 runtime 准备中 |
| `loadingEntry` | `hasWorkspace=false` 或业务无工作区 | `noWorkspace` | 显示无工作区提示 |
| `loadingEntry` | 失败 | `entryFailed` | 展示重试 |

冻结规则：

- 此页不依赖 `runtime/status` 决定跳转
- 最终跳转判断只看 `workspace-entry.ready`

## 4.5 用户工作台状态机

路由：`/app`

状态机分两层。

第一层为页面初始化：

1. `dashboardBooting`
2. `dashboardReady`
3. `dashboardError`

第二层为 runtime 卡片：

1. `runtimeUnknown`
2. `runtimeCreating`
3. `runtimeRunning`
4. `runtimeStopped`
5. `runtimeError`
6. `runtimeDeleting`

映射规则：

| 字段组合 | runtime UI 状态 |
| --- | --- |
| 无数据或首次加载 | `runtimeUnknown` |
| `observedState=creating` | `runtimeCreating` |
| `observedState=running` 且 `ready=false` | `runtimeCreating` |
| `observedState=running` 且 `ready=true` | `runtimeRunning` |
| `observedState=stopped` | `runtimeStopped` |
| `observedState=deleted` | `runtimeStopped` |
| `observedState=error` 或 `lastError!=null` | `runtimeError` |

动作互斥冻结：

- `start` 进行中时，禁用 `stop/delete/open`
- `stop` 进行中时，禁用 `start/delete/open`
- `delete` 进行中时，禁用 `start/stop/open`
- `ready=false` 时禁用“进入工作区”

## 4.6 管理后台状态机

### 用户列表页

状态：

- `loading`
- `ready`
- `empty`
- `error`
- `patchingStatus`

冻结字段：

- `userId`
- `subjectId`
- `role`
- `status`
- `authMethod`
- `runtimeObservedState`
- `lastLoginAt`

### 用户详情页

状态：

- `loadingProfile`
- `profileReady`
- `loadingRuntime`
- `runtimeReady`
- `error`

冻结动作：

- 禁用 / 启用用户
- 查看 runtime 详情

### invitation 管理页

状态：

- `loadingList`
- `listReady`
- `creatingInvitation`
- `revokingInvitation`
- `resendingInvitation`
- `error`

冻结字段建议：

- `invitationId`
- `targetEmail`
- `workspaceId`
- `role`
- `status`
- `expiresAt`
- `consumedAt`
- `consumedByUserId`
- `lastError`

### 模型治理页

状态：

- `loadingModels`
- `modelsReady`
- `savingModelPolicy`
- `error`

由于后端文档未展开更细字段，前端冻结到“列表 + 开关/策略编辑抽屉”层级，不冻结更细表单布局。

### provider 凭据页

状态：

- `loadingCredentials`
- `credentialsReady`
- `creatingCredential`
- `verifyingCredential`
- `deletingCredential`
- `error`

### usage 汇总页

状态：

- `loadingSummary`
- `summaryReady`
- `empty`
- `error`

---

## 5. 组件级共享状态模型

### 5.1 AsyncResource 统一模型

建议所有查询型资源统一为：

```ts
type AsyncResource<T> =
  | { phase: 'idle'; data: null; error: null }
  | { phase: 'loading'; data: T | null; error: null }
  | { phase: 'success'; data: T; error: null }
  | { phase: 'error'; data: T | null; error: AppError };
```

### 5.2 MutationState 统一模型

```ts
type MutationState =
  | { phase: 'idle' }
  | { phase: 'submitting' }
  | { phase: 'success' }
  | { phase: 'error'; error: AppError };
```

### 5.3 AppError 冻结模型

```ts
type AppError = {
  httpStatus: number;
  code: string;
  message: string;
};
```

冻结要求：

- 任何页面错误展示都必须保留 `code`
- 不允许只保留中文文案而丢失错误码

---

## 6. 前端必须处理的错误码

| code | UI 处理策略 |
| --- | --- |
| `UNAUTHENTICATED` | 回登录入口 |
| `ACCESS_DENIED` | 403 无权限页 |
| `USER_DISABLED` | 账号禁用拦截页 |
| `USER_NOT_FOUND` | 显示账号未同步错误并允许重试 |
| `INVITATION_NOT_FOUND` | invitation 无效页 |
| `INVITATION_ALREADY_CONSUMED` | invitation 已使用页 |
| `INVITATION_REVOKED` | invitation 已撤销页 |
| `INVITATION_EXPIRED` | invitation 已过期页 |
| `INVITATION_EMAIL_MISMATCH` | post-login 邮箱不匹配页 |
| `INVITATION_WORKSPACE_INVALID` | 工作区无效页 |
| `INVITATION_ERROR` | invitation 系统错误页 |
| `USER_SYNC_ERROR` | 登录收口失败页 |
| `RUNTIME_ACTION_CONFLICT` | runtime 动作冲突 toast + 刷新状态 |
| `RUNTIME_CONTRACT_DRIFT` | runtime contract 漂移告警页 |
| `RUNTIME_START_FAILED` | 启动失败态 |
| `RUNTIME_STOP_FAILED` | 停止失败态 |
| `RUNTIME_DELETE_FAILED` | 删除失败态 |
| `MODEL_NOT_FOUND` | 模型配置失效提醒 |
| `QUOTA_EXCEEDED` | quota 限额提示 |

---

## 7. 页面可冻结的最终结论

前端可以据此冻结以下事实：

1. 登录后所有业务页先跑 `auth/me -> auth/access`
2. invitation 链路固定为 `preview -> start -> Authentik -> post-login`
3. `post-login` 是前端主动调用的 BFF 收口接口
4. runtime 展示、任务进度、最终跳转是三种不同状态源
5. `workspace-entry` 是唯一跳转工作区入口
6. 用户工作台与管理后台均可按本文状态切片直接建模
7. 错误码、字段名、状态枚举无需等待后端二次澄清

若后端实现与本文不一致，以后端三份冻结文档中的同名字段和错误码为准，但不得突破本文列出的前端边界。
