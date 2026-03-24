
# RuntimeManager 开发契约（V1 冻结修订）

## 1. 定位

RuntimeManager（RM）是**内部容器执行层**，只负责：

- 创建 / 启动 / 停止 / 删除 OpenClaw runtime 容器
- 执行宿主机目录初始化与权限修正
- 挂载用户目录、配置文件、secret 文件
- 把 runtime 接入平台共享网络
- 返回**当前观测状态**与稳定的内部访问地址
- 检测关键 contract drift 并显式返回冲突

它**不负责**：

- 不生成 `runtimeId / volumeId`
- 不访问业务数据库验证 `runtimeId` 是否真实存在
- 不创建外层异步任务，不维护用户可见任务状态机
- 不决定 `workspace / role / quota`
- 不管理 Traefik 路由
- 不解析 `secretFilePath` 内容，也不把 secret 文件自动转成环境变量
- 不暴露给浏览器或公网

---

## 2. 上下游契约

### 2.1 上游：Orchestrator 必须提供

V1 中，RM 的 internal API 请求体由 Orchestrator 下发，**不接受调用方覆盖 `imageRef`**。

Orchestrator 必须提供：

- `userId`
- `runtimeId`
- `volumeId`
- `routeHost`
- `retentionPolicy`（本次请求的 **effectiveRetentionPolicy**）
- `compat.openclawConfigDir`
- `compat.openclawWorkspaceDir`
- `configMount`（可选增强挂载）
- `env`（可选）
- `envOverrides`（可选）

### 2.2 下游：RuntimeManager 必须负责

- 基于 label 识别受管容器
- 在宿主机上执行 `mkdir/chown/chmod` 等 `init-perms` 等价动作
- 创建并启动容器
- 挂载目录与只读配置/secret 文件
- 接入共享网络并绑定固定 network alias
- 同步停止 / 删除容器
- 查询容器当前事实状态
- 在关键配置漂移时返回 `409 RUNTIME_CONTRACT_DRIFT`

### 2.3 外层任务边界

冻结规则：

- **Orchestrator 对用户侧接口仍可异步返回 `taskId`**
- **RuntimeManager 内部接口全部为同步执行**
- RM 立即返回当前 `observedState` / `message` / `internalEndpoint`
- RM 不创建第二套异步 task 状态机，避免与 Orchestrator 双重编排

---

## 3. 第一版必须保留的启动兼容契约

这是 V1 的唯一真相；若其他文档与本节冲突，以本节为准。

### 3.1 镜像与启动命令冻结

V1 的 OpenClaw runtime 镜像固定为：

```text
ghcr.io/openclaw/openclaw@sha256:a5a4c83b773aca85a8ba99cf155f09afa33946c0aa5cc6a9ccb6162738b5da02
```

启动命令固定为：

```text
node dist/index.js gateway --bind lan --port 18789
```

冻结说明：

- `imageRef` 由 **Orchestrator 的服务端配置**固定，并隐式作用于 RM
- RM **不接受调用方通过请求体覆盖 `imageRef`**
- `command` 也不接受调用方覆盖

### 3.2 容器内目录结构冻结

必须保证容器内可用目录为：

- `/home/node/.openclaw`
- `/home/node/.openclaw/workspace`
- `/home/node/.openclaw/canvas`
- `/home/node/.openclaw/cron`

### 3.3 权限初始化冻结

启动前必须执行与 `init-perms` 等价的宿主机动作：

- 创建目标目录
- `chown -R 1000:1000 <openclawConfigDir>`
- `chmod -R u+rwX,g+rwX <openclawConfigDir>`
- 若 `openclawWorkspaceDir` 与 `openclawConfigDir` 非同一路径，也要对 workspace 路径执行同等处理

冻结说明：

- 这些动作在**容器创建前、直接针对宿主机挂载目录执行**
- 不通过 OpenClaw 容器内脚本完成
- 不依赖 sidecar
- 若 RM 对目标 host path 不具备权限，则 `ensure-running` 失败并返回 `RUNTIME_START_FAILED`

### 3.4 环境变量兼容要求

RM 必须保证下列基础环境变量存在：

- `HOME=/home/node`
- `TERM=xterm-256color`
- `TZ=UTC`
- `OPENAI_BASE_URL=http://litellm:4000`

兼容支持的显式下发环境变量包括：

- `OPENCLAW_GATEWAY_TOKEN`
- `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS`
- `CLAUDE_AI_SESSION_KEY`
- `CLAUDE_WEB_SESSION_KEY`
- `CLAUDE_WEB_COOKIE`

冻结说明：

- `OPENAI_BASE_URL` 在 V1 固定为 `http://litellm:4000`，**不允许请求方覆盖**
- 部署层必须保证 LiteLLM 在 `clawloops_shared` 上的可解析名称为 `litellm`
- RM 只负责把 runtime 接入该网络，不负责创建或修复 LiteLLM 别名

