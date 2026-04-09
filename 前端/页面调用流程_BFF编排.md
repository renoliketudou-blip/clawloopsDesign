# ClawLoops 页面调用流程（BFF 编排冻结版，轻量认证修订）

## 1. 文档目标

本文定义每个前端页面在 MVP 首版中的调用顺序、BFF 边界、重试策略、跳转规则与不可越界事项，用于冻结：

- 页面初始化请求顺序
- 用户操作触发的接口编排
- 页面与 BFF 的职责分界
- 跳转与轮询的唯一标准

本版默认前提：

- 平台使用业务内轻量认证
- 登录真相来自平台 session
- invitation 在站内完成首次设密与接入
- 种子管理员首次登录后必须立刻强制改密
- 首版不做通用改密与找回密码

---

## 2. 编排总原则

### 2.1 前端与 BFF 的边界

前端只调用：

- `/api/v1/*`

前端绝不调用：

- `/internal/*`
- Runtime Manager internal API
- 任何第三方 IAM API

session 约束：

- 前端不读取 `clawloops_session`
- 前端不自己设置、续期或删除 session cookie
- 前端不向 `browserUrl` 拼接 token、ticket、`userId` 或其他鉴权参数

前端唯一允许直接离开当前控制面域的动作：

- 从 `workspace-entry` 获得 `browserUrl` 后整页跳转到 workspace 子域名

### 2.2 编排规则

1. 页面可见状态以 BFF 返回为准，不靠前端推理补业务结论。
2. `accept` 视为幂等接口，前端可以安全重试。
3. 非管理员用户的默认控制面是 `/app`，`/workspace-entry` 只负责最终跳转。
4. runtime 动作先拿 `taskId`，再轮询任务，再刷新 runtime 状态。
5. workspace 跳转只信 `GET /api/v1/workspace-entry`。
6. `browserUrl` 不等于可访问，`ready=true` 才允许跳转。
7. 邀请、登录、runtime 等失败页必须给出明确下一步，不允许只停留在说明页。
8. `admin` 首页只信 `GET /api/v1/admin/home`。
9. session cookie 由浏览器和服务端维护，前端不读取、不拼接、不覆盖。

---

## 3. 路由总表

| 路由 | 页面名称 | 权限 | 初始化接口 | 主动作 |
| --- | --- | --- | --- | --- |
| `/login` | 登录入口页 | 公开 | `/api/v1/auth/options` | `/auth/login` |
| `/disabled` | 账号禁用说明页 | 公开 | 无 | logout / 回登录 |
| `/403` | 无权限页 | 公开 | 无 | 回首页 / 返回上一页 |
| `/force-password-change` | 首登强制改密页 | 已登录且 `mustChangePassword=true` | `/api/v1/auth/me` | `/auth/password/change` |
| `/invite/:token` | invitation 接入页 | 公开 | `/api/v1/public/invitations/{token}` | `/accept` |
| `/app` | 用户工作台 | 已登录且 allowed | `/auth/me`、`/auth/access`、`/users/me/runtime/status`、`/models` | start/stop/delete/open |
| `/workspace-entry` | 工作区入口页 | 已登录且 allowed | `/auth/me`、`/auth/access`、`/workspace-entry` | 跳 workspace |
| `/files` | 文件管理 | 已登录且 allowed | `/auth/me`、`/auth/access`、`/files/list` | 上传/下载/编辑/保存 |
| `/admin` | 管理后台首页 | admin | `/auth/me`、`/auth/access`、`/admin/home` | 跳各后台子页 |
| `/admin/users` | 用户列表页 | admin | `/auth/me`、`/auth/access`、`/admin/users` | 改用户状态 |
| `/admin/users/:userId` | 用户详情页 | admin | `/auth/me`、`/auth/access`、`/admin/users/{userId}`、`/admin/users/{userId}/runtime` | 启停状态治理 |
| `/admin/invitations` | invitation 列表页 | admin | `/auth/me`、`/auth/access`、`/admin/invitations` | 创建/撤销/重发 |
| `/admin/invitations/:invitationId` | invitation 详情页 | admin | `/auth/me`、`/auth/access`、`/admin/invitations/{invitationId}` | 撤销/重发 |
| `/admin/models` | 模型治理页 | admin | `/auth/me`、`/auth/access`、`/admin/models` | 更新模型策略 |
| `/admin/provider-credentials` | provider 凭据页 | admin | `/auth/me`、`/auth/access`、`/admin/provider-credentials` | 创建/验证/删除 |
| `/admin/user-files/:username` | 用户文件列表 | admin | `/auth/me`、`/auth/access`、`/admin/user-files/:username/list` | 删除 |
| `/admin/usage` | usage 汇总页 | admin | `/auth/me`、`/auth/access`、`/admin/usage/summary` | 刷新统计 |

