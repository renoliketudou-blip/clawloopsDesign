# ClawLoops 页面调用流程（BFF 编排冻结版）

## 1. 文档目标

本文定义每个前端页面在 MVP 首版中的调用顺序、BFF 边界、重试策略、跳转规则与不可越界事项，用于冻结：

- 页面初始化请求顺序
- 用户操作触发的接口编排
- 页面与 BFF 的职责分界
- 跳转与轮询的唯一标准

---

## 2. 编排总原则

### 2.1 前端与 BFF 的边界

前端只调用：

- `/api/v1/*`

前端绝不调用：

- `/internal/*`
- Runtime Manager internal API
- Authentik 用户/邀请管理 API

前端唯一允许直接离开应用域的动作：

- 接收 `/api/v1/public/invitations/{token}/start` 返回的 `redirectUrl`
- 浏览器整页跳转到 Authentik enrollment flow
- 从 `workspace-entry` 获得 `browserUrl` 后整页跳转到 workspace 子域名

### 2.2 编排规则

1. 页面可见状态以 BFF 返回为准，不靠前端推理补业务结论。
2. `post-login` 与 `start` 视为幂等接口，前端可以安全重试。
3. 非管理员用户的默认控制面是 `/app`，`/workspace-entry` 只负责最终跳转。
4. runtime 动作先拿 `taskId`，再轮询任务，再刷新 runtime 状态。
5. workspace 跳转只信 `GET /api/v1/workspace-entry`。
6. `browserUrl` 不等于可访问，`ready=true` 才允许跳转。
7. 邀请、登录、runtime 等失败页必须给出明确下一步，不允许只停留在说明页。
8. `admin` 首页只信 `GET /api/v1/admin/home`。

---

## 3. 路由总表

| 路由 | 页面名称 | 权限 | 初始化接口 | 主动作 |
| --- | --- | --- | --- | --- |
| `/login` | 登录入口页 | 公开 | `/api/v1/auth/options` | 跳 Authentik 登录 |
| `/invite/:token` | invitation 接入页 | 公开 | `/api/v1/public/invitations/{token}` | `/start` |
| `/post-login` | 登录完成收口页 | 已登录 | `/api/v1/auth/post-login` | 收口后按 `redirectTo` 跳转 |
| `/app` | 用户工作台 | 已登录且 allowed | `/auth/me`、`/auth/access`、`/users/me/runtime/status`、`/models` | start/stop/delete/open |
| `/workspace-entry` | 工作区入口页 | 已登录且 allowed | `/auth/me`、`/auth/access`、`/workspace-entry` | 跳 workspace |
| `/admin` | 管理后台首页 | admin | `/auth/me`、`/auth/access`、`/admin/home` | 跳各后台子页 |
| `/admin/users` | 用户列表页 | admin | `/auth/me`、`/auth/access`、`/admin/users` | 改用户状态 |
| `/admin/users/:userId` | 用户详情页 | admin | `/auth/me`、`/auth/access`、`/admin/users/{userId}`、`/admin/users/{userId}/runtime` | 启停状态治理 |
| `/admin/invitations` | invitation 列表页 | admin | `/auth/me`、`/auth/access`、`/admin/invitations` | 创建/撤销/重发 |
| `/admin/invitations/:invitationId` | invitation 详情页 | admin | `/auth/me`、`/auth/access`、`/admin/invitations/{invitationId}` | 撤销/重发 |
| `/admin/models` | 模型治理页 | admin | `/auth/me`、`/auth/access`、`/admin/models` | 更新模型策略 |
| `/admin/provider-credentials` | provider 凭据页 | admin | `/auth/me`、`/auth/access`、`/admin/provider-credentials` | 创建/验证/删除 |
| `/admin/usage` | usage 汇总页 | admin | `/auth/me`、`/auth/access`、`/admin/usage/summary` | 刷新统计 |

---

