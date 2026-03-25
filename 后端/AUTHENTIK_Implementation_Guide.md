# ClawLoops × 官方 Authentik 实施文档（组映射修订）

这份文档是给你直接在 Cursor 里开干用的，不再停留在“讨论方案”，而是明确到：

- 官方 Authentik 首版该怎么接
- ClawLoops 自己还要补哪些业务层能力
- RuntimeManager / Orchestrator 的新冻结边界是什么
- 部署、流程、接口、代码落点如何安排

---

## 1. 先说结论

### 1.1 你的文档要不要改？

**要改，而且这次要把 Authentik 与 runtime V1 一起改到能冻结开发。**

本次必须冻结的点至少包括：

1. 把 Authentik 从“只是登录入口”升级为“正式身份系统”
2. 把 invitation 从一句需求升级成完整的业务对象与时序
3. 把“首版只开本地账号密码”写成硬规则
4. 把“密码管理全部交给 Authentik”写进架构、契约、接口三份文档
5. 把应用管理员角色来源统一冻结为 Authentik Groups 映射
6. 把 invitation 生命周期统一冻结为“双层模型 + 延迟创建”
7. 把 `post-login`、`start` 的幂等性和事务边界写清楚
8. 把 workspace 子域名保护、`ready=true` 跳转规则、字段命名冻结写成强规则
9. 把 runtime 的 `imageRef / 18789 / clawloops_shared / compat / drift / 同步边界` 一次性定死

### 1.2 官方 Authentik 能否满足你的需求？

**能满足大部分，而且首版足够用。**

能直接利用的官方能力：

- 首次初始化管理员密码
- 本地账号密码登录
- Enrollment Flow
- Invitation Stage
- User Write Stage
- User Login Stage
- Recovery / User Settings / 强制改密能力
- Proxy Outpost + Traefik Forward Auth

不能直接只靠 Authentik 独立解决、建议由 ClawLoops 负责的部分：

- invitation 绑定 `workspace / workspaceRole`
- invitation 的业务有效性与是否已消费
- 用户第一次进来后该进哪个 workspace
- runtime 是否允许启动
- disabled / quota / runtime 资源治理
- `volumeId -> host path` 解析与 runtime drift 重建决策

可以直接利用、并且本次改为冻结设计的部分：

- Authentik Group 作为应用级角色来源
- 例如 `clawloops-admins -> admin`
- 后端读取 `X-Authentik-Groups` 后映射 `appRole / isAdmin`

### 1.3 首版推荐方案

推荐的首版不是“平台自己写认证”，也不是“把所有业务角色都塞进 Authentik”，而是下面这套：

- Authentik 负责身份、密码、会话、enrollment flow
- Authentik Groups 负责应用级角色输入
- ClawLoops 负责 `workspace membership / invitation / runtime` 业务真相
- Traefik + Outpost 负责前置鉴权
- 首版只启用本地账号密码
- 邀请链接走“平台 token + enrollment flow”模式
- 身份侧 invitation 在 `start` 阶段延迟创建，不在管理员创建 invitation 时预生成
- 后端读取 `X-Authentik-Groups`，命中 `clawloops-admins` 时把应用内角色设为 `admin`
- Orchestrator 对用户侧暴露异步任务
- RuntimeManager 对内部执行层暴露同步接口

---

## 2. 你这套需求的推荐解释

你的原始诉求里最容易卡住的是这句：

> 首次可用免密码链接进入 - 进入让用户修改帐号密码

### 2.1 不推荐的理解

不推荐理解成：

1. 先 magic link 直接登录
2. 进入系统后再弹一个强制改密流程
3. 平台自己再维护一个“首次登录状态机”

### 2.2 推荐的理解

推荐把它理解成：

1. 用户第一次不需要预先知道密码
2. 用户通过**一次性 invitation 链接**进入
3. 这个 invitation 链接本身就是“首次免密码入口”
4. 在 Authentik 的 enrollment flow 里，用户填写资料并设置密码
5. 完成后由 Authentik 自动登录
6. ClawLoops 根据 invitation 完成 `workspace / workspaceRole` 绑定

