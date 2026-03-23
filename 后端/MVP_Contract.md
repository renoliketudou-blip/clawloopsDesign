
# ClawLoops 平台 MVP 开发基线总契约（Authentik 接入版，运行时冻结修订）

统一 1 到 6 模块在“官方 Authentik 首版接入 + runtime V1 冻结”条件下的职责边界、字段约定、状态枚举、错误码和联调流程。

| 文档定位 | 模块协作总契约 |
| --- | --- |
| 适用阶段 | MVP 首版上线 |
| 本版重点 | 增加管理员初始化冻结口径、邀请制接入生命周期、首登密码流程、post-login 幂等、runtime 跳转规则与 runtime V1 contract |
| 本版原则 | 不改上游源码、先跑通首版、边界清晰、字段冻结、便于直接落地 |
| 当前版本 | v0.8-authentik-runtime-frozen |

---

## 1. MVP 总体原则

| 项 | 说明 |
| --- | --- |
| 身份系统 | 官方 Authentik |
| 首版登录方式 | 仅本地用户名/邮箱 + 密码 |
| 管理员初始化 | 首个官方 bootstrap 管理员统一按默认 `akadmin` 处理 |
| 平台模式 | 管理员提供服务，普通用户只使用自己的 workspace |
| 邀请方式 | 平台 invitation + Authentik enrollment flow，首版延迟创建身份侧 invitation |
| 用户首次接入 | 通过一次性 invitation 链接进入 enrollment flow 并直接设置密码 |
| 工作区鉴权 | Traefik + Authentik Proxy Outpost + Forward Auth |
| 密码归属 | 统一交给 Authentik 管理 |
| runtime V1 | 固定镜像、固定端口、固定网络、固定 alias、`compat` 必填 |
| 首版非目标 | 第三方登录、企业目录同步、复杂审批流、复杂共享空间权限、复杂费用管理 |

---

## 2. 全局统一字段与枚举

### 2.1 用户相关

| 字段 | 说明 |
| --- | --- |
| `userId` | ClawLoops 平台内部用户唯一标识，例如 `u_001` |
| `subjectId` | 外部身份唯一标识，例如 `authentik:12345` |
| `tenantId` | MVP 固定为 `t_default` |
| `role` | `user / admin` |
| `user.status` | `active / disabled` |
| `auth.method` | `local_password`（首版固定） |
| `auth.provider` | `authentik` |

### 2.2 runtime 相关

| 字段 | 说明 |
| --- | --- |
| `runtimeId` | 平台范围内全局唯一 |
| `desiredState` | `running / stopped / deleted` |
| `observedState` | `creating / running / stopped / error / deleted` |
| `browserUrl` | 浏览器可访问地址 |
| `internalEndpoint` | 平台内部访问地址，V1 固定为 `http://rt-<runtimeId>:18789` |
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
| `targetEmail` | 被邀请邮箱 |
| `workspaceId` | 目标工作区 |
| `invitation.role` | 被邀请后获得的业务角色 |
| `authentikInvitationRef` | Authentik 侧 invitation 引用，首版允许为空 |
| `consumedByUserId` | 最终消费该 invitation 的用户 |
| `expiresAt` | 过期时间；`expired` 由其派生，不入库 |

### 2.4 任务状态

| 字段 | 说明 |
| --- | --- |
| `task.status` | `pending / running / succeeded / failed / canceled` |

### 2.5 字段冻结清单

以下字段名称不得漂移：

- `subjectId`
- `invitationId`
- `inviteTokenHash`
- `workspaceId`
- `role`
- `desiredState`
- `observedState`
- `browserUrl`
- `internalEndpoint`
- `invitation.status`

---

