# CrewClaw × 官方 Authentik 实施文档

这份文档是给你直接在 **Cursor** 里开干用的，不再停留在“讨论方案”，而是明确到：

- 你现在的文档应该怎么改；
- 官方 Authentik 能做到什么；
- 哪些地方要由 CrewClaw 自己补一层；
- 首版应该如何拆任务；
- 接口、部署、流程、代码落点如何安排。

---

## 1. 先说结论

### 1.1 你的文档要不要改？

**要改，而且建议一次性改到位。**

必须改动的点有 4 个：

1. 把 Authentik 从“只是登录入口”升级为“正式身份系统”。
2. 把 invitation 从一句需求，升级成完整的业务对象与时序。
3. 把“首版只开本地账号密码”写成硬规则。
4. 把“密码修改机制交给 Authentik”写进架构、契约、接口三份文档。

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

不能直接只靠 Authentik 独立解决、建议由你平台负责的部分：

- invitation 绑定 `workspace / role`
- invitation 的业务有效性与是否已消费
- 用户第一次进来后该进哪个 workspace
- runtime 是否允许启动
- disabled / quota / runtime 资源治理

### 1.3 首版推荐方案

**推荐的首版不是“平台自己写认证”，也不是“把业务全塞进 Authentik”，而是下面这套：**

- Authentik 负责身份、密码、会话、enrollment flow。
- CrewClaw 负责 workspace / role / invitation / runtime 业务真相。
- Traefik + Outpost 负责前置鉴权。
- 首版只启用本地账号密码。
- 邀请链接走“一次性 token + enrollment flow”模式。

---

## 2. 你这套需求的推荐解释

你的原始诉求里最容易卡住的是这句：

> 首次可用免密码链接进入 - 进入让用户修改帐号密码

### 2.1 不推荐的理解

不推荐理解成：

1. 先 magic link 直接登录；
2. 进入系统后再弹一个强制改密流程；
3. 平台自己再维护一个“首次登录状态机”。

因为首版这样做会把流程拉长，而且平台和 Authentik 会出现职责重叠。

### 2.2 推荐的理解

推荐把它理解成：

1. 用户第一次不需要预先知道密码；
2. 用户通过**一次性 invitation 链接**进入；
3. 这个 invitation 链接本身就是“首次免密码入口”；
4. 在 Authentik 的 enrollment flow 里，用户填写资料并设置密码；
5. 完成后由 Authentik 自动登录；
6. CrewClaw 根据 invitation 完成 workspace / role 绑定。

这样你既满足了“首登免密码”，又没有引入额外魔法状态机。

---

## 3. 首版边界（必须冻结）

```text
首版直接用官方 Authentik
不改源码
Docker 分容器、同网络
只开本地账号密码
后续再加微信/飞书/钉钉/Google/GitHub/企业 SSO
```

把这 5 条当成首版硬边界，不要在开发中途漂移。

---

## 4. 组件与部署方式

### 4.1 推荐容器清单

建议至少拆成：

- `traefik`
- `crewclaw-web`
- `crewclaw-api`
- `runtime-manager`
- `authentik-postgresql`
- `authentik-redis`
- `authentik-server`
- `authentik-worker`
- `authentik-proxy-outpost`
- `litellm`
- `litellm-postgresql`

### 4.2 Docker 网络

建议统一加入一个内部网络，例如：

```yaml
networks:
  crewclaw_net:
    driver: bridge
```

### 4.3 为什么一定要同网络

因为首版需要同时满足：

- Traefik 能访问 CrewClaw 与 Outpost
- Outpost 能访问 Authentik Core
- CrewClaw 能访问 Authentik 管理接口（若你用 API/脚本初始化）
- runtime manager 能访问内部服务

同网络最省事，也最符合你当前单机部署模型。

---

## 5. Authentik 首次安装策略

### 5.1 重要现实约束

如果你坚持：

- 用官方 Authentik
- 不改源码

那么**首个初始化管理员账号默认是 `akadmin`**，而不是 `admin`。

### 5.2 这对你意味着什么

#### 情况 A：你只是想要“首个登录的是管理员”

那官方能力直接够用：

- 第一次进入初始化流程
- 设置管理员密码
- 后续管理员再登录

#### 情况 B：你要求“用户名必须字面叫 admin”

那就不能靠官方初始化页直接做到。

推荐处理方式：

1. 首次用官方初始化管理员完成配置；
2. 在 Authentik 里新增本地管理员 `admin`；
3. `admin` 成为日常使用管理员；
4. 原始 bootstrap 管理员只保留 break-glass 用途。

### 5.3 首版正式建议

**建议你接受“管理员角色首登”这个定义，而不是执着用户名必须写死 `admin`。**

这是不改源码前提下最稳的路径。

---

## 6. Authentik 首版需要配的对象

首版最少要配下面这些对象：

1. Brand
2. default authentication flow（本地密码）
3. invitation enrollment flow
4. recovery / user settings flow（可沿用官方默认）
5. CrewClaw 平台 Application
6. Proxy Provider
7. Proxy Outpost

