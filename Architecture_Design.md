# CrewClaw 平台设计文档

基于已确认策略“**用户只使用，管理员提供服务；用户只用，不配置**”后的统一重构版本。

| 文档定位 | 系统架构与 MVP 落地设计 |
| -------- | ---------------------- |
| 适用范围 | 单机部署、多用户访问、容器级隔离、统一模型网关 |
| 本版重点 | 收紧 MVP 为团队平台模式；用户侧仅保留启动/进入工作区；模型、凭据与用量治理统一收敛到管理员侧；继续冻结 `browserUrl` / `internalEndpoint` 与 runtime 启动闭环 |
| 关联文档 | 《MVP 开发基线总契约》《MVP 统一总接口》《开源与涨星策略》 |

### 本次修订摘要

MVP 明确采用“管理员提供服务、用户只使用”的团队平台模式；普通用户不再管理 provider 凭据、不再切换模型绑定、不再进行复杂配置；平台统一提供默认模型服务与网关访问；管理员侧负责模型可见性、上游凭据、平台级 usage summary 与用户治理；普通用户侧不提供 usage 统计展示。；UserRuntimeBinding 继续包含 `volumeId`、`imageRef`、`browserUrl`、`internalEndpoint`、`retentionPolicy`、`lastError`；runtime 启动仍由模块 3 先渲染配置文件与 secret file，再交由 runtime manager 挂载；统一入口继续采用 Traefik 子域名路由，并由 Authentik 对 workspace 子域名做前置鉴权；首次启动时由模块 2 负责初始化 UserRuntimeBinding 并分配 `runtimeId / volumeId / default imageRef`。

## 1. 目标与范围

目标是在一台服务器上部署公司内部 CrewClaw 平台，使 OpenClaw 能以**团队平台**方式运行：支持多用户访问、账号密码登录、容器级隔离、统一模型接入、个人 runtime 工作区、平台统一凭据托管以及管理员视角的用量治理。

MVP 不追求复杂多租户能力，不做复杂共享空间 UI，不实现复杂套餐、财务结算和复杂策略中心；普通用户侧不提供模型绑定、provider 凭据管理等自服务配置能力。这些能力保留到后续路线图。

| 范围 | MVP 结论 |
| ---- | -------- |
| 多租户 | 仅保留 `tenantId` 基础字段，MVP 默认固定为 `t_default`，不做复杂租户治理。 |
| 共享空间 | MVP 不提供复杂共享空间 UI；仅保留后续扩展接口空间。 |
| 隔离边界 | 采用每用户一个 runtime 容器的容器级隔离，不宣称强安全沙箱级隔离。 |
| 模型治理 | 提供统一模型网关、平台默认模型、平台托管上游 provider 凭据。 |
| 用户配置 | 普通用户只启动并使用自己的 OpenClaw workspace，不管理模型、凭据和绑定。 |
| 费用管理 | MVP 不做复杂费用结算；用量仅保留管理员侧 usage summary 视图。 |

## 2. 整体架构

- 统一入口：Traefik，采用子域名路由，把浏览器入口和内部服务地址分开管理。
- 统一认证：Authentik，平台负责会话鉴权与身份上下文注入；所有 workspace 子域名统一经过 Authentik 前置鉴权。
- 平台层：控制面 API + 极简 Web UI，负责用户、状态、资源真相与管理后台。
- 运行时：每用户一个 OpenClaw runtime 容器。
- 生命周期管理：独立 runtime manager / provisioner，负责容器创建、停止、删除、挂载与状态查询。
- 模型网关：LiteLLM + PostgreSQL，统一接公司模型、本地模型和平台托管的 provider 凭据。
- 本地模型：vLLM 为主，Ollama 作为补充。
- 密钥管理：MVP 使用 secret file 注入；后续可接 OpenBao。