---

## 3. 首版边界（必须冻结）

```text
首版直接用官方 Authentik
不改源码
Docker 分容器部署
平台运行链路统一使用 clawloops_shared
首个初始化管理员按默认 akadmin
应用管理员角色由 Authentik Groups 映射
只开本地账号密码
后续再加微信/飞书/钉钉/Google/GitHub/企业 SSO
RuntimeManager internal API 同步执行
Orchestrator 对外异步返回 taskId
```

把这几条当成首版硬边界，不要在开发中途漂移。

---

## 4. 组件与部署方式

### 4.1 推荐容器清单

建议至少拆成：

- `traefik`
- `clawloops-web`
- `clawloops-api`
- `runtime-manager`
- `authentik-postgresql`
- `authentik-redis`
- `authentik-server`
- `authentik-worker`
- `authentik-proxy-outpost`
- `litellm`
- `litellm-postgresql`

### 4.2 Docker 网络

建议统一定义：

```yaml
networks:
  clawloops_shared:
    driver: bridge
```

最低必须接入该网络的容器：

- `traefik`
- `clawloops-api`
- `runtime-manager`
- `litellm`
- per-user runtime

关于 Authentik：

- 不要求所有 Authentik 组件都进入 `clawloops_shared`
- 最低要求是 `authentik-proxy-outpost` 与 Traefik / 受保护应用网络可达
- `authentik-postgresql / authentik-redis` 通常不需要暴露到该共享网络

### 4.3 为什么一定要有共享网络

因为首版需要同时满足：

- Traefik 能访问 ClawLoops 与 Outpost
- Outpost 能访问 Authentik Core
- runtime 能通过 `http://litellm:4000` 访问 LiteLLM
- RuntimeManager 能把 per-user runtime 接入统一服务发现平面

---

## 5. Authentik 首次安装策略

### 5.1 重要现实约束

如果你坚持：

- 用官方 Authentik
- 不改源码

那么 **首个初始化管理员账号默认是 `akadmin`**，而不是 `admin`。

### 5.2 这对你意味着什么

#### 情况 A：你只是想要“首个登录的是管理员”

那官方能力直接够用：

- 第一次进入初始化流程
- 设置管理员密码
- 后续管理员再登录

#### 情况 B：你要求“用户名必须字面叫 admin”

那就不能靠官方初始化页直接做到。

推荐处理方式：

1. 首次用官方初始化管理员完成配置
2. 在 Authentik 里新增本地管理员 `admin`
3. `admin` 成为日常使用管理员
4. 原始 bootstrap 管理员只保留 break-glass 用途

### 5.3 首版正式建议

**建议接受“管理员角色首登”这个定义，而不是执着用户名必须写死 `admin`。**

---

## 6. Authentik 首版需要配的对象

首版最少要配下面这些对象：

1. Brand
2. default authentication flow（本地密码）
3. invitation enrollment flow
4. recovery / user settings flow（可沿用官方默认）
5. ClawLoops 平台 Application
6. Proxy Provider
7. Proxy Outpost

---

## 7. 登录流怎么配

### 7.1 首版登录方式

只开：

- 用户名优先输入（兼容邮箱输入）
- 密码

不开放：

- Google
- GitHub
- 企业 SSO
- 微信
- 钉钉
- 飞书

### 7.2 Authentik 侧流配置目标

你要得到的效果是：

- 登录页只看到本地账号密码
- 不出现其他 Source 按钮
- 登录成功后能回到 ClawLoops

### 7.3 你在 Cursor 里实现时对应的系统边界

- Authentik 决定“这个人登录成功没有”
- Authentik Groups 提供“这个人具备什么应用级角色”的输入
- ClawLoops 决定“这个人虽然登录了，但是否可访问业务”

也就是说：

- 认证成功 ≠ 业务允许
- `appRole` 可由 `X-Authentik-Groups` 映射得出
- ClawLoops 还要检查 `user.status / workspace membership / runtime rules`