## 4. 公开页面编排

## 4.1 登录入口页 `/login`

目标：

- 呈现首版唯一登录方式
- 若已登录则不重复展示登录按钮

调用顺序：

1. `GET /api/v1/auth/options`
2. 可选：`GET /api/v1/auth/me`

页面规则：

- 登录方式只展示 `local_password`
- 登录入口文案以 `/auth/options.methods[0].label` 为准（当前固定为 `账号密码登录`）
- 不展示 Google、GitHub、企业 SSO 等入口
- 若 `/auth/me` 已表明已登录，则按角色跳 `/admin` 或 `/app`

前端职责：

- 只负责展示入口按钮和说明文案
- 不负责拼装身份参数

## 4.2 invitation 接入页 `/invite/:token`

### 初始化流程

1. 解析路由参数 `token`
2. `GET /api/v1/public/invitations/{token}`
3. 可选：`GET /api/v1/auth/me`，用于识别当前浏览器是否已有登录态
4. 根据返回渲染有效页或失效页

### 点击“继续接入”流程

前置判断：

- 若页面已识别出“当前登录账号”与“邀请目标邮箱”看起来不一致，则默认不直接触发 `/start`
- 此时页面先展示错账号风险卡片，主 CTA 为“切换账号后继续接入”
- 用户完成切换账号后，再回到本页继续正常的 `/start` 流程

1. `POST /api/v1/public/invitations/{token}/start`
2. 拿到：
   - `status=accepted`
   - `pendingInvitationSession.ttlSeconds`
   - `redirectUrl`
3. 浏览器整页跳转 `redirectUrl`

页面提示要求：

- 若当前浏览器已有登录态，页面应显式展示“当前登录账号”和“邀请目标邮箱”
- 若两者看起来不一致，前端不做业务校验结论，但必须把它做成可操作流程，而不是只停留在提示
- 错账号风险卡片最少要提供：
  - `切换账号后继续接入` 作为主 CTA
  - `返回登录入口` 作为次 CTA
  - `联系管理员` 作为兜底动作
- invitation 失败页必须提供回登录入口、联系管理员或稍后重试等可恢复动作

### 失败收口

| 场景 | 处理 |
| --- | --- |
| token 无效 | 留在当前页，渲染无效 invitation |
| `INVITATION_REVOKED` | 渲染已撤销 |
| `INVITATION_EXPIRED` | 渲染已过期 |
| `INVITATION_ALREADY_CONSUMED` | 渲染已使用 |
| `INVITATION_ERROR` | 允许重试 |

前端不做：

- 不自行写 pending invitation session
- 不在前端缓存 `itoken`
- 不在前端做邮箱校验

## 4.3 post-login 收口页 `/post-login`

### 冻结事实

`POST /api/v1/auth/post-login` 是前端主动调用的 BFF 接口，不是 Authentik callback。

### 调用顺序

1. 页面加载后，确认浏览器已带 Authentik 登录态
2. `POST /api/v1/auth/post-login`
3. BFF 内部完成：
   - `/internal/users/sync`
   - 检查 pending invitation 上下文
   - 邮箱强校验
   - membership 绑定
   - invitation consume
4. 前端根据 BFF 返回决定下一跳

### 前端跳转策略

| 返回结果 | 下一步 |
| --- | --- |
| `entryType=admin_console` | 跳 `/admin` |
| `entryType=workspace` 且 `hasWorkspace=true` | 跳 `/app` |
| `hasWorkspace=false` | 进入无工作区页，并给出联系管理员或返回入口动作 |

用户体验规则：

- `post-login` 对普通用户的目标是把人送到“可继续操作的页面”，而不是直接送到中转页
- 若 `invitationApplied=true`，`/app` 首屏必须先展示“你已成功加入 <workspaceName>”的强确认，再展示单一主 CTA“开始准备工作区”或“继续准备工作区”
- 首登成功后的确认区必须说明：
  - 接入已完成
  - 当前加入的 workspace
  - 下一步只需要准备工作区