### 3.5 网络契约冻结

V1 的统一共享网络固定为：

```text
clawloops_shared
```

首版最低要求：

- `Traefik`
- `ClawLoops API`
- `RuntimeManager`
- `LiteLLM`
- `per-user runtime`

都必须在 `clawloops_shared` 上互通。

关于 Authentik：

- **不是要求 Authentik 全家桶所有容器都加入同一网络**
- 最低要求是与受保护应用链路相关、需要互通的组件可达
- 通常至少保证 `authentik-proxy-outpost` 与 Traefik / 受保护应用网络可达

冻结结果：

- 原 `compat.networkName` 从必填删除
- RM 不再接受上游通过请求体切换网络名
- V1 只连 `clawloops_shared`

### 3.6 端口、network alias 与 `internalEndpoint` 冻结

冻结规则：

- `18789`：唯一主服务端口，也是唯一 readiness 判定端口
- `18790`：兼容保留端口，不作为 readiness 条件，不要求 `internalEndpoint` 使用
- 稳定通信别名固定为：`rt-<runtimeId>`
- 稳定内部地址固定为：`http://rt-<runtimeId>:18789`

补充说明：

- 容器查找**靠 label**，不是靠容器名
- 容器通信**靠固定 network alias**
- `containerName` 是实现细节，不是契约字段
- 若 `runtimeId` 自身已包含业务前缀，也不做二次规范化；直接生成 `rt-<runtimeId>`

---

## 4. 挂载契约

### 4.1 `volumeId` 的语义冻结

`volumeId` 是**平台逻辑卷 ID**，用于：

- 业务数据库记录
- label 追踪
- 平台侧审计 / 排障

RM **不**根据 `volumeId` 猜测宿主机路径。

真正的宿主机路径由 Orchestrator / Binding 层解析，并通过 `compat` 下发给 RM。

### 4.2 `compat` 字段冻结

`compat` 在 V1 的 `ensure-running` 请求中**必填**。

#### 冻结字段

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `openclawConfigDir` | 是 | 宿主机上的配置根目录，对应容器 `/home/node/.openclaw` |
| `openclawWorkspaceDir` | 是 | 宿主机上的 workspace 目录，对应容器 `/home/node/.openclaw/workspace` |


明确删除：

- `networkName`
- `gatewayPort`

### 4.3 双挂载目录模型冻结

V1 继续支持“双挂载”：

- `openclawConfigDir -> /home/node/.openclaw`
- `openclawWorkspaceDir -> /home/node/.openclaw/workspace`

目录边界冻结如下：

- `workspace`：独立数据目录
- `canvas / cron / config`：都归 `openclawConfigDir`
- `wipe_workspace` 时的删除范围必须按第 7 节矩阵执行，不能模糊外扩

### 4.4 `configMount` 固定挂载点

`configMount` 是可选增强挂载，**不替代 `compat`**。

#### 固定容器内落点

| 输入字段 | 容器内落点 | 权限 |
| --- | --- | --- |
| `configMount.configFilePath` | `/home/node/.openclaw/openclaw.json` | `:ro` |
| `configMount.secretFilePath` | `/run/clawloops/secrets/gateway.token` | `:ro` |

冻结说明：

- V1 容器内标准配置文件落点固定为 `/home/node/.openclaw/openclaw.json`
- 上游不保留容器内文件名自定义能力

### 4.5 配置与 secret 注入规则

配置解释权属于 **OpenClaw / 上游渲染层**，不属于 RM。

RM 的行为冻结为：

- 只保证文件被挂载到固定位置
- 只把上游显式提供的 `env` / `envOverrides` 注入容器
- **不会**读取 `secretFilePath` 内容并自动转成环境变量
- `OPENCLAW_GATEWAY_TOKEN` 若要走 env，值必须由上游显式提供

#### 配置优先级（机器可执行规则）

| 顺位 | 来源 |
| --- | --- |
| 1 | 上游显式下发的 `env` / `envOverrides` |
| 2 | 本次 request 指定并挂载的 `configMount` 文件 |
| 3 | 持久化目录中既有配置 |
| 4 | 否则启动失败 |

---

## 5. 状态契约与启动窗口

### 5.1 对外状态

RM 对外只暴露以下 `observedState`：

- `creating`
- `running`
- `stopped`
- `error`
- `deleted`

### 5.2 状态判断规则

- 容器不存在：`deleted`
- 容器已创建但未稳定：`creating`
- 容器运行中且 **18789** 在启动窗口内达成可用：`running`
- 容器已停止：`stopped`
- 容器异常退出、启动失败、启动窗口超时：`error`

### 5.3 启动成功时间窗

RM 内部启动判定冻结为：

- `30s grace`
- `1s poll`
- `3 consecutive successes`（针对 18789）