---

## 8. 邀请流的推荐实现

### 8.1 业务对象设计

建议在 ClawLoops 数据库增加 `invitations` 表，至少包含：

```sql
id
invite_token_hash
target_email
workspace_id
role
status
expires_at
consumed_at
consumed_by_user_id
authentik_invitation_ref
created_by_user_id
created_at
updated_at
```

> `status` 首版只存 `pending / consumed / revoked`；`expired` 由 `expires_at` 派生，不单独落库。

### 8.2 为什么不能只靠 Authentik invitation

因为真正要绑定的是：

- `workspaceId`
- `workspaceRole`
- 业务用户状态

这些不是身份系统真相，而是平台真相。

所以推荐模式是：

- **ClawLoops invitation = 业务真相**
- **Authentik invitation = 身份执行入口**

### 8.3 外部发给用户的链接

外部统一发：

```text
https://clawloops.example.com/invite/{platform_token}
```

不要直接把 Authentik 的原始 `itoken` 链接当成对外业务链接。

### 8.4 推荐流程

#### 第一步：管理员创建 invitation

后台表单：

- `targetEmail`（真实邮箱或代理邮箱）
- `loginUsername`（无真实邮箱用户必填）
- 目标 workspace
- role
- 有效期

系统动作：

- 创建 platform invitation
- 生成一次性 platform token
- 返回对外 inviteUrl

#### 第二步：用户打开 invitation 页

前端展示：

- 你被邀请加入哪个 workspace
- 你将获得什么角色
- 推荐登录账号（优先展示 `loginUsername`）
- 链接是否有效
- 一个“继续接入”按钮
- 若用户属于无真实邮箱场景，不把代理邮箱作为主文案暴露给普通用户

#### 第三步：前端点继续接入

后端动作：

1. 校验 platform token
2. 校验 invitation 业务状态
3. 写入 pending invitation cookie / session
4. 延迟创建或换取 Authentik enrollment URL
5. 返回 `redirectUrl`

#### 第四步：跳转到 Authentik enrollment flow

Enrollment Flow 建议顺序：

1. Invitation Stage
2. Prompt Stage（用户名/姓名/邮箱槽位/密码/重复密码）
3. User Write Stage
4. User Login Stage

无真实邮箱用户的体验建议：

- `email` 使用系统分配的代理邮箱预填
- 若 Authentik Flow 支持配置，代理邮箱字段应尽量只读或隐藏，不要求用户理解其技术含义
- 用户界面优先显示“登录用户名”，而不是让用户记住代理邮箱

#### 第五步：登录成功回到 ClawLoops

系统动作：

1. 用 Authentik 会话识别当前用户
2. 调 `/internal/users/sync`
3. 执行身份邮箱槽位强校验
4. 从 `X-Authentik-Groups` 映射 `appRole`
5. 检查 pending invitation
6. 绑定 workspace / workspaceRole
7. 标记 invitation consumed
8. 清理 cookie / session
9. 若 `appRole=admin` 则跳转管理后台；否则进入工作台

### 8.5 `start` 与 `post-login` 的关键规则

- `start` 必须幂等
- 同一浏览器会话只保留一个有效 pending invitation session
- pending session TTL 建议 10–30 分钟
- `post-login` 必须幂等
- 同一 `invitationId + userId` 只能成功消费一次
- `consume invitation` 与 `workspace membership binding` 必须原子，或定义清晰补偿逻辑

### 8.6 身份邮箱槽位校验固定位置

- 身份邮箱槽位强校验统一放在 `post-login` 阶段执行
- 即已经拿到 `subjectId` 与 `email` 之后再比对 `targetEmail`
- `targetEmail` 可为真实邮箱或代理邮箱
- 不要把校验漂移到 preview、start 或前端页面逻辑里

---

## 9. 首次密码策略怎么做最稳

### 9.1 推荐基线

**在 invitation enrollment flow 里直接设置密码。**

也就是：

- 用户不需要提前知道密码
- 但会在“完成接入”页当场设置密码
- 这一步由 Authentik Flow 完成

