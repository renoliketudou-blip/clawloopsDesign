# CrewClaw 平台设计文档

基于用户已确认的取舍，对原始设计文档进行统一重构后的版本。

| 文档定位 | 系统架构与 MVP 落地设计                                      |
| -------- | ------------------------------------------------------------ |
| 适用范围 | 单机部署、多用户访问、容器级隔离、统一模型网关               |
| 本版重点 | 收紧 MVP 边界、补齐 runtime 启动闭环、拆分 browserUrl 与 internalEndpoint |
| 关联文档 | 《MVP 开发基线总契约》《MVP 统一总接口》《开源与涨星策略》   |

### 本次修订摘要

MVP 明确不追求复杂多租户和复杂共享空间 UI；UserRuntimeBinding 增加  volumeId、imageRef、browserUrl、internalEndpoint、retentionPolicy、lastError；runtime 启动改为模块 3 先渲染配置文件与 secret file，再交由 runtime manager 挂载；统一入口采用 Traefik  子域名路由。

## 1. 目标与范围

目标是在一台服务器上部署公司内部 CrewClaw 平台，使 OpenClaw 能以团队平台方式运行：支持多用户访问、账号密码登录、容器级隔离、统一模型接入、个人空间、最小凭据托管闭环以及用量摘要。

MVP 不追求复杂多租户能力，不做复杂共享空间 UI，不实现复杂套餐、财务结算和复杂策略中心。这些能力保留到后续路线图。

| 范围     | MVP 结论                                                     |
| -------- | ------------------------------------------------------------ |
| 多租户   | 仅保留 tenantId 这一基础字段，MVP 默认固定为 t_default，不做复杂租户治理。 |
| 共享空间 | MVP 不提供复杂共享空间 UI；只保留后续扩展接口空间。          |
| 隔离边界 | 采用每用户一个 runtime 容器的容器级隔离，不宣称强安全沙箱级隔离。 |
| 模型治理 | 提供统一网关、默认模型、用户自有凭据托管、最小 usage summary。 |
| 费用管理 | MVP 不做费用看板与复杂计费，仅保留 usage summary。           |

## 2. 整体架构

- 统一入口：Traefik，采用子域名路由，把浏览器入口和内部服务地址分开管理。
- 统一认证：Authentik，平台负责会话鉴权与身份上下文注入。
- 平台层：控制面 API + 极简 Web UI，负责用户、状态、资源真相与管理后台。
- 运行时：每用户一个 OpenClaw runtime 容器。
- 生命周期管理：独立 runtime manager /provisioner，负责容器创建、停止、删除、挂载与状态查询。
- 模型网关：LiteLLM + PostgreSQL，统一接公司模型、本地模型和用户自有凭据。
- 本地模型：vLLM 为主，Ollama 作为补充。
- 密钥管理：MVP 使用 secret file 注入；后续可接 OpenBao。

| 层级     | 核心组件                               | 职责                                                         |
| -------- | -------------------------------------- | ------------------------------------------------------------ |
| 入口层   | Traefik + Authentik                    | 统一外部访问、登录、会话识别与转发。                         |
| 平台层   | CrewClaw 控制面                        | 用户同步、UserRuntimeBinding 管理、管理后台、工作台。        |
| 编排层   | Runtime Orchestrator + Runtime Manager | 将 desiredState 落为容器实际状态。                           |
| 运行时层 | Per-user OpenClaw runtime              | 用户自己的 workspace、state、auth profiles 与 agent runtime。 |
| 模型层   | LiteLLM + PostgreSQL + vLLM/Ollama     | 统一模型出口、默认模型、凭据代理与 usage 归集。              |

## 3. 关键设计决策