## 3. UserRuntimeBinding 冻结结构

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `runtimeId` | 是 | 用户唯一 runtime 标识；平台范围全局唯一 |
| `volumeId` | 是 | 平台逻辑卷标识 |
| `imageRef` | 是 | runtime 实际生效镜像引用；V1 由平台固定，不允许调用方覆盖 |
| `desiredState` | 是 | 平台目标状态 |
| `observedState` | 是 | 宿主机观测状态 |
| `browserUrl` | 否 | 浏览器入口 |
| `internalEndpoint` | 否 | 内部访问地址；V1 固定为 `http://rt-<runtimeId>:18789` |
| `retentionPolicy` | 是 | 默认 `preserve_workspace` |
| `lastError` | 否 | 最近一次失败信息 |

**新增说明**：

`browserUrl` 能否真正返回给前端，不仅取决于 runtime 状态，也取决于：

1. 当前请求已通过 Authentik 前置鉴权
2. 当前 ClawLoops 用户状态为 `active`
3. 当前用户已具备合法 workspace 绑定
4. 当前 runtime 属于该用户
5. 当前入口接口返回 `ready=true`

**语义冻结**：

- `task.status` = 操作生命周期
- `observedState` = 资源状态
- `ready` = 最终可访问状态
- 前端跳转只看 `ready`

---

## 4. 新增冻结对象：Invitation

本版把 `Invitation` 也提升为冻结对象。

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `invitationId` | 是 | 平台邀请唯一标识 |
| `inviteTokenHash` | 是 | 一次性 token 哈希 |
| `targetEmail` | 是 | 受邀邮箱 |
| `workspaceId` | 是 | 邀请目标工作区 |
| `role` | 是 | 邀请后绑定角色 |
| `status` | 是 | `pending / consumed / revoked` |
| `expiresAt` | 是 | 过期时间 |
| `consumedAt` | 否 | 消费时间 |
| `consumedByUserId` | 否 | 消费用户 |
| `authentikInvitationRef` | 否 | Authentik 侧 invitation 标识 |
| `lastError` | 否 | 最近一次失败原因 |

**冻结原则**：

- Invitation 是 ClawLoops 的业务对象
- Authentik invitation 只是身份层执行引用
- `workspaceId / role / status` 以 ClawLoops 为真相
- `expired` 仅通过 `expiresAt < now>` 派生
- `用户创建 / 密码设置 / 会话建立` 以 Authentik 为真相

---

## 5. 模块间依赖关系（修订版）

| 模块 | 职责 |
| --- | --- |
| 模块 1：身份与访问接入 | 对接 Authentik 会话、读取前置鉴权上下文、首次登录触发 `/internal/users/sync`、处理 invitation 完成后的 post-login 收口、执行邮箱强校验 |
| 模块 2：租户与用户资源控制 | 维护 User / Invitation / WorkspaceMembership / UserRuntimeBinding 真相；负责首次 binding 初始化；保证 invitation 消费与 membership 绑定的原子性或补偿逻辑 |
| 模块 3：Runtime 编排 | 只在用户已通过认证且业务绑定合法的前提下处理 runtime 启停删；对外返回异步 task；对内调用 RM 同步接口 |
| 模块 4：模型接入、平台凭据代理与用量归集 | 与 Authentik 解耦，不处理密码与 invitation，只处理模型治理 |
| 模块 5：管理后台 | 负责 invitation 创建、查看、撤销、用户治理、runtime 查看 |
| 模块 6：用户工作台 | 负责用户首次接入完成后的工作台承接、runtime 状态展示与 `workspace-entry` 跳转 |
| RuntimeManager | 同步执行容器动作、目录初始化、挂载、网络接入、事实状态查询；不维护外层任务状态机 |

---

## 6. 关键跨模块规则

### 6.1 统一身份归属

- 身份认证由 Authentik 负责
- ClawLoops 不自行校验用户密码
- ClawLoops 只消费 Authentik 已认证后的会话与身份上下文

### 6.2 统一登录方式

首版只允许：

- 本地用户目录
- 本地密码登录

首版禁止：

- Google / GitHub / 企业 SSO / 微信 / 钉钉 / 飞书
- 混合登录入口同时上线

### 6.3 统一管理员初始化口径

