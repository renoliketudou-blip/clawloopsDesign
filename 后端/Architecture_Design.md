# ClawLoops 平台设计文档（业务内轻量认证版，运行时冻结修订）

基于既有 MVP 架构，按“**认证真相收回业务后端、邀请制首次设密、平台自管 session、工作区入口继续受保护、runtime V1 contract 保持冻结**”重构后的统一版本。

| 文档定位 | 系统架构与 MVP 落地设计 |
| --- | --- |
| 适用范围 | 单机部署、多用户访问、容器级隔离、统一模型网关、业务内轻量认证 |
| 本版重点 | 冻结登录与邀请接入链路、统一 session 校验边界、删除外部 IAM 依赖、明确不做改密与找回密码、保持 runtime V1 contract 不漂移 |
| 关联文档 | 《MVP 开发基线总契约》《MVP 统一总接口》《RuntimeManager 开发契约》《轻量认证实施文档》 |
| 当前版本 | v0.12-lightweight-auth |

---

## 0. 本次修订摘要

本版明确采用如下落地结论：

- 首版身份系统不再依赖第三方 IAM，统一采用 **ClawLoops 业务内轻量认证**
- 平台后端自己维护 `User / PasswordHash / Session / Invitation`
- 首版只开放 **本地账号密码**
- 普通用户继续采用 **邀请制**
- invitation 改为 **单层模型**
- 用户在平台站内完成首次设密与 invitation 接受
- 登录成功后直接进入 `/admin` 或 `/app`，不再经过 `post-login` 中转页
- 所有 workspace 子域名继续受保护，但保护方式改为 **平台 session + 自有鉴权中间层**
- 首版明确 **不做改密、不做找回密码**
- runtime V1 仍采用 **固定镜像 + 固定命令 + 固定端口 + 固定 alias + `compat` 必填** 的冻结契约
- RuntimeManager 是**同步执行器**；Orchestrator 才是对用户侧暴露任务的**异步编排器**

---

## 1. 目标与范围

目标是在一台服务器上部署公司内部 ClawLoops 平台，使平台满足以下能力：

1. 平台自带用户名优先登录能力
2. 平台管理员可完成首版初始化与用户邀请
3. 普通用户通过 invitation 链接进入系统，并在站内完成首次设密
4. invitation 被接受后，平台完成指定 `workspace / role` 绑定
5. 用户进入平台后，继续按既有设计使用个人 runtime 工作区
6. workspace 入口统一由 Traefik 暴露，但访问权限由平台 session 校验决定
7. 平台仍坚持“管理员提供服务、用户只使用”的团队平台模式

MVP 仍然不追求：

- 第三方登录
- 企业目录同步
- 改密流程
- 找回密码流程
- 复杂多租户隔离 UI
- 用户自助 provider key 管理
- 用户自助模型绑定
- 复杂费用管理

| 范围 | MVP 结论 |
| --- | --- |
| 身份系统 | ClawLoops 业务内轻量认证 |
| 管理员初始化 | 平台启动后由种子管理员账号或初始化脚本完成 |
| 登录方式 | 仅本地账号密码 |
| 邀请机制 | 平台单层 invitation，站内完成首次设密与消费 |
| 工作区访问 | Traefik + 平台 session 鉴权中间层 |
| 用户密码 | 平台认证模块保存密码哈希并验证 |
| 密码扩展 | 首版不做改密与找回密码 |
| runtime V1 | 固定镜像、固定端口、固定网络、固定 alias、`compat` 必填 |
| 外部身份源 | 后续扩展，不在首版启用 |

---

## 2. 为什么本版要切到业务内轻量认证

之前方案把外部 IAM 同时用作：

- 身份目录
- 密码管理
- enrollment flow 编排
- 前置鉴权入口

这会导致首版出现大量额外概念：

- 双层 invitation
- `itoken`
- enrollment flow slug
- `post-login` 收口
- 外部头透传与组映射

而当前 MVP 的真实需求并没有这么复杂，真正需要冻结的能力只有：

1. 用户能登录
2. 用户能通过邀请首次进入系统
3. 平台能知道该用户属于哪个 workspace
4. 管理员与普通用户能稳定分流
5. workspace 入口对未登录用户不可访问

因此本版把认证缩回业务边界，把复杂度留在自己真正需要控制的地方。

---

## 3. 核心设计原则

### 3.1 身份真相归属

- `User`、`PasswordHash`、`Session`、`Invitation` 都由 ClawLoops 保存
- 平台自己校验用户名密码
- 平台自己设置和撤销登录 session
- 平台自己完成 invitation 消费与 membership 绑定

### 3.2 单层 invitation

- 对外只发 `https://clawloops.example.com/invite/{platform_token}`
- 不再引入第二层身份侧 token
- invitation 自身就是首登入口
- 用户在站内完成设密后立即建立 session

### 3.3 业务和 runtime 继续解耦