| 层级 | 核心组件 | 职责 |
| ---- | -------- | ---- |
| 入口层 | Traefik + Authentik | 统一外部访问、登录、会话识别、workspace 子域名前置鉴权与转发。 |
| 平台层 | CrewClaw 控制面 | 用户同步、UserRuntimeBinding 管理、管理后台、最小用户工作台。 |
| 编排层 | Runtime Orchestrator + Runtime Manager | 将 desiredState 落为容器实际状态。 |
| 运行时层 | Per-user OpenClaw runtime | 每个用户自己的 workspace、state、auth profiles 与 agent runtime。 |
| 模型层 | LiteLLM + PostgreSQL + vLLM/Ollama | 统一模型出口、平台默认模型、平台托管凭据与 usage 归集。 |

## 3. 关键设计决策

| 主题 | 修订后的确定方案 |
| ---- | ---------------- |
| 产品模式 | 采用团队平台模式：管理员提供环境、模型与凭据；用户只启动并使用自己的 OpenClaw runtime。 |
| 多租户 / 共享空间 | MVP 暂不追求复杂多租户能力，不做复杂共享空间 UI。 |
| 仓库策略 | 不纳入 OpenClaw 主仓源码；固定官方镜像 tag；允许基于官方镜像构建 runtime wrapper image，或通过 entrypoint / sidecar / runtime manager 注入配置与卷挂载；fork / patch 仅作为预案。 |
| runtime 配置注入 | 采用方案 A：模块 3 先渲染模型网关配置文件与 secret file，再由 runtime manager 作为只读挂载注入容器。 |
| 网关凭据协议 | `gatewayAccessTokenRef` 只作为平台内部引用；真正给 runtime 使用的是启动时挂载的 secret file。 |
| 用户侧配置边界 | 普通用户不配置 provider key，不切换模型绑定，不接触上游凭据；用户侧只看到平台开放的模型服务结果。 |
| 管理员治理边界 | 管理员统一管理平台模型、provider 凭据、模型开关、usage summary 与用户状态。 |
| 统一入口 | 用户侧只使用 `browserUrl`；平台内部与 runtime manager 只使用 `internalEndpoint`。 |
| 路由策略 | Traefik 使用子域名，例如 `u-001.crewclaw.example.com`。 |
| browserUrl 安全模型 | `browserUrl` 不等于匿名可访问地址；所有 workspace 子域名统一经 Traefik + Authentik 前置鉴权。即使知道子域名，也不能绕过登录直接访问 runtime。 |
| 首次创建 binding | 首次启动时由模块 2 负责初始化 UserRuntimeBinding，并分配 `runtimeId`、`volumeId`、默认 `imageRef` 与默认 `retentionPolicy`；模块 3 只消费 binding 并负责编排。 |
| 禁用策略 | 除 `/api/v1/auth/me` 外，disabled 用户访问业务接口统一返回 `403 USER_DISABLED`；同时把 desiredState 收敛到 `stopped` 并立即使已签发 gateway token 失效。 |
| 删除语义 | 引入 `retentionPolicy = preserve_workspace / wipe_workspace`。MVP 默认 `preserve_workspace`。 |
| OpenClaw 本地 profile | runtime 保留本地 profile，但首次进入时由平台自动创建并建立映射。 |

## 4. UserRuntimeBinding 统一对象

本次修订中，UserRuntimeBinding 是必须同步更新的核心对象。它同时影响设计文档、总契约和总接口文档。

| 字段 | 说明 |
| ---- | ---- |
| runtimeId | 平台内部 runtime 唯一标识。 |
| volumeId | 该 runtime 绑定的持久化卷标识，用于 workspace / state / auth profiles 保留。 |
| imageRef | 当前使用的 runtime 镜像引用，可为 wrapper image 或固定官方镜像引用。 |
| desiredState | 目标状态：`running / stopped / deleted`。 |
| observedState | 观测状态：`creating / running / stopped / error / deleted`。 |
| browserUrl | 面向浏览器访问的地址，由 Traefik 子域名提供；访问时统一经过 Authentik 前置鉴权。 |
| internalEndpoint | 面向平台内部服务的容器地址或内部路由地址。 |
| retentionPolicy | 删除策略：`preserve_workspace / wipe_workspace`。 |
| lastError | 最近一次编排失败信息，供前端和后台展示。 |