### 9.2 为什么这比“进来后再改密”更好

因为会少掉一整层状态管理：

- 不需要“首次登录成功但尚未改密”的中间态
- 不需要平台自己写“首次改密强跳页”
- 不需要多一次跳转

### 9.3 平台明确禁止的事

- 保存密码
- 生成正式临时密码
- 提供密码落库接口
- 实现独立改密 API

---

## 10. 平台与 Authentik 的职责边界

### 10.1 Authentik 负责

- 本地账号目录
- 密码存储与校验
- 登录会话
- 用户组真相
- enrollment flow
- invitation stage
- user settings / recovery / reset password
- forward auth

### 10.2 ClawLoops 负责

- 用户业务状态（active / disabled）
- 应用级角色映射策略（例如 `clawloops-admins -> admin`）
- invitation 的业务有效性
- workspace / workspaceRole 绑定
- runtime 资源真相
- quota / usage / 模型治理
- 后台 invitation 管理

### 10.3 Runtime Orchestrator 负责

- 用户侧异步 task 生命周期
- `volumeId -> host path` 解析
- V1 固定镜像 / 固定命令 / effectiveRetentionPolicy 决策
- 收到 `RUNTIME_CONTRACT_DRIFT` 后决定是否重建
- 把 RM 返回结果写回 binding / task

### 10.4 RuntimeManager 负责

- 宿主机目录初始化
- 容器创建 / 启动 / 停止 / 删除
- 固定挂载、固定网络、固定 alias 接入
- 启动探测与当前事实状态返回
- 关键 contract drift 检测

### 10.5 不要混淆的点

不要让：

- Authentik 成为 `workspace membership / workspaceRole` 的唯一真相库
- 误以为“应用级角色来自 group 映射”就等于“workspace 权限也应该全部来自 group”
- RuntimeManager 成为外层任务系统
- RM 自动解析 secret 文件并注入 env
- RM 自动 stop+delete+recreate 以“修复” drift

---

## 11. Traefik + Outpost + Forward Auth 怎么接

### 11.1 用途

这部分是为了保护：

- 平台控制面
- 用户 workspace 子域名

### 11.2 目标效果

- 用户访问平台或 workspace 时，先过 Authentik
- 未登录就被送去登录
- 已登录才进入应用
- 业务层还要再检查用户状态与归属

### 11.3 典型思路

Traefik 给业务路由挂一个 `forwardAuth` middleware，指向 Outpost：

```yaml
http:
  middlewares:
    authentik:
      forwardAuth:
        address: http://authentik-proxy:9000/outpost.goauthentik.io/auth/traefik
        trustForwardHeader: true
        authResponseHeaders:
          - X-authentik-username
          - X-authentik-email
          - X-authentik-name
          - X-authentik-uid
          - X-Authentik-Groups
```

然后平台主域名和所有 workspace 子域名都挂这个 middleware。

### 11.4 你的应用层要做什么

模块 1 负责读取认证后的身份上下文，最少映射出：

- `subjectId`
- `email`
- `username`
- `name`
- `groups`
- `appRole`

再触发用户同步与业务校验。

推荐规则：

- 命中 `X-Authentik-Groups` 中的 `clawloops-admins`，则 `appRole=admin`
- 未命中时默认 `appRole=user`
- `isAdmin = appRole === 'admin'`

### 11.5 安全硬规则

- 所有 workspace 子域名必须统一经过 Traefik + Authentik Forward Auth
- `browserUrl` 属于受保护入口，不是匿名公开地址
- 前端只有在 `ready=true` 时才允许跳转
- `admin` 登录后默认进入 `/admin`
- `workspace-entry` 只负责非管理员用户的工作区跳转

### 11.6 admin 首页规则

- `/admin` 不是空白占位页
- `/admin` 首页由后端聚合接口提供摘要数据
- 管理员即使没有 workspace membership，也必须能稳定进入 `/admin`
- 首页最小包含用户摘要、invitation 摘要、runtime 摘要与待处理事项