---

## 4. 公开页面编排

## 4.1 登录入口页 `/login`

目标：

- 呈现首版唯一登录方式
- 若已登录则不重复展示登录表单

调用顺序：

1. `GET /api/v1/auth/options`
2. 可选：`GET /api/v1/auth/me`

页面规则：

- 登录方式只展示 `local_password`
- 登录入口文案以 `/auth/options.methods[0].label` 为准（当前固定为 `用户名优先登录`）
- 密码提示文案以 `/auth/options.passwordPolicy` 为准，不允许前端手写另一套规则
- 对无真实邮箱用户，登录页辅助文案应明确“请优先使用管理员提供的用户名登录”
- 不展示 Google、GitHub、企业 SSO 等入口
- 不展示通用改密和找回密码入口
- 若 `/auth/me` 已表明 `mustChangePassword=true`，则优先跳 `/force-password-change`
- 否则按角色跳 `/admin` 或 `/app`

点击“登录”流程：

1. 收集 `username`
2. 收集 `password`
3. `POST /api/v1/auth/login`
4. 成功后按 `redirectTo` 跳 `/force-password-change`、`/admin` 或 `/app`

前端职责：

- 只负责展示表单、校验必填和错误态
- 不负责持久化密码

## 4.1A 账号禁用说明页 `/disabled`

页面规则：

- 该页用于承接 `/auth/access.reason=USER_DISABLED`
- 不再发起业务页初始化请求
- 可提供 logout 与返回 `/login` 入口

## 4.1B 无权限页 `/403`

页面规则：

- 该页用于承接 `ACCESS_DENIED`
- 非 admin 进入 `/admin/*` 时统一跳转到这里
- 不复用为账号禁用页

## 4.2 首登强制改密页 `/force-password-change`

### 初始化流程

1. `GET /api/v1/auth/me`
2. 若 `mustChangePassword=false`，直接按角色跳回 `/admin` 或 `/app`

### 页面展示规则

- 该页不是后台导航页，不进入 `/admin` 壳层
- 必须明确提示“当前为默认初始密码，需先修改后才能继续”
- 密码规则提示必须来自 `/auth/options.passwordPolicy` 或其缓存结果
- 必须提供当前密码、新密码、确认新密码三个输入项
- 除 logout 外，不提供跳过入口

### 点击“更新密码并继续”流程

1. 收集 `currentPassword`
2. 收集 `newPassword`
3. 收集 `newPasswordConfirm`
4. `POST /api/v1/auth/password/change`
5. 成功后按 `redirectTo` 进入 `/admin`

失败分支要求：

- `CURRENT_PASSWORD_INCORRECT`：高亮当前密码输入并提示重新输入
- `PASSWORD_CHANGE_INVALID`：展示密码规则或“新旧密码不能相同”

## 4.3 invitation 接入页 `/invite/:token`

### 初始化流程

1. 解析路由参数 `token`
2. `GET /api/v1/public/invitations/{token}`
3. 根据返回渲染有效页或失效页

### 页面展示规则

- 必须展示 `workspaceName`
- 必须展示 `role`
- 必须展示 `expiresAt`
- 密码规则提示必须来自 `/auth/options.passwordPolicy` 或其缓存结果
- 若存在 `loginUsername`，必须将其作为主说明文案
- 若存在 `targetEmail` 且为代理邮箱，不应把它作为主提示文案
- 页内必须提供“设置初始密码并继续”主 CTA

### 点击“继续接入”流程

1. 收集 `username`
2. 收集 `password`
3. 收集 `passwordConfirm`
4. `POST /api/v1/public/invitations/{token}/accept`
5. 若返回 `accepted=true`，无论 `replayed=false` 还是 `replayed=true`，都按 `redirectTo` 进入 `/app`

失败分支要求：

- `INVITATION_NOT_FOUND`、`INVITATION_EXPIRED`、`INVITATION_REVOKED`、`INVITATION_ALREADY_CONSUMED` 都要有独立说明
- `INVITATION_USERNAME_MISMATCH` 要明确提示“请使用管理员提供的用户名完成接入”
- `INVITATION_PASSWORD_INVALID` 要在表单区直接给出可操作提示

---

## 5. 已登录页面编排

