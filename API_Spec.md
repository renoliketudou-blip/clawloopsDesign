# CrewClaw 平台 MVP 统一总接口

供前后端、平台服务与 runtime manager 统一联调使用。

| 文档定位 | 接口与字段基线                                               |
| -------- | ------------------------------------------------------------ |
| 适用范围 | 用户侧 / 管理员侧 / 内部服务侧 /runtime manager 内部接口     |
| 修订重点 | 删去对普通用户暴露的 gateway-config；补充 task 状态机；引入 browserUrl/internalEndpoint 与 retentionPolicy |
| 响应原则 | 所有响应均采用 JSON；字段名直接作为开发基线，不再自行改名    |

**接口修订摘要**：普通用户侧不再暴露 /api/v1/models/gateway-config；工作区入口改为返回 browserUrl；runtime binding 与  runtime-manager 启动接口新增  volumeId/imageRef/routeHost/configMount/retentionPolicy/lastError 等关键字段。

# 1. 通用响应约定

| 项         | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 成功状态码 | 读取接口默认 200；创建或异步动作默认 201 / 202。             |
| 错误体     | 统一至少包含 code 和 message 字段。                          |
| 异步任务   | runtime 启停删返回 taskId；前端通过 /api/v1/runtime/tasks/{taskId} 轮询。 |
| 任务状态   | pending / running / succeeded / failed / canceled。          |
| 权限约定   | 用户侧接口依赖当前登录用户；管理员侧接口仅 admin 可访问；internal 接口仅服务间访问。 |
| usage 口径 | MVP 展示以 OpenClaw 上报为主；后续审计 / 计费以网关记录为准。 |

# 2. 统一错误码

| HTTP    | code                     | 用途                     |
| ------- | ------------------------ | ------------------------ |
| 401     | UNAUTHENTICATED          | 未登录或 token 无效。    |
| 403     | ACCESS_DENIED            | 权限不足。               |
| 403     | USER_DISABLED            | 用户已禁用。             |
| 403     | CREDENTIAL_ACCESS_DENIED | 无权操作该凭据。         |
| 404     | USER_NOT_FOUND           | 用户不存在。             |
| 404     | RUNTIME_NOT_FOUND        | runtime 不存在。         |
| 404     | MODEL_NOT_FOUND          | 模型不存在。             |
| 404     | CREDENTIAL_NOT_FOUND     | 凭据不存在。             |
| 409     | RUNTIME_ACTION_CONFLICT  | runtime 正忙或状态冲突。 |
| 422     | QUOTA_EXCEEDED           | 超出 quota。             |
| 422     | CREDENTIAL_INVALID       | 凭据校验失败或不可用。   |
| 500/502 | *_ERROR                  | 内部模块或上游错误。     |

# 3. 接口总览（修订后）

| 方法   | 路径                                                | 用途                         | 权限        |
| ------ | --------------------------------------------------- | ---------------------------- | ----------- |
| GET    | /api/v1/auth/me                                     | 获取当前登录用户             | 用户        |
| GET    | /api/v1/auth/access                                 | 检查当前用户是否可访问业务   | 用户        |
| GET    | /api/v1/users/me/quota                              | 获取当前用户 quota           | 用户        |
| GET    | /api/v1/users/me/runtime                            | 获取当前用户 runtime binding | 用户        |
| POST   | /api/v1/users/me/runtime/start                      | 启动或创建 runtime           | 用户        |
| POST   | /api/v1/users/me/runtime/stop                       | 停止 runtime                 | 用户        |
| DELETE | /api/v1/users/me/runtime                            | 删除 runtime                 | 用户        |
| GET    | /api/v1/users/me/runtime/status                     | 查询当前用户 runtime 状态    | 用户        |
| GET    | /api/v1/runtime/tasks/{taskId}                      | 查询 runtime 任务状态        | 用户 /admin |
| GET    | /api/v1/models                                      | 获取当前用户可见模型列表     | 用户        |
| GET    | /api/v1/models/bindings                             | 获取当前用户模型绑定关系     | 用户        |
| PUT    | /api/v1/models/{modelId}/binding                    | 设置某模型的凭据绑定         | 用户        |
| GET    | /api/v1/credentials                                 | 获取当前用户凭据列表         | 用户        |
| POST   | /api/v1/credentials                                 | 新增用户凭据                 | 用户        |
| POST   | /api/v1/credentials/{credentialId}/verify           | 验证用户凭据                 | 用户        |
| DELETE | /api/v1/credentials/{credentialId}                  | 删除用户凭据                 | 用户        |
| GET    | /api/v1/usage/summary                               | 获取当前用户用量摘要         | 用户        |
| GET    | /api/v1/workspace-entry                             | 获取当前用户工作区入口       | 用户        |
| PATCH  | /api/v1/admin/users/{userId}/status                 | 启用 / 禁用用户              | admin       |
| GET    | /api/v1/admin/models                                | 获取全局模型列表与策略       | admin       |
| GET    | /api/v1/admin/usage/summary                         | 获取全局 usage 汇总          | admin       |
| POST   | /internal/users/sync                                | 首次登录同步 / 创建用户      | internal    |
| PUT    | /internal/users/{userId}/runtime-binding            | 创建或更新 runtime binding   | internal    |
| PATCH  | /internal/users/{userId}/runtime-binding/state      | 更新 runtime 状态            | internal    |
| GET    | /internal/model-config/users/{userId}               | 获取运行时模型配置           | internal    |
| POST   | /internal/usage/records                             | 接收 OpenClaw usage 上报     | internal    |
| POST   | /internal/runtime-manager/containers/ensure-running | 确保容器运行                 | internal    |
| POST   | /internal/runtime-manager/containers/stop           | 停止容器                     | internal    |
| POST   | /internal/runtime-manager/containers/delete         | 删除容器                     | internal    |
| GET    | /internal/runtime-manager/containers/{runtimeId}    | 查询容器状态                 | internal    |