超过启动窗口仍未达成 3 次连续成功时：

- `ensure-running` 返回 `observedState=error`
- 错误码为 `RUNTIME_START_FAILED`

### 5.4 `18790` 的语义

V1 明确：

- `18789` 是唯一必检端口
- `18790` 仅保留兼容，不作为 readiness 条件
- 不要求额外暴露为 `internalEndpoint`

---

## 6. contract drift 处理

### 6.1 漂移总原则

若容器已存在，但关键 contract 与 V1 基线不一致，RM **不得自行重建**。

RM 必须：

- 检测关键漂移项
- 返回 `409 RUNTIME_CONTRACT_DRIFT`
- 由 Orchestrator 决定是否执行 `stop + delete + recreate`

### 6.2 关键漂移项冻结清单

以下任一项不一致，即视为关键 drift：

- 固定 `imageRef`
- 固定 `command`
- `clawloops_shared` 网络接入
- 固定 `networkAlias`
- 必需挂载的 source / target
- 必需环境变量
- 必需 labels
- `gateway` 主端口（18789）

### 6.3 非关键漂移项

以下变化**不属于关键 drift**：

- `routeHost` 变化

`routeHost` 在 RM 中只用于 label 追踪：

- 可在后续 `ensure-running` 时更新 label
- 或保留原值
- **不得因此触发重建**

---

## 7. 删除语义、幂等矩阵与删除范围

### 7.1 retentionPolicy 优先级

删除的最终语义由 Orchestrator 先计算出本次 **effectiveRetentionPolicy**，再传给 RM。

RM 的规则冻结为：

- 只认本次请求体中的 `retentionPolicy`
- 不回查数据库真相
- 不与历史 binding 做二次合并

### 7.2 幂等矩阵

| 操作 | 容器不存在 | 容器已停止 | 容器运行中 |
| --- | --- | --- | --- |
| `ensure-running` | 创建并启动 | 启动 | 返回 `running` / `already running` |
| `stop` | 返回 `stopped` / `already stopped` | 返回 `stopped` / `already stopped` | 停止并返回 `stopped` |
| `delete` | 返回 `deleted` / `already deleted` | 删除并返回 `deleted` | 删除并返回 `deleted` |

### 7.3 `wipe_workspace` 删除矩阵

| retentionPolicy | 删除容器 | 删除 `openclawConfigDir` | 删除 `openclawWorkspaceDir` |
| --- | --- | --- | --- |
| `preserve_workspace` | 是 | 否 | 否 |
| `wipe_workspace` | 是 | 是 | 是 |

补充规则：

- 若 `openclawConfigDir` 与 `openclawWorkspaceDir` 存在父子重叠，按**去重后的根路径集合**执行删除
- 不允许越过请求给定的根路径向上删除
- `canvas / cron / config` 视为 `openclawConfigDir` 的一部分
- `workspace` 视为独立数据目录

---

## 8. 容器识别、查询与唯一性契约

### 8.1 唯一性

总契约冻结为：

- `runtimeId` 在平台范围内**全局唯一**

### 8.2 受管容器 label

每个 runtime 容器必须带上：

- `clawloops.managed=true`
- `clawloops.userId=<userId>`
- `clawloops.runtimeId=<runtimeId>`
- `clawloops.volumeId=<volumeId>`
- `clawloops.routeHost=<routeHost>`
- `clawloops.retentionPolicy=<retentionPolicy>`

### 8.3 查询规则

RM 只基于容器事实返回状态：

- 只按 label `clawloops.runtimeId=<runtimeId>` 查询
- 不访问业务数据库
- 不验证 `runtimeId` 是否存在于平台真相

冻结结果：

- `GET /containers/{runtimeId}` 找不到容器时，返回 `200 + observedState=deleted`
- 若同一 `runtimeId` 匹配到多个受管容器，返回 `409 RUNTIME_ACTION_CONFLICT`
- RM 不做隐式选择

---

## 9. 错误码与 HTTP 映射

| HTTP | code | 说明 |
| --- | --- | --- |
| 409 | `RUNTIME_CONTRACT_DRIFT` | 已有容器与冻结 contract 不一致 |
| 409 | `RUNTIME_ACTION_CONFLICT` | 同一 `runtimeId` 命中多个受管容器，或当前动作冲突 |
| 500/502 | `RUNTIME_START_FAILED` | 创建、权限初始化、启动探测失败 |
| 500 | `RUNTIME_STOP_FAILED` | 停止容器失败 |
| 500 | `RUNTIME_DELETE_FAILED` | 删除容器或清理目录失败 |

---

# RuntimeManager API 文档（V1 冻结修订）

所有接口均为 **internal only**，并且全部是**同步接口**。

统一响应为 JSON。

---

## 1. 确保容器运行

### `POST /internal/runtime-manager/containers/ensure-running`