- 认证成功只代表用户有权进入平台控制面
- runtime 启停删仍只由业务状态决定
- RuntimeManager 不承担 session 与 invitation 逻辑

### 3.4 密码边界

- 平台只存密码哈希，不存密码明文
- 首版只做登录验证和首次设密
- 首版不做改密、找回密码、邮箱验证

### 3.5 工作区入口安全边界

- `browserUrl` 只是入口地址
- workspace 子域名受平台 session 保护
- 知道 URL 不等于可访问
- 只有 `ready=true` 时前端才允许跳转

---

## 4. 核心组件

首版组件保持尽量简单：

- `traefik`
- `clawloops-web`
- `clawloops-api`
- `runtime-manager`
- `litellm`
- `litellm-postgresql`
- per-user runtime

新增或明确的认证相关职责：

| 组件 | 职责 |
| --- | --- |
| `clawloops-api` | 登录、session、invitation、用户同步、membership 绑定、权限判断 |
| `clawloops-web` | 登录页、邀请接入页、用户工作台、管理后台 |
| `traefik` | 主域与 workspace 子域统一入口 |
| `workspace auth middleware` | 基于平台 session 校验 workspace 子域访问权限 |

这里的 `workspace auth middleware` 可以是：

- Traefik `ForwardAuth` 指向平台自有鉴权接口
- 或部署在 workspace 前的轻量代理

无论选哪种实现，语义都必须一致：

- 只认平台 session
- 不依赖第三方身份头
- 只把“该用户是否允许访问该 workspace”作为判断依据

---

## 5. 认证与会话模型

### 5.1 用户模型

平台用户最少应包含：

- `userId`
- `subjectId`
- `username`
- `passwordHash`
- `role`
- `status`
- `createdAt`
- `lastLoginAt`

字段语义冻结：

- `subjectId` 是平台认证主体标识，例如 `clawloops:u_001`
- `username` 是首版主登录标识
- `role` 只认 `admin / user`
- `status` 只认 `active / disabled`

### 5.2 session 模型

平台 session 最少应包含：

- `sessionId`
- `userId`
- `issuedAt`
- `expiresAt`
- `revokedAt`

首版规则：

- 浏览器使用 HttpOnly session cookie
- session 过期由服务端校验
- logout 通过服务端撤销 session
- 不要求前端解析 token

### 5.3 管理员初始化

首版不再引入外部系统初始化管理员的概念。

推荐做法：

1. 首次部署时通过环境变量或初始化脚本写入一个 `admin` 用户
2. 初始密码只用于平台管理员第一次登录
3. 后续管理员继续通过平台登录页登录

冻结要求：

- 文档统一使用“种子管理员账号”口径
- 不再混用 `akadmin` 等外部系统初始化概念

---

## 6. invitation 与首次接入流程

### 6.1 业务目标

管理员希望做到：

1. 创建一个 invitation
2. invitation 绑定目标 `workspace / role`
3. invitation 链接带一次性平台 token
4. 用户打开链接后可直接完成首次接入
5. 用户首次不需要预先知道密码
6. 接入成功后进入平台
7. 之后按普通用户名密码方式登录

### 6.2 推荐实现方式

采用 **单层 invitation**：

- 平台保存 `invitationId`
- 平台保存 `inviteTokenHash`
- 平台保存 `targetEmail`
- 平台保存 `loginUsername`
- 平台保存 `workspaceId`
- 平台保存 `role`
- 平台保存 `status`
- 平台保存 `expiresAt`
- 平台保存 `consumedByUserId`

### 6.3 首版推荐时序

1. 管理员在 ClawLoops 创建 invitation
2. ClawLoops 保存业务 invitation + platform token
3. 用户打开 `/invite/{token}`
4. 页面读取 invitation 预览信息
5. 用户在站内填写初始密码并确认接入
6. API 校验 invitation 状态、用户名口径与密码规则
7. 平台创建或激活用户账号并写入密码哈希
8. 平台完成 `workspace / role` 绑定
9. 平台把 invitation 标记为 `consumed`
10. 平台建立 session
11. 浏览器直接进入 `/app` 或 `/admin`

### 6.4 为什么不再需要 `post-login`

因为首版登录收口动作已经内聚在 invitation 接受接口里：

- 不再需要等待外部 IAM 完成登录
- 不再需要 pending invitation session
- 不再需要浏览器从外部身份页跳回平台收口

### 6.5 用户名与邮箱口径

首版仍保留无真实邮箱用户兼容策略：

- `loginUsername` 是用户侧主识别信息
- `targetEmail` 可以为空，或仅作管理员侧记录
- 前端默认优先展示 `loginUsername`
- 用户登录时优先输入用户名

---

## 7. 登录流程

### 7.1 普通登录

1. 用户打开 `/login`
2. 提交 `username + password`
3. 平台校验用户状态与密码哈希
4. 平台建立 session
5. `admin` 进入 `/admin`
6. 普通用户进入 `/app`

