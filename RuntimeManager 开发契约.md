# RuntimeManager 开发契约（V1）

## 1. 定位

runtimeManager 是 **内部容器执行层**，只负责：

- 创建 / 启动 / 停止 / 删除 OpenClaw runtime 容器
- 挂载用户目录、配置文件、secret
- 连接共享网络
- 返回容器状态和内部访问地址

它**不负责**：

- 不生成 `runtimeId / volumeId`
- 不决定模型列表
- 不保存业务真相
- 不管理普通用户配置
- 不直接暴露给浏览器

------

## 2. 上下游契约

### 上游：模块 3 / Orchestrator 必须提供

- `userId`
- `runtimeId`
- `imageRef`
- `volumeId`
- `routeHost`
- `configMount.configFilePath`
- `configMount.secretFilePath`
- `retentionPolicy`

### 下游：runtimeManager 必须负责

- Docker 容器存在性检查
- 目录初始化与权限修正
- 挂载目录和配置
- 启动容器
- 停止容器
- 删除容器
- 查询容器状态

------

## 3. 第一版必须保留的启动兼容契约

这是根据你现有**已验证能启动**的配置提炼出来的，V1 不要改坏。

### 3.1 镜像与启动命令冻结

runtimeManager 创建 OpenClaw 容器时，必须支持：

- 镜像：`ghcr.io/openclaw/openclaw@sha256:a5a4c83b773aca85a8ba99cf155f09afa33946c0aa5cc6a9ccb6162738b5da02`
- 启动命令：

```
node dist/index.js gateway --bind lan --port 18789
```

### 3.2 容器内目录结构冻结

必须保证容器内存在：

- `/home/node/.openclaw`
- `/home/node/.openclaw/workspace`
- `/home/node/.openclaw/canvas`
- `/home/node/.openclaw/cron`

### 3.3 权限初始化冻结

启动前必须执行与 `init-perms` 等价的动作：

- 创建上述目录
- `chown -R 1000:1000 /home/node/.openclaw`
- `chmod -R u+rwX,g+rwX /home/node/.openclaw`

也就是说，**runtimeManager 要内建 `init-perms` 逻辑**。

### 3.4 环境变量兼容要求

第一版至少支持这些环境变量：

- `HOME=/home/node`
- `TERM=xterm-256color`
- `TZ=UTC`
- `OPENAI_BASE_URL=http://litellm:4000`

兼容支持：

- `OPENCLAW_GATEWAY_TOKEN`
- `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS`
- `CLAUDE_AI_SESSION_KEY`
- `CLAUDE_WEB_SESSION_KEY`
- `CLAUDE_WEB_COOKIE`

其中真正必须保留的是：

- `HOME`
- `TERM`
- `TZ`
- `OPENAI_BASE_URL`

### 3.5 网络契约冻结

OpenClaw runtime 容器必须加入：

- `crewclaw_shared`

原因很简单：你现在 `openclaw.json` 和环境变量都依赖容器内通过 `http://litellm:4000` 访问 LiteLLM。

### 3.6 端口契约冻结

当前已验证端口是：

- `18789`：gateway 主端口
- `18790`：bridge/辅助端口

所以 runtimeManager 第一版应以：

- `internalEndpoint = http://<containerName>:18789`

作为返回值。

**不要再返回 3000。**
 你现有文档里的 `3000` 示例要改成 `18789`，否则会和真实可启动配置冲突。

------

## 4. 挂载契约

根据你当前配置，第一版应兼容这两种挂载：

### 必需挂载

- `OPENCLAW_CONFIG_DIR -> /home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR -> /home/node/.openclaw/workspace`

### 额外挂载

- `configMount.configFilePath -> 只读挂载到容器`
- `configMount.secretFilePath -> 只读挂载到容器`

### V1 建议

为了不破坏你已验证方案，runtimeManager 第一版先继续支持“双挂载”：

- 配置根目录挂 `.openclaw`
- workspace 单独挂 `.openclaw/workspace`

后续再收敛成单 volume root。

------

## 5. 配置优先级契约

你现在的 OpenClaw 启动有两类配置来源：

### A. 持久化目录中的 `openclaw.json`

里面已经包含：

- LiteLLM provider
- models
- gateway.port / bind
- gateway.auth.token

### B. 容器环境变量

里面包含：

- `OPENAI_BASE_URL`
- `OPENCLAW_GATEWAY_TOKEN`
- `CLAUDE_*`

所以 runtimeManager 第一版要遵守这个原则：

### 原则

**不主动改写用户持久化目录里的 `openclaw.json`，只负责把平台渲染好的配置挂进去或注入环境变量。**

### 建议优先级

