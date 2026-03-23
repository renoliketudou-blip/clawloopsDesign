# 2. 页面级交互状态文档

## 2.1 文档目标

本文档用于统一前端页面级状态、按钮行为、加载态、空态、错误态、异步任务态和关键页面状态机。

目标不是罗列所有视觉稿，而是让 UI 和前端对以下内容保持一致：

- 页面首次加载时显示什么
- API 成功 / 失败时如何反馈
- 哪些按钮可点击，哪些要禁用
- 哪些页面需要轮询
- 发生 `UNAUTHENTICATED / USER_DISABLED / INVITATION_*` 时怎么收口

---

## 2.2 全局页面状态基线

所有业务页面建议统一以下页面级状态：

| 状态 | 说明 | UI 建议 |
| --- | --- | --- |
| initializing | 页面首次进入，尚未完成首屏请求 | 全页 skeleton 或页面级 loading |
| ready | 数据加载成功，可正常操作 | 正常内容态 |
| empty | 请求成功但无可展示内容 | empty state |
| submitting | 当前有写操作进行中 | 禁用重复提交、显示按钮 loading |
| polling | 有异步任务在执行中 | 显示进度提示、保留核心信息可读 |
| partial-error | 页面主体可显示，但局部请求失败 | 局部 error block + retry |
| fatal-error | 页面主请求失败，页面不可正常展示 | 全页错误态 |

---

## 2.3 全局错误收口规则

### 2.3.1 认证错误

| 错误码 | 前端动作 |
| --- | --- |
| `UNAUTHENTICATED` | 清空本地会话缓存，跳 `/login` 或登录入口 |
| `ACCESS_DENIED` | 跳 `/403` |
| `USER_DISABLED` | 强制跳 `/account-disabled` |

### 2.3.2 invitation 错误

| 错误码 | 前端动作 |
| --- | --- |
| `INVITATION_NOT_FOUND` | 邀请页展示“链接不存在” |
| `INVITATION_ALREADY_CONSUMED` | 邀请页展示“链接已使用” |
| `INVITATION_REVOKED` | 邀请页展示“链接已撤销” |
| `INVITATION_EXPIRED` | 邀请页展示“链接已过期” |
| `INVITATION_EMAIL_MISMATCH` | 登录完成页或邀请页展示邮箱不匹配提示 |
| `INVITATION_WORKSPACE_INVALID` | 展示 workspace 无效，建议联系管理员 |

### 2.3.3 runtime 错误

| 错误码 | 前端动作 |
| --- | --- |
| `RUNTIME_NOT_FOUND` | 展示未创建状态，引导启动 |
| `RUNTIME_ACTION_CONFLICT` | toast 提示“任务处理中，请稍后再试” |
| `*_ERROR` | 展示最近错误信息与重试按钮 |

---

## 2.4 全局异步交互原则

1. 所有写操作按钮在请求发起后必须进入 loading 状态。
2. 写操作成功后优先局部刷新，不建议整页强刷。
3. 涉及 runtime 的异步动作要以 `taskId` 为中心处理轮询。
4. 轮询结束后必须再刷新一次最终状态接口，不能只相信 task 结果文本。
5. 对 destructive action（删除、撤销、禁用）必须二次确认。

---

## 2.5 页面状态机：登录说明页 `/login`

### 2.5.1 首屏状态

1. `initializing`：请求 `GET /api/v1/auth/options`
2. `ready`：展示当前登录方式
3. `fatal-error`：接口失败时展示“暂时无法获取登录方式”

### 2.5.2 页面元素

- 登录方式列表
- 首版说明文案
- 主按钮：前往登录
- 次级信息：未来支持的登录方式

### 2.5.3 按钮行为

| 按钮 | 状态 | 规则 |
| --- | --- | --- |
| 前往登录 | 默认可点击 | 跳转 Authentik 登录 |
| 前往登录 | loading | 若需要先请求登录 URL，则按钮进入 loading |

---

## 2.6 页面状态机：邀请预览 / 接入页 `/invite/:token`

### 2.6.1 首屏请求

- `GET /api/v1/public/invitations/{token}`

### 2.6.2 状态机

