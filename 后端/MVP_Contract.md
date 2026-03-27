# ClawLoops 平台 MVP 开发基线总契约（轻量认证版，运行时 V2.2 修订）

统一 1 到 6 模块在“业务内轻量认证 + runtime V2.2”条件下的职责边界、字段约定、状态枚举、错误码和联调流程。

| 文档定位 | 模块协作总契约 |
| --- | --- |
| 适用阶段 | MVP 首版上线 |
| 本版重点 | 增加站内登录与 invitation 首设密码流程、统一 session 鉴权边界、删除外部 IAM 依赖、补充种子管理员首登强制改密，并把 runtime 收敛为后端渲染配置 + RM 执行启动 |
| 本版原则 | 先跑通首版、边界清晰、字段冻结、实现尽量简单、便于直接落地 |
| 当前版本 | v0.14-runtime-v2.2 |

---

## 1. MVP 总体原则

| 项 | 说明 |
| --- | --- |
| 身份系统 | ClawLoops 业务内轻量认证 |
| 首版登录方式 | 仅本地用户名优先登录（兼容邮箱输入）+ 密码 |
| 管理员初始化 | 首个管理员由平台种子数据或初始化脚本创建 |
| 平台模式 | 管理员提供服务，普通用户只使用自己的 workspace |
| 邀请方式 | 平台单层 invitation，用户在站内完成首次设密与接入 |
| 用户首次接入 | 通过一次性 invitation 链接进入站内接入页并直接设置初始密码 |
| 工作区鉴权 | Traefik + 平台 session 鉴权中间层 |
| 密码归属 | 平台认证模块保存密码哈希并执行验证 |
| 密码扩展 | 首版仅支持种子管理员首登强制改密，不做找回密码 |
| runtime V2.2 | 固定镜像、固定内部端口、固定网络、固定 alias、`compat` 必填、后端渲染 `openclaw.json` |
| 用户体验基线 | 普通用户登录后默认进入 `/app`，由工作台承接首次使用与回访使用；无真实邮箱用户也必须可按用户名顺畅接入 |
| workspace 关系基线 | 普通用户首版只会绑定 0 或 1 个 workspace，不存在前端 workspace 选择分支 |
| 首版非目标 | 第三方登录、企业目录同步、复杂审批流、复杂共享空间权限、复杂费用管理 |

---

## 2. 全局统一字段与枚举

### 2.1 用户相关

| 字段 | 说明 |
| --- | --- |
| `userId` | ClawLoops 平台内部用户唯一标识，例如 `u_001` |
| `subjectId` | 平台认证主体标识，例如 `clawloops:u_001` |
| `username` | 平台本地登录用户名；无真实邮箱用户默认优先使用它登录 |
| `tenantId` | MVP 固定为 `t_default` |
| `role` | `user / admin` |
| `user.status` | `active / disabled` |
| `auth.method` | `local_password`（首版固定） |
| `auth.provider` | `clawloops` |

### 2.2 runtime 相关

| 字段 | 说明 |
| --- | --- |
| `runtimeId` | 平台范围内全局唯一 |
| `desiredState` | `running / stopped / deleted` |
| `observedState` | `creating / running / stopped / error / deleted` |
| `browserUrl` | 浏览器可访问地址 |
| `internalEndpoint` | 平台内部访问地址，V2.2 固定为 `http://rt-<runtimeId>:18789` |
| `retentionPolicy` | `preserve_workspace / wipe_workspace` |
| `ready` | 最终可访问状态，不等同于 `observedState` |
| `volumeId` | 平台逻辑卷 ID，不等同于宿主机路径 |
| `imageRef` | 平台记录的实际生效镜像；V1 固定，但不允许调用方通过 RM 覆盖 |

### 2.3 invitation 相关

| 字段 | 说明 |
| --- | --- |
| `invitationId` | 平台邀请唯一标识 |
| `inviteTokenHash` | 平台一次性 token 哈希 |
| `invitation.status` | `pending / consumed / revoked` |
| `targetEmail` | 被邀请用户邮箱槽位；可为空或为代理邮箱 |
| `loginUsername` | 用户首次接入与后续登录优先使用的用户名 |
| `workspaceId` | invitation 目标 workspace |
| `role` | invitation 对应 workspace 角色 |
| `expiresAt` | invitation 过期时间 |
| `consumedByUserId` | 实际消费 invitation 的用户 ID |

---

## 3. 模块边界

### 3.1 模块 1：认证与访问接入

负责：

- 用户名密码登录
- session 建立与撤销
- 公开 invitation 预览
- invitation 接受与首次设密
- 当前登录身份与访问状态查询
- workspace 子域访问的统一鉴权判定

不负责：

- runtime 启停删业务编排
- 模型治理
- provider 凭据治理

### 3.2 模块 2：租户与用户资源控制

负责：

- `User / Invitation / WorkspaceMembership / UserRuntimeBinding` 真相
- invitation 消费与 membership 绑定原子性
- disabled 用户治理

### 3.3 模块 3：Runtime 编排

负责：

- 只在用户已登录且业务绑定合法时处理 runtime 启停删
- 对外返回异步任务
- 对内调用 RM 同步 internal API