- 首个官方初始化管理员固定按默认 `akadmin` 处理
- 业务上视为管理员角色账号
- 如需额外日常管理员，可在初始化后创建
- 文档、接口、测试口径不得再混用 `admin` 与 `akadmin` 作为首个初始化账号

### 6.4 统一 invitation 语义

- 管理员创建 invitation 必须绑定目标 `workspaceId` 与 `role`
- invitation 必须有一次性平台 token
- invitation 消费后必须不可再次使用
- 平台 invitation 一旦撤销，对应入口必须立即失效
- 平台 token 是业务入口真相；Authentik `itoken` 只是身份执行入口

### 6.5 invitation 生命周期冻结

- 创建 invitation 时只生成 ClawLoops 业务 token
- 用户调用 `start` 时再延迟创建或换取 Authentik enrollment URL
- 平台状态是最终业务真相
- `revoked / expired / consumed` 必须由平台优先校验
- 即使身份侧 token 尚有效，也必须以平台状态阻断 `start` 或 `post-login`

### 6.6 post-login 收口规则

用户通过 Authentik 完成 enrollment 或登录后，模块 1 必须执行：

1. `/internal/users/sync`
2. 检查是否存在待消费 invitation 上下文
3. 若存在，先执行邮箱强校验
4. 调模块 2 完成 workspace / role 绑定
5. 把 invitation 标记为 `consumed`
6. 清理待消费上下文

**幂等要求**：

- `post-login` 必须幂等
- 同一 `invitationId + userId` 只能成功消费一次
- 刷新页面、浏览器重试、网络抖动不得产生重复 membership 或重复 side effect
- `consume invitation` 与 `workspace membership binding` 必须原子，或有清晰补偿逻辑

### 6.7 invitation start 规则

- `start` 是幂等操作
- 可重复调用，但只产生一个有效 pending invitation 会话
- pending invitation session 需绑定当前浏览器会话并具备 TTL（建议 10–30 分钟）
- `start` 不直接消费 invitation

### 6.8 runtime 启动前置条件

在模块 3 接收 `ensure_running` 前，必须满足：

- 用户已登录
- 用户不为 `disabled`
- 用户具备合法 workspace 绑定
- 若当前页面来自 invitation 完成流程，则 invitation 已成功收口

### 6.9 browserUrl 安全模型

- `browserUrl` 只表示浏览器入口
- 所有 workspace 子域名必须经 Traefik + Authentik Forward Auth
- 知道 URL 不等于可访问
- 只有 `ready=true` 时前端才能跳转
- `workspace-entry` 是唯一跳转入口；`runtime/status` 只用于展示状态

### 6.10 disabled 收口规则

- 除 `/api/v1/auth/me` 外，disabled 用户访问业务接口统一返回 `403 USER_DISABLED`
- `/api/v1/auth/access` 永远返回 `200`，仅用于状态判断
- disabled 用户不可继续消费 invitation
- disabled 用户若已有运行中 runtime，系统应尽快收敛到 `stopped`

### 6.11 internal API 安全规则

- `/internal/*` 接口只允许服务间访问
- 必须使用 mTLS、internal token 或同等级服务鉴权
- 禁止公网暴露

### 6.12 Orchestrator 与 RuntimeManager 边界

冻结规则：

- **Orchestrator 决策，RuntimeManager 执行**
- Orchestrator 负责：
  - 对用户侧暴露 `taskId`
  - 计算 effectiveRetentionPolicy
  - 解析 `volumeId -> host path`
  - 固定 V1 `imageRef` / `command`
  - 决定在 `RUNTIME_CONTRACT_DRIFT` 后是否重建
- RuntimeManager 负责：
  - 同步执行容器动作
  - 执行宿主机目录初始化
  - 检测关键 drift
  - 返回最小事实状态
- RM **不得**自己做复杂补偿编排或自动重建

### 6.13 runtime V1 输入契约

RM 的 `ensure-running` 请求中：

