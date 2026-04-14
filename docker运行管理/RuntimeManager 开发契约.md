# RuntimeManager 开发契约（V2.2 后端渲染配置修订）

## 0. 本次修订摘要

本版把 RuntimeManager 从“参与启动引导与配置生成”收敛为“**后端渲染完整配置，RM 只写盘、挂载并启动 runtime**”。

核心变化只有 5 条：

- LiteLLM 改为**平台后端共享服务**，不再跟每个 user runtime 一起启动
- Orchestrator 在 `ensure-running` 前**渲染完整 `openclaw.json`**
- RM 只负责把上游提供的配置写入标准路径并挂载
- 用户不再需要执行 `docker compose run --rm openclaw-cli onboard`
- runtime 固定容器内监听 `18789`，默认**不直接暴露宿主机端口**

---

## 1. 定位

RuntimeManager（RM）是**内部容器执行层**，只负责：

- 创建 / 启动 / 停止 / 删除 OpenClaw runtime 容器
- 执行宿主机目录初始化与权限修正
- 把上游提供的完整 `openclaw.json` 写入宿主机配置目录
- 挂载用户目录与配置目录
- 把 runtime 接入平台共享网络
- 返回**当前观测状态**与稳定内部地址
- 检测关键 contract drift 并显式返回冲突

它**不负责**：

- 不生成 `runtimeId / volumeId`
- 不访问业务数据库验证 `runtimeId` 是否真实存在
- 不创建外层异步任务，不维护用户可见任务状态机
- 不决定 `workspace / role / quota`
- 不管理 Traefik 路由
- 不负责启动 LiteLLM 进程；LiteLLM 属于平台后端共享服务
- 不渲染业务配置模板，不理解 token 生命周期，只把上游配置视为不透明内容
- 不暴露给浏览器或公网域名；浏览器入口统一由平台网关负责

---

## 2. 上下游契约

### 2.1 上游：Orchestrator 必须提供

V2.2 中，RM 的 internal API 请求体由 Orchestrator 下发，**不接受调用方覆盖 `imageRef` / `command` / `networkName` / 容器内端口**。

Orchestrator 必须提供：

- `userId`
- `runtimeId`
- `volumeId`
- `routeHost`
- `retentionPolicy`（本次请求的 **effectiveRetentionPolicy**）
- `compat.openclawConfigDir`
- `compat.openclawWorkspaceDir`
- `renderedConfig.openclawJson`
- `renderedConfig.configVersion`

### 2.2 下游：RuntimeManager 必须负责

- 基于 label 识别受管容器
- 在宿主机上执行 `mkdir/chown/chmod` 等 `init-perms` 等价动作
- 把上游提供的完整 `openclaw.json` 原样写盘
- 创建并启动容器
- 挂载目录与配置
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

## 3. 必须保留的启动兼容契约

这是 V2.2 的唯一真相；若其他文档与本节冲突，以本节为准。

### 3.1 镜像与启动命令冻结

OpenClaw runtime 镜像固定为：

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
- runtime 一律以 `lan + 18789` 的组合启动，不再做每用户容器内端口差异化

### 3.2 容器内目录结构冻结

必须保证容器内可用目录为：

- `/home/node/.openclaw`
- `/home/node/.openclaw/workspace`
- `/home/node/.openclaw/canvas`
- `/home/node/.openclaw/cron`

### 3.3 权限初始化与配置生成冻结

启动前必须执行与 `init-perms` 等价的宿主机动作：

- 创建目标目录
- `chown -R 1000:1000 <openclawConfigDir>`
- `chmod -R u+rwX,g+rwX <openclawConfigDir>`
- 若 `openclawWorkspaceDir` 与 `openclawConfigDir` 非同一路径，也要对 workspace 路径执行同等处理

冻结说明：

- 这些动作在**容器创建前、直接针对宿主机挂载目录执行**
- 不通过 OpenClaw 容器内脚本完成
- 不依赖 sidecar
- `openclaw.json` 的写盘发生在权限初始化之后、容器创建之前
- 若 RM 对目标 host path 不具备权限，则 `ensure-running` 失败并返回 `RUNTIME_START_FAILED`

### 3.4 环境变量兼容要求

RM 必须保证下列基础环境变量存在：

