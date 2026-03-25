
# ClawLoops 平台设计文档（Authentik 官方首版接入版，运行时冻结修订）

基于既有 MVP 架构，按“**首版直接用官方 Authentik、不改源码、Docker 分容器、共享网络固定、当前仅本地账号密码、后续再接外部身份源**”重构后的统一版本。

| 文档定位 | 系统架构与 MVP 落地设计 |
| --- | --- |
| 适用范围 | 单机部署、多用户访问、容器级隔离、统一模型网关、官方 Authentik 身份接入 |
| 本版重点 | 冻结管理员初始化口径、邀请制接入规则、首登密码流程、无真实邮箱用户兼容策略、Traefik + Outpost 前置鉴权，以及 runtime V1 contract |
| 关联文档 | 《MVP 开发基线总契约》《MVP 统一总接口》《RuntimeManager 开发契约》《Authentik 实施文档》 |
| 当前版本 | v0.11-no-real-email-friendly |

---

## 0. 本次修订摘要

本版明确采用如下落地结论：

- 首版身份系统直接采用 **官方 Authentik**
- **不修改 OpenClaw / 上游源码**；只改 ClawLoops 控制面、部署层、网关层与业务绑定逻辑
- 平台运行链路统一使用共享网络 **`clawloops_shared`**
- 首版只开放 **本地账号密码**
- 无真实邮箱用户允许使用“用户名 + 代理邮箱槽位”接入，且前端体验必须优先围绕用户名展开
- 所有 workspace 子域名继续统一经 **Traefik + Authentik Proxy Outpost + Forward Auth** 做前置鉴权
- 平台主域名下的 invitation 承接入口保持公开，不先走默认登录流
- 普通用户接入采用 **邀请制**
- 首版 invitation 采用 **双层模型 + 延迟创建**
- 首版 runtime 采用 **固定镜像 + 固定命令 + 固定端口 + 固定 alias + `compat` 必填** 的冻结契约
- RuntimeManager 是**同步执行器**；Orchestrator 才是对用户侧暴露任务的**异步编排器**

---

## 1. 目标与范围

目标是在一台服务器上部署公司内部 ClawLoops 平台，使平台满足以下能力：

1. 官方 Authentik 统一身份接入
2. 平台管理员可完成首版初始化与用户邀请
3. 普通用户通过邀请链接接入系统，并绑定指定 workspace / role
4. 用户进入平台后，继续按既有设计使用个人 runtime 工作区
5. workspace 入口统一由 Traefik 暴露，由 Authentik 做前置鉴权
6. 平台仍坚持“管理员提供服务、用户只使用”的团队平台模式

MVP 仍然不追求：

- 复杂多租户隔离 UI
- 用户自助 provider key 管理
- 用户自助模型绑定
- 复杂费用管理
- 企业级目录同步首版接入
- 微信 / 钉钉 / 飞书等第三方登录首版上线

| 范围 | MVP 结论 |
| --- | --- |
| 身份系统 | 采用官方 Authentik |
| 首次管理员初始化 | 使用官方初始化流程，首个 bootstrap 管理员按默认 `akadmin` 处理 |
| 登录方式 | 仅本地账号密码 |
| 邀请机制 | 平台 invitation + Authentik enrollment flow 组合，且首版延迟创建身份侧 invitation |
| 工作区访问 | Traefik + Authentik Forward Auth |
| 用户密码 | 全部交给 Authentik 管理 |
| runtime V1 | 固定镜像、固定端口、固定网络、固定 alias、`compat` 必填 |
| 外部身份源 | 后续扩展，不在首版启用 |

---

## 2. 为什么本版必须冻结这些规则

原因不是因为原方案错误，而是因为原文档虽已引入 Authentik，但还没有把以下关键事实冻结成可开发基线：

1. **首次启动管理员引导** 没有统一写死为 `akadmin`
2. **邀请制接入** 缺少明确生命周期：平台 token 与 Authentik `itoken` 的边界、何时创建身份侧 invitation、何时消费业务 invitation
3. **本地账号密码首版、外部身份源后加** 没有写成不能漂移的硬规则
4. **用户密码修改与强制改密由 Authentik Flow 管理** 没有落成接口级禁区
5. **post-login 收口** 缺少幂等与事务定义
6. **Traefik / Outpost / Forward Auth / workspace 子域名** 的关系需要从概念接入提升为硬安全规则
7. **runtime 基线** 仍存在 `imageRef`、端口、网络名、同步 / 异步边界、挂载与删除语义冲突

