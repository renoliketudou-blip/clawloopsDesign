# 4. 接口到页面的数据映射文档

## 4.1 文档目标

本文档用于把后端接口字段映射到前端页面、前端视图模型、交互动作和展示组件，减少以下问题：

- 前后端对字段理解不一致
- 前端重复命名或命名漂移
- 页面拿到了接口却不知道如何组织成 UI
- 一个字段在不同页面显示不同文案

本文档默认以前端首版页面体系为基线。

---

## 4.2 全局映射原则

### 4.2.1 不改后端字段名

首版禁止前端在核心领域字段上自行改名后再到处混用。

例如：

- `subjectId` 不改成 `externalUid`
- `invitationId` 不改成 `inviteId`
- `browserUrl` 不改成 `workspaceUrl`
- `observedState` 不改成 `status`

### 4.2.2 前端允许新增 view model

允许前端基于接口响应组装展示用字段，例如：

- `isRuntimeReady`
- `canStartRuntime`
- `canRevokeInvitation`
- `userDisplayName`

但这些 view model 不得回写后端，不得替代原始字段。

### 4.2.3 枚举统一映射

所有枚举建议通过统一 mapper 转中文文案与视觉语义，不要在页面里散落 if-else。

---

## 4.3 认证域数据映射

## 4.3.1 `GET /api/v1/auth/me`

### 原始响应

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

### 页面映射

| 页面 | 用途 |
| --- | --- |
| 所有受保护页面 | 顶层鉴权 |
| `/workspace` | 身份摘要 |
| `/admin/*` | 管理员守卫 |
| `/account-disabled` | 判断是否禁用 |

### 推荐前端 view model

```ts
interface CurrentUserVM {
  authenticated: boolean
  userId?: string
  subjectId?: string
  role?: 'user' | 'admin'
  status?: 'active' | 'disabled'
  authProviderLabel?: string
  authMethodLabel?: string
  isAdmin?: boolean
  isDisabled?: boolean
}
```

### 字段映射规则

| 后端字段 | 前端字段 / 展示 |
| --- | --- |
| `authenticated` | 是否进入受保护区 |
| `user.userId` | 顶层用户标识 |
| `user.subjectId` | 管理后台详情展示 |
| `user.role` | 路由守卫、角色 badge |
| `user.status` | 顶层禁用判断 |
| `user.auth.provider` | “身份提供方”显示 |
| `user.auth.method` | “登录方式”显示 |
| `user.isAdmin` | 导航显隐 |
| `user.isDisabled` | 强制跳转 `/account-disabled` |

---

## 4.3.2 `GET /api/v1/auth/access`

### 原始响应

```json
{
  "allowed": true,
  "reason": null
}
```

### 页面映射

| 页面 | 用途 |
| --- | --- |
| 所有受保护页面 | 二次访问校验 |

### 规则

| 条件 | 前端动作 |
| --- | --- |
| `allowed=true` | 放行业务页 |
| `allowed=false && reason=USER_DISABLED` | 跳 `/account-disabled` |
| `allowed=false` 其他原因 | 跳 `/403` |

---

## 4.3.3 `GET /api/v1/auth/options`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/login` | 显示首版登录方式 |

### 推荐 view model

```ts
interface AuthMethodVM {
  type: string
  enabled: boolean
  label: string
}
```

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `provider` | 登录说明文案 |
| `methods[]` | 登录方式列表 |
| `futureMethods[]` | 次级说明，不作为实际入口 |

---

## 4.3.4 `GET /api/v1/auth/post-login`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/auth/landing` | 登录完成收口 |

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `status` | 是否完成收口 |
| `userId` | 当前用户确认 |
| `invitationApplied` | 显示“邀请已应用”提示 |
| `redirectTo` | 跳转目标 |

---

## 4.4 invitation 域数据映射

## 4.4.1 `GET /api/v1/public/invitations/{token}`

### 原始响应（有效）

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

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/invite/:token` | 邀请预览页 |

### 推荐 view model

```ts
interface InvitationPreviewVM {
  valid: boolean
  targetEmail?: string
  workspaceId?: string
  workspaceName?: string
  role?: string
  expiresAt?: string
  invalidCode?: string
  invalidMessage?: string
}
```

### 字段映射规则

| 后端字段 | 前端展示 |
| --- | --- |
| `valid` | 页面走有效态还是无效态 |
| `invitation.targetEmail` | 被邀请邮箱 |
| `invitation.workspaceName` | 目标工作区名称 |
| `invitation.role` | 将获得的角色 |
| `invitation.expiresAt` | 到期时间 |
| `code` | 无效原因 |
| `message` | 无效态补充说明 |

---

## 4.4.2 `POST /api/v1/public/invitations/{token}/start`

### 原始响应

```json
{
  "status": "accepted",
  "redirectUrl": "https://auth.example.com/if/flow/clawloops-invitation-enrollment/?itoken=xxxx"
}
```

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/invite/:token` | 点击继续接入后的跳转 |

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `status` | 判断是否可继续 |
| `redirectUrl` | 浏览器跳转地址 |