```text
initializing
├─ success(valid=true) -> ready-valid
└─ success(valid=false) -> ready-invalid
   └─ error code -> invalid subtype
```

### 2.6.3 valid 状态下展示内容

- workspace 名称
- role
- targetEmail
- expiresAt
- 主按钮：继续接入

### 2.6.4 invalid 状态下展示内容

按错误码切换文案：

- 已过期
- 已撤销
- 已使用
- 链接不存在
- workspace 无效

### 2.6.5 “继续接入”按钮状态

1. 点击后进入 `submitting`
2. 调 `POST /api/v1/public/invitations/{token}/start`
3. 成功后立刻浏览器跳转 `redirectUrl`
4. 失败则回到 `ready-valid`，并展示错误提示

### 2.6.6 文案要求

邀请页必须明确告诉用户：

- 首次接入将在身份系统中完成账号资料与密码设置
- 该链接通常为一次性链接
- 若链接失效需联系管理员

---

## 2.7 页面状态机：登录完成承接页 `/auth/landing`

### 2.7.1 目标

把“登录成功但前端还没进入业务区”的短暂过程可视化。

### 2.7.2 推荐状态序列

```text
initializing
-> syncing-user
-> applying-invitation (optional)
-> redirecting
```

### 2.7.3 后端交互建议

优先使用后端收口接口：

- `GET /api/v1/auth/post-login`

### 2.7.4 前端展示文案建议

| 状态 | 文案 |
| --- | --- |
| syncing-user | 正在同步账号信息 |
| applying-invitation | 正在应用邀请并绑定工作区 |
| redirecting | 正在进入工作台 |
| fatal-error | 登录已完成，但平台初始化失败 |

### 2.7.5 异常收口

| 场景 | 处理 |
| --- | --- |
| `USER_DISABLED` | 跳 `/account-disabled` |
| `INVITATION_EMAIL_MISMATCH` | 停留页内展示错误，并给出“返回登录 / 联系管理员” |
| 其他失败 | 展示错误块和重试按钮 |

---

## 2.8 页面状态机：账户禁用页 `/account-disabled`

### 2.8.1 页面目标

统一收口 disabled 用户。

### 2.8.2 展示内容

- 账户已禁用标题
- 原因说明（若后端未给明确原因，使用通用文案）
- 联系管理员说明
- 按钮：刷新状态 / 退出登录

### 2.8.3 页面规则

- 不显示业务导航
- 不继续请求普通业务接口
- 仅保留状态刷新和退出动作

---

## 2.9 页面状态机：用户工作台首页 `/workspace`

### 2.9.1 首屏推荐请求

1. `GET /api/v1/auth/me`
2. `GET /api/v1/users/me/runtime/status`
3. `GET /api/v1/models`
4. 可选：`GET /api/v1/users/me/quota`

### 2.9.2 页面分区建议

- 用户身份摘要
- runtime 状态卡
- 进入工作区主入口
- 模型摘要
- quota 摘要

### 2.9.3 runtime 状态渲染规则

| observedState | UI 呈现 | 主操作 |
| --- | --- | --- |
| `creating` | 启动中 | 查看进度 / 禁用重复启动 |
| `running` | 运行中 | 进入工作区 / 停止 |
| `stopped` | 已停止 | 启动 |
| `error` | 启动失败 | 查看错误 / 重试启动 |
| `deleted` | 已删除 | 创建并启动 |

### 2.9.4 页面刷新策略

- 首页不建议常驻高频轮询
- 仅当用户刚触发 runtime 动作时才进入短期 polling
- 页面重新聚焦时可轻量刷新 runtime 状态

---

## 2.10 页面状态机：工作区入口页 `/workspace/entry`

### 2.10.1 首屏请求

- `GET /api/v1/workspace-entry`

### 2.10.2 状态与 UI

| 状态 | 条件 | UI |
| --- | --- | --- |
| ready | `ready=true` | 展示进入工作区主按钮 |
| not-ready | `ready=false` 且无 fatal error | 展示“尚未就绪”与下一步引导 |
| fatal-error | 接口失败 | 展示错误态和重试 |

### 2.10.3 行为规则

- 点击“进入工作区”直接跳 `browserUrl`
- 前端不能缓存旧 `browserUrl` 长期复用，应以最新接口结果为准