### 用途

- 若容器不存在，则创建并启动
- 若容器存在但停止，则启动
- 若容器已运行且 contract 未漂移，则幂等返回
- 若容器已存在但关键 contract 漂移，则返回 `409 RUNTIME_CONTRACT_DRIFT`

### 请求体

```json
{
  "userId": "u_001",
  "runtimeId": "rt_001",
  "volumeId": "vol_001",
  "routeHost": "u-001.clawloops.example.com",
  "configMount": {
    "configFilePath": "/var/lib/clawloops/runtime-configs/u_001/openclaw.json",
    "secretFilePath": "/var/lib/clawloops/runtime-secrets/u_001/gateway.token"
  },
  "retentionPolicy": "preserve_workspace",
  "compat": {
    "openclawConfigDir": "/var/lib/clawloops/users/u_001/config",
    "openclawWorkspaceDir": "/var/lib/clawloops/users/u_001/workspace",
  },
  "env": {
    "OPENCLAW_GATEWAY_TOKEN": "<redacted>"
  },
  "envOverrides": {
    "OPENCLAW_ALLOW_INSECURE_PRIVATE_WS": "true"
  }
}
```

### 字段说明

| 字段 | 说明 |
| --- | --- |
| `compat` | **必填**；RM 不再猜目录、端口或网络 |
| `configMount` | 可选增强挂载；不替代 `compat` |
| `env` / `envOverrides` | 仅对显式下发值负责注入；RM 不解析 secret 文件内容 |
| `imageRef` | **已从 V1 请求体删除**；由 Orchestrator 服务端固定 |

### 成功响应（创建中）

```json
{
  "runtimeId": "rt_001",
  "observedState": "creating",
  "internalEndpoint": "http://rt-rt_001:18789",
  "message": "creating"
}
```

### 幂等成功响应（已运行）

```json
{
  "runtimeId": "rt_001",
  "observedState": "running",
  "internalEndpoint": "http://rt-rt_001:18789",
  "message": "already running"
}
```

### 漂移响应

```json
{
  "code": "RUNTIME_CONTRACT_DRIFT",
  "message": "existing container contract drift detected"
}
```

### 启动失败响应

```json
{
  "code": "RUNTIME_START_FAILED",
  "message": "failed to create or start container"
}
```

---

## 2. 停止容器

### `POST /internal/runtime-manager/containers/stop`

### 请求体

```json
{
  "userId": "u_001",
  "runtimeId": "rt_001"
}
```

### 成功响应

```json
{
  "runtimeId": "rt_001",
  "observedState": "stopped",
  "message": "stopped"
}
```

### 幂等响应

若容器已不存在或已停止，也返回成功：

```json
{
  "runtimeId": "rt_001",
  "observedState": "stopped",
  "message": "already stopped"
}
```

### 失败响应

```json
{
  "code": "RUNTIME_STOP_FAILED",
  "message": "failed to stop container"
}
```

---

## 3. 删除容器

### `POST /internal/runtime-manager/containers/delete`

### 请求体

```json
{
  "userId": "u_001",
  "runtimeId": "rt_001",
  "retentionPolicy": "preserve_workspace"
}
```

### 成功响应

```json
{
  "runtimeId": "rt_001",
  "observedState": "deleted",
  "message": "deleted"
}
```

### 幂等响应

若容器已不存在，也返回成功：

```json
{
  "runtimeId": "rt_001",
  "observedState": "deleted",
  "message": "already deleted"
}
```

### 说明

- `preserve_workspace`：删容器，不删目录
- `wipe_workspace`：删容器，并按第 7.3 节矩阵删除目录

### 失败响应

```json
{
  "code": "RUNTIME_DELETE_FAILED",
  "message": "failed to delete container or cleanup directories"
}
```

---

## 4. 查询容器状态

### `GET /internal/runtime-manager/containers/{runtimeId}`

### 成功响应（运行中）

```json
{
  "runtimeId": "rt_001",
  "observedState": "running",
  "internalEndpoint": "http://rt-rt_001:18789",
  "message": "ok"
}
```

### 成功响应（已删除或不存在容器事实）

```json
{
  "runtimeId": "rt_001",
  "observedState": "deleted",
  "internalEndpoint": null,
  "message": "not found as container fact"
}
```

### 冲突响应（命中多个容器）

```json
{
  "code": "RUNTIME_ACTION_CONFLICT",
  "message": "multiple managed containers matched the same runtimeId"
}
```

---

## 5. 健康检查

### `GET /healthz`

```json
{
  "status": "healthy"
}
```

### `GET /readyz`

```json
{
  "status": "ready"
}
```

说明：

- `/readyz` **仅表示 RuntimeManager 服务本身可用**
- **不代表任何单个 runtime 已 ready**

---

v2.0-runtime-frozen  
reno  
2026-03-23