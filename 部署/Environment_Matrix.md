# CrewClaw 环境矩阵

## 1. 目的

本文件用于统一发布、自测和排障时的环境口径，避免域名、Cookie、Traefik、共享网络、模型网关和运行时入口写成多套说法。

---

## 2. 当前仓库里的环境事实

### 2.1 Compose 默认变量

来源：`infra/compose/.env.example`

| 变量 | 默认值 | 用途 | 发布前要求 |
| --- | --- | --- | --- |
| `CLAWLOOPS_DOMAIN` | `clawloops.localhost` | 主站域名 | 与实际 Traefik 路由保持一致 |
| `RUNTIME_MANAGER_DOMAIN` | `runtime-manager.clawloops.localhost` | Runtime Manager 域名 | 与实际 Traefik 路由保持一致 |
| `RUNTIME_PUBLIC_HOST` | `localhost` | runtime 对外暴露主机 | 仅用于当前实现，不应替代正式入口 |
| `LITELLM_MASTER_KEY` | `sk-local-master` | API 与 LiteLLM 的共享密钥 | 生产环境必须替换 |
| `CLAWLOOPS_MODEL_GATEWAY_BASE_URL` | `http://litellm:4000` | 模型网关地址 | 与共享网络服务名一致 |
| `CLAWLOOPS_MODEL_GATEWAY_DEFAULT_MODELS` | `qwen-max-proxy` | 默认模型 | 必须与 `litellm.config.yaml` 保持一致 |
| `DASHSCOPE_API_KEY` | 空 | 模型供应商密钥 | 本地联调和发布前必填 |

### 2.2 Compose 服务

来源：`infra/compose/docker-compose.yml`

| 服务 | 端口 / 地址 | 所在网络 | 备注 |
| --- | --- | --- | --- |
| `traefik` | `80` / `443` / `8080` | `clawloops_shared` | 当前启用 insecure dashboard |
| `clawloops-web` | 容器内 `3000` | `clawloops_shared` | 主站 UI |
| `clawloops-api` | 容器内 `8000` | `clawloops_shared` | `/api` 入口 |
| `runtime-manager` | 容器内 `18080` | `clawloops_shared` | 依赖 Docker socket 和 `/var/lib/clawloops` |
| `litellm` | 容器内 `4000` | `clawloops_shared` | 平台共享模型网关 |

### 2.3 Runtime 固定契约

来源：`docs/docker运行管理/RuntimeManager 开发契约.md`

| 项 | 固定值 | 说明 |
| --- | --- | --- |
| 共享网络 | `clawloops_shared` | 不允许按请求动态切换 |
| runtime readiness 端口 | `18789` | 唯一必检端口 |
| runtime 内部地址 | `http://rt-<runtimeId>:18789` | 平台内部通信使用 |
| 模型网关地址 | `http://litellm:4000` | runtime 固定消费 |

---

## 3. 推荐环境矩阵

### 3.1 本地开发 / 联调环境

| 项 | 推荐值 |
| --- | --- |
| 主域 | `clawloops.localhost` |
| Runtime Manager 域名 | `runtime-manager.clawloops.localhost` |
| 协议 | `http` |
| Cookie `Secure` | `false` |
| Cookie `Domain` | 可留空或按本地方案配置 |
| 主要目标 | 快速启动、联调、人工冒烟 |

### 3.2 预发布 / 生产环境

| 项 | 推荐值 |
| --- | --- |
| 主域 | `clawloops.example.com` |
| workspace 子域 | `<routeHost>.clawloops.example.com` |
| Runtime Manager 域名 | `runtime-manager.clawloops.example.com` |
| 协议 | `https` |
| Cookie `Secure` | `true` |
| Cookie `Domain` | `.clawloops.example.com` |
| 主要目标 | 正式发布、受保护 workspace 入口 |

---

## 4. 域名与入口矩阵