---

## 2.11 页面状态机：runtime 管理页 `/workspace/runtime`

### 2.11.1 首屏请求

- `GET /api/v1/users/me/runtime`

### 2.11.2 页面核心状态

```text
initializing
-> ready
   ├─ start-submitting -> polling-task -> refresh-runtime -> ready
   ├─ stop-submitting -> polling-task -> refresh-runtime -> ready
   └─ delete-submitting -> polling-task -> refresh-runtime -> ready
```

### 2.11.3 启动 runtime

1. 点击启动
2. 调 `POST /api/v1/users/me/runtime/start`
3. 返回 `taskId`
4. 轮询 `GET /api/v1/runtime/tasks/{taskId}`
5. 成功后刷新：`GET /api/v1/users/me/runtime/status` 或 `/runtime`

### 2.11.4 停止 runtime

流程同启动，仅 action 为 stop。

### 2.11.5 删除 runtime

1. 点击删除
2. 弹确认框
3. 用户选择 `retentionPolicy`
4. 调 `DELETE /api/v1/users/me/runtime`
5. 进入轮询
6. 轮询结束刷新 runtime 状态

### 2.11.6 按钮禁用规则

| 场景 | 启动 | 停止 | 删除 |
| --- | --- | --- | --- |
| `creating` | 禁用 | 禁用 | 禁用 |
| `running` | 禁用 | 可用 | 可用 |
| `stopped` | 可用 | 禁用 | 可用 |
| `error` | 可用（重试） | 视情况禁用 | 可用 |
| 当前有 task 轮询中 | 全部禁用 | 全部禁用 | 全部禁用 |

### 2.11.7 页面错误呈现

- `lastError` 显示在 runtime 状态卡底部
- 如果 task 失败，toast + 错误块双反馈
- 页面不因局部动作失败而整页崩掉

---

## 2.12 页面状态机：模型列表页 `/workspace/models`

### 2.12.1 首屏状态

- `initializing`：请求 `GET /api/v1/models`
- `ready`：表格 / 卡片方式展示模型列表
- `empty`：无可见模型时展示空态
- `fatal-error`：加载失败

### 2.12.2 页面交互原则

- 首版只读
- 不提供编辑
- 不提供新增 / 删除

---

## 2.13 页面状态机：quota 页 `/workspace/quota`

### 2.13.1 首屏状态

- `initializing`：请求 `GET /api/v1/users/me/quota`
- `ready`
- `empty`
- `fatal-error`

### 2.13.2 UI 建议

- quota 卡片
- 进度条或统计条
- 超额提示

---

## 2.14 页面状态机：管理员用户列表页 `/admin/users`

### 2.14.1 首屏状态

- `initializing`：请求 `GET /api/v1/admin/users`
- `ready`
- `empty`
- `fatal-error`

### 2.14.2 页面交互

- 搜索
- 状态筛选
- 角色筛选
- 行点击进入详情
- 可选：列表内切换用户状态

### 2.14.3 用户状态变更

若支持列表内直接禁用 / 启用：

1. 点击后弹确认框
2. 调 `PATCH /api/v1/admin/users/{userId}/status`
3. 行级 loading
4. 成功后局部刷新当前行或刷新列表

---

## 2.15 页面状态机：管理员用户详情页 `/admin/users/:userId`

### 2.15.1 首屏请求

并发请求：

- `GET /api/v1/admin/users/{userId}`
- `GET /api/v1/admin/users/{userId}/runtime`

### 2.15.2 页面区块建议

- 用户基本信息
- 身份与登录方式
- 用户状态
- runtime 详情
- 危险操作区（禁用 / 启用）

### 2.15.3 局部失败策略

- 用户基本信息成功但 runtime 详情失败时，页面进入 `partial-error`
- runtime 区块单独展示错误并支持重试

---

## 2.16 页面状态机：管理员邀请列表页 `/admin/invitations`

### 2.16.1 首屏请求

- `GET /api/v1/admin/invitations`

### 2.16.2 页面区块建议

- 筛选区：`status / workspaceId / targetEmail`
- 列表区
- 创建 invitation 抽屉 / 弹窗

