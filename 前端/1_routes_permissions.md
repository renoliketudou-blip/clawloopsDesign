# 1. 前端页面清单与路由权限文档

## 1.1 文档目标

本文档用于给 UI 与前端开发统一首版页面边界、推荐路由、访问权限、页面职责和跳转关系。

首版前端必须遵守以下原则：

- 身份认证由 **Authentik** 负责，前端不自建密码体系。
- 首版只开放 **本地账号密码** 登录。
- 邀请制接入是普通用户首版接入主链路。
- `workspace / role / user.status` 以 CrewClaw 平台为准。
- `disabled` 用户不能继续访问业务页面。
- `browserUrl` 是受保护入口，不是匿名地址。

---

## 1.2 用户角色与访问级别

| 级别 | 说明 | 典型页面 |
| --- | --- | --- |
| Public | 未登录也可访问 | 登录说明、邀请预览、错误页 |
| Authenticated | 已登录普通用户或管理员 | 工作台、runtime、模型列表、workspace 入口 |
| Admin | 已登录且 `role=admin` | 用户管理、邀请管理、模型治理、凭据治理、用量汇总 |
| Disabled | 已登录但 `user.status=disabled` | 仅允许进入禁用说明页 / 登出 |

---

## 1.3 全局路由守卫规则

### 1.3.1 启动时统一校验顺序

前端应用启动后，推荐按以下顺序处理全局鉴权：

1. 请求 `GET /api/v1/auth/me`
2. 若 `authenticated=false`，进入 Public 路由
3. 若 `authenticated=true`，继续请求 `GET /api/v1/auth/access`
4. 若 `allowed=false && reason=USER_DISABLED`，强制跳转到 `/account-disabled`
5. 若已登录且权限正常，则根据用户角色放行页面

### 1.3.2 管理员路由守卫

管理员页面必须满足：

- `authenticated=true`
- `user.role=admin`
- `user.status=active`

否则：

- 未登录：跳登录
- 已登录但非管理员：跳 `/403`
- 已禁用：跳 `/account-disabled`

### 1.3.3 邀请页面特殊规则

邀请页属于 Public 页面，但内部状态依赖 invitation token 校验结果。前端不得仅根据 URL 中 token 存在与否显示“可接入”，必须先调用：

- `GET /api/v1/public/invitations/{token}`

### 1.3.4 workspace 进入规则

用户不能直接以前端拼接 URL 方式进入工作区。进入工作区前必须先通过：

- `GET /api/v1/workspace-entry`

仅当 `ready=true` 时才允许跳转到 `browserUrl`。

---

## 1.4 推荐路由树

> 说明：以下是前端推荐路由，不等于后端 API 路径。

```text
/
├─ /login
├─ /invite/:token
├─ /auth/landing
├─ /account-disabled
├─ /403
├─ /404
├─ /workspace
│  ├─ /workspace/entry
│  ├─ /workspace/runtime
│  ├─ /workspace/models
│  └─ /workspace/quota
└─ /admin
   ├─ /admin/users
   ├─ /admin/users/:userId
   ├─ /admin/invitations
   ├─ /admin/invitations/:invitationId
   ├─ /admin/models
   ├─ /admin/provider-credentials
   └─ /admin/usage
```

---

## 1.5 页面清单与权限矩阵