---

## 7. 登录流怎么配

### 7.1 首版登录方式

只开：

- 用户名 / 邮箱
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

- 登录页只看到本地账号密码；
- 不出现其他 Source 按钮；
- 登录成功后能回到 CrewClaw。

### 7.3 你在 Cursor 里实现时对应的系统边界

- Authentik 决定“这个人登录成功没有”。
- CrewClaw 决定“这个人虽然登录了，但是否可访问业务”。

换句话说：

- 认证成功 ≠ 业务允许
- CrewClaw 还要检查 `user.status / workspace membership / runtime rules`

---

## 8. 邀请流的推荐实现

这是首版的核心。

### 8.1 业务对象设计

建议你在 CrewClaw 数据库增加 `invitations` 表，至少包含：

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

### 8.2 为什么不能只靠 Authentik invitation

因为你真正要绑定的是：

- `workspaceId`
- `role`
- 业务用户状态

这些不是身份系统真相，而是平台真相。

所以推荐模式是：

- **CrewClaw invitation = 业务真相**
- **Authentik invitation = 身份执行入口**

### 8.3 外部发给用户的链接

外部统一发：

```text
https://crewclaw.example.com/invite/{platform_token}
```

不要直接把 Authentik 的原始 `itoken` 链接当成你对外的业务链接。

原因：

- 你要先做业务校验；
- 你要先知道这个 invitation 绑定哪个 workspace / role；
- 你还要留出未来“撤销 / 审批 / 重发 / 追踪”的空间。

### 8.4 推荐流程

#### 第一步：管理员创建 invitation

后台表单：

- 目标邮箱
- 目标 workspace
- role
- 有效期

系统动作：

- 创建 platform invitation
- 生成一次性 token
- 准备 Authentik invitation（或延迟生成）
- 返回对外 inviteUrl

#### 第二步：用户打开 invitation 页

前端展示：

- 你被邀请加入哪个 workspace
- 你将获得什么角色
- 链接是否有效
- 一个“继续接入”按钮

#### 第三步：前端点继续接入

后端动作：

1. 校验 platform token
2. 写入 pending invitation cookie / session
3. 生成 Authentik enrollment URL
4. 返回 `redirectUrl`

#### 第四步：跳转到 Authentik enrollment flow

Enrollment Flow 建议顺序：

1. Invitation Stage
2. Prompt Stage（用户名/姓名/邮箱/密码/重复密码）
3. User Write Stage
4. User Login Stage

#### 第五步：登录成功回到 CrewClaw

回到 `/api/v1/auth/post-login` 或等价的 BFF 路由。

系统动作：

1. 用 Authentik 会话识别当前用户
2. 调 `/internal/users/sync`
3. 检查 pending invitation
4. 绑定 workspace / role
5. 标记 invitation consumed
6. 清理 cookie / session
7. 跳工作台

---

## 9. 首次密码策略怎么做最稳

### 9.1 推荐基线

**在 invitation enrollment flow 里直接设置密码。**

也就是：

- 用户不需要提前知道密码
- 但会在“完成接入”页当场设置密码
- 这一步由 Authentik Flow 完成

### 9.2 为什么这比“进来后再改密”更好

因为你会少掉一整层状态管理：

- 不需要“首次登录成功但尚未改密”的中间态
- 不需要你平台自己写“首次改密强跳页”
- 不需要多一次跳转

### 9.3 如果你以后坚持“下一次登录强制改密”

那就用 Authentik 官方的“Force password reset on next login”方案，不要在平台重写。

你平台只做一件事：

- 在需要时给用户打上 `reset_password=true` 或等价策略标记

---

## 10. 平台与 Authentik 的职责边界

### 10.1 Authentik 负责

- 本地账号目录
- 密码存储与校验
- 登录会话
- enrollment flow
- invitation stage
- user settings / recovery / reset password
- forward auth

### 10.2 CrewClaw 负责

- 用户业务状态（active / disabled）
- invitation 的业务有效性
- workspace / role 绑定
- runtime 资源真相
- quota / usage / 模型治理
- 后台 invitation 管理

### 10.3 不要混淆的点

不要让 Authentik 成为 `workspace / role` 的唯一真相库。

首版最稳的做法是：

- Authentik 可以带一些辅助属性或 group
- 但最终平台业务授权仍以 CrewClaw 为准

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
          - X-authentik-jwt
```

然后平台主域名和 workspace 子域名都挂这个 middleware。

### 11.4 你的应用层要做什么

模块 1 负责读取认证后的身份上下文，最少映射出：

- `subjectId`
- `email`
- `username`
- `name`

再触发用户同步与业务校验。

---

## 12. 推荐的数据模型补充

除了你已有的 `UserRuntimeBinding`，建议补两个对象：

### 12.1 Invitation

```ts
interface Invitation {
  invitationId: string
  inviteTokenHash: string
  targetEmail: string
  workspaceId: string
  role: string
  status: 'pending' | 'consumed' | 'revoked' | 'expired'
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
  role: string
  status: 'active' | 'disabled'
  source: 'invitation' | 'manual' | 'sync'
}
```

---

## 13. 推荐 API 落点

### 13.1 公开接口

```text
GET  /api/v1/auth/options
GET  /api/v1/public/invitations/{token}
POST /api/v1/public/invitations/{token}/start
GET  /api/v1/auth/post-login
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