### 交互要求

- 该接口成功后直接 `window.location.href = redirectUrl`
- 不在前端缓存 `redirectUrl`

---

## 4.4.3 `GET /api/v1/admin/invitations`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/invitations` | 列表页 |

### 推荐列表行 view model

```ts
interface InvitationRowVM {
  invitationId: string
  targetEmail: string
  workspaceId: string
  role: string
  status: 'pending' | 'consumed' | 'revoked' | 'expired'
  expiresAt: string
  statusLabel: string
  canRevoke: boolean
  canResend: boolean
}
```

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `invitationId` | 行主键、详情跳转 |
| `targetEmail` | 主识别字段 |
| `workspaceId` | 辅助信息 |
| `role` | badge 或文本 |
| `status` | 状态 badge |
| `expiresAt` | 到期时间 |

---

## 4.4.4 `POST /api/v1/admin/invitations`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/invitations` | 创建 invitation |

### 请求字段到表单映射

| 表单字段 | 后端字段 |
| --- | --- |
| 目标邮箱 | `targetEmail` |
| 目标工作区 | `workspaceId` |
| 角色 | `role` |
| 有效期（小时） | `expiresInHours` |

### 响应字段到结果 UI 映射

| 后端字段 | 前端展示 |
| --- | --- |
| `invitationId` | 创建成功结果 |
| `status` | 初始状态 |
| `inviteUrl` | 复制链接 |
| `targetEmail` | 成功结果卡 |
| `workspaceId` | 成功结果卡 |
| `role` | 成功结果卡 |
| `expiresAt` | 成功结果卡 |

---

## 4.4.5 `GET /api/v1/admin/invitations/{invitationId}`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/invitations/:invitationId` | 详情页 |

### 推荐详情 view model

```ts
interface InvitationDetailVM {
  invitationId: string
  targetEmail: string
  workspaceId: string
  role: string
  status: string
  authentikInvitationRef?: string | null
  expiresAt: string
  consumedAt?: string | null
  consumedByUserId?: string | null
  lastError?: string | null
  canRevoke: boolean
  canResend: boolean
}
```

---

## 4.5 runtime 域数据映射

## 4.5.1 `GET /api/v1/users/me/runtime`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/workspace/runtime` | runtime 管理页 |

### 原始结构要点

```json
{
  "userId": "u_001",
  "runtime": {
    "runtimeId": "rt_001",
    "volumeId": "vol_001",
    "imageRef": "clawloops-runtime-wrapper:openclaw-1.0.0",
    "desiredState": "running",
    "observedState": "running",
    "browserUrl": "https://u-001.clawloops.example.com",
    "retentionPolicy": "preserve_workspace",
    "lastError": null
  }
}
```

### 推荐 view model

```ts
interface RuntimeDetailVM {
  runtimeId: string
  volumeId: string
  imageRef: string
  desiredState: string
  observedState: string
  browserUrl?: string
  retentionPolicy: 'preserve_workspace' | 'wipe_workspace'
  lastError?: string | null
  canStart: boolean
  canStop: boolean
  canDelete: boolean
  isReady: boolean
}
```

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `runtime.runtimeId` | 详情标识 |
| `runtime.volumeId` | 排障 / 详情展示 |
| `runtime.imageRef` | 镜像版本展示 |
| `runtime.desiredState` | 目标状态 |
| `runtime.observedState` | 当前状态 badge |
| `runtime.browserUrl` | 进入工作区 |
| `runtime.retentionPolicy` | 删除对话框默认值 |
| `runtime.lastError` | 错误块 |

---

## 4.5.2 `GET /api/v1/users/me/runtime/status`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/workspace` | 首页 runtime 摘要 |
| `/workspace/runtime` | 轻量刷新 |

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `runtimeId` | 当前 runtime 标识 |
| `desiredState` | 状态文案 |
| `observedState` | 状态 badge |
| `ready` | 是否可进入工作区 |
| `browserUrl` | ready 时显示“进入工作区” |
| `reason` | 不可进入原因 |
| `lastError` | 错误说明 |

---

## 4.5.3 `POST /api/v1/users/me/runtime/start`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/workspace` | 首页快捷启动 |
| `/workspace/runtime` | 正式启动动作 |

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `taskId` | 进入轮询 |
| `action` | 任务说明 |
| `status` | 结果 toast |

---