## 5.1 用户工作台 `/app`

初始化顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. `GET /api/v1/users/me/runtime/status`
4. `GET /api/v1/models`

页面规则：

- 首次接入成功后，页面首屏必须先确认“你已成功加入某个 workspace”
- 该确认态要和普通回访态共存，不单独再做 `/post-login`
- 若 `allowed=false`，进入禁用拦截态

用户操作：

- 启动 runtime：`POST /api/v1/users/me/runtime/start`
- 停止 runtime：`POST /api/v1/users/me/runtime/stop`
- 删除 runtime：`POST /api/v1/users/me/runtime/delete`
- 打开工作区：`GET /api/v1/workspace-entry`

## 5.2 工作区入口页 `/workspace-entry`

初始化顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. `GET /api/v1/workspace-entry`

规则：

- 只在 `ready=true` 时允许整页跳转 `browserUrl`
- `observedState` 只用于展示，不作为最终跳转依据
- 用户若无合法 workspace，应回到 `/app` 的承接态
- 跳转时不附加任何额外鉴权参数，workspace 子域访问权限完全依赖浏览器已持有的平台 session

## 5.3 文件管理 `/files`

初始化顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. `GET /api/v1/files/list`

用户操作：

- 上传文件：`POST /api/v1/files/upload`
- 下载文件：`GET /api/v1/files/download/{runtimeId}`
- 读取文件内容：`GET /api/v1/files/read/{runtimeId}`
- 保存文件：`POST /api/v1/files/write/{runtimeId}`

---

## 6. 管理后台编排

## 6.1 `/admin`

初始化顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. `GET /api/v1/admin/home`

规则：

- 若 `mustChangePassword=true`，不得进入该页，必须回 `/force-password-change`
- 非 admin 用户统一进入 403 无权限页
- `/admin` 不是中转页，必须是可用首页

## 6.2 `/admin/invitations`

初始化顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. `GET /api/v1/admin/invitations`

动作编排：

- 创建 invitation：`POST /api/v1/admin/invitations`
- 撤销 invitation：`POST /api/v1/admin/invitations/{invitationId}/revoke`
- 重发 invitation：`POST /api/v1/admin/invitations/{invitationId}/resend`

创建成功后页面至少要展示：

- `inviteUrl`
- `loginUsername`
- `workspaceName`
- `expiresAt`

---

## 7. 页面级错误收口

### 7.1 登录页

- `INVALID_CREDENTIALS`：用户名或密码错误
- `USER_DISABLED`：账号已禁用，请联系管理员
- `PASSWORD_CHANGE_REQUIRED`：立即跳到 `/force-password-change`
- `SESSION_ERROR`：系统繁忙，请稍后重试

### 7.2 强制改密页

- `CURRENT_PASSWORD_INCORRECT`：当前密码错误，请重新输入
- `PASSWORD_CHANGE_INVALID`：留在当前页并展示密码规则或“新旧密码不能相同”
- `UNAUTHENTICATED`：回 `/login`

### 7.3 invitation 页

- invitation 无效：展示专门失效页
- 用户名不匹配：保持在当前页并高亮用户名输入
- 密码不合法：保持在当前页并展示密码规则
- `accepted=true` 且 `replayed=true`：按普通成功分支处理，不额外弹“重复提交”错误

### 7.4 工作台与后台

- `UNAUTHENTICATED`：回 `/login`
- `USER_DISABLED`：进入 `/disabled`
- `ACCESS_DENIED`：进入 `/403`

---

## 8. 不可越界事项

- 不解析 cookie 或 token 作为业务真相
- 不读取、写入或删除 `clawloops_session`
- 不把 `browserUrl` 直接当可访问条件，必须先看 `workspace-entry.ready`
- 不直接调用 `/internal/*`
- 不自己拼装 `subjectId`
- 不向 `browserUrl` 拼接任何鉴权参数
- 不新增 `/post-login`
- 不在首版前端实现通用改密、找回密码或第三方登录入口

---

## 9. 最终编排结论

前端需要记住的唯一认证结论是：

1. 登录走 `/api/v1/auth/login`
2. 若命中种子管理员首登，必须先走 `/api/v1/auth/password/change`
3. invitation 接入走 `/api/v1/public/invitations/{token}/accept`
4. 成功后直接进入 `/app`、`/admin` 或 `/force-password-change`
5. 工作区跳转只信 `/api/v1/workspace-entry`
6. 首版不做通用改密和找回密码

---

v0.5-轻量认证修订  
2026-03-25