---

## 12. 推荐的数据模型补充

### 12.1 Invitation

```ts
interface Invitation {
  invitationId: string
  inviteTokenHash: string
  targetEmail: string
  loginUsername?: string | null
  workspaceId: string
  workspaceRole: string
  status: 'pending' | 'consumed' | 'revoked'
  expiresAt: string
  consumedAt?: string | null
  consumedByUserId?: string | null
  authentikInvitationRef?: string | null
  lastError?: string | null
}
```

### 12.2 WorkspaceMembership

```ts
interface WorkspaceMembership {
  workspaceId: string
  userId: string
  workspaceRole: string
  status: 'active' | 'disabled'
  source: 'invitation' | 'manual' | 'sync'
}
```

### 12.3 AuthContext

```ts
interface AuthContext {
  subjectId: string
  email: string
  username?: string | null
  name?: string | null
  groups?: string[]
  appRole: 'admin' | 'user'
  isAdmin: boolean
  provider: 'authentik'
  method: 'local_password'
}
```

### 12.4 RuntimeEnsureRunningRequest（V1）

```ts
interface RuntimeEnsureRunningRequest {
  userId: string
  runtimeId: string
  volumeId: string
  routeHost: string
  retentionPolicy: 'preserve_workspace' | 'wipe_workspace'
  compat: {
    openclawConfigDir: string
    openclawWorkspaceDir: string
  }
  configMount?: {
    configFilePath?: string
    secretFilePath?: string
  }
  env?: Record<string, string>
  envOverrides?: Record<string, string>
}
```

冻结说明：

- V1 中没有 `imageRef`
- `compat` 是必填
- `networkName / gatewayPort` 不再出现在请求体中

---

## 13. 推荐 API 落点

### 13.1 公开接口

```text
GET  /api/v1/auth/options
GET  /api/v1/public/invitations/{token}
POST /api/v1/public/invitations/{token}/start
POST /api/v1/auth/post-login
```

### 13.2 管理员接口

```text
GET  /api/v1/admin/invitations
POST /api/v1/admin/invitations
GET  /api/v1/admin/invitations/{id}
POST /api/v1/admin/invitations/{id}/revoke
POST /api/v1/admin/invitations/{id}/resend
```

### 13.3 内部接口

```text
POST /internal/users/sync
POST /internal/invitations
POST /internal/invitations/{id}/consume
POST /internal/invitations/{id}/revoke
POST /internal/users/{userId}/runtime-binding/ensure
```

### 13.4 runtime 删除接口建议

首版建议改为：

```text
POST /api/v1/users/me/runtime/delete
```

不要继续依赖 `DELETE` body。

### 13.5 RuntimeManager internal 接口说明

```text
POST /internal/runtime-manager/containers/ensure-running
POST /internal/runtime-manager/containers/stop
POST /internal/runtime-manager/containers/delete
GET  /internal/runtime-manager/containers/{runtimeId}
```

冻结边界：

- RM internal 接口全部同步
- `GET` 查不到容器事实时返回 `200 + observedState=deleted`
- 命中多个容器时返回 `409 RUNTIME_ACTION_CONFLICT`
- 关键 drift 返回 `409 RUNTIME_CONTRACT_DRIFT`

---

## 14. 在 Cursor 里应该怎么拆任务

### 14.1 第一组：文档与类型先行

先做：

1. 更新架构文档
2. 更新 MVP 契约
3. 更新 API 规范
4. 更新 RuntimeManager 文档
5. 增加类型定义：
   - `Invitation`
   - `WorkspaceMembership`
   - `AuthContext`
   - `RuntimeEnsureRunningRequest`

### 14.2 第二组：后端真相对象

先落库：

1. `invitations` 表
2. `workspace_memberships` 表（如果还没有清晰实体）
3. `users` 表补 `subject_id / auth_provider / auth_method`

### 14.3 第三组：Auth 入口

做：

1. `/api/v1/auth/options`
2. `/api/v1/auth/me`
3. `POST /api/v1/auth/post-login`
4. `internal/users/sync`
5. `X-Authentik-Groups -> appRole/isAdmin` 映射