### 3.4 模块 4：模型接入、平台凭据代理与用量归集

负责：

- 模型治理
- 平台 provider 凭据
- usage 聚合

不处理：

- 密码
- session
- invitation

### 3.5 模块 5：管理后台

负责：

- invitation 创建、查看、撤销、重发
- 用户治理
- runtime 查看
- 作为 `admin` 默认首页入口

### 3.6 模块 6：用户工作台

负责：

- 普通用户登录后的承接页
- runtime 状态展示
- `workspace-entry` 跳转

---

## 4. 统一身份与登录规则

### 4.1 登录方式

首版只允许：

- 本地用户目录
- 本地密码登录

首版禁止：

- Google / GitHub / 企业 SSO / 微信 / 钉钉 / 飞书
- 多身份源切换
- magic link 登录

### 4.2 管理员初始化

- 首个管理员账号由平台种子数据或初始化脚本创建
- 种子管理员默认初始密码固定为 `admin`
- 初始化时必须写入 `mustChangePassword=true`
- 首次登录成功后必须立刻跳转 `/force-password-change`
- 改密完成前不得进入 `/admin`
- 管理员身份真相保存在平台数据库
- 文档、接口、测试口径不得再混用任何外部 IAM 初始化账号概念

### 4.3 session 规则

- session 由平台服务端签发与校验
- 浏览器通过 HttpOnly cookie 持有 session
- `/auth/me` 是当前登录用户唯一真相
- `/auth/access` 是当前业务可访问性唯一真相

cookie 冻结口径：

- cookie 名称统一为 `clawloops_session`
- `HttpOnly=true`
- 生产环境 `Secure=true`
- `SameSite=Lax`
- `Path=/`
- 生产环境 `Domain` 必须覆盖主域与 workspace 子域，例如 `.clawloops.example.com`
- 登录成功、invitation 接受成功、强制改密成功后若发生 session 创建或轮换，必须写入同一套 cookie 属性
- logout 清 cookie 时必须复用完全一致的 `Domain / Path`

---

## 5. invitation 生命周期冻结

### 5.1 invitation 语义

- 管理员创建 invitation 必须绑定目标 `workspaceId` 与 `role`
- invitation 必须有一次性平台 token
- invitation 消费后必须不可再次用于首次接入
- 平台 invitation 一旦撤销，对应入口必须立即失效

### 5.2 首版推荐流程

1. 管理员创建 invitation
2. 用户打开 `/invite/{token}`
3. 前端调 `GET /api/v1/public/invitations/{token}`
4. 用户提交 `username + password + passwordConfirm`
5. 后端校验 invitation 与密码规则
6. 后端完成用户激活或用户创建
7. 后端完成 `workspace / role` 绑定
8. 后端把 invitation 标记为 `consumed`
9. 后端建立 session
10. 普通用户进入 `/app`；管理员进入 `/admin`

### 5.3 幂等要求

- `accept` 是幂等操作
- 同一 `invitationId + userId` 只能成功消费一次
- 刷新页面、浏览器重试、网络抖动不得产生重复 membership 或重复 side effect
- `consume invitation` 与 `workspace membership binding` 必须原子，或有清晰补偿逻辑
- 首次成功消费返回 `200`
- 若同一 `token` 已被同一 `loginUsername` 对应用户成功消费，再次提交仍返回 `200`，且最终响应语义必须稳定；可额外返回 `replayed=true`
- 若 invitation 已被其他用户消费，返回 `409 INVITATION_ALREADY_CONSUMED`
- 若 invitation 已撤销，返回 `409 INVITATION_REVOKED`
- 若 invitation 已过期，返回 `410 INVITATION_EXPIRED`
- 若提交的 `username` 与 invitation 冻结的 `loginUsername` 不一致，返回 `422 INVITATION_USERNAME_MISMATCH`

---

## 6. 用户名与无真实邮箱兼容规则

- `loginUsername` 是首版主登录标识
- `targetEmail` 可以为空，也可以是代理邮箱
- 前端体验必须优先围绕 `loginUsername`
- 用户首次接入后，后续登录优先用用户名，不要求其记住代理邮箱

---

## 7. 工作区访问与安全边界

### 7.1 控制面安全

- `/admin/*` 仅 `admin`
- `/app` 与 `/workspace-entry` 仅允许已登录且 `allowed=true`
- disabled 用户访问业务接口统一阻断

### 7.2 workspace 子域安全

- `browserUrl` 只表示浏览器入口
- 所有 workspace 子域名必须先通过平台 session 鉴权
- 知道 URL 不等于可访问
- `workspace-entry` 是唯一工作区跳转入口；仅服务非管理员用户

网关鉴权冻结口径：

- 网关统一调用 `GET /internal/auth/workspace-access`
- 网关必须透传浏览器原始 session cookie 与 `Host`
- 网关必须透传 `X-Forwarded-Proto`、`X-Forwarded-Host`、`X-Forwarded-Uri`、`X-Forwarded-Method`
- 平台依据 host 路由规则自行解析目标 `workspaceId`
- 返回 `200` 才允许继续转发到 runtime
- 返回 `401` 表示 session 缺失、失效或已撤销
- 返回 `403` 表示用户已登录但命中 `USER_DISABLED`、`PASSWORD_CHANGE_REQUIRED`、无 workspace membership 或 runtime 当前不可进入
- 网关不得把下游 runtime 的可达性当作权限放行依据，权限真相只来自平台鉴权接口