- `HOME=/home/node`
- `TERM=xterm-256color`
- `TZ=UTC`
- `OPENAI_BASE_URL=http://litellm:4000`

冻结说明：

- `OPENAI_BASE_URL` 固定为 `http://litellm:4000`，**不允许请求方覆盖**
- gateway token 由 Orchestrator 负责生成、持久化并渲染进完整 `openclaw.json`
- gateway token 的生命周期默认为**长期有效**，直到用户被删除或平台显式轮换
- RM 不解析、校验、推导 token，只负责写入上游已渲染好的配置内容
- 正常链路不再要求调用方手工准备 `openclaw.json`
- 正常链路不再要求用户手工执行 `openclaw onboard`

### 3.5 网络契约冻结

统一共享网络固定为：

```text
clawloops_shared
```

最低必须接入该网络的容器：

- `traefik`
- `clawloops-api`
- `runtime-manager`
- `litellm`
- per-user runtime

冻结结果：

- 原 `compat.networkName` 从必填删除
- RM 不再接受上游通过请求体切换网络名
- LiteLLM 必须由平台后端共享 compose / stack 启动，不应由 per-user runtime compose 单独启动
- RM 只负责把 runtime 接入 `clawloops_shared`，不负责创建 LiteLLM 本身

### 3.6 端口、network alias 与 `internalEndpoint` 冻结

冻结规则：

- `18789`：唯一主服务端口，也是唯一 readiness 判定端口
- `18790`：兼容保留端口，不作为 readiness 条件，也不要求宿主机映射
- 容器内监听端口固定为 `18789`
- 稳定通信别名固定为：`rt-<runtimeId>`
- 稳定内部地址固定为：`http://rt-<runtimeId>:18789`

补充说明：

- 容器查找**靠 label**，不是靠容器名
- 容器通信**靠固定 network alias**
- `containerName` 是实现细节，不是契约字段
- 若 `runtimeId` 自身已包含业务前缀，也不做二次规范化；直接生成 `rt-<runtimeId>`
- 浏览器入口统一走平台 `browserUrl` / workspace 子域名
- 若部署层未来需要额外启用调试直连端口，应作为**非默认运维能力**单独设计，不属于 V2.2 主契约

---

## 4. 挂载与配置契约

### 4.1 `volumeId` 的语义冻结

`volumeId` 是**平台逻辑卷 ID**，用于：

- 业务数据库记录
- label 追踪
- 平台侧审计 / 排障

RM **不**根据 `volumeId` 猜测宿主机路径。

真正的宿主机路径由 Orchestrator / Binding 层解析，并通过 `compat` 下发给 RM。

### 4.2 `compat` 字段冻结

`compat` 在 `ensure-running` 请求中**必填**。

#### 冻结字段

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `openclawConfigDir` | 是 | 宿主机上的配置根目录，对应容器 `/home/node/.openclaw` |
| `openclawWorkspaceDir` | 是 | 宿主机上的 workspace 目录，对应容器 `/home/node/.openclaw/workspace` |

明确删除：

- `networkName`
- `gatewayPort`
- `configMount`
- `secretFilePath`

### 4.3 双挂载目录模型冻结

V2.2 继续支持“双挂载”：

- `openclawConfigDir -> /home/node/.openclaw`
- `openclawWorkspaceDir -> /home/node/.openclaw/workspace`

目录边界冻结如下：

- `workspace`：独立数据目录
- `canvas / cron / config`：都归 `openclawConfigDir`
- `wipe_workspace` 时的删除范围必须按第 7 节矩阵执行，不能模糊外扩

### 4.4 平台托管 `openclaw.json` 固定落点

V2.2 的标准配置文件落点固定为：

| 产物 | 容器内落点 | 生成方 |
| --- | --- | --- |
| `openclaw.json` | `/home/node/.openclaw/openclaw.json` | Orchestrator 渲染，RuntimeManager 写盘 |

冻结说明：

- 上游不再要求预先复制 `openclaw.json`
- 用户不再需要手工执行 `docker compose run --rm openclaw-cli onboard`
- RM 必须在每次 `ensure-running` 前校验并写入上游渲染结果
- 对平台托管 runtime 而言，`openclaw.json` 由 RM 视为**受管文件**

### 4.5 配置渲染与写盘规则

配置解释权与渲染执行权属于 **Orchestrator / 后端配置渲染层**，RM 只负责写盘与挂载。