因此本版文档不再只把 Authentik 看成“登录网关”，也不再把 RuntimeManager 看成“模糊执行器”，而是一起冻结为：

- Authentik：身份、密码、会话、enrollment flow、前置鉴权
- ClawLoops：业务真相、workspace / role、runtime 真相
- Orchestrator：对外异步编排
- RuntimeManager：对内同步执行 + 最小事实观测

---

## 3. 整体架构（修订版）

### 3.1 组件分层

- **入口层**：Traefik
- **身份层**：Authentik Server、Authentik Worker、Authentik PostgreSQL、Authentik Redis、Authentik Proxy Outpost
- **平台层**：ClawLoops 控制面 API + 极简 Web UI
- **编排层**：Runtime Orchestrator + Runtime Manager
- **运行时层**：Per-user OpenClaw runtime
- **模型层**：LiteLLM + PostgreSQL + vLLM / Ollama

| 层级 | 核心组件 | 职责 |
| --- | --- | --- |
| 入口层 | Traefik | 统一暴露平台域名与 workspace 子域名；对受保护路由挂载 Authentik Forward Auth |
| 身份层 | Authentik Core + Proxy Outpost | 本地用户、会话、密码、邀请制 enrollment、前置鉴权、未来外部身份源接入 |
| 平台层 | ClawLoops 控制面 | 用户同步、邀请绑定、workspace/role 真相、runtime 真相、后台治理 |
| 编排层 | Runtime Orchestrator + Runtime Manager | runtime 启停删、配置挂载、状态回写；其中 Orchestrator 异步，RM 同步 |
| 运行时层 | Per-user OpenClaw runtime | 用户个人工作区与本地运行环境 |
| 模型层 | LiteLLM + PostgreSQL + vLLM/Ollama | 平台统一模型出口与用量归集 |

### 3.2 部署约束

- 所有服务使用**独立容器**部署
- 平台运行链路统一依赖共享网络 **`clawloops_shared`**
- `Traefik / ClawLoops API / RuntimeManager / LiteLLM / per-user runtime` 必须加入 `clawloops_shared`
- Authentik 不要求所有组件都进入同一共享网络；最低要求是 `authentik-proxy-outpost` 与 Traefik / 受保护应用链路可达
- ClawLoops 控制面保存业务真相；Authentik 保存身份真相
- internal API 必须通过服务间鉴权（mTLS 或 internal token）并禁止公网访问

---

## 4. 与官方 Authentik 的联合方式

### 4.1 为什么首版用官方 Authentik

首版需求本质分成两层：

1. **身份能力**：用户是谁、如何登录、密码怎么管理、enrollment 怎么完成、会话怎么生效
2. **业务能力**：用户应该进哪个 workspace、拿什么 role、是不是已经被禁用、能不能打开 runtime

Authentik 非常适合承接第一层；ClawLoops 承接第二层。这样职责边界清晰、开发量最小、后续扩展最顺。

### 4.2 推荐联合策略

首版采用以下组合：

- 平台控制面与 workspace 子域名统一由 Traefik 暴露
- Traefik 通过 Authentik Proxy Provider 的 **Forward Auth** 中间件保护受保护业务路由
- `/invite/{token}`、invitation preview、invitation start 这些 invitation 承接入口保持公开
- Authentik 当前只启用 **本地用户名优先登录（兼容邮箱输入）+ 密码** 登录
- 管理员在 ClawLoops 后台创建 invitation，ClawLoops 保存业务邀请码对象
- ClawLoops 在用户调用 `/start` 时延迟创建或换取 Authentik enrollment invitation
- 用户点击平台 invitation 链接后，被重定向进入 Authentik enrollment flow
- Authentik 负责创建用户、设置密码、建立登录会话
- 登录成功后，ClawLoops 在 `post-login` 根据 invitation 记录完成 `workspace / role` 绑定
- runtime 启停删由 Orchestrator 发起异步任务，再调用 RM 的同步 internal API 完成实际执行