| 场景 | 正确口径 | 不应使用 |
| --- | --- | --- |
| 平台主站 | `https://clawloops.example.com` | 与 workspace 使用完全不同主域 |
| 工作区入口 | `https://ws-001.clawloops.example.com` | `http://localhost:<hostPort>` 作为正式入口 |
| Runtime Manager | `https://runtime-manager.clawloops.example.com` | 混用多套域名别名却不更新文档 |
| 内部 runtime 地址 | `http://rt-<runtimeId>:18789` | 提供给浏览器直接访问 |
| 模型网关地址 | `http://litellm:4000` | 在 runtime 中写死其他宿主机地址 |

---

## 5. Cookie 与鉴权矩阵

来源：`docs/后端/Architecture_Design.md`、`docs/部署/Deployment_Prerequisites.md`

| 项 | 本地联调 | 预发布 / 生产 |
| --- | --- | --- |
| Cookie 名称 | `clawloops_session` | `clawloops_session` |
| `HttpOnly` | `true` | `true` |
| `SameSite` | `Lax` | `Lax` |
| `Secure` | `false` 或按本地 HTTPS 配置 | `true` |
| `Path` | `/` | `/` |
| `Domain` | 视本地域名策略而定 | 必须覆盖主域与 workspace 子域 |
| workspace 鉴权 | 仍应经过平台 session | 必须经过平台 session |

---

## 6. Traefik 矩阵

### 6.1 当前仓库状态

| 配置位置 | 当前事实 | 风险 |
| --- | --- | --- |
| `infra/traefik/static/traefik.yml` | 只启用 `file` provider | Compose labels 目前不是唯一真相 |
| `infra/compose/docker-compose.yml` | 同时写了 Traefik labels | 容易让团队误以为 labels 已生效 |
| `infra/traefik/dynamic/middlewares.yml` | 当前写死 `192.168.0.99.nip.io` / `192.168.0.99` | 与 `.env.example` 的 `.localhost` 口径不一致 |

### 6.2 发布前必须统一

| 检查项 | 要求 |
| --- | --- |
| 生效 provider | 团队必须明确当前是 `file` provider 还是 Docker provider 真正生效 |
| 主站域名 | 必须与 `CLAWLOOPS_DOMAIN` 一致 |
| Runtime Manager 域名 | 必须与 `RUNTIME_MANAGER_DOMAIN` 一致 |
| workspace 鉴权 | 必须统一接入 `ForwardAuth` |
| 透传头 | 必须保留原始 cookie、`Host`、`X-Forwarded-*` |

---

## 7. 安装与启动体验矩阵

| 项 | 当前仓库口径 | 发布前建议 |
| --- | --- | --- |
| 主启动方式 | `docker compose up -d --build` | 保持为唯一推荐入口 |
| 环境变量来源 | `infra/compose/.env` | 所有关键变量必须在 `.env.example` 有说明 |
| 本地域名说明 | `infra/compose/README.md` | 与 Traefik 动态路由示例保持一致 |
| 前端构建 | `pnpm` + `vite build` | 文档中明确 node / pnpm 要求 |
| 后端测试 | `pytest` | 作为发布门禁之一固定下来 |

---

## 8. 发布前需要人工确认的差异

以下差异若存在，必须在测试记录中明确写出：

| 差异 | 影响 |
| --- | --- |
| `.env` 使用 `.localhost`，Traefik 动态路由仍是 `nip.io` | 主站与 API 可能访问不到正确路由 |
| `browserUrl` 仍返回 hostPort | workspace 入口与主设计不一致 |
| Cookie 只能在主域可见 | workspace 子域无法复用登录态 |
| LiteLLM 模型名与默认模型不一致 | 模型链路直接失败 |

---

## 9. 推荐使用方式

每次发版时，先填本文件中的实际值，再执行：

1. `docs/部署/Release_Checklist.md`
2. `apps/clawloops-api/TESTING.md`
3. `docs/测试/Test_Run_Log.md`

如果环境矩阵与部署实际不一致，应先修正文档或配置，再进行发布测试。
