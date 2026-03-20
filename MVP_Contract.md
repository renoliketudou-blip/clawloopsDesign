# CrewClaw 平台 MVP 开发基线总契约

统一 1 到 6 模块的职责边界、字段约定、状态枚举、错误码和联调流程。

| 文档定位 | 模块协作总契约                                               |
| -------- | ------------------------------------------------------------ |
| 适用阶段 | MVP 首版上线                                                 |
| 变更重点 | 采用“用户只使用，管理员提供服务”模式；冻结 UserRuntimeBinding / 任务状态机 / `browserUrl` / `internalEndpoint`、禁用与删除语义，并补齐首次 binding 初始化边界与 workspace 前置鉴权模型 |
| 本版原则 | 先上线、后演进；能并行开发、能稳定联调、能支持首版发布       |

**边界声明**：本契约覆盖 MVP 必需能力：能登录、能识别用户、能拉起 runtime、能让用户进入工作区、能展示平台开放模型、能由管理员托管平台凭据、能提供管理员侧 usage summary、能做最小治理。复杂多租户、复杂共享空间、复杂费用管理以及普通用户自助模型配置不纳入本版基线。

## 1. MVP 总体原则

| 项         | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 总体范围   | 一台服务器部署，多用户访问，Authentik 登录，Traefik 统一入口，LiteLLM 统一模型出口，每用户一个 OpenClaw runtime。 |
| 产品模式   | 采用团队平台模式：管理员提供 runtime 环境、模型服务和 provider 凭据；普通用户只启动并使用自己的 OpenClaw workspace。 |
| MVP 目标   | 登录、用户同步、runtime 启停、工作区进入、平台模型服务、平台凭据托管、管理后台最小治理、用户工作台最小可用。 |
| 统一限制   | 每用户最多 1 个 runtime；角色只保留 `user / admin`；用户状态只保留 `active / disabled`。 |
| MVP 非目标 | 复杂多租户、复杂共享空间 UI、复杂套餐和费用结算、复杂策略中心、普通用户自助 provider key 管理。 |

## 2. 全局统一字段与枚举

| 字段                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| userId                    | CrewClaw 平台内部用户唯一标识，例如 `u_001`。                |
| subjectId                 | 外部身份唯一标识，例如 `authentik:12345`。                   |
| tenantId                  | 租户标识；MVP 默认固定为 `t_default`。                       |
| role                      | `user / admin`。                                             |
| user.status               | `active / disabled`。                                        |
| desiredState              | `running / stopped / deleted`。                              |
| observedState             | `creating / running / stopped / error / deleted`。           |
| browserUrl                | 浏览器访问 runtime 的地址，由 Traefik 子域名暴露，并统一经过 Authentik 前置鉴权。 |
| internalEndpoint          | 平台内部服务访问 runtime 的地址。                            |
| retentionPolicy           | `preserve_workspace / wipe_workspace`。                      |
| task.status               | `pending / running / succeeded / failed / canceled`。        |
| model.source              | `shared / local`。                                           |
| model.enabled             | `true / false`。                                             |
| providerCredential.status | `active / invalid / disabled`。                              |
| usage.totalTokens         | MVP 展示口径优先采用 OpenClaw 上报；后续审计 / 计费口径以网关记录为准。 |

## 3. UserRuntimeBinding 冻结结构

| 字段             | 必填 | 说明                                                   |
| ---------------- | ---- | ------------------------------------------------------ |
| runtimeId        | 是   | 用户唯一 runtime 标识。                                |
| volumeId         | 是   | 持久化卷标识，挂载 workspace / state / auth profiles。 |
| imageRef         | 是   | 运行时镜像引用；可为官方镜像或 wrapper image。         |
| desiredState     | 是   | 平台目标状态。                                         |
| observedState    | 是   | 宿主机观测状态。                                       |
| browserUrl       | 否   | 浏览器入口地址；runtime ready 前可为空。               |
| internalEndpoint | 否   | 内部访问地址；创建完成前可为空。                       |
| retentionPolicy  | 是   | 默认 `preserve_workspace`。                            |
| lastError        | 否   | 最近一次失败信息。                                     |

**同步要求**：UserRuntimeBinding 的结构变化必须同时更新设计文档、总契约和总接口文档，不允许只改某一份。