冻结规则：

- Orchestrator 必须渲染完整、可直接启动的 `openclaw.json`
- 渲染结果中必须包含 gateway token、固定 `gateway.bind=lan`、固定 `gateway.port=18789`
- 渲染结果中必须把模型代理地址指向 `http://litellm:4000`
- RM 必须把收到的 `renderedConfig.openclawJson` 写入 `compat.openclawConfigDir`
- RM 不再要求额外的 `configMount` / `secretFilePath`
- RM 不允许对上游配置内容做业务级重写，只允许做落盘前的格式校验与原子写入

#### 配置优先级（机器可执行规则）

| 顺位 | 来源 |
| --- | --- |
| 1 | 本次 request 的 `renderedConfig.openclawJson` |
| 2 | 受管目录中的既有 `openclaw.json`（仅允许被同一 `configVersion` 覆盖或替换） |
| 3 | 否则启动失败 |

### 4.6 token 生命周期冻结

gateway token 的平台语义冻结为：

- 由 Orchestrator / 后端生成并持久化
- 默认长期有效，不做短期自动轮换
- 删除用户时必须同步失效
- 如需主动轮换，由 Orchestrator 生成新 token、重渲染 `openclaw.json` 并触发 runtime 重建或重启
- RM 不得在日志、错误消息、label 中输出 token 明文

### 4.7 公共区域挂载策略（增量能力）

本节用于并入公共区域能力，不改变 RM 在 runtime V2.2 中的执行边界。

冻结规则：

- 宿主机公共区域根目录固定为 `/var/lib/clawloops/shared/public/files/`
- `user runtime` 启动时：RM 需将公共区域复制到用户容器私有副本目录
- `admin runtime` 启动时：RM 也需将公共区域复制到管理员容器私有副本目录
- 无论 `user/admin`，容器内（OpenClaw）对公共区域副本的变更都不得回写宿主机
- 宿主机公共区写入影响仅允许通过工作台管理员管理接口产生，不由 RM 容器副本链路产生

实现边界：

- 公共区域复制/挂载属于容器启动准备动作
- RM 仍不承担公共区域业务权限判断（权限判断在 `clawloops-api`）
- 该能力不改变 RM 同步接口语义与错误码主集合

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

V2.2 明确：

- `18789` 是唯一必检端口
- `18790` 仅保留兼容，不作为 readiness 条件
- `18790` 不要求宿主机映射
- 不要求额外暴露为 `internalEndpoint`

---

## 6. contract drift 处理

### 6.1 漂移总原则

若容器已存在，但关键 contract 与 V2.2 基线不一致，RM **不得自行重建**。

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
- `renderedConfig.configVersion`
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
- `clawloops.configVersion=<configVersion>`

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
| 500/502 | `RUNTIME_START_FAILED` | 创建、权限初始化、配置写盘、启动探测失败 |
| 500 | `RUNTIME_STOP_FAILED` | 停止容器失败 |
| 500 | `RUNTIME_DELETE_FAILED` | 删除容器或清理目录失败 |

---

# RuntimeManager API 文档（V2.2 后端渲染配置修订）

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
  "retentionPolicy": "preserve_workspace",
  "compat": {
    "openclawConfigDir": "/var/lib/clawloops/users/u_001/config",
    "openclawWorkspaceDir": "/var/lib/clawloops/users/u_001/workspace"
  },
  "renderedConfig": {
    "configVersion": "cfg_u001_v1",
    "openclawJson": {
      "gateway": {
        "bind": "lan",
        "port": 18789,
        "auth": {
          "mode": "token",
          "token": "<redacted>"
        }
      },
      "models": {
        "providers": {
          "litellm": {
            "baseUrl": "http://litellm:4000"
          }
        }
      }
    }
  }
}
```

### 字段说明

| 字段 | 说明 |
| --- | --- |
| `compat` | **必填**；RM 不再猜目录 |
| `renderedConfig.openclawJson` | **必填**；由 Orchestrator 渲染的完整配置内容 |
| `renderedConfig.configVersion` | **必填**；用于 drift 判断与审计 |
| `imageRef` | **已从请求体删除**；由 Orchestrator 服务端固定 |

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
  "message": "failed to prepare config or start container"
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

v2.2-runtime-rendered-config  
reno  
2026-03-27