## 5. runtime 启动闭环

1. 用户在工作台点击“启动 Runtime”，模块 6 调用模块 3 的 `ensure_running` 接口。
2. 模块 3 先向模块 2 调用“确保 binding 存在”的内部能力；若该用户尚无 UserRuntimeBinding，由模块 2 首次分配 `runtimeId`、`volumeId`、默认 `imageRef` 和默认 `retentionPolicy` 并落库。
3. 模块 3 从模块 2 读取 UserRuntimeBinding 和 quota，从模块 4 拉取该用户可用的**平台统一运行时模型配置**。
4. 模块 3 把 gateway-config 渲染为配置文件，并把网关访问令牌写入 secret file；两者存放在平台受控目录。
5. 模块 3 组装 runtime 启动模板，包括 `imageRef`、`volumeId`、`routeHost`、`configMount`、`retentionPolicy` 等参数。
6. runtime manager 接收启动模板，创建或启动容器，挂载 volume、只读配置文件、只读 secret file，并返回 `observedState` 与 `internalEndpoint`。
7. Traefik 为该 runtime 下发或激活对应子域名路由，并对该子域名启用 Authentik 前置鉴权；模块 3 回写 `browserUrl` 与 `internalEndpoint` 到模块 2。
8. 用户通过 `browserUrl` 进入工作区；首次进入时，平台在 runtime 内自动创建并映射本地 profile。

**实现约束**：平台主服务在生产架构中不直接操作 Docker socket。所有容器创建、停止、删除与挂载操作都通过 runtime manager 承担。

## 6. 入口、路由与访问地址

| 字段 | 使用对象 | 示例 | 说明 |
| ---- | -------- | ---- | ---- |
| browserUrl | 浏览器 / 前端 | `https://u-001.crewclaw.example.com` | 统一入口，由 Traefik 子域名路由暴露；访问时统一经过 Authentik 前置鉴权。 |
| internalEndpoint | 平台内部服务 / runtime manager | `http://crewclaw-u001:3000` | 容器网络或内部服务访问地址，不直接暴露给浏览器。 |
| routeHost | Traefik / runtime manager | `u-001.crewclaw.example.com` | 用于生成 `browserUrl` 和反代规则。 |

用户侧接口和工作区入口不得再直接返回内部 Docker 网络地址。前端只使用 `browserUrl`；`internalEndpoint` 只在内部接口、调度和诊断中出现。

### 6.1 browserUrl 安全模型

- `browserUrl` 只表示“浏览器入口地址”，不表示“匿名可访问”。
- 所有 `*.crewclaw.example.com` 的 workspace 请求先到 Traefik，再经过 Authentik 前置鉴权。
- 只有在 Authentik 会话有效、用户状态为 active、且该子域名与当前用户绑定关系合法时，才允许继续转发到对应 runtime。
- 因此，即使用户知道其他人的 `browserUrl`，也不能绕过平台登录直接访问对方 workspace。

## 7. 模型、凭据与用量设计

- OpenClaw runtime 一律通过 LiteLLM / CrewClaw 统一网关访问模型。
- 平台统一托管上游 provider 凭据；普通用户与 runtime 均不直接接触 provider 明文 API Key。
- 普通用户不管理模型绑定、不上传 provider 凭据；管理员负责平台模型开关、默认模型路由与 provider 凭据维护。
- runtime 在启动后通过挂载的配置文件知道 `baseUrl`、可见模型与默认路由；通过挂载的 secret file 使用网关访问令牌。
- MVP 展示口径以 OpenClaw 上报的 usage 为主；平台仅向管理员侧提供汇总口径，普通用户侧不展示 usage 统计。