## 4. 模块间依赖关系

- 模块 1 为所有受保护接口提供统一身份上下文，并读取 disabled 状态。
- 模块 2 是用户、状态、quota、UserRuntimeBinding 的业务真相层，也是首次 runtime binding 初始化的唯一归口。
- 模块 3 读取模块 2 的 binding，读取模块 4 的模型配置，渲染 config / secret，再调用 runtime manager。
- 模块 4 提供平台模型配置、平台 provider 凭据治理、内部 gateway-config 和 usage 归集。
- 模块 5 仅给 admin 使用，消费模块 2 与模块 4 的治理数据。
- 模块 6 仅给当前用户使用，消费模块 1 / 3 / 4 的最小用户侧能力。

| 模块                                     | 职责                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| 模块 1：身份与访问接入                   | 对接 Authentik、登录态校验、首次登录触发 `/internal/users/sync`、输出 AuthContext、识别 `admin / disabled`。 |
| 模块 2：租户与用户资源控制               | 维护 User / Tenant / Quota / UserRuntimeBinding；负责首次分配 `runtimeId / volumeId / default imageRef / default retentionPolicy`；不碰 Docker。 |
| 模块 3：Runtime 编排                     | 提供 `ensure_running / stop / delete / inspect`；先向模块 2 确保 binding 存在；生成启动模板；调用 runtime manager；回写状态。 |
| 模块 4：模型接入、平台凭据代理与用量归集 | 维护平台模型服务、托管平台 provider 凭据、内部生成 gateway-config、归集 usage summary。 |
| 模块 5：管理后台                         | 用户治理、runtime 查看、模型开关、provider 凭据管理、usage summary。 |
| 模块 6：用户工作台                       | 展示用户信息、runtime 状态、只读模型列表、工作区入口。       |

## 5. 关键跨模块规则

| 规则                | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| 统一入口            | 前端和浏览器只能使用 `browserUrl`；`internalEndpoint` 仅内部使用。 |
| browserUrl 安全模型 | 所有 workspace 子域名统一经 Traefik + Authentik 前置鉴权；知道 URL 不等于可访问。 |
| runtime 启动        | 模块 3 必须先向模块 2 确保 binding 已存在，再渲染模型配置文件和 secret file，再调用 runtime manager 挂载。 |
| binding 首建归口    | 首次创建 UserRuntimeBinding 时，由模块 2 分配 `runtimeId / volumeId / default imageRef / default retentionPolicy`；模块 3 不直接发号。 |
| 模型与凭据归口      | 平台模型与 provider 凭据统一由模块 4 管理；普通用户不直接管理 provider key、不做模型绑定。 |
| gateway-config 权限 | 仅 internal 可读，不向普通用户前端暴露。                     |
| disabled 收口       | 除 `/api/v1/auth/me` 外，disabled 用户访问业务接口统一返回 `403 USER_DISABLED`；`/workspace-entry` 不再返回 `ready=false`。 |
| 删除语义            | 删除 runtime 时必须显式携带 `retentionPolicy`；MVP 默认 `preserve_workspace`。 |
| 本地 profile        | runtime 保留本地 profile，但首次进入由平台自动创建并映射。   |
| 用量口径            | MVP 以 OpenClaw 上报为主；平台仅向管理员侧提供 usage 汇总与治理视角。 |

## 6. 各模块详细定义（修订要点）

### 6.1 模块 1：身份与访问接入

- 未登录返回 `401 UNAUTHENTICATED`。
- 已禁用用户可识别身份，`/api/v1/auth/me` 可返回身份信息；但除该接口外，业务接口统一返回 `403 USER_DISABLED`。
- 首次登录自动触发用户同步。
- workspace 子域名访问统一复用 Authentik 前置鉴权，不允许绕过平台登录直接访问 runtime。

### 6.2 模块 2：租户与用户资源控制

- 以 `subjectId` 幂等创建用户。
- 每用户最多一条 UserRuntimeBinding。
- binding 中必须包含 `volumeId / imageRef / retentionPolicy`。
- 首次 runtime 启动前，模块 2 通过 `runtime-binding/ensure` 幂等创建 binding 并分配 `runtimeId / volumeId / default imageRef / default retentionPolicy`。

### 6.3 模块 3：Runtime 编排