### 7.2 invitation 首次接入

1. 用户打开 `/invite/{token}`
2. 平台展示 invitation 预览
3. 用户提交初始密码
4. 平台完成 invitation 消费、membership 绑定与 session 建立
5. 进入 `/app`

### 7.3 logout

1. 前端调用 `POST /api/v1/auth/logout`
2. 平台撤销当前 session
3. 浏览器回到 `/login`

---

## 8. 授权与路由规则

### 8.1 控制面权限

- `/admin/*` 只允许 `admin`
- `/app`、`/workspace-entry` 只允许已登录且 `allowed=true`
- disabled 用户不能继续进入业务页面

### 8.2 workspace 访问规则

工作区访问语义冻结为：

```text
*.clawloops.app -> Traefik -> 平台 session 鉴权 -> runtime workspace
```

鉴权中间层至少要能判断：

- 当前是否存在有效 session
- 当前用户是否具备对应 workspace membership
- 当前 runtime 是否处于可访问状态

### 8.3 `browserUrl` 安全模型

- `browserUrl` 只表示浏览器入口
- URL 本身不等于权限
- 只有 session 校验通过后，workspace 才真正可访问

---

## 9. 模块职责（修订版）

| 模块 | 职责 |
| --- | --- |
| 模块 1：认证与访问接入 | 处理登录、logout、session、invitation 接受、权限判定 |
| 模块 2：租户与用户资源控制 | 维护 User / Invitation / WorkspaceMembership / UserRuntimeBinding 真相；负责 invitation 消费与 binding 初始化 |
| 模块 3：Runtime 编排 | 只在用户已通过认证且业务绑定合法的前提下处理 runtime 启停删；对外返回异步 task；对内调用 RM 的同步 internal API |
| 模块 4：模型接入、平台凭据代理与用量归集 | 与认证解耦，不处理密码与 invitation，只处理模型治理 |
| 模块 5：管理后台 | 负责 invitation 创建、查看、撤销、用户治理、runtime 查看，并作为 `admin` 登录后的默认首页 |
| 模块 6：用户工作台 | 负责普通用户登录后的工作台承接、runtime 状态展示与 `workspace-entry` 跳转 |
| RuntimeManager | 同步执行容器动作、目录初始化、挂载、网络接入、事实状态查询；不维护外层任务状态机 |

---

## 10. 数据对象与状态冻结

### 10.1 User

- `role = admin | user`
- `status = active | disabled`

### 10.2 Invitation

- `status = pending | consumed | revoked`
- `expired` 通过 `expiresAt < now` 派生

### 10.3 Runtime

- `desiredState = running | stopped | deleted`
- `observedState = creating | running | stopped | error | deleted`
- `ready` 是最终可访问态，不等同于 `observedState`

---

## 11. 部署与网络

### 11.1 共享网络

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

### 11.2 认证相关部署要求

- `clawloops-api` 必须能读写用户、session、invitation 相关数据
- workspace 鉴权中间层必须能访问平台认证校验接口
- 主域名与 workspace 子域名必须共用可验证的平台 session

---

## 12. 首版明确不做

- 第三方登录
- 企业 SSO
- SCIM / 目录同步
- 改密页
- 找回密码页
- 邮箱验证码
- 多因素认证

这些能力后续都可以加，但不能反向污染首版开发边界。

---

## 13. 与 runtime V1 的关系

本次修订不改变 runtime V1 的冻结结论：

- `runtimeId` 平台范围内全局唯一
- RM internal 接口同步执行
- 对用户侧仍由 Orchestrator 返回 `taskId`
- 固定共享网络 `clawloops_shared`
- `internalEndpoint = http://rt-<runtimeId>:18789`
- `18789` 是唯一必检端口
- `compat.openclawConfigDir / compat.openclawWorkspaceDir` 仍为必填

---

## 14. 最终设计结论

首版最稳妥的方案不是继续堆叠外部 IAM，也不是把 runtime 和认证耦合在一起，而是：

- **ClawLoops 自己负责登录、密码哈希、session 与 invitation**
- **ClawLoops 自己负责 membership 绑定与资源治理**
- **Traefik + 平台自有鉴权中间层负责 workspace 入口保护**
- **首版只开本地账号密码**
- **首版不做改密与找回密码**
- **`admin` 登录后默认进入 `/admin`；普通用户登录后默认进入 `/app`；`workspace-entry` 只负责最终跳转与短时等待**
- **Orchestrator 负责对外异步编排，RuntimeManager 负责对内同步执行**
- **runtime V1 统一固定为 `clawloops_shared + 18789 + rt-<runtimeId> + compat 必填`**

这样既满足当前 MVP 的简单落地目标，也给后续扩展第三方登录留出清晰边界。

---

v0.12-轻量认证修订
reno  
2026-03-25