### 14.4 第四组：invitation 主链路

做：

1. 管理员创建 invitation
2. invitation 预览页接口
3. invitation start 接口
4. pending invitation cookie / session
5. post-login consume 逻辑
6. 身份邮箱槽位强校验
7. Authentik → ClawLoops 错误映射

### 14.5 第五组：Traefik / Outpost 接入

做：

1. 平台域名挂 forward auth
2. workspace 子域名挂 forward auth
3. `/outpost.goauthentik.io/*` 路由放行到 outpost
4. 验证 header 是否透传到应用

### 14.6 第六组：runtime 收敛

做：

1. Orchestrator 改为对外异步 task
2. RM `ensure-running` 请求体删除 `imageRef`
3. 把 `compat` 提升为必填
4. 固定 `clawloops_shared`
5. 固定 `internalEndpoint = http://rt-<runtimeId>:18789`
6. 实现 `RUNTIME_CONTRACT_DRIFT`
7. 实现 `stop(nonexistent)=stopped / delete(nonexistent)=deleted`
8. 实现 `GET not found => observedState=deleted`

### 14.7 第七组：联调验收

验收 13 条：

1. 首次管理员能初始化成功
2. 登录页只有本地密码
3. invitation 可创建
4. invitation 链接一次性有效
5. 用户完成接入后能绑定 workspace / workspaceRole
6. `start` 与 `post-login` 可安全重复调用
7. workspace 子域名会被 Authentik 保护
8. 前端只在 `ready=true` 时跳转 `browserUrl`
9. RM internal API 不再接收 `imageRef`
10. runtime 统一跑在 `clawloops_shared + 18789 + rt-<runtimeId>`
11. `clawloops-admins` 组成员登录后得到 `appRole=admin`
12. `admin` 登录后默认进入 `/admin`，且不依赖 workspace membership
13. `/api/v1/admin/home` 能返回首页摘要与待处理事项

---

## 15. 你可以直接交给 Cursor 的开发任务清单

下面这段可以直接复制到 Cursor 作为任务说明。

```md
目标：在不修改上游源码的前提下，把 ClawLoops 接入官方 Authentik，并把 runtime V1 contract 一起冻结到可开发状态。

边界：
- 使用官方 Authentik
- Docker 分容器部署
- 平台运行链路统一使用 clawloops_shared
- 首个初始化管理员按默认 akadmin
- 应用管理员角色通过 Authentik Groups 映射
- 首版只开本地账号密码
- 邀请制接入
- 平台保持 workspace membership 业务真相
- 所有 workspace 子域名必须经过 Traefik + Authentik Forward Auth
- Orchestrator 对外异步返回 taskId
- RuntimeManager internal API 同步执行

需要完成：
1. 新增 Invitation 数据模型与迁移
2. 新增 WorkspaceMembership 绑定逻辑
3. 新增 /api/v1/auth/options
4. 新增 /api/v1/public/invitations/{token}
5. 新增 /api/v1/public/invitations/{token}/start（幂等）
6. 新增 POST /api/v1/auth/post-login（幂等）
7. 新增 /api/v1/admin/invitations 系列接口
8. 新增 /internal/invitations 与 /internal/invitations/{id}/consume
9. 保持现有 runtime 对外接口主体不变，但删除改为 POST /runtime/delete
10. 把 subjectId/authProvider/authMethod 接入用户同步
11. 解析 `X-Authentik-Groups` 并实现 `clawloops-admins -> admin`
12. 在 Traefik 上为平台域名和 workspace 子域名挂 Authentik forward auth
13. RuntimeManager V1 请求体删除 imageRef，compat 必填
14. 固定 runtime 网络为 clawloops_shared，固定 internalEndpoint 为 http://rt-<runtimeId>:18789
15. 增加 RUNTIME_CONTRACT_DRIFT / RUNTIME_START_FAILED / RUNTIME_STOP_FAILED / RUNTIME_DELETE_FAILED
16. 平台禁止保存密码、禁止实现独立改密 API
17. `admin` 登录后默认进入 `/admin`；`workspace-entry` 是唯一工作区跳转入口，前端只在 ready=true 时跳转
18. 新增 `GET /api/v1/admin/home`，作为 `/admin` 的首屏聚合接口

推荐链路：
- 平台 token 负责业务入口
- Authentik invitation/enrollment flow 负责身份接入
- start 阶段延迟创建或换取 enrollment URL
- 登录完成后由 post-login 收口，先映射 `appRole`，再绑定 workspace/workspaceRole
- 身份邮箱槽位强校验在 post-login 阶段执行
- 无真实邮箱用户的 UI 默认优先展示 `loginUsername`
- Orchestrator 决策，RuntimeManager 执行
```