---

## 5. 首次启动：管理员账号如何处理

### 5.1 现实边界

如果坚持“**不改官方源码**”，那么官方 Authentik 首次初始化默认管理员用户是 **`akadmin`**。因此：

- “第一次登录的是管理员账号，并进入后设置自己的密码”——官方可以直接支持
- “这个账号的用户名必须字面叫 `admin`”——官方首个初始化账号默认不是 `admin`”

### 5.2 首版冻结做法

统一解释为：**第一次登录的是管理员角色账号，而不是用户名必须叫 `admin`**。

首版落地流程：

1. 启动 Authentik Core
2. 管理员访问 Authentik 官方 `initial-setup` 流程
3. 在浏览器中设置 bootstrap 管理员密码
4. 初始管理员进入后台后完成：
   - 基础品牌配置
   - 本地认证流确认
   - Proxy Outpost / Provider 创建
   - ClawLoops 平台应用接入
   - invitation enrollment flow 导入或配置
5. 如有需要，可在后续额外创建日常使用的本地管理员账号 `admin`；但这不改变首个官方初始化管理员口径

---

## 6. 本地账号密码模式（首版硬规则）

首版只开本地账号密码，不启用任何外部身份源。

### 6.1 规则

- 登录页仅展示用户名优先登录 + 密码
- 不显示 Google / GitHub / 企业 SSO / 微信 / 钉钉 / 飞书按钮
- 不配置外部 Source
- 平台首版用户全部为 Authentik 本地用户

### 6.2 后续如何扩展

后续接入外部身份源时，不需要推翻首版方案，只需要：

- 在 Authentik 中新增对应 Source
- 把 Source 挂入 Identification Stage
- 在用户同步时扩展 `auth.method` 与 `subjectId` 映射
- 保持 ClawLoops 的业务真相不变

---

## 7. 邀请制接入设计（核心改动）

### 7.1 业务目标

管理员希望做到：

1. 创建一个 invitation
2. invitation 绑定一个目标 `workspace / role`
3. invitation 链接带一次性平台 token
4. 用户点击后进入“完成接入”页
5. 用户首次不需要提前知道密码
6. 用户完成接入后进入平台
7. 之后再按普通账号密码方式登录

### 7.2 推荐实现方式

采用 **双层 invitation**：

#### 第 1 层：ClawLoops 业务邀请

ClawLoops 自己保存业务真相：

- `invitationId`
- `inviteTokenHash`
- `targetEmail`
- `loginUsername`
- `workspaceId`
- `role`
- `status`
- `expiresAt`
- `consumedByUserId`
- `authentikInvitationRef`

#### 第 2 层：Authentik enrollment invitation

Authentik 负责执行身份侧 enrollment：

- 执行 Invitation Stage / Enrollment Flow
- 在 flow 中完成用户创建 / 密码设置 / 登录会话建立
- 返回 `itoken` 形式的 enrollment URL

### 7.3 为什么要双层 invitation

因为：

- `workspace / role` 是**业务真相**，应该在 ClawLoops
- 用户密码 / 用户创建 / 会话是**身份真相**，应该在 Authentik
- 这样未来你切换 invitation UI、增加审批、增加多 workspace 分发时，不需要重写身份系统

### 7.4 首版推荐时序（延迟创建）

1. 管理员在 ClawLoops 创建 invitation
2. ClawLoops 保存业务 invitation + platform token
3. 用户打开 `/invite/{token}`
4. 用户触发 `/public/invitations/{token}/start`
5. ClawLoops 写入 pending invitation session
6. ClawLoops 延迟创建 / 换取 Authentik enrollment URL
7. 用户进入 enrollment flow
8. Authentik 完成用户创建、密码设置与自动登录
9. 浏览器回到 ClawLoops `post-login`
10. ClawLoops 完成 user sync、身份邮箱槽位强校验、membership 绑定与 invitation consume
11. 若 `appRole=admin` 则进入管理后台；否则进入工作台

### 7.5 token 语义冻结