## 4.5.4 `POST /api/v1/users/me/runtime/stop`

映射规则与 start 相同，仅 action 变为 `stop`。

---

## 4.5.5 `DELETE /api/v1/users/me/runtime`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/workspace/runtime` | 删除 runtime |

### 请求字段到表单映射

| 前端字段 | 后端字段 |
| --- | --- |
| 删除策略 | `retentionPolicy` |

---

## 4.5.6 `GET /api/v1/runtime/tasks/{taskId}`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/workspace/runtime` | task 轮询 |
| `/workspace` | 首页快捷动作轮询 |

### 推荐 view model

```ts
interface RuntimeTaskVM {
  taskId: string
  userId: string
  runtimeId: string
  action: string
  status: 'pending' | 'running' | 'succeeded' | 'failed' | 'canceled'
  message?: string
}
```

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `taskId` | 轮询键 |
| `action` | 显示当前动作 |
| `status` | 轮询状态 |
| `message` | 状态说明 |

---

## 4.5.7 `GET /api/v1/workspace-entry`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/workspace/entry` | 工作区入口 |
| `/workspace` | 首页快捷进入 |

### 字段映射规则

| 后端字段 | 前端用途 |
| --- | --- |
| `ready` | 是否可进入 |
| `runtimeId` | 当前 workspace 对应 runtime |
| `browserUrl` | 跳转地址 |

### 前端派生字段建议

```ts
const canEnterWorkspace = ready === true && !!browserUrl
```

---

## 4.6 模型与 quota 域数据映射

## 4.6.1 `GET /api/v1/models`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/workspace/models` | 模型列表 |
| `/workspace` | 首页模型摘要 |

### 说明

后端文档当前强调“普通用户侧只读展示当前用户可见模型”，前端应按以下方式处理：

- 模型列表页：完整展示
- 首页：只展示前 N 个模型或汇总数量

### 建议 view model

```ts
interface ModelItemVM {
  modelId: string
  displayName: string
  provider?: string
  enabled?: boolean
  visibilityLabel?: string
}
```

> 具体字段以实际接口响应为准，但前端不要自行推断可编辑能力。

---

## 4.6.2 `GET /api/v1/users/me/quota`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/workspace/quota` | quota 详情 |
| `/workspace` | quota 摘要 |

### 建议 view model

```ts
interface QuotaVM {
  total?: number
  used?: number
  remaining?: number
  unit?: string
  exceeded?: boolean
}
```

> 该接口在契约中存在，但示例字段未完全展开，前端应根据后端最终响应做一层 adapter，不应把页面直接绑死在猜测字段上。

---

## 4.7 管理员用户域数据映射

## 4.7.1 `GET /api/v1/admin/users`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/users` | 用户列表 |

### 后端建议最小字段

- `userId`
- `subjectId`
- `role`
- `status`
- `authMethod`
- `runtimeObservedState`
- `lastLoginAt`

### 推荐列表行 view model

```ts
interface AdminUserRowVM {
  userId: string
  subjectId: string
  role: 'user' | 'admin'
  status: 'active' | 'disabled'
  authMethod: string
  runtimeObservedState?: string
  lastLoginAt?: string | null
}
```

---

## 4.7.2 `GET /api/v1/admin/users/{userId}`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/users/:userId` | 用户详情 |

### 后端建议最小字段

- `userId`
- `subjectId`
- `tenantId`
- `role`
- `status`
- `createdAt`
- `updatedAt`
- `auth.provider`
- `auth.method`

### 推荐详情 view model

```ts
interface AdminUserDetailVM {
  userId: string
  subjectId: string
  tenantId: string
  role: 'user' | 'admin'
  status: 'active' | 'disabled'
  createdAt: string
  updatedAt: string
  authProvider: string
  authMethod: string
  canDisable: boolean
  canEnable: boolean
}
```

---

## 4.7.3 `PATCH /api/v1/admin/users/{userId}/status`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/users` | 行内操作（可选） |
| `/admin/users/:userId` | 主操作 |

### 请求字段映射

| 前端动作 | 后端字段 |
| --- | --- |
| 启用用户 | `status=active` |
| 禁用用户 | `status=disabled` |

---

## 4.7.4 `GET /api/v1/admin/users/{userId}/runtime`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/users/:userId` | runtime 详情区块 |

### 映射原则

该接口可以复用用户侧 runtime 详情卡组件，但必须切换为“只读管理视角”，不提供用户侧启动 / 停止 / 删除按钮，除非产品明确授权管理员控制。

---

## 4.8 管理员模型与凭据域数据映射

## 4.8.1 `GET /api/v1/admin/models`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/models` | 模型治理 |

### 映射原则