- `compat.openclawConfigDir`：必填
- `compat.openclawWorkspaceDir`：必填
- `configMount.configFilePath / secretFilePath`：可选增强挂载，不替代 `compat`
- `env / envOverrides`：用于显式注入环境变量
- `imageRef / networkName / gatewayPort`：**不得再出现在 V1 请求体**

### 6.14 runtime V1 固定网络与地址

- 统一共享网络固定为 `clawloops_shared`
- 固定通信别名为 `rt-<runtimeId>`
- 固定内部地址为 `http://rt-<runtimeId>:18789`
- `18789` 是唯一 readiness 端口
- `18790` 仅兼容保留，不作为 readiness 条件
- 查找容器靠 label；通信靠固定 alias；不靠容器名猜测

### 6.15 drift 与删除规则

- 关键 drift 项包括：固定镜像、固定命令、网络接入、network alias、必需挂载、必需 env、必需 labels、18789 主端口
- `routeHost` 变化不属于关键 drift
- `GET /internal/runtime-manager/containers/{runtimeId}` 找不到容器事实时，返回 `200 + observedState=deleted`
- 若同一 `runtimeId` 命中多个受管容器，返回 `409 RUNTIME_ACTION_CONFLICT`
- `delete(nonexistent)=deleted`
- `stop(nonexistent)=stopped`
- `wipe_workspace` 删除 `compat.openclawConfigDir` 与 `compat.openclawWorkspaceDir` 指向的数据；若父子重叠，按去重后的根路径集合执行

---

## 7. Authentik 首次管理员初始化契约

### 7.1 首次初始化基线

- 首次管理员初始化使用官方流程完成
- 平台不自建“初始化管理员密码设置页”
- 平台只承接完成后的登录态与身份同步

### 7.2 正式口径

- 首个官方 bootstrap 管理员为 `akadmin`
- 业务上视为管理员角色账户
- 不再把“管理员账号必须字面叫 `admin`”作为首版要求

### 7.3 日常管理员策略

推荐：

- 首次初始化管理员用于 bootstrap
- 日常平台管理由额外创建的本地管理员账号承担
- bootstrap 账号只保留 break-glass 应急用途

---

## 8. 邀请制接入契约

### 8.1 邀请创建

管理员创建 invitation 时必须提供：

- `targetEmail`
- `workspaceId`
- `role`
- `expiresAt` 或默认有效期

系统必须生成：

- 平台一次性 token
- 对应 `Invitation` 记录

系统在首版 **不要求** 同步预创建 Authentik invitation。

### 8.2 邀请预览

用户打开 invitation 链接后，平台必须能返回：

- 邀请是否有效
- 目标邮箱
- 目标 workspace
- 目标 role
- 是否已消费 / 已撤销 / 已过期

### 8.3 邀请启动

用户点击“继续接入”后：

- 平台先验证 platform token
- 再验证平台 invitation 状态
- 再写入短期 pending invitation 会话
- 再创建或换取 Authentik enrollment flow URL
- `start` 必须幂等

### 8.4 邀请完成

Authentik 完成 enrollment / login 后：

- ClawLoops 必须以当前 `subjectId` 关联用户
- `post-login` 阶段执行邮箱强校验
- 完成目标 workspace / role 绑定
- 标记 invitation `consumed`
- 后续同一 token 不再可用

### 8.5 冲突策略

若出现以下情况，必须拒绝接入并显示明确错误：

- token 不存在
- token 已过期
- token 已撤销
- token 已消费
- 当前登录用户与 invitation 目标邮箱不一致
- invitation 指向的 workspace 不存在或已关闭

---

## 9. 密码与首登策略契约

### 9.1 首版正式基线

本版冻结采用：

- invitation enrollment flow 内直接设置密码
- 完成后自动登录
- 后续密码修改走 Authentik 自身 flow

### 9.2 平台侧禁止行为

- 禁止在 ClawLoops 数据库保存用户密码
- 禁止平台自己生成“临时密码”并作为正式密码存储
- 禁止平台自己提供密码落库接口
- 禁止绕开 Authentik 自行实现第二套密码修改 API