- 邮箱不匹配、邀请失效、无工作区都必须提供明确下一步

### 错误处理

| code | 页面行为 |
| --- | --- |
| `INVITATION_EMAIL_MISMATCH` | 显示账号与邀请不匹配，并提供“切换账号后继续接入”“返回登录入口”动作 |
| `INVITATION_REVOKED` | 显示邀请失效，并提供联系管理员动作 |
| `INVITATION_EXPIRED` | 显示邀请过期，并提供联系管理员动作 |
| `INVITATION_WORKSPACE_INVALID` | 显示目标工作区无效，并提供联系管理员动作 |
| `USER_SYNC_ERROR` | 显示登录收口失败，可重试 |
| `USER_DISABLED` | 显示账号已禁用 |
| `INVITATION_ERROR` | 显示系统错误，可重试 |

---

## 5. 用户控制面编排

## 5.1 用户工作台 `/app`

### 初始化流程

按后端联调建议固定顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. `GET /api/v1/users/me/runtime/status`
4. `GET /api/v1/models`

建议执行方式：

- `auth/me` 先跑
- 已登录后并发执行 `auth/access`、`runtime/status`、`models`

### 页面展示切面

| 切面 | 数据来源 |
| --- | --- |
| 账号信息 | `/auth/me` |
| 访问是否允许 | `/auth/access` |
| runtime 卡片 | `/users/me/runtime/status` |
| 模型只读列表 | `/models` |

工作台交互要求：

- 首次接入用户与回访用户共用 `/app`，但首屏提示不同
- 首次接入且 runtime 尚未就绪时，首屏必须先展示“接入成功确认区”，再给出唯一主 CTA“开始准备工作区”或“继续准备工作区”
- 回访用户优先看到最近状态和“进入工作区”入口
- runtime 状态文案以“未启动 / 准备中 / 可进入 / 启动失败”为主，不直接暴露底层实现术语
- runtime 准备态不能只显示一个抽象状态词，必须展示阶段化进度与当前下一步

### 点击“启动 runtime”

1. `POST /api/v1/users/me/runtime/start`
2. 拿到 `taskId`
3. 进入轮询：
   - `GET /api/v1/runtime/tasks/{taskId}`
4. 每次任务状态变化后或定时补偿刷新：
   - `GET /api/v1/users/me/runtime/status`
5. 当 `task.status` 进入终态时停止任务轮询
6. 以最新 `runtime.status.ready` 更新页面

终态判断：

| task.status | 页面处理 |
| --- | --- |
| `succeeded` | 刷新状态，若 `ready=true` 则允许进入工作区 |
| `failed` | 刷新状态并展示 `lastError` |
| `canceled` | 刷新状态并解除按钮禁用 |

等待体验要求：

- 页面要把 runtime 准备做成真正的进度体验，而不是只显示 loading 或“准备中”
- 最少展示 4 个连续步骤：
  - 已接入 workspace
  - 正在创建运行环境
  - 正在启动服务
  - 正在验证工作区入口
- 当前步骤必须高亮，已完成步骤必须可见，用户要能知道“现在做到哪一步了”
- 页面要明确提示“准备仍在继续，可稍后回来”
- 轮询中允许用户离开当前页，返回工作台后继续根据最新状态渲染，并保留上次已完成的进度步骤
- 超过预期时间时，要从“加载中”切换为“仍在准备，可稍后回来”的可恢复文案
- 失败展示应优先给出产品化文案，再保留错误码用于排障
- 失败后至少提供：
  - `重新准备工作区`
  - `返回工作台稍后再试`
  - `联系管理员`

### 点击“停止 runtime”

1. `POST /api/v1/users/me/runtime/stop`
2. 拿到 `taskId`
3. 轮询 `/runtime/tasks/{taskId}`
4. 补充刷新 `/users/me/runtime/status`