- **ClawLoops token 是业务入口真相**
- **Authentik `itoken` 是身份执行入口**
- 对外永远发 `https://clawloops.example.com/invite/{platform_token}`
- 不直接把 Authentik 原始 `itoken` 当成业务链接对外发送
- `start` 生成 `redirectUrl` 时必须显式引用平台配置的 `AUTHENTIK_ENROLLMENT_FLOW_SLUG`
- 首版禁止在该配置缺失时静默回退到 `default-authentication-flow`

### 7.6 “首次免密码链接进入”如何解释最合理

冻结解释如下：

- 用户第一次进入系统时，不需要预先知道密码
- 只需要打开 invitation 一次性链接
- 该 invitation 链接就是“首登免密码入口”
- 在 enrollment flow 中由用户自己设置密码
- 完成后自动登录

### 7.7 身份邮箱槽位校验规则

- 首版统一为 **强身份邮箱槽位校验**
- 当前认证身份的邮箱槽位必须匹配 invitation `targetEmail`
- 校验点固定放在 **post-login 阶段**
- 不支持“允许覆盖”“管理员放行”等例外逻辑
- `targetEmail` 可为真实邮箱或代理邮箱；若为代理邮箱，用户侧体验仍应优先展示 `loginUsername`

### 7.8 pending session 生命周期

- `start` 后会写入 pending invitation session
- 该 session 必须具备 TTL（建议 10–30 分钟）
- 只绑定当前浏览器会话
- `start` 必须幂等：重复调用只复用一个有效 pending invitation session，不应生成多个并发会话
- `start` 若发现 `AUTHENTIK_ENROLLMENT_FLOW_SLUG` 缺失、错误或对应 flow 不存在，必须直接返回配置错误

### 7.9 撤销、过期、消费的联动规则

- 平台 invitation 状态是最终业务真相
- `revoked / consumed / expired` 均需先由平台校验
- 即使身份侧 token 尚未过期，平台状态一旦失效也必须阻断 `start` 与 `post-login`
- 这也是首版推荐**延迟创建 Authentik invitation**的直接原因

---

## 8. 密码策略

### 8.1 管理原则

密码相关逻辑全部交给 Authentik Flow 与本地用户体系管理，平台不自建第二套密码表。

### 8.2 管理员密码

- 首次由官方初始化流程设置
- 后续通过 Authentik 自身用户设置或管理员操作修改

### 8.3 普通用户密码

Enrollment Flow 建议顺序：

1. Invitation Stage
2. Prompt Stage（收集 username / name / email / password / password_repeat）
3. User Write Stage
4. User Login Stage

### 8.4 平台级禁区

ClawLoops 明确禁止：

- 保存用户密码
- 生成正式临时密码
- 提供密码落库接口
- 实现独立改密 API

---

## 9. 新增核心对象：Invitation

除了原有 `UserRuntimeBinding`，本版新增 `Invitation` 作为必须冻结的业务对象。

| 字段 | 说明 |
| --- | --- |
| `invitationId` | 平台内部邀请唯一标识 |
| `inviteTokenHash` | 平台一次性邀请码哈希 |
| `targetEmail` | 目标身份邮箱槽位；首版必填，可为真实邮箱或代理邮箱 |
| `loginUsername` | 推荐登录用户名；无真实邮箱用户应提供 |
| `workspaceId` | 目标 workspace |
| `role` | 目标角色，例如 `admin / user / workspace_member` |
| `status` | `pending / consumed / revoked` |
| `expiresAt` | 过期时间 |
| `consumedAt` | 消费时间 |
| `consumedByUserId` | 最终接入的用户 |
| `authentikInvitationRef` | Authentik 侧 invitation 引用，首版延迟创建时允许为空 |
| `lastError` | 最近一次生成或消费失败原因 |

---

## 10. UserRuntimeBinding 冻结结构与 runtime V1 基线

`UserRuntimeBinding` 继续冻结如下：