- 创建和启动合并为 `ensure_running`。
- 异步动作返回 `202 Accepted` 和 `taskId`。
- `task.status` 最小枚举为 `pending / running / succeeded / failed / canceled`。
- 首次启动时，模块 3 必须先调用模块 2 的 `runtime-binding/ensure`，获取或创建 binding 后，才能继续渲染配置与编排。
- 若用户已 `disabled`，必须拒绝新动作，并触发运行中 runtime 的收敛停止。

### 6.4 模块 4：模型接入、平台凭据代理与用量归集

- 平台默认模型对新用户开箱即用。
- 平台统一管理 provider 凭据，普通用户不可读取也不可维护 provider 明文 key。
- `gateway-config` 为 internal only，并通过文件 + secret 注入 runtime。
- 归集平台 usage summary，仅向管理员提供治理视角，不向普通用户提供 usage 视图。

### 6.5 模块 5：管理后台

- MVP 以查看、启停开关和状态治理为主。
- 不展示复杂费用看板，不做财务结算。
- 后台最小接口集应覆盖用户列表、用户详情、用户状态修改、runtime 详情、模型策略、provider 凭据状态与 usage summary。

### 6.6 模块 6：用户工作台

- 只展示当前用户自己的资源。
- OpenClaw runtime `running` 且 `browserUrl` 非空时才可进入工作区。
- 用户不管理 provider 凭据，不切换模型绑定。
- 首页或任务轮询场景优先消费 runtime 轻量状态接口；详情与删除确认场景消费完整 binding 接口。

## 7. 统一联调主流程

1. 首次登录：Authentik 登录 -> 模块 1 调 `/internal/users/sync` -> 模块 2 幂等创建用户 -> 模块 6 获取 `/auth/me`。
2. 启动 runtime：模块 6 调 `/users/me/runtime/start` -> 模块 3 先调模块 2 的 `runtime-binding/ensure` -> 模块 3 再读取 binding 和 model config -> 渲染 `config / secret` -> 调 runtime manager `ensure-running` -> 模块 2 回写 binding。
3. 进入工作区：模块 6 调 `/workspace-entry` -> `ready=true` 且 `browserUrl` 非空 -> 前端跳转；子域名访问继续经过 Authentik 前置鉴权。
4. 管理模型服务：模块 5 配置平台模型和 provider 凭据 -> 模块 4 更新网关策略 -> 新启动或刷新后的 runtime 生效。
5. 禁用用户：模块 5 改状态 -> 模块 2 置为 `disabled` -> 编排层收敛 `desiredState=stopped` -> token 立即失效 -> 后续业务访问统一返回 `403 USER_DISABLED`。

## 8. 错误码与验收口径

| 类别       | 冻结要求                                                     |
| ---------- | ------------------------------------------------------------ |
| 字段一致性 | 不得把 `browserUrl / internalEndpoint` 再合并回单一 `endpoint`；不得把 `observedState` 与 `desiredState` 合并为单一 `status`。 |
| 接口一致性 | 前端只依赖统一接口完成工作台、后台、runtime 控制与模型服务展示；`/users/me/runtime` 与 `/users/me/runtime/status` 必须保持边界清晰。 |
| 状态一致性 | 任务状态机、用户状态、runtime 状态在 1 到 6 模块内保持一致。 |
| 功能验收   | runtime 能正常启停、查询、删除，并按 `retentionPolicy` 正确处理数据保留。 |
| 安全验收   | 禁用用户后运行中 runtime 停止，已签发 token 立即失效；workspace 子域名必须经过 Authentik 前置鉴权。 |
| 权限验收   | 普通用户不能访问 provider 凭据治理与模型绑定修改能力；管理员可以统一治理平台模型与凭据。 |

## 9. MVP 之外暂不纳入基线的内容

- 复杂 RBAC。
- 一个用户多个 runtime。
- 复杂共享空间权限与 UI。
- 普通用户自助 API Key 管理和策略中心。
- 费用看板、复杂计费、审计报表和财务结算。
- 复杂灰度升级、自动迁移和弹性扩缩容。


v 0.5
reno 
2026-03-19 14:04

# === v0.5 修订整合（认证与邀请扩展） ===

（新增 auth.method、invitation 规则与首登策略）