- 若提供 `secretFilePath`，优先把 token 注入为 `OPENCLAW_GATEWAY_TOKEN`
- 若未提供，则依赖现有 `openclaw.json.gateway.auth.token`
- 若两者都没有，视为启动条件不足

------

## 6. 状态契约

runtimeManager 对外只暴露这些状态：

- `creating`
- `running`
- `stopped`
- `error`
- `deleted`

### 状态判断规则

- 容器不存在：`deleted`
- 容器已创建但未稳定：`creating`
- 容器运行中且 18789 可用：`running`
- 容器已停止：`stopped`
- 容器异常退出或启动失败：`error`

------

## 7. 删除契约

删除时必须遵守：

- `preserve_workspace`
- `wipe_workspace`

### 行为

- `preserve_workspace`：删容器，不删用户目录
- `wipe_workspace`：删容器，也删用户数据目录

------

## 8. 容器识别契约

runtimeManager 不能靠“猜名字”找容器，必须靠 label。

建议每个 runtime 容器带上：

- `crewclaw.managed=true`
- `crewclaw.userId=<userId>`
- `crewclaw.runtimeId=<runtimeId>`
- `crewclaw.volumeId=<volumeId>`
- `crewclaw.routeHost=<routeHost>`
- `crewclaw.retentionPolicy=<retentionPolicy>`

------

# RuntimeManager API 文档（V1）

所有接口均为 **internal only**。

统一响应为 JSON。

------

## 1. 确保容器运行

### `POST /internal/runtime-manager/containers/ensure-running`

### 用途

- 若容器不存在，则创建并启动
- 若容器存在但停止，则启动
- 若容器已运行，则幂等返回

### 请求体

```
{
  "userId": "u_001",
  "runtimeId": "rt_001",
  "imageRef": "ghcr.io/openclaw/openclaw@sha256:a5a4c83b773aca85a8ba99cf155f09afa33946c0aa5cc6a9ccb6162738b5da02",
  "volumeId": "vol_001",
  "routeHost": "u-001.crewclaw.example.com",
  "configMount": {
    "configFilePath": "/var/lib/crewclaw/runtime-configs/u_001/openclaw.json",
    "secretFilePath": "/var/lib/crewclaw/runtime-secrets/u_001/gateway.token"
  },
  "retentionPolicy": "preserve_workspace",
  "compat": {
    "openclawConfigDir": "/var/lib/crewclaw/users/u_001/config",
    "openclawWorkspaceDir": "/var/lib/crewclaw/users/u_001/workspace",
    "networkName": "crewclaw_shared",
    "gatewayPort": 18789,
    "bridgePort": 18790
  }
}
```

### 字段说明

- `compat`：这是我建议你补充的字段。因为你现有启动依赖目录结构和端口，不建议 runtimeManager 自己瞎猜。

### 成功响应

```
{
  "runtimeId": "rt_001",
  "observedState": "creating",
  "internalEndpoint": "http://crewclaw-rt-u_001:18789",
  "message": "creating"
}
```

### 幂等成功响应

```
{
  "runtimeId": "rt_001",
  "observedState": "running",
  "internalEndpoint": "http://crewclaw-rt-u_001:18789",
  "message": "already running"
}
```

### 失败响应

```
{
  "code": "RUNTIME_MANAGER_ERROR",
  "message": "failed to create container"
}
```

------

## 2. 停止容器

### `POST /internal/runtime-manager/containers/stop`

### 请求体

```
{
  "userId": "u_001",
  "runtimeId": "rt_001"
}
```

### 成功响应

```
{
  "runtimeId": "rt_001",
  "observedState": "stopped",
  "message": "stopped"
}
```

### 幂等响应

如果容器已不存在或已停止，也返回成功：

```
{
  "runtimeId": "rt_001",
  "observedState": "stopped",
  "message": "already stopped"
}
```

------

## 3. 删除容器

### `POST /internal/runtime-manager/containers/delete`

### 请求体

```
{
  "userId": "u_001",
  "runtimeId": "rt_001",
  "retentionPolicy": "preserve_workspace"
}
```

### 成功响应

```
{
  "runtimeId": "rt_001",
  "observedState": "deleted",
  "message": "deleted"
}
```

### 说明

- `preserve_workspace`：只删容器
- `wipe_workspace`：删容器并删目录

------

## 4. 查询容器状态

### `GET /internal/runtime-manager/containers/{runtimeId}`

### 成功响应

```
{
  "runtimeId": "rt_001",
  "observedState": "running",
  "internalEndpoint": "http://crewclaw-rt-u_001:18789",
  "message": "ok"
}
```

### 未找到

```
{
  "code": "RUNTIME_NOT_FOUND",
  "message": "runtime not found"
}
```

------

## 5. 健康检查

### `GET /healthz`

```
{
  "status": "ok"
}
```

### `GET /readyz`

```
{
  "status": "ready"
}
```