### 2.16.3 创建 invitation 状态机

```text
idle-form
-> validating
-> submitting
-> success(created)
-> optional copy-url / continue-create
```

### 2.16.4 创建成功后的 UI

- 展示 `inviteUrl`
- 提供复制按钮
- 提供“查看详情”按钮
- 保持表单关闭并刷新列表

### 2.16.5 创建失败处理

- 表单字段错误：行内提示
- 服务错误：表单顶部错误块 + toast

---

## 2.17 页面状态机：管理员邀请详情页 `/admin/invitations/:invitationId`

### 2.17.1 首屏请求

- `GET /api/v1/admin/invitations/{invitationId}`

### 2.17.2 状态对应按钮

| invitation.status | 撤销 | 重发 |
| --- | --- | --- |
| `pending` | 可用 | 可用 |
| `consumed` | 禁用 | 可用（按产品策略决定） |
| `revoked` | 禁用 | 可用 |
| `expired` | 禁用 | 可用 |

### 2.17.3 撤销流程

1. 点击撤销
2. 弹确认框
3. 调 `POST /api/v1/admin/invitations/{invitationId}/revoke`
4. 成功后刷新详情

### 2.17.4 重发流程

1. 点击重发
2. 调 `POST /api/v1/admin/invitations/{invitationId}/resend`
3. 返回新 `inviteUrl`
4. 弹结果提示并允许复制

---

## 2.18 页面状态机：模型治理页 `/admin/models`

### 2.18.1 首屏状态

- `initializing`：请求 `GET /api/v1/admin/models`
- `ready`
- `empty`
- `fatal-error`

### 2.18.2 编辑状态

每次修改单模型策略时：

1. 行进入 `submitting`
2. 调 `PUT /api/v1/admin/models/{modelId}`
3. 成功后更新该行
4. 失败则还原并提示

---

## 2.19 页面状态机：平台凭据治理页 `/admin/provider-credentials`

### 2.19.1 首屏请求

- `GET /api/v1/admin/provider-credentials`

### 2.19.2 页面交互

- 新增凭据
- 校验凭据
- 删除凭据

### 2.19.3 危险操作规则

- 删除必须二次确认
- 校验按钮应有独立 loading，不影响其他行

---

## 2.20 页面状态机：用量汇总页 `/admin/usage`

### 2.20.1 首屏状态

- `initializing`：请求 `GET /api/v1/admin/usage/summary`
- `ready`
- `empty`
- `fatal-error`

### 2.20.2 UI 建议

- 汇总卡片
- 明细表格
- 筛选条件区（若后端已支持）

---

## 2.21 轮询统一规范

### 2.21.1 适用范围

当前首版必须轮询的主要场景：

- runtime 启动
- runtime 停止
- runtime 删除

### 2.21.2 轮询规则建议

- 轮询间隔：2s ~ 3s
- 单次最长轮询：60s ~ 120s
- 页面离开后停止轮询
- 轮询结束后再请求一次最终状态接口

### 2.21.3 轮询状态映射

| task.status | 前端状态 |
| --- | --- |
| `pending` | 等待执行 |
| `running` | 执行中 |
| `succeeded` | 成功，准备刷新最终状态 |
| `failed` | 失败，展示 message |
| `canceled` | 已取消 |

---

## 2.22 Toast 与反馈分层规范

### 2.22.1 成功反馈

适合 toast 的场景：

- invitation 创建成功
- invitation 撤销成功
- provider 凭据校验成功
- runtime 动作已提交

### 2.22.2 需要页面内错误块的场景

- 邀请已失效
- 登录完成但 invitation 应用失败
- runtime 最近错误
- 首屏主请求失败

### 2.22.3 二者同时使用的场景

- runtime task 失败
- invitation start 失败
- 管理后台写操作失败

---

## 2.23 最终结论

首版前端真正难点不在页面数量，而在以下 4 类状态要处理准确：

1. **登录与禁用用户收口**
2. **邀请链路有效性与异常分支**
3. **runtime 异步任务状态机**
4. **管理员写操作的局部刷新与危险操作确认**

只要这些状态统一，UI 与前端实现就不会在联调阶段反复返工。


v 0.1
reno 
2026-03-23 10:54