- 列表字段由后端实际返回决定
- 前端至少应支持：模型名、状态、可见性、默认策略展示与编辑

### 建议 view model

```ts
interface AdminModelRowVM {
  modelId: string
  displayName: string
  enabled: boolean
  visibility?: string
  defaultPolicy?: string
}
```

---

## 4.8.2 `PUT /api/v1/admin/models/{modelId}`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/models` | 行内编辑或抽屉编辑 |

### 前端原则

- 保存前做最小前端校验
- 保存后刷新单行或整表
- 不做乐观更新到无法回滚的程度

---

## 4.8.3 `GET /api/v1/admin/provider-credentials`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/provider-credentials` | 凭据列表 |

### 建议 view model

```ts
interface ProviderCredentialRowVM {
  credentialId: string
  provider: string
  displayName: string
  status?: string
  lastVerifiedAt?: string | null
}
```

---

## 4.8.4 `POST /api/v1/admin/provider-credentials`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/provider-credentials` | 新增凭据 |

### 映射原则

- 表单字段完全以后端要求为准
- 成功后刷新列表
- 敏感值不做回显

---

## 4.8.5 `POST /api/v1/admin/provider-credentials/{credentialId}/verify`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/provider-credentials` | 行内校验 |

### 前端规则

- 每行独立 loading
- 成功后刷新该行状态

---

## 4.8.6 `DELETE /api/v1/admin/provider-credentials/{credentialId}`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/provider-credentials` | 删除凭据 |

### 前端规则

- 危险操作
- 必须二次确认
- 删除成功后从列表移除或刷新列表

---

## 4.8.7 `GET /api/v1/admin/usage/summary`

### 页面映射

| 页面 | 用途 |
| --- | --- |
| `/admin/usage` | 用量汇总 |

### 建议 view model

```ts
interface UsageSummaryVM {
  summaryCards: Array<{ label: string; value: string | number }>
  rows: Array<Record<string, unknown>>
}
```

> 由于示例字段未展开，前端应以“汇总卡片 + 表格明细”的可适配结构接入。

---

## 4.9 页面级接口编排建议

## 4.9.1 `/workspace` 首页

### 建议首屏组合

```ts
Promise.all([
  getAuthMe(),
  getMyRuntimeStatus(),
  getModels(),
  getMyQuota().catch(() => null),
])
```

### 页面 view model 建议

```ts
interface WorkspaceHomeVM {
  currentUser: CurrentUserVM
  runtime: RuntimeStatusVM | null
  models: ModelItemVM[]
  quota?: QuotaVM | null
}
```

---

## 4.9.2 `/workspace/runtime` 页面

### 建议数据组合

- 首屏：`GET /api/v1/users/me/runtime`
- 动作后：轮询 task -> 刷新 runtime/status

---

## 4.9.3 `/admin/users/:userId` 页面

### 建议数据组合

```ts
Promise.all([
  getAdminUserDetail(userId),
  getAdminUserRuntime(userId).catch(() => null),
])
```

---

## 4.9.4 `/admin/invitations` 页面

### 建议数据组合

- 首屏：`GET /api/v1/admin/invitations`
- 创建成功：本地插入新行或刷新列表
- 重发 / 撤销后：刷新当前行或刷新列表

---

## 4.10 前端 adapter 层建议

建议前端建立统一 adapter / mapper 层：

```text
api/
  auth.ts
  invitations.ts
  runtime.ts
  adminUsers.ts
  adminInvitations.ts
mappers/
  auth.mapper.ts
  invitation.mapper.ts
  runtime.mapper.ts
  user.mapper.ts
```

目的：

- 屏蔽页面层对后端原始响应结构的直接依赖
- 统一枚举文案
- 统一状态可操作性判断

---

## 4.11 可操作性派生规则建议

以下字段建议只在前端 view model 中派生，不落库存储：

### 4.11.1 runtime

```ts
canStart = observedState === 'stopped' || observedState === 'error' || observedState === 'deleted'
canStop = observedState === 'running'
canDelete = observedState !== 'creating'
isReady = observedState === 'running' && !!browserUrl
```

### 4.11.2 invitation

```ts
canRevoke = status === 'pending'
canResend = ['pending', 'revoked', 'expired', 'consumed'].includes(status)
```

### 4.11.3 user

```ts
canDisable = status === 'active'
canEnable = status === 'disabled'
```

---

## 4.12 最终结论

首版接口到页面映射要抓住三件事：

1. **核心字段不改名**
2. **页面通过 adapter 层生成 view model**
3. **所有可操作按钮都基于后端状态派生，而不是前端猜测**

这样可以保证 UI、前端、后端在联调时用同一套语言，不会因为字段命名和状态理解偏差而反复返工。


v 0.1
reno 
2026-03-23 10:54