### 9.3 可选扩展

若后续要启用“下次登录强制改密”，应通过 Authentik Flow 完成，而不是在平台 API 中手写状态机。

---

## 10. 联调主流程（修订版）

### 10.1 首次管理员初始化

1. 启动 Authentik
2. 管理员完成官方初始化流程
3. Authentik 管理后台完成应用、Provider、Outpost、Flow 配置
4. 访问 ClawLoops 控制面
5. 模块 1 同步管理员用户

### 10.2 管理员创建邀请

1. 模块 5 提交 invitation 创建请求
2. 模块 2 落库 `Invitation`
3. 返回 invite URL

### 10.3 用户完成接入

1. 用户打开 `/invite/{token}`
2. 模块 2 校验 invitation
3. 模块 1 生成跳转到 Authentik enrollment flow 的入口
4. 用户在 Authentik 中完成资料和密码设置
5. Authentik 登录成功后回到 ClawLoops
6. 模块 1 调 `/internal/users/sync`
7. 模块 1 执行邮箱强校验
8. 模块 2 根据 pending invitation 幂等完成绑定
9. 模块 6 跳转工作台

### 10.4 正常登录并进入工作区

1. 用户访问平台域名
2. Traefik + Authentik 完成前置鉴权
3. 模块 1 获取 AuthContext
4. 模块 6 获取 `/workspace-entry`
5. `ready=true` 时才允许跳转 `browserUrl`

### 10.5 runtime 启动链路

1. 用户调 `POST /api/v1/users/me/runtime/start`
2. 模块 3 返回 `taskId`
3. 模块 3 调 RM `ensure-running`
4. RM 同步执行并返回当前 `observedState`
5. 模块 3 更新 task 与 binding
6. 前端用 `/runtime/tasks/{taskId}` + `/runtime/status` 轮询

---

## 11. 错误码与验收口径

### 11.1 新增错误码

| HTTP | code | 说明 |
| --- | --- | --- |
| 404 | `INVITATION_NOT_FOUND` | invitation 不存在 |
| 409 | `INVITATION_ALREADY_CONSUMED` | invitation 已使用 |
| 409 | `INVITATION_REVOKED` | invitation 已撤销 |
| 410 | `INVITATION_EXPIRED` | invitation 已过期（派生值） |
| 422 | `INVITATION_EMAIL_MISMATCH` | 当前接入邮箱与邀请目标不匹配 |
| 422 | `INVITATION_WORKSPACE_INVALID` | invitation 指向的 workspace 无效 |
| 409 | `RUNTIME_ACTION_CONFLICT` | runtime 状态冲突或命中多个容器 |
| 409 | `RUNTIME_CONTRACT_DRIFT` | 容器 contract drift |
| 500/502 | `INVITATION_ERROR` | invitation / enrollment 执行失败 |
| 500/502 | `USER_SYNC_ERROR` | 登录后用户同步失败 |
| 500/502 | `RUNTIME_START_FAILED` | runtime 启动失败 |
| 500 | `RUNTIME_STOP_FAILED` | runtime 停止失败 |
| 500 | `RUNTIME_DELETE_FAILED` | runtime 删除失败 |

### 11.2 功能验收

必须满足：

1. 管理员可完成首次初始化
2. 平台首版只显示本地密码登录
3. 管理员可创建绑定 workspace / role 的 invitation
4. invitation 为一次性 platform token
5. 用户可通过 invitation 完成接入并在 flow 中直接设置密码
6. 用户首次接入后能进入平台
7. workspace 子域名受 Authentik 前置鉴权保护
8. 用户密码由 Authentik 管理
9. `post-login` 与 `start` 均支持幂等重试
10. runtime 统一使用 `clawloops_shared + 18789 + rt-<runtimeId>`
11. RM internal API 不再接收 `imageRef`

### 11.3 安全验收

必须满足：