## 8. 用户禁用、删除与收口语义

| 场景 | 平台动作 |
| ---- | -------- |
| 禁用用户 | 立即把 `desiredState` 收敛为 `stopped`；调用 runtime manager 停止容器；立即使已签发 gateway token 失效；除 `/api/v1/auth/me` 外，后续业务访问统一返回 `403 USER_DISABLED`，`/workspace-entry` 也不再返回 `ready=false`。 |
| 删除 runtime（preserve_workspace） | 删除容器和路由，保留 volume 与本地 profile 数据；再次启动时可恢复工作区。 |
| 删除 runtime（wipe_workspace） | 删除容器、路由和持久化 volume，清除工作区及 profile 数据。MVP 默认不作为默认动作。 |

## 9. 开发与部署方式

开发阶段可使用“容器化开发环境”，即开发工作在一个开发容器中完成，依赖、构建链路和调试工具都在容器里统一。开发容器不等于生产架构；生产仍采用平台主服务 + 独立 runtime manager 的职责边界。

- 允许在安全边界清晰的前提下，由 runtime manager 负责 Docker-in-Docker 或 Docker-outside-of-Docker 方案。
- 平台主服务本身不因开发便利而直接承担 Docker 控制职责。
- 构建完成后，开发容器中的产物即可沉淀为发布镜像；临时文件在发版前清理。

## 10. 调用故事（修订版）

### 10.1 首次登录并启动 runtime

1. 用户访问平台域名，Traefik 把请求导向控制面和 Authentik。
2. 模块 1 完成登录态校验，首次登录时调用 `/internal/users/sync`。
3. 模块 2 幂等创建用户并写入默认 `tenantId=t_default`。
4. 用户在工作台点击启动 runtime，模块 3 先向模块 2 确保 runtime binding 已存在；若不存在，则由模块 2 首次分配 `runtimeId`、`volumeId`、默认 `imageRef` 与默认 `retentionPolicy`。
5. 模块 3 获取 binding、统一模型配置与启动模板。
6. 模块 3 渲染配置文件与 secret file，并调用 runtime manager `ensure-running`。
7. runtime manager 拉起 runtime 容器、挂载 volume 与配置、回传 `internalEndpoint`。
8. Traefik 启用子域名路由并对该子域名开启 Authentik 前置鉴权，模块 3 回写 `browserUrl`。前端通过 `/workspace-entry` 获取 `browserUrl` 并跳转。

### 10.2 管理员调整模型服务

1. 管理员在后台维护 provider 凭据、模型开关与默认模型策略。
2. 模块 4 更新平台模型配置与网关路由策略。
3. 新启动的 runtime 直接消费新配置；已运行 runtime 可通过后续重启或配置刷新机制生效。
4. 普通用户无需执行任何模型配置动作，继续通过 OpenClaw 使用平台提供的模型服务。

### 10.3 用户禁用

1. 管理员在后台把用户状态改为 `disabled`。
2. 模块 2 更新用户状态并触发 runtime 收敛。
3. 模块 3 或后台治理任务把 `desiredState` 置为 `stopped`，runtime manager 停止容器。
4. 网关访问令牌立即失效；该用户除 `/api/v1/auth/me` 外的业务访问统一返回 `403 USER_DISABLED`。

## 11. MVP 之外的后续路线

- 复杂多租户治理与 `tenant_admin` 等 RBAC。
- 复杂共享空间及其权限模型。
- 用户侧高级模型自定义、个人 provider 凭据接入与自助绑定。
- 复杂 API Key 治理、策略中心、费用看板与财务结算。
- 自动扩缩容、跨机迁移、灰度升级。
- 必要时的 OpenClaw fork / patch 深度定制。


v 0.4
reno 
2026-03-16 14:04