| 页面 | 推荐路由 | 权限 | 是否首版必须 | 说明 |
| --- | --- | --- | --- | --- |
| 登录说明页 | `/login` | Public | 建议 | 首版用于展示登录方式说明与登录入口 |
| 邀请预览 / 接入页 | `/invite/:token` | Public | 必须 | 展示 invitation 是否有效、workspace、role、接入按钮 |
| 登录完成承接页 | `/auth/landing` | Authenticated | 建议 | 前端承接后端登录收口结果与跳转 |
| 账户禁用页 | `/account-disabled` | Disabled | 必须 | 展示禁用状态与联系管理员说明 |
| 无权限页 | `/403` | Authenticated | 建议 | 普通用户进入管理员页面时展示 |
| 未找到页 | `/404` | Public | 建议 | 常规系统页 |
| 用户工作台首页 | `/workspace` | Authenticated | 必须 | 登录后的默认业务首页 |
| 工作区入口页 | `/workspace/entry` | Authenticated | 必须 | 进入个人 runtime 前的承接页 |
| runtime 管理页 | `/workspace/runtime` | Authenticated | 必须 | 启动 / 停止 / 删除 runtime |
| 模型列表页 | `/workspace/models` | Authenticated | 必须 | 查看当前用户可见模型 |
| quota 查看页 | `/workspace/quota` | Authenticated | 建议 | 查看额度 / 配额 |
| 用户管理页 | `/admin/users` | Admin | 必须 | 管理员查看用户列表 |
| 用户详情页 | `/admin/users/:userId` | Admin | 必须 | 查看单个用户与 runtime 详情 |
| 邀请管理页 | `/admin/invitations` | Admin | 必须 | 创建 / 查询 / 筛选 invitation |
| 邀请详情页 | `/admin/invitations/:invitationId` | Admin | 建议 | 查看详情、撤销、重发 |
| 模型治理页 | `/admin/models` | Admin | 必须 | 模型启停、可见性、默认策略 |
| 平台凭据治理页 | `/admin/provider-credentials` | Admin | 必须 | 录入 / 校验 / 删除 provider 凭据 |
| 用量汇总页 | `/admin/usage` | Admin | 建议 | 查看全局 usage 汇总 |

---

## 1.6 页面职责明细

### 1.6.1 `/login` 登录说明页

**职责**：

- 调 `GET /api/v1/auth/options`
- 展示首版仅支持“账号密码”
- 展示“前往登录”主按钮
- 告知第三方登录尚未开放

**首版建议**：

- 不自建账号密码表单
- 直接使用 Authentik 托管登录页
- 若部署层已通过 Forward Auth 自动拉起登录，可将本页做成说明页或空壳跳转页

### 1.6.2 `/invite/:token` 邀请预览 / 接入页

**职责**：

- 校验 token 是否有效
- 展示被邀请邮箱、目标 workspace、角色、过期时间
- 在无效场景下展示已过期 / 已撤销 / 已使用等错误态
- 用户点击“继续接入”后，调用 start 接口获取 `redirectUrl` 并跳转到 Authentik enrollment flow

### 1.6.3 `/auth/landing` 登录完成承接页

**职责**：

- 处理登录完成后的前端承接
- 展示“正在完成登录 / 正在应用邀请 / 正在跳转工作台”等中间态
- 根据后端返回结果决定跳转工作台、禁用页或错误页

**说明**：

后端基线存在 `GET /api/v1/auth/post-login`。如果部署采用 BFF 直接重定向，也可以不保留独立可见页面，但前端至少应保留 landing 组件能力，便于处理中间态、异常态和埋点。

### 1.6.4 `/workspace` 用户工作台首页

**职责**：

- 作为登录后的默认业务首页
- 聚合展示当前用户身份、runtime 状态、进入 workspace 的主入口、模型列表摘要、quota 摘要
- 对不同 runtime 状态给出对应 CTA

### 1.6.5 `/workspace/entry` 工作区入口页

**职责**：

- 调 `GET /api/v1/workspace-entry`
- 判断当前用户 workspace 是否可进入
- 当 `ready=true` 时展示“进入工作区”主按钮
- 当 `ready=false` 时给出启动 runtime 或查看错误信息的引导

### 1.6.6 `/workspace/runtime` runtime 管理页

**职责**：

- 调用 runtime 详情与状态接口
- 支持启动 / 停止 / 删除 runtime
- 展示异步任务进度与最近错误
- 删除时弹出确认框并选择 `retentionPolicy`

### 1.6.7 `/workspace/models` 模型列表页

**职责**：

- 只读展示当前用户可见模型
- 不提供普通用户侧 provider 凭据管理
- 不提供模型开关写操作

### 1.6.8 `/workspace/quota` quota 页

**职责**：

- 展示当前用户 quota 使用情况
- 展示配额说明、超额说明、管理员联系信息（可选）

### 1.6.9 `/admin/users` 用户管理页

**职责**：

- 查看用户列表
- 按状态、角色、运行态筛选
- 跳转用户详情页
- 快捷禁用 / 启用用户（若设计采用列表内操作）

### 1.6.10 `/admin/users/:userId` 用户详情页

**职责**：

- 查看用户身份信息、登录方式、状态、runtime 详情
- 修改用户状态（active / disabled）
- 查看该用户 runtime 详情