- 知道 `browserUrl` 不能绕过登录直接访问
- invitation token 不能重复消费
- 平台 `revoked / expired / consumed` 状态优先于身份侧 token
- disabled 用户不能继续访问业务接口
- internal API 不得公网暴露
- 平台数据库不保存 Authentik 密码明文或散列
- RM 不读取 secret 文件内容并自动注入 env

---

## 12. MVP 之外暂不纳入基线的内容

- Google / GitHub / 企业 SSO / 微信 / 钉钉 / 飞书登录
- SCIM / LDAP / AD 同步
- MFA 强制上线
- invitation 审批流
- 多 runtime / 多 workspace 自助切换
- 用户自助 provider key 管理
- 复杂组织架构同步

---

## 13. 最终契约结论

本版 MVP 契约的核心不是“让 Authentik 接管一切”，也不是“让 RuntimeManager 自己兜底所有补偿”，而是：

- **Authentik 接管身份、密码、会话和 enrollment flow**
- **ClawLoops 接管用户业务状态、workspace / role 绑定、runtime 与资源治理**
- **Invitation 采用业务真相与身份执行解耦，首版统一为延迟创建模式**
- **首版只开本地密码，后续再向外扩展**
- **工作区跳转统一由 `workspace-entry` 收口，前端只在 `ready=true` 时跳转**
- **Orchestrator 对外异步，RuntimeManager 对内同步**
- **runtime V1 统一固定为 `clawloops_shared + 18789 + rt-<runtimeId> + compat 必填`**

这套边界一旦冻结，前后端与平台服务就可以并行开发，而不需要在开发中途反复重谈认证模型和 runtime contract。

---

# 冻结附录

> 本附录用于将实现层关键细节去歧义化，作为 MVP 开发冻结基线的一部分。  
> 若与正文存在冲突，以本附录为准。

---

## A1. 数据库约束（强制）

### Users

- `subject_id` → **UNIQUE NOT NULL**
- `email` → **UNIQUE NOT NULL（lowercase 规范化）**

### Invitations

- `invite_token_hash` → **UNIQUE NOT NULL**
- `target_email` → **NOT NULL（lowercase）**
- `status` → 不存储 `expired`，仅存：
  - `pending | consumed | revoked`
- `expires_at` → 必填

#### 状态判定（只读规则）

```text
expired = (status == pending && now > expires_at)
```

### Workspace Memberships

- 唯一约束：

```text
UNIQUE (workspace_id, user_id)
```

- `status` 枚举（冻结）：

```text
active | disabled
```

### Invitation 消费幂等约束（必须实现）

必须保证以下逻辑具备幂等性：

```text
(invitation_id, user_id) -> 只能成功绑定一次
```

---

## A2. 枚举冻结（不可扩展）

### Platform Role（users.role）

```text
platform_admin
platform_user
```

### Workspace Role（workspace_memberships.role）

```text
workspace_admin
workspace_member
```

### Membership Status

```text
active
disabled
```

### Invitation Error Code（统一返回）

```text
INVALID_TOKEN
EXPIRED
REVOKED
EMAIL_MISMATCH
ALREADY_CONSUMED
```

---

## A3. `post-login` 流程最终定义（冻结）

### 定义

`POST /api/v1/auth/post-login` 是：

> **前端在 Authentik 登录完成后主动调用的 BFF 接口（非 Authentik callback）**

### 流程时序（唯一标准）

1. 用户完成 Authentik 登录
2. 浏览器返回前端应用
3. 前端调用：

```text
POST /api/v1/auth/post-login
```

### 请求来源

- 用户身份来自：
  - Authentik session / Forward Auth header
- invitation 上下文来自：
  - cookie 或 server session（推荐）
  - 不依赖前端 body 传递

### 行为逻辑（必须幂等）

```text
IF 有 pending invitation:
    校验 email 匹配
    创建 / 更新 membership
    标记 invitation = consumed（原子或补偿）
```

### 返回格式（冻结）

```json
{
  "hasWorkspace": true,
  "workspaceId": "ws_xxx",
  "needsWorkspaceSelection": false
}
```