| 字段 | 说明 |
| --- | --- |
| `runtimeId` | 平台内部 runtime 唯一标识（平台范围全局唯一） |
| `volumeId` | 平台逻辑卷标识；不等同于宿主机路径 |
| `imageRef` | 平台记录的 runtime 实际生效镜像；V1 固定为 digest，不允许调用方覆盖 |
| `desiredState` | `running / stopped / deleted` |
| `observedState` | `creating / running / stopped / error / deleted` |
| `browserUrl` | 浏览器访问入口 |
| `internalEndpoint` | 平台内部访问地址，V1 固定为 `http://rt-<runtimeId>:18789` |
| `retentionPolicy` | `preserve_workspace / wipe_workspace` |
| `lastError` | 最近一次失败信息 |

### 10.1 访问前置条件

用户只有在下列条件同时满足时，才能把 `browserUrl` 当作可用入口：

1. Authentik 会话有效
2. ClawLoops 用户状态为 `active`
3. 已完成合法 workspace 绑定
4. runtime 已达到最终可访问状态 `ready=true`

### 10.2 runtime V1 冻结实施规则

- V1 镜像固定为  
  `ghcr.io/openclaw/openclaw@sha256:a5a4c83b773aca85a8ba99cf155f09afa33946c0aa5cc6a9ccb6162738b5da02`
- 启动命令固定为  
  `node dist/index.js gateway --bind lan --port 18789`
- 共享网络固定为 `clawloops_shared`
- `networkAlias` 固定为 `rt-<runtimeId>`
- 主服务端口固定为 `18789`
- `18790` 仅兼容保留，不作为 readiness 条件
- `compat.openclawConfigDir / compat.openclawWorkspaceDir` 必填
- `configMount` 是可选增强挂载，不替代 `compat`
- `configMount.configFilePath -> /home/node/.openclaw/openclaw.json:ro`
- `configMount.secretFilePath -> /run/clawloops/secrets/gateway.token:ro`
- `routeHost` 在 RM 中只作 label 追踪，不参与容器启动逻辑，也不属于关键 drift
- RM 只检测 drift 并返回 `409 RUNTIME_CONTRACT_DRIFT`，不自己做 stop+delete+recreate

### 10.3 状态语义冻结

- `task.status` = 操作生命周期
- `observedState` = 资源状态
- `ready` = 最终可访问状态
- `browserUrl` 跳转只看 `ready`

### 10.4 登录后首页规则

- `appRole=admin` 的用户登录后默认进入 `/admin`
- 平台管理员首页不依赖 workspace membership
- 非管理员用户登录后默认进入 `/app`
- `workspace-entry` 只负责普通用户主动进入工作区时的最终跳转与短时等待，不承担默认首页决策

### 10.5 管理后台首页最小能力

`/admin` 不是空白壳路由，也不是前端自行拼装的跳板页；首版冻结为真正可用的管理后台首页。

首页最小目标：

- 让管理员在 1 屏内看到平台当前治理重点
- 让管理员能直接进入高频任务，而不是先被迫进入任意二级列表页
- 即使管理员没有任何 workspace membership，也能稳定落到可操作页面

首页最小信息块：

- 用户总览：`totalUsers / activeUsers / disabledUsers`
- invitation 总览：`pendingInvitations / expiringInvitations24h`
- runtime 总览：`runningRuntimes / runtimeErrors`
- 待处理 invitation 列表：用于快速进入邀请治理
- runtime 异常列表：用于快速进入用户详情排障

实现边界：

- 管理后台首页数据由后端聚合接口提供，避免前端首屏并发拼装多组查询
- 首版聚合接口固定为 `GET /api/v1/admin/home`
- 首页只做摘要与跳转，不承担复杂编辑表单
- 写操作仍在 `/admin/users`、`/admin/invitations` 等专页完成

---

## 11. 路由、前置鉴权与 workspace 子域名

### 11.1 首版选择

首版继续采用：

- Traefik 统一路由
- Authentik Proxy Outpost
- Forward Auth
- workspace 子域名前置鉴权

### 11.2 路由模型

| 路由 | 示例 | 鉴权方式 |
| --- | --- | --- |
| 平台受保护业务入口 | `https://clawloops.example.com/app`、`https://clawloops.example.com/admin` | Traefik + Authentik |
| 平台公开邀请入口 | `https://clawloops.example.com/invite/{token}`、公开 invitation API | 平台 token 校验，不走 Forward Auth |
| 工作区入口 | `https://u-001.clawloops.example.com` | Traefik + Authentik Forward Auth |
| Outpost 回调路径 | `/outpost.goauthentik.io/*` | 由 Outpost 直接处理 |