# 4. 用户侧接口

## 4.1 获取当前登录用户

GET /api/v1/auth/me，返回 authenticated 与 user 信息，包含 userId/subjectId/tenantId/role/isAdmin/isDisabled。

## 4.2 获取当前用户 runtime binding

GET /api/v1/users/me/runtime。

| 字段            | 说明                            |
| --------------- | ------------------------------- |
| runtimeId       | runtime 唯一标识。              |
| volumeId        | 绑定的持久化卷。                |
| imageRef        | 当前镜像引用。                  |
| desiredState    | 目标状态。                      |
| observedState   | 当前观测状态。                  |
| browserUrl      | 浏览器入口；未 ready 时可为空。 |
| retentionPolicy | 删除策略。                      |
| lastError       | 最近错误信息。                  |

**示例**：{"userId":"u_001","runtime":{"runtimeId":"rt_001","volumeId":"vol_001","imageRef":"crewclaw-runtime-wrapper:openclaw-1.0.0","desiredState":"running","observedState":"running","browserUrl":"https://u-001.crewclaw.example.com","retentionPolicy":"preserve_workspace","lastError":null}}

## 4.3 启动或创建 runtime

POST /api/v1/users/me/runtime/start，返回 202 Accepted。若 runtime 已 running，可幂等成功。

**响应示例**：{"taskId":"rtask_001","action":"ensure_running","status":"accepted"}

## 4.4 停止与删除 runtime

POST /api/v1/users/me/runtime/stop 为幂等停止接口。DELETE /api/v1/users/me/runtime 用于删除当前用户 runtime。

删除接口请求体建议支持 retentionPolicy，未显式提供时使用 binding 中的 retentionPolicy；MVP 默认 preserve_workspace。

## 4.5 查询 runtime 任务状态

GET /api/v1/runtime/tasks/{taskId}。

| 字段      | 说明                                                |
| --------- | --------------------------------------------------- |
| taskId    | 任务标识。                                          |
| userId    | 所属用户。                                          |
| runtimeId | 所属 runtime。                                      |
| action    | ensure_running / stop / delete。                    |
| status    | pending / running / succeeded / failed / canceled。 |
| message   | 任务说明。                                          |

## 4.6 获取模型列表与绑定

GET /api/v1/models 返回当前用户可见模型列表及绑定来源。GET /api/v1/models/bindings 返回模型与凭据的实际绑定关系。

**修订点**：普通用户侧不再提供 /api/v1/models/gateway-config；runtime 所需 gateway-config 改由 internal 接口提供给模块 3。

## 4.7 凭据与用量摘要

- GET /api/v1/credentials：只返回凭据元数据，不返回明文 key。
- POST /api/v1/credentials：创建时 secret 仅在请求体中出现一次，落入 secret store 后不再返回。
- POST /api/v1/credentials/{credentialId}/verify：返回 verified/status 与 lastValidatedAt。
- GET /api/v1/usage/summary：展示当前用户 usage summary。

## 4.8 获取工作区入口

GET /api/v1/workspace-entry。

| 字段       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| ready      | 当前用户是否可以进入工作区。                                 |
| runtimeId  | runtime 标识。                                               |
| browserUrl | 前端跳转目标。                                               |
| reason     | 当 ready=false 时，可为 runtime_not_found /runtime_not_running/runtime_starting /user_disabled。 |

**响应示例**：{"ready":true,"runtimeId":"rt_001","browserUrl":"https://u-001.crewclaw.example.com"}

# 5. 管理员侧接口

- PATCH /api/v1/admin/users/{userId}/status：status 仅允许 active/disabled；修改后应立即影响前台访问控制。
- GET /api/v1/admin/users/{userId}/runtime：可查看 runtime 的 desiredState/observedState/browserUrl/internalEndpoint/lastError。
- GET /api/v1/admin/users/{userId}/credentials：仅返回凭据元数据和状态。
- GET /api/v1/admin/usage/summary：提供平台 usage 汇总，不等于费用结算。