---

## 14. 在 Cursor 里应该怎么拆任务

这是最关键的一段。

### 14.1 第一组：文档与类型先行

先做：

1. 更新架构文档
2. 更新 MVP 契约
3. 更新 API 规范
4. 增加 TypeScript / Go / Python 类型定义：
   - `Invitation`
   - `WorkspaceMembership`
   - `AuthContext`

### 14.2 第二组：后端真相对象

先落库：

1. `invitations` 表
2. `workspace_memberships` 表（如果还没有清晰实体）
3. `users` 表补 `subject_id / auth_provider / auth_method`

### 14.3 第三组：Auth 入口

做：

1. `/api/v1/auth/options`
2. `/api/v1/auth/me`
3. `/api/v1/auth/post-login`
4. `internal/users/sync`

### 14.4 第四组：invitation 主链路

做：

1. 管理员创建 invitation
2. invitation 预览页接口
3. invitation start 接口
4. pending invitation cookie/session
5. post-login consume 逻辑

### 14.5 第五组：Traefik / Outpost 接入

做：

1. 平台域名挂 forward auth
2. workspace 子域名挂 forward auth
3. `/outpost.goauthentik.io/*` 路由放行到 outpost
4. 验证 header / jwt 是否透传到应用

### 14.6 第六组：联调验收

验收 6 条：

1. 首次管理员能初始化成功
2. 登录页只有本地密码
3. invitation 可创建
4. invitation 链接一次性有效
5. 用户完成接入后能绑定 workspace / role
6. 进入 workspace 子域名会被 Authentik 保护

---

## 15. 你可以直接交给 Cursor 的开发任务清单

下面这段你可以直接复制到 Cursor 作为任务说明。

```md
目标：在不修改上游源码的前提下，把 CrewClaw 接入官方 Authentik。

边界：
- 使用官方 Authentik
- Docker 分容器，同网络
- 首版只开本地账号密码
- 邀请制接入
- 平台保持 workspace/role 业务真相

需要完成：
1. 新增 Invitation 数据模型与迁移
2. 新增 WorkspaceMembership 绑定逻辑
3. 新增 /api/v1/auth/options
4. 新增 /api/v1/public/invitations/{token}
5. 新增 /api/v1/public/invitations/{token}/start
6. 新增 /api/v1/auth/post-login
7. 新增 /api/v1/admin/invitations 系列接口
8. 新增 /internal/invitations 与 /internal/invitations/{id}/consume
9. 保持现有 runtime 接口不变
10. 把 subjectId/authProvider/authMethod 接入用户同步
11. 在 Traefik 上为平台域名和 workspace 子域名挂 Authentik forward auth
12. 接口返回和状态机必须与文档保持一致

推荐链路：
- 平台 invitation token 负责业务入口
- Authentik invitation/enrollment flow 负责身份接入
- 登录完成后由 post-login 入口收口并绑定 workspace/role
```

---

## 16. 最小可用实现顺序

如果你只想最快跑通首版，按这个顺序就行：

### 第 1 步
先把 Authentik 跑起来，并完成管理员初始化。

### 第 2 步
把 Traefik + Outpost 前置鉴权接到平台主域名。

### 第 3 步
实现 `/auth/me` 与 `/internal/users/sync`。

### 第 4 步
实现 invitation 表和管理员创建 invitation。

### 第 5 步
做 invitation preview + start + post-login consume。

### 第 6 步
把 workspace 子域名也挂上 forward auth。

### 第 7 步
联调 runtime / workspace-entry。

---

## 17. 常见误区

### 误区 1：把 Authentik 当成 workspace 权限中心

不对。它是身份系统，不是你平台业务授权的唯一真相。

### 误区 2：对外直接发 Authentik invitation 链接

不推荐。你会失去业务可控性和追踪性。

### 误区 3：平台自己再写一套密码修改 API

不推荐。首版会变复杂，而且会破坏职责边界。

### 误区 4：首版同时上本地密码 + 多个第三方登录

不推荐。会让联调复杂度暴涨。

---

## 18. 最后的正式建议

如果你的目标是：

- 尽快可落地
- 不改上游源码
- 以后还能扩展到 Google / GitHub / 企业 SSO / 微信 / 钉钉 / 飞书

那么首版最稳的方案就是：

1. **官方 Authentik 负责身份、密码、会话**
2. **CrewClaw 负责 invitation 业务真相和 workspace/role 绑定**
3. **Traefik + Outpost 负责统一前置鉴权**
4. **首版只做本地密码**
5. **把 invitation 链接定义成一次性免密码接入入口**
6. **在 enrollment flow 里直接设置用户密码**

这套方案是最符合你当前需求、实施成本最低、后续扩展阻力最小的组合。



v 0.5
reno 
2026-03-23 10:04