### 11.3 安全模型

- `browserUrl` 不是匿名公开地址
- 知道 URL 不等于可以访问
- 所有 workspace 子域名必须统一经过 Traefik + Authentik Forward Auth
- invitation 承接入口必须保持公开，否则会被错误送入普通登录流
- invitation 承接入口虽然公开，但仍必须通过平台 token 和后端状态校验
- ClawLoops 仍需在业务层检查用户状态、workspace 归属和 runtime 状态

### 11.4 管理员一次性配置要求

管理员在部署期至少手动完成一次：

1. 在 Authentik 中创建或导入 `ClawLoops Enrollment Flow`
2. 固定其 `slug`
3. 在 `clawloops-api` 配置 `AUTHENTIK_ENROLLMENT_FLOW_SLUG=<该 slug>`
4. 重启 API 服务
5. 用测试 invitation 验证 `start` 返回的 `redirectUrl` 命中该 flow

这属于平台一次性配置，不属于最终用户操作。

---

## 12. 模块职责修订

| 模块 | 修订后的职责 |
| --- | --- |
| 模块 1：身份与访问接入 | 接收 Authentik 会话上下文；同步 / 创建 ClawLoops 用户；识别 admin / disabled；处理 post-login invitation 收口；执行身份邮箱槽位强校验 |
| 模块 2：租户与用户资源控制 | 维护 User / Invitation / WorkspaceMembership / UserRuntimeBinding 真相；保证 invitation consume 与 membership binding 的原子性或补偿逻辑 |
| 模块 3：Runtime 编排 | 对用户侧暴露异步任务；决定何时调用 RM；在收到 drift 后决定是否 stop+delete+recreate |
| 模块 4：模型接入 | 不变 |
| 模块 5：管理后台 | 新增 invitation 创建、查看、撤销；仍负责用户治理，并作为 `admin` 登录后的默认首页；提供首页摘要视图与高频治理入口 |
| 模块 6：用户工作台 | 新增“邀请完成后首次进入”承接逻辑；仅负责非管理员用户的工作区承接，工作区跳转只依赖 `workspace-entry` |
| RuntimeManager | 同步执行容器动作、目录初始化、最小事实观测；不维护外层任务状态机 |

---

## 13. 关键调用故事（修订版）

### 13.1 首次安装与管理员初始化

1. 运维启动 Authentik Core 与依赖容器
2. 管理员打开官方初始化入口
3. 设置 bootstrap 管理员 `akadmin` 的密码
4. 登录 Authentik 后创建：
   - ClawLoops 平台应用
   - Proxy Provider / Outpost
   - Enrollment Flow
   - 本地登录规则
5. ClawLoops 控制面开始接收 Authentik 会话并同步管理员用户

### 13.2 管理员创建 invitation

1. 管理员在 ClawLoops 后台选择目标 `workspace / role`，填写 `targetEmail`，并尽量填写 `loginUsername`
2. 模块 5 调模块 2 创建 `Invitation`
3. 模块 2 生成一次性平台 token，并保存 invitation
4. 系统返回可发送的 invite URL
5. 不在此时创建 Authentik invitation

### 13.3 用户完成接入

1. 用户打开 invitation URL
2. ClawLoops 校验平台 token，确认未过期、未撤销、未消费
3. 用户点击继续接入，ClawLoops 把当前 invitation 上下文写入短期 session / cookie
4. ClawLoops 延迟创建或换取 Authentik enrollment URL，并重定向
5. Authentik 完成 invitation 校验、用户写入、密码写入与自动登录
6. 浏览器回到 ClawLoops `post-login` 入口
7. 模块 1 完成 `/internal/users/sync`
8. 模块 1 执行身份邮箱槽位强校验
9. 模块 2 幂等完成 `workspace / role` 绑定，并把 invitation 标记为 `consumed`
10. 若为 `admin` 则进入管理后台；若为普通用户则进入工作台并可启动 runtime

### 13.4 用户启动 runtime