### 点击“删除 runtime”

1. 用户确认删除策略：
   - `preserve_workspace`
   - `wipe_workspace`
2. `POST /api/v1/users/me/runtime/delete`
3. 请求体带 `retentionPolicy`
4. 轮询 `/runtime/tasks/{taskId}`
5. 刷新 `/users/me/runtime/status`

### 点击“进入工作区”

固定顺序：

1. `GET /api/v1/workspace-entry`
2. 仅当 `ready=true` 时整页跳转 `browserUrl`

禁止做法：

- 不允许直接使用 `/users/me/runtime/status.browserUrl` 进行跳转

## 5.2 工作区入口页 `/workspace-entry`

页面目标：

- 做最终跳转裁决
- 承接“立即进入工作区”动作或短时等待

调用顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. `GET /api/v1/workspace-entry`

页面分支：

| 条件 | 页面行为 |
| --- | --- |
| `ready=true` | 立即跳工作区子域 |
| `ready=false` 且 runtime 存在 | 停留在本页，展示与 `/app` 一致的阶段化准备进度，并允许返回工作台 |
| `ready=false` 且 runtime 尚未准备 | 引导返回 `/app` 启动或继续准备 |
| `USER_DISABLED` | 进入禁用页 |
| 无 workspace | 进入无工作区页，并提供联系管理员动作 |

### 轮询建议

若业务希望本页自动等待 runtime ready，可采用：

1. 首次调 `/workspace-entry`
2. 若 `ready=false`，每 2 秒重试一次
3. 60 秒后改为每 3 秒重试一次
4. 最长 180 秒
5. 超时后保留手动刷新，并允许用户返回 `/app` 稍后再试

此轮询只允许发生在 `/workspace-entry` 页，不建议在任意页面后台常驻。
此页应被设计成短时等待页，而不是普通用户登录后的默认首页。
此页若处于等待态，应复用 `/app` 的阶段化进度展示与恢复动作，不再退化成单句文案页。

---

## 6. 管理后台编排

## 6.1 管理后台首页 `/admin`

调用顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. 校验 `role=admin`
4. `GET /api/v1/admin/home`

首页最小渲染块：

- 摘要卡片：用户、invitation、runtime
- 待办列表：`attention.pendingInvitations`
- 异常列表：`attention.runtimeAlerts`
- 快捷导航：用户管理、邀请管理、Usage 汇总

冻结要求：

- 首页不依赖 workspace membership
- 首页不直接执行写操作
- 首页只承载摘要、待办与跳转

## 6.2 用户列表页 `/admin/users`

调用顺序：

1. `GET /api/v1/auth/me`
2. `GET /api/v1/auth/access`
3. 校验 `role=admin`
4. `GET /api/v1/admin/users`

动作流程：

- 修改用户状态：
  1. `PATCH /api/v1/admin/users/{userId}/status`
  2. 成功后回刷 `GET /api/v1/admin/users`

冻结要求：

- 用户列表页最小字段就按后端建议字段展示
- 前端不猜更多筛选条件，除非后端后续补充接口

## 6.3 用户详情页 `/admin/users/:userId`

调用顺序：

1. `GET /api/v1/admin/users/{userId}`
2. `GET /api/v1/admin/users/{userId}/runtime`

动作流程：

- 启用/禁用：
  1. `PATCH /api/v1/admin/users/{userId}/status`
  2. 回刷详情
  3. 回刷 runtime 区块

## 6.4 invitation 列表页 `/admin/invitations`

调用顺序：

1. `GET /api/v1/admin/invitations`

动作流程：

- 创建 invitation：
  1. 打开创建弹窗
  2. `POST /api/v1/admin/invitations`
  3. 成功后回刷列表

- 撤销 invitation：
  1. `POST /api/v1/admin/invitations/{invitationId}/revoke`
  2. 成功后回刷列表