### 1.6.11 `/admin/invitations` 邀请管理页

**职责**：

- 查看 invitation 列表
- 按 `status / workspaceId / targetEmail` 筛选
- 创建 invitation
- 复制 invite URL
- 进入 invitation 详情页

### 1.6.12 `/admin/invitations/:invitationId` 邀请详情页

**职责**：

- 查看 invitation 基本信息、Authentik 引用、消费状态
- 执行撤销
- 执行重发

### 1.6.13 `/admin/models` 模型治理页

**职责**：

- 查看全局模型列表与策略
- 变更模型开关、可见性与默认策略

### 1.6.14 `/admin/provider-credentials` 平台凭据治理页

**职责**：

- 查看凭据列表与状态
- 新增 provider 凭据
- 校验凭据
- 删除凭据

### 1.6.15 `/admin/usage` 用量汇总页

**职责**：

- 展示全局 usage 汇总
- 按时间范围、模型、用户维度筛选（如后端已支持）

---

## 1.7 默认跳转规则

### 1.7.1 未登录用户

- 访问受保护页面时，跳转 `/login` 或直接进入 Authentik 登录

### 1.7.2 已登录普通用户

- 默认进入 `/workspace`
- 若访问 `/admin/*`，跳 `/403`

### 1.7.3 已登录管理员

- 默认仍建议进入 `/workspace`
- 管理后台作为二级导航进入 `/admin/*`

### 1.7.4 disabled 用户

- 强制跳转 `/account-disabled`
- 顶层只保留“刷新状态 / 退出登录 / 联系管理员”操作

---

## 1.8 页面与后端 API 的最小对应关系

| 页面 | 首屏最小 API |
| --- | --- |
| `/login` | `GET /api/v1/auth/options` |
| `/invite/:token` | `GET /api/v1/public/invitations/{token}` |
| `/auth/landing` | `GET /api/v1/auth/post-login` 或页面启动后调 `GET /api/v1/auth/me` |
| `/workspace` | `GET /api/v1/auth/me` + `GET /api/v1/users/me/runtime/status` + `GET /api/v1/models` |
| `/workspace/entry` | `GET /api/v1/workspace-entry` |
| `/workspace/runtime` | `GET /api/v1/users/me/runtime` |
| `/workspace/models` | `GET /api/v1/models` |
| `/workspace/quota` | `GET /api/v1/users/me/quota` |
| `/admin/users` | `GET /api/v1/admin/users` |
| `/admin/users/:userId` | `GET /api/v1/admin/users/{userId}` + `GET /api/v1/admin/users/{userId}/runtime` |
| `/admin/invitations` | `GET /api/v1/admin/invitations` |
| `/admin/invitations/:invitationId` | `GET /api/v1/admin/invitations/{invitationId}` |
| `/admin/models` | `GET /api/v1/admin/models` |
| `/admin/provider-credentials` | `GET /api/v1/admin/provider-credentials` |
| `/admin/usage` | `GET /api/v1/admin/usage/summary` |

---

## 1.9 首版不建议前端新增的页面

首版不建议额外做以下页面，以避免前后端边界漂移：

- 自建注册页
- 自建重置密码页
- 自建修改密码页
- 自建第三方登录选择页
- 自助管理 workspace / role 页面
- 普通用户侧 provider 凭据页面

这些能力在首版基线下分别属于：

- Authentik
- 管理后台
- 平台后端绑定逻辑

---

## 1.10 路由命名建议

### 1.10.1 路由 path 建议

- 业务区统一用 `/workspace/*`
- 管理区统一用 `/admin/*`
- 系统类页面统一放根级

### 1.10.2 路由 name 建议

```text
login
invite-preview
auth-landing
account-disabled
workspace-home
workspace-entry
workspace-runtime
workspace-models
workspace-quota
admin-users
admin-user-detail
admin-invitations
admin-invitation-detail
admin-models
admin-provider-credentials
admin-usage
```

---

## 1.11 最终结论

首版前端页面体系应围绕四个中心组织：

1. **Authentik 登录接入**
2. **邀请制接入**
3. **用户工作台与 runtime 管理**
4. **管理员治理后台**

页面不宜过多拆散，重点是把路由权限、禁用用户收口、邀请流程和 runtime 异步态处理清楚。