1. 用户调用 `POST /api/v1/users/me/runtime/start`
2. 模块 3 创建异步任务并返回 `taskId`
3. 模块 3 计算 effectiveRetentionPolicy、host path 绑定、固定镜像与固定命令
4. 模块 3 调 RM 的 `POST /internal/runtime-manager/containers/ensure-running`
5. RM 同步执行目录初始化、挂载、网络接入与启动探测
6. RM 立即返回当前 `observedState`
7. 模块 3 把结果写回 runtime task 与 binding
8. 前端通过 `/api/v1/runtime/tasks/{taskId}` + `/api/v1/users/me/runtime/status` 观察最终状态

### 13.5 post-login 幂等要求

- 用户刷新页面、浏览器重试、网络抖动时，不得产生重复 membership
- 同一 `invitationId + userId` 只能成功消费一次
- 重复调用返回已消费 / 已绑定的最终结果

---

## 14. 开发与部署方式

### 14.1 部署建议

推荐拆成以下容器：

- `traefik`
- `clawloops-web`
- `clawloops-api`
- `runtime-manager`
- `authentik-server`
- `authentik-worker`
- `authentik-postgresql`
- `authentik-redis`
- `authentik-proxy-outpost`
- `litellm`
- `litellm-postgresql`

### 14.2 共享网络建议

推荐统一定义：

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

- 最低要求是 `authentik-proxy-outpost` 与受保护应用链路可达
- 不强制 `authentik-postgresql / authentik-redis` 暴露到同一共享网络

### 14.3 不改源码的真正含义

这里的“不改源码”指：

- 不 fork / patch Authentik
- 不改 OpenClaw 上游源码
- 只在你自己的平台代码中增加：
  - invitation 业务对象
  - Authentik 回调承接
  - 用户同步
  - workspace / role 绑定
  - Orchestrator 任务编排
  - RuntimeManager 执行器
  - Traefik / Outpost 部署配置

---

## 15. 首版明确不做的事

- 不把 workspace / role 业务真相塞进 Authentik 作为唯一真相
- 不在首版开放 Google / GitHub / 企业 SSO / 微信 / 钉钉 / 飞书
- 不做多因素认证强制
- 不做多 runtime / 多 workspace 自助切换
- 不做复杂邀请审批流
- 不做平台自研密码系统
- 不把“先 magic link、后强制改密”作为首版流程
- 不让 RuntimeManager 自己做复杂补偿编排或自动重建

---

## 16. 后续路线图

### 16.1 第二阶段

- 接入 Google / GitHub 登录
- 在 Authentik 中新增 Source，并挂入 Identification Stage
- 扩展 `auth.method` 与 `subjectId` 兼容逻辑

### 16.2 第三阶段

- 企业 SSO（OIDC / SAML）
- SCIM / 目录同步
- 更复杂的用户生命周期管理

### 16.3 中国企业身份源阶段

- 微信登录
- 钉钉登录
- 飞书登录

此时仍不需要推翻首版架构，只是把 Authentik 从“本地身份中心”扩展成“统一身份入口”。

---

## 17. 最终设计结论

首版最稳妥的方案不是“把所有认证逻辑都写进 ClawLoops”，也不是“让 RuntimeManager 自己补偿一切”，而是：

- **Authentik 负责身份与密码**
- **ClawLoops 负责业务绑定与资源治理**
- **Traefik + Outpost 负责统一前置鉴权**
- **Invitation 用双层模型把业务 token 与身份 enrollment 解耦，并在首版采用延迟创建**
- **首版只开本地账号密码，后续再逐步加外部身份源**
- **`admin` 登录后默认进入 `/admin`；普通用户登录后默认进入 `/app`；`workspace-entry` 只负责最终跳转与短时等待，且前端只在 `ready=true` 时跳转**
- **Orchestrator 负责对外异步编排，RuntimeManager 负责对内同步执行**
- **runtime V1 统一固定为 `clawloops_shared + 18789 + rt-<runtimeId> + compat 必填`**

这样既满足当前诉求，也不会把首版做成难以演进的临时拼装方案。

---

v0.11-无真实邮箱用户友好修订
reno  
2026-03-25 16:05