# 6. 内部服务接口

## 6.1 同步 / 创建用户

POST /internal/users/sync，仍以 subjectId 幂等；默认 tenantId=t_default，role=user。

## 6.2 创建或更新 runtime binding

PUT /internal/users/{userId}/runtime-binding。

**请求示例**：{"runtimeId":"rt_001","volumeId":"vol_001","imageRef":"crewclaw-runtime-wrapper:openclaw-1.0.0","desiredState":"running","observedState":"creating","browserUrl":null,"internalEndpoint":null,"retentionPolicy":"preserve_workspace","lastError":null}

## 6.3 更新 runtime binding 状态

PATCH /internal/users/{userId}/runtime-binding/state。

**请求示例**：{"desiredState":"running","observedState":"running","browserUrl":"https://u-001.crewclaw.example.com","internalEndpoint":"http://crewclaw-u001:3000","lastError":null}

## 6.4 获取运行时模型配置

GET /internal/model-config/users/{userId}，仅 internal 可用。

| 字段                  | 说明                                  |
| --------------------- | ------------------------------------- |
| baseUrl               | LiteLLM 或 CrewClaw 网关地址。        |
| models                | 当前用户可用模型列表。                |
| gatewayAccessTokenRef | 平台内部引用，不直接给 runtime 使用。 |
| configRenderVersion   | 本次渲染版本号，用于排障和重试。      |

模块 3 拿到该配置后，必须渲染为配置文件和 secret file，再通过 runtime manager 挂载给容器。

## 6.5 接收 OpenClaw usage 上报

POST /internal/usage/records，MVP 主口径使用 reportedBy=openclaw 的 token 数据。

# 7. Runtime Manager 内部接口

## 7.1 确保容器运行

POST /internal/runtime-manager/containers/ensure-running。

| 字段                       | 说明                                         |
| -------------------------- | -------------------------------------------- |
| userId                     | 所属用户。                                   |
| runtimeId                  | runtime 标识。                               |
| imageRef                   | 要启动的 runtime 镜像。                      |
| volumeId                   | 要挂载的持久化卷。                           |
| routeHost                  | Traefik 子域名主机名。                       |
| configMount.configFilePath | 模块 3 预渲染的模型配置文件路径。            |
| configMount.secretFilePath | 模块 3 预渲染的网关 token secret file 路径。 |
| retentionPolicy            | 删除策略。                                   |

**请求示例**：{"userId":"u_001","runtimeId":"rt_001","imageRef":"crewclaw-runtime-wrapper:openclaw-1.0.0","volumeId":"vol_001","routeHost":"[u-001.crewclaw.example.com](https://u-001.crewclaw.example.com)","configMount":{"configFilePath":"/var/lib/crewclaw/runtime-configs/u_001/model-gateway.json","secretFilePath":"/var/lib/crewclaw/runtime-secrets/u_001/gateway.token"},"retentionPolicy":"preserve_workspace"}

**成功示例**：{"runtimeId":"rt_001","observedState":"creating","internalEndpoint":"http://crewclaw-u001:3000","message":"creating"}

## 7.2 停止、删除、查询容器

- POST /internal/runtime-manager/containers/stop：停止指定 runtime 容器。
- POST /internal/runtime-manager/containers/delete：删除容器，必要时按 retentionPolicy 清理 volume。
- GET /internal/runtime-manager/containers/{runtimeId}：返回 observedState/internalEndpoint 与 message。

# 8. 前端联调建议

- 工作台首页初始化顺序：/auth/me -> /users/me/runtime/status -> /models -> /credentials -> /usage/summary。
- 点击启动 runtime：POST /users/me/runtime/start -> 轮询 /runtime/tasks/{taskId} -> 刷新 /users/me/runtime/status。
- 进入工作区前先调 /workspace-entry；ready=true 才跳转到 browserUrl。
- 新增凭据后，先 /verify，再通过 /models/{modelId}/binding 切换。
- 管理后台初始化：/admin/users//admin/models//admin/usage/summary；用户详情页再补 /admin/users/{userId} 与 /admin/users/{userId}/credentials。

# 9. 字段冻结清单

| 禁止行为     | 冻结要求                                                     |
| ------------ | ------------------------------------------------------------ |
| 命名漂移     | 不要把 userId 改成 uid，不要把 runtimeId 改成 containerId。  |
| 状态合并     | 不要把 observedState 和 desiredState 混成单一 status。       |
| 地址混用     | 不要再用一个 endpoint 同时表示 browserUrl 和 internalEndpoint。 |
| 凭据误解     | gatewayAccessTokenRef 是内部引用，不是上游 provider key，也不是直接返回给浏览器的 token。 |
| 删除语义漂移 | 删除 runtime 时必须遵守 retentionPolicy，不得在无说明情况下清空工作区。 |


v 0.1