- 重发 invitation：
  1. `POST /api/v1/admin/invitations/{invitationId}/resend`
  2. 成功后回刷列表

创建请求冻结字段：

```json
{
  "targetEmail": "user@example.com",
  "workspaceId": "ws_001",
  "role": "workspace_member",
  "expiresInHours": 72
}
```

## 6.5 invitation 详情页 `/admin/invitations/:invitationId`

调用顺序：

1. `GET /api/v1/admin/invitations/{invitationId}`

动作流程：

- 撤销：`POST /api/v1/admin/invitations/{invitationId}/revoke`
- 重发：`POST /api/v1/admin/invitations/{invitationId}/resend`

## 6.6 模型治理页 `/admin/models`

调用顺序：

1. `GET /api/v1/admin/models`

动作流程：

1. 修改某模型策略
2. `PUT /api/v1/admin/models/{modelId}`
3. 成功后回刷 `GET /api/v1/admin/models`

冻结边界：

- 只冻结“列表 + 单项保存”编排
- 不冻结更复杂的批量编排

## 6.7 provider 凭据页 `/admin/provider-credentials`

调用顺序：

1. `GET /api/v1/admin/provider-credentials`

动作流程：

- 新增：
  1. `POST /api/v1/admin/provider-credentials`
  2. 成功后回刷列表

- 校验：
  1. `POST /api/v1/admin/provider-credentials/{credentialId}/verify`
  2. 成功后更新当前项状态

- 删除：
  1. `DELETE /api/v1/admin/provider-credentials/{credentialId}`
  2. 成功后回刷列表

## 6.8 usage 汇总页 `/admin/usage`

调用顺序：

1. `GET /api/v1/admin/usage/summary`

冻结边界：

- 只冻结为查询页
- 不引入前端二次统计真相

---

## 7. 统一轮询策略

## 7.1 适用范围

只允许两类轮询：

1. runtime 任务轮询
2. workspace-entry ready 轮询

## 7.2 runtime 任务轮询策略

建议：

1. 接到 `taskId` 后立即请求一次
2. 前 10 秒每 2 秒轮询
3. 10 秒后每 3 秒轮询
4. 最长 90 秒
5. 结束后补一次 `/users/me/runtime/status`

停止条件：

- `task.status in [succeeded, failed, canceled]`
- 页面卸载
- 用户主动取消当前动作追踪

## 7.3 workspace-entry 轮询策略

建议：

1. 首次加载立即请求
2. `ready=false` 时每 2 秒请求
3. 60 秒后每 3 秒请求
4. 最长 180 秒
5. 超时后改为手动刷新，并提示返回工作台

---

## 8. 页面到 BFF 的标准编排模板

适用于任意业务页：

1. 公开页只调用公开接口。
2. 鉴权页先过 `auth/me -> auth/access`。
3. 页面初始化只拉“当前页最低必要数据”。
4. 提交型动作先置为提交中，禁止重复点击。
5. 成功后优先回刷当前页所依赖的最小数据集。
6. 错误必须按 `code` 收口，不允许只按文案模糊判断。

---

## 9. 冻结结论

前端可以按本文直接冻结 BFF 编排：

1. invitation 链路固定为 `preview -> start -> redirect -> post-login`
2. 普通用户登录后默认落到 `/app`，由工作台承接首次使用与回访使用
3. 用户工作台固定为 `me -> access -> runtime/status -> models`
4. runtime 动作固定为 `action -> task polling -> runtime/status refresh`
5. workspace 跳转固定为 `workspace-entry -> browserUrl`
6. 管理后台首页固定为 `/admin -> /api/v1/admin/home`
7. 管理后台其余页面均采用“提交后回刷列表/详情”的简单编排
8. 前端无需等待 internal API 细节即可并行开发


v0.3 用户自然走完修订
reno  
2026-03-25 16:05