| 主题                  | 修订后的确定方案                                             |
| --------------------- | ------------------------------------------------------------ |
| 多租户 / 共享空间     | MVP 暂不追求复杂多租户能力，不做复杂共享空间 UI。            |
| 仓库策略              | 不纳入 OpenClaw 主仓源码；固定官方镜像 tag；允许基于官方镜像构建 runtime wrapper image，或通过 entrypoint/sidecar/runtime manager 注入配置与卷挂载；fork/patch 仅作为预案。 |
| runtime 配置注入      | 采用方案 A：模块 3 先渲染模型网关配置文件与 secret file，再由 runtime manager 作为只读挂载注入容器。 |
| 网关凭据协议          | gatewayAccessTokenRef 只作为平台内部引用；真正给 runtime 使用的是启动时挂载的 secret file。 |
| 统一入口              | 用户侧只使用 browserUrl；平台内部与 runtime manager 只使用 internalEndpoint。 |
| 路由策略              | Traefik 使用子域名，例如 [u-001.crewclaw.example.com](https://u-001.crewclaw.example.com)。 |
| 禁用策略              | 禁用用户后立即将 desiredState 收敛到 stopped，并立即使已签发 token 失效。 |
| 删除语义              | 引入 retentionPolicy = preserve_workspace /wipe_workspace。MVP 默认 preserve_workspace。 |
| OpenClaw 本地 profile | runtime 保留本地 profile，但首次进入时由平台自动创建并建立映射。 |

## 4. UserRuntimeBinding 统一对象

本次修订中，UserRuntimeBinding 是必须同步更新的核心对象。它同时影响设计文档、总契约和总接口文档。

| 字段             | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| runtimeId        | 平台内部 runtime 唯一标识。                                  |
| volumeId         | 该 runtime 绑定的持久化卷标识，用于 workspace/state/auth profiles 保留。 |
| imageRef         | 当前使用的 runtime 镜像引用，可为 wrapper image 或固定官方镜像引用。 |
| desiredState     | 目标状态：running /stopped/deleted。                         |
| observedState    | 观测状态：creating /running/stopped /error/deleted。         |
| browserUrl       | 面向浏览器访问的地址，由 Traefik 子域名提供。                |
| internalEndpoint | 面向平台内部服务的容器地址或内部路由地址。                   |
| retentionPolicy  | 删除策略：preserve_workspace /wipe_workspace。               |
| lastError        | 最近一次编排失败信息，供前端和后台展示。                     |

## 5. runtime 启动闭环

1. 用户在工作台点击 “启动 Runtime”，模块 6 调用模块 3 的 ensure_running 接口。
2. 模块 3 从模块 2 读取 UserRuntimeBinding 和 quota，从模块 4 拉取运行时模型配置。
3. 模块 3 把 gateway-config 渲染为配置文件，并把网关访问令牌写入 secret file；两者存放在平台受控目录。
4. 模块 3 组装 runtime 启动模板，包括 imageRef、volumeId、routeHost、configMount、retentionPolicy 等参数。
5. runtime manager 接收启动模板，创建或启动容器，挂载 volume、只读配置文件、只读 secret file，并返回 observedState 与 internalEndpoint。
6. Traefik 为该 runtime 下发或激活对应子域名路由，模块 3 回写 browserUrl 与 internalEndpoint 到模块 2。
7. 用户通过 browserUrl 进入工作区；首次进入时，平台在 runtime 内自动创建并映射本地 profile。

**实现约束**：平台主服务在生产架构中不直接操作 Docker socket。所有容器创建、停止、删除与挂载操作都通过 runtime manager 承担。

## 6. 入口、路由与访问地址

| 字段             | 使用对象                      | 示例                                                         | 说明                                             |
| ---------------- | ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| browserUrl       | 浏览器 / 前端                 | https://u-001.crewclaw.example.com                           | 统一入口，由 Traefik 子域名路由暴露。            |
| internalEndpoint | 平台内部服务 /runtime manager | http://crewclaw-u001:3000                                    | 容器网络或内部服务访问地址，不直接暴露给浏览器。 |
| routeHost        | Traefik / runtime manager     | [u-001.crewclaw.example.com](https://u-001.crewclaw.example.com) | 用于生成 browserUrl 和反代规则。                 |

用户侧接口和工作区入口不得再直接返回内部 Docker 网络地址。前端只使用 browserUrl；internalEndpoint 只在内部接口、调度和诊断中出现。

## 7. 凭据与模型访问设计

- OpenClaw runtime 一律通过 LiteLLM / CrewClaw 统一网关访问模型。
- MVP 允许平台默认模型与用户自有 provider 凭据并存，但不向用户或 runtime 暴露上游 provider 明文 API Key。
- runtime 在启动后通过挂载的配置文件知道 baseUrl、可见模型与默认路由；通过挂载的 secret file 使用网关访问令牌。
- MVP 展示口径以 OpenClaw 上报的 usage 为主；后续审计与计费口径以网关日志为准。

## 8. 用户禁用、删除与收口语义

| 场景                               | 平台动作                                                     |
| ---------------------------------- | ------------------------------------------------------------ |
| 禁用用户                           | 立即把 desiredState 收敛为 stopped；调用 runtime manager 停止容器；立即使已签发 gateway token 失效；后续业务访问统一返回 403 USER_DISABLED。 |
| 删除 runtime（preserve_workspace） | 删除容器和路由，保留 volume 与本地 profile 数据；再次启动时可恢复工作区。 |
| 删除 runtime（wipe_workspace）     | 删除容器、路由和持久化 volume，清除工作区及 profile 数据。MVP 默认不作为默认动作。 |

## 9. 开发与部署方式

开发阶段可使用 “容器化开发环境”，即开发工作在一个开发容器中完成，依赖、构建链路和调试工具都在容器里统一。开发容器不等于生产架构；生产仍采用平台主服务 + 独立 runtime manager 的职责边界。

- 允许在安全边界清晰的前提下，由 runtime manager 负责 Docker-in-Docker 或 Docker-outside-of-Docker 方案。
- 平台主服务本身不因开发便利而直接承担 Docker 控制职责。
- 构建完成后，开发容器中的产物即可沉淀为发布镜像；临时文件在发版前清理。

## 10. 调用故事（修订版）

### 10.1 首次登录并启动 runtime

1. 用户访问平台域名，Traefik 把请求导向控制面和 Authentik。
2. 模块 1 完成登录态校验，首次登录时调用 /internal/users/sync。
3. 模块 2 幂等创建用户并写入默认 tenantId=t_default。
4. 用户在工作台点击启动 runtime，模块 3 获取 binding、模型配置与启动模板。
5. 模块 3 渲染配置文件与 secret file，并调用 runtime manager ensure-running。
6. runtime manager 拉起 runtime 容器、挂载 volume 与配置、回传 internalEndpoint。
7. Traefik 启用子域名路由，模块 3 回写 browserUrl。前端通过 /workspace-entry 获取 browserUrl 并跳转。

### 10.2 用户禁用

1. 管理员在后台把用户状态改为 disabled。
2. 模块 2 更新用户状态并触发 runtime 收敛。
3. 模块 3 或后台治理任务把 desiredState 置为 stopped，runtime manager 停止容器。
4. 网关访问令牌立即失效；该用户后续访问平台统一返回 403 USER_DISABLED。

## 11. MVP 之外的后续路线

- 复杂多租户治理与 tenant_admin 等 RBAC。
- 复杂共享空间及其权限模型。
- 复杂 API Key 治理、策略中心、费用看板与财务结算。
- 自动扩缩容、跨机迁移、灰度升级。
- 必要时的 OpenClaw fork/patch 深度定制。