---

## A4. 网关与鉴权信任边界（冻结）

### Traefik + Authentik Forward Auth

统一入口：

```text
*.clawloops.app -> Traefik -> Authentik Forward Auth -> App
```

### 应用层唯一信任来源（必须）

应用层只信任以下 header：

```text
X-authentik-uid
X-authentik-email
X-authentik-username
```

### 禁止

- 不解析 JWT（MVP）
- 不信任前端传 `userId`
- 不直连 Authentik API 获取用户身份

---

## A5. Workspace / 用户关系规则（冻结）

### 关系定义

- 一个 user：
  - 可以属于多个 workspace
- 一个 workspace：
  - 可以有多个 user

### 登录后跳转规则

```text
IF membership == 1:
    自动进入 workspace

IF membership > 1:
    进入 /workspace-entry

IF membership == 0:
    hasWorkspace = false
```

### 禁用规则（强制）

```text
user.disabled -> 拒绝所有访问
membership.disabled -> 不可进入该 workspace
```

---

## A6. 事务与一致性模型（冻结）

### 强一致推荐路径（优先）

```text
单事务：
    consume invitation
    create membership
```

### 若无法单事务（允许）

必须实现补偿：

```text
IF membership 创建成功但 invitation 未标记：
    后续 post-login 自动修复
```

### 不允许

- invitation consumed 但 membership 不存在（无补偿）
- 非幂等写入

---

## A7. 最小联调用例（必须通过）

1. 新用户邀请：`start -> 注册 -> post-login -> membership 创建`
2. 已登录用户接受邀请：`start -> post-login -> 直接绑定`
3. token 重复点击：不报错，不重复绑定
4. post-login 重试：不重复写入
5. expired token：返回 `EXPIRED`
6. revoked token：返回 `REVOKED`
7. email mismatch：返回 `EMAIL_MISMATCH`
8. disabled user：登录后拒绝访问

---

## A8. RuntimeManager 冻结附录（新增）

### A8.1 V1 固定镜像与命令

```text
imageRef = ghcr.io/openclaw/openclaw@sha256:a5a4c83b773aca85a8ba99cf155f09afa33946c0aa5cc6a9ccb6162738b5da02
command  = node dist/index.js gateway --bind lan --port 18789
```

### A8.2 RM internal API 规则

- `ensure-running` 请求体**不再包含 `imageRef`**
- `compat.openclawConfigDir / openclawWorkspaceDir` 为必填
- `networkName / gatewayPort` 从 V1 请求体删除
- `configMount.configFilePath / secretFilePath` 为可选增强挂载
- `env / envOverrides` 才是显式环境变量注入来源
- RM 不读取 secret 文件内容并自动变成 env

### A8.3 固定网络、端口、别名

```text
network        = clawloops_shared
networkAlias   = rt-<runtimeId>
internalEP     = http://rt-<runtimeId>:18789
readinessPort  = 18789
compatPortOnly = 18790
```

### A8.4 状态与时间窗

- `creating / running / stopped / error / deleted`
- 启动成功窗口：`30s grace + 1s poll + 3 consecutive successes`
- 超过窗口仍未就绪：`observedState=error` + `RUNTIME_START_FAILED`

### A8.5 drift 与查询规则

- 关键 drift：镜像、命令、网络接入、network alias、挂载、必需 env、labels、18789 主端口
- `routeHost` 变化不属于关键 drift
- `GET /containers/{runtimeId}` 找不到容器事实时：`200 + observedState=deleted`
- 若匹配多个容器：`409 RUNTIME_ACTION_CONFLICT`

### A8.6 删除矩阵

- `stop(nonexistent)=stopped`
- `delete(nonexistent)=deleted`
- `wipe_workspace` 删除 `compat.openclawConfigDir` 与 `compat.openclawWorkspaceDir`
- 若两者父子重叠，按去重后的根路径集合执行，避免重复删除与越界删除

---

v0.8-authentik-runtime-frozen  
reno  
2026-03-23