### 7.3 前后端统一口径

- 前端不解析 token 作为业务真相
- 前端不直接构造用户身份
- 后端不把外部 IAM 头作为信任边界

---

## 8. 首版密码边界

平台统一允许：

- 在 invitation 接受时设置初始密码
- 在登录时校验密码
- 在已登录态下执行受限的首登强制改密

首版统一密码规则：

- 长度 `8-64`
- 至少包含 1 个字母与 1 个数字
- 不允许与 `username` 相同
- 不允许继续使用默认管理员密码 `admin`

平台统一禁止：

- 保存密码明文
- 提供对所有用户开放的通用改密入口
- 提供找回密码 API
- 在日志、审计或错误体中输出密码信息

---

## 9. 联调主流程

### 9.1 普通登录主流程

1. 前端读取 `/api/v1/auth/options`
2. 用户提交 `/api/v1/auth/login`
3. 后端建立 session
4. 前端读取 `/api/v1/auth/me`
5. 若 `mustChangePassword=true`，优先进入 `/force-password-change`
6. 其他场景按角色进入 `/admin` 或 `/app`


### 9.2 invitation 接入主流程

1. 前端进入 `/invite/{token}`
2. 读取 `/api/v1/public/invitations/{token}`
3. 用户提交 `/api/v1/public/invitations/{token}/accept`
4. 后端完成用户激活、membership 绑定、invitation 消费、session 建立
5. 前端依据稳定成功响应进入 `/app`
6. 若 token 无效，则留在 `/invite/{token}` 渲染对应失效页面态

### 9.3 工作区进入主流程

1. 前端读取 `/api/v1/users/me/runtime/status`
2. 用户点击进入工作区
3. 前端读取 `/api/v1/workspace-entry`
4. 仅当 `ready=true` 时整页跳转 `browserUrl`
5. workspace 入口再次进行平台 session 校验

---

## 10. 与 runtime V2.2 的冻结边界

本次认证改造不改变 runtime 的执行边界，但把启动方式收敛为后端渲染配置、RM 执行启动。关键规则如下：

- `runtimeId` 在平台范围内全局唯一
- 用户侧 runtime 启停删是 **Orchestrator 异步任务**
- RM internal 接口是 **同步执行器**
- runtime 镜像固定
- `compat.openclawConfigDir / compat.openclawWorkspaceDir` 是 `ensure-running` 必填
- `renderedConfig.openclawJson` 是 `ensure-running` 必填
- `renderedConfig.configVersion` 是 `ensure-running` 必填
- 统一共享网络为 `clawloops_shared`
- LiteLLM 由平台后端共享 stack 统一启动，并以服务名 `litellm` 对多个 runtime 提供服务
- `internalEndpoint` 固定为 `http://rt-<runtimeId>:18789`
- `18789` 是唯一必检端口；`18790` 仅兼容保留
- 浏览器正式入口统一走平台 session 鉴权，不以 runtime 直连端口作为主路径
- gateway token 由 Orchestrator 持久化，默认长期有效，删除用户时失效

---

## 11. 禁止行为清单

| 禁止行为 | 原因 |
| --- | --- |
| 把 `subjectId` 改成其他命名 | 会造成接口与前端状态模型漂移 |
| 在首版再引入外部 IAM token | 会把单层 invitation 重新变复杂 |
| 让前端自己消费 invitation | 会破坏事务边界 |
| 让 RM 校验用户登录态 | RM 不应承担认证职责 |
| 增加通用改密或找回密码接口 | 超出首版范围，联调成本上升 |
| 让普通用户登录后默认先进 `workspace-entry` | 普通用户默认落点应为 `/app` |

---

## 12. 最终契约结论

本版 MVP 契约的核心不是“让外部 IAM 接管一切”，也不是“让 RuntimeManager 自己兜底所有补偿”，而是：

- **ClawLoops 接管身份、密码哈希、会话和 invitation**
- **ClawLoops 统一完成 invitation 消费与 workspace membership 绑定**
- **Traefik + 平台自有鉴权中间层保护 workspace 子域**
- **`admin` 登录后默认进入 `/admin`；普通用户登录后默认进入 `/app`**
- **种子管理员默认密码为 `admin`，首次登录必须先完成强制改密**
- **`workspace-entry` 只负责最终跳转，不承担登录收口**
- **LiteLLM 由平台后端共享托管，runtime 不再各自启动 LiteLLM**
- **Orchestrator 负责渲染完整 `openclaw.json`，RuntimeManager 只写盘和挂载**
- **runtime V2.2 contract 继续冻结，不随认证改造漂移**
- **首版不做通用改密与找回密码**

这套边界一旦冻结，前后端与平台服务就可以并行开发，而不需要在开发中途反复重谈认证模型。

---

v0.14-runtime
reno  
2026-03-27