---

## 16. 最小可用实现顺序

1. 先把 Authentik 跑起来，并完成管理员初始化
2. 把 Traefik + Outpost 前置鉴权接到平台主域名
3. 实现 `/auth/me` 与 `/internal/users/sync`
4. 实现 invitation 表和管理员创建 invitation
5. 做 invitation preview + start + post-login consume
6. 把 workspace 子域名也挂上 forward auth
7. 把 runtime V1 contract 收敛到固定镜像 / 固定网络 / 固定端口 / `compat` 必填
8. 联调 `workspace-entry`

---

## 17. 常见误区

### 误区 1：把 Authentik Group 映射扩大成全部授权真相

不对。应用级角色可以来自 Authentik Groups，但 `workspace membership / workspaceRole` 仍然应该留在平台业务侧。

### 误区 2：对外直接发 Authentik invitation 链接

不推荐。你会失去业务可控性和追踪性。

### 误区 3：平台自己再写一套密码修改 API

不推荐。首版会变复杂，而且会破坏职责边界。

### 误区 4：首版同时上本地密码 + 多个第三方登录

不推荐。会让联调复杂度暴涨。

### 误区 5：所有人登录后都先进 `workspace-entry`

不对。`admin` 登录后默认应进入 `/admin`；只有非管理员用户才进入 `workspace-entry` 流程。

### 误区 6：拿到 `browserUrl` 就直接跳

不对。前端只能在 `workspace-entry.ready=true` 时跳转，且 URL 始终受前置鉴权保护。

### 误区 7：让 RuntimeManager 自己“顺手修好” drift

不对。RM 只能检测并返回 `RUNTIME_CONTRACT_DRIFT`，真正是否 stop+delete+recreate 由 Orchestrator 决策。

---

## 18. 最后的正式建议

如果目标是：

- 尽快可落地
- 不改上游源码
- 以后还能扩展到 Google / GitHub / 企业 SSO / 微信 / 钉钉 / 飞书

那么首版最稳的方案就是：

1. **官方 Authentik 负责身份、密码、会话**
2. **Authentik Groups 提供应用级角色输入，`clawloops-admins` 映射到 `admin`**
3. **ClawLoops 负责 invitation 业务真相和 workspace/workspaceRole 绑定**
4. **Traefik + Outpost 负责统一前置鉴权**
5. **首版只做本地密码**
6. **把 invitation 链接定义成一次性免密码接入入口**
7. **在 enrollment flow 里直接设置用户密码**
8. **采用延迟创建 Authentik invitation 的模式**
9. **把管理员首页与工作区跳转分开：`admin` 默认进 `/admin`，非管理员用户再走 `workspace-entry`**
10. **给 `/admin` 一个真正可用的首页，并用聚合接口返回摘要与待办**
11. **把 runtime V1 明确定成 `clawloops_shared + 18789 + rt-<runtimeId> + compat 必填`**
12. **让 Orchestrator 负责异步任务，让 RuntimeManager 只做同步执行器**

这套方案更贴近统一 IAM 接入思路，也能直接解决“用户已认证但 `isAdmin=false`”这类应用管理员判定问题。

---

v0.10-无真实邮箱用户友好修订
reno  
2026-03-25 14:47