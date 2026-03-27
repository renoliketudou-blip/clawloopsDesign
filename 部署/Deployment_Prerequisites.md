# ClawLoops 部署前置条件清单

## 0. 文档定位

这份清单用于在平台上线前一次性核对以下前置条件是否已经满足：

- 域名与子域规划
- session cookie 可见范围
- Traefik `ForwardAuth` 接入方式
- Traefik 需要透传的请求头
- 平台共享网络与关键服务连通性

本清单的目标不是重复描述业务流程，而是把“部署层必须先配对，否则系统即使代码正确也会表现异常”的前提一次写全。

适用范围：

- `traefik`
- `clawloops-web`
- `clawloops-api`
- `runtime-manager`
- `litellm`
- per-user runtime

---

## 1. 一页结论

在当前 MVP 中，workspace 正式访问链路固定为：

```text
浏览器 -> Traefik -> ForwardAuth -> clawloops-api -> per-user runtime
```

要让这条链路稳定工作，必须同时满足以下 5 条：

1. 主域与 workspace 子域属于同一可共享 session 的域名体系
2. 平台签发的 `clawloops_session` cookie 对主域与 workspace 子域都可见
3. Traefik 对所有 workspace 子域统一启用 `ForwardAuth`
4. Traefik 向 `GET /internal/auth/workspace-access` 原样透传约定头与原始 cookie
5. `traefik`、`clawloops-api`、`runtime-manager`、`litellm`、per-user runtime 全部加入 `clawloops_shared`

只要其中任一项不成立，就可能出现以下问题：

- 主站可登录，但 workspace 子域始终 `401`
- workspace URL 能打开，但绕过了平台 session 保护
- runtime 已运行，但浏览器仍无法进入
- runtime 容器启动成功，但无法访问 `litellm`

---

## 2. 域名与入口前置条件

### 2.1 主域与 workspace 子域

推荐使用同一主域下的两类入口：

- 平台主域：`clawloops.example.com`
- workspace 子域：`<routeHost>.clawloops.example.com`

冻结要求：

- 主域与 workspace 子域必须属于同一顶级可控域名体系
- 不允许把平台主站放在一个域名、workspace 放在另一个完全独立域名
- `browserUrl` 只能指向受平台网关保护的 workspace 子域
- 不允许把 runtime 宿主机端口直链地址作为正式 `browserUrl`

推荐示例：

```text
主站: https://clawloops.example.com
工作区: https://ws-001.clawloops.example.com
```

不推荐示例：

```text
主站: https://clawloops.example.com
工作区: https://ws-001.other-domain.com
```

原因：

- session cookie 无法自然跨域共享
- `ForwardAuth` 与 workspace 保护链路会变得脆弱
- 登录态、退出态与强制改密态更容易出现不一致

### 2.2 `browserUrl` 约束

`browserUrl` 只表示浏览器入口，不表示匿名可访问。

部署前必须确认：

- `browserUrl` 指向的是 Traefik 暴露的 workspace 子域
- `browserUrl` 对应的域名会先经过 `ForwardAuth`
- `browserUrl` 不附加 token、ticket、`userId` 或其他鉴权参数
- runtime 默认不直接暴露宿主机端口作为正式入口

---

## 3. TLS 与 Cookie 前置条件

### 3.1 基础要求

平台 session cookie 固定为：

- 名称：`clawloops_session`
- `HttpOnly=true`
- 生产环境 `Secure=true`
- `SameSite=Lax`
- `Path=/`

部署前必须确认：

- 主域与 workspace 子域都走 HTTPS
- 生产环境证书同时覆盖主域与 workspace 子域
- 生产环境 cookie `Domain` 能覆盖主域与 workspace 子域，例如 `.clawloops.example.com`

### 3.2 Cookie Domain 检查清单

以下条件必须同时成立：

- 登录接口签发 cookie 时使用统一的 `Domain`
- invitation 接受成功后写 cookie 时使用同一套 `Domain / Path / SameSite / Secure`
- 强制改密后轮换 session 时仍使用同一套 cookie 属性
- logout 清 cookie 时复用完全一致的 `Domain / Path`

常见错误：

- 登录页可用，但 workspace 子域不带 cookie
- 登录成功时未设置 `Domain`，导致 cookie 只在主域可见
- logout 清 cookie 时属性不一致，导致浏览器残留旧 session
- 某些接口写 cookie 时 `SameSite` 或 `Secure` 与其他接口不一致

### 3.3 域名示例必须统一

所有部署配置、示例 compose、Traefik 路由、运维说明、API 示例中的域名口径应统一。

建议在一次部署中只使用一套示例：

- `clawloops.example.com`

不要在同一套部署说明中混用：

- `clawloops.example.com`
- `clawloops.app`

否则很容易把 cookie `Domain`、证书或 DNS 配错。

---

## 4. ForwardAuth 前置条件

### 4.1 统一接入要求

所有 workspace 子域都必须在 Traefik 统一接入 `ForwardAuth`，并指向：

```text
GET /internal/auth/workspace-access
```

冻结要求：

- `ForwardAuth` 只保护 workspace 子域，不要求保护平台主站的所有页面
- 所有 workspace 子域都必须走同一放行判断接口
- 不允许某些 workspace 子域绕过 `ForwardAuth` 直连 runtime
- `workspace-access` 只返回允许或拒绝，不承担浏览器跳转编排

### 4.2 放行语义

`GET /internal/auth/workspace-access` 的部署语义必须与代码契约一致：

- `200`：允许继续转发到 runtime
- `401 UNAUTHENTICATED`：当前 session 不存在、失效或已撤销
- `403 USER_DISABLED`：当前用户已禁用
- `403 PASSWORD_CHANGE_REQUIRED`：当前用户必须先完成强制改密
- `403 ACCESS_DENIED`：无 workspace membership，或 runtime 当前不可进入

Traefik 前置要求：

- 对所有非 `2xx` 响应一律不得继续转发到 runtime
- 不得把下游 runtime 的返回结果当成鉴权真相
- 不得在网关层伪造用户身份头替代平台 session 校验

### 4.3 只读透传头

当 `workspace-access` 返回 `200` 时，允许附带只读透传头：

- `X-Clawloops-User-Id`
- `X-Clawloops-Subject-Id`
- `X-Clawloops-Workspace-Id`

部署与实现都必须遵守：

- 这些头仅供下游审计或日志使用
- runtime 不得把这些头重新视为新的信任边界
- 真实放行依据仍然只能是平台 session + 平台鉴权接口

---

## 5. Traefik 透传头前置条件

Traefik 在调用 `GET /internal/auth/workspace-access` 时，必须原样透传以下信息：

- 浏览器原始 cookie
- `Host`
- `X-Forwarded-Proto`
- `X-Forwarded-Host`
- `X-Forwarded-Uri`
- `X-Forwarded-Method`

部署前必须确认：

- Traefik 没有丢失浏览器原始 cookie
- Traefik 没有把 workspace host 改写成内部服务名
- `X-Forwarded-*` 头在鉴权子请求里仍然可见
- 平台能根据 host 路由规则解析目标 `workspaceId`

常见错误：

- `ForwardAuth` 请求里没有原始 cookie，导致所有 workspace 子域都变成未登录
- `Host` 被改写成 `clawloops-api` 之类的内部服务名，导致平台无法识别目标 workspace
- `X-Forwarded-Uri` 缺失，排障时无法判断真实访问路径

---

## 6. 共享网络前置条件

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

冻结要求：

- 不允许在请求体中切换共享网络名
- `litellm` 必须由平台后端共享 stack 启动，而不是由 per-user runtime 自行启动
- per-user runtime 只接入平台统一共享网络，不单独约定另一套正式入口网络

### 6.1 服务名与连通性

部署前必须确认：

- `litellm` 在 `clawloops_shared` 中可通过服务名 `litellm` 访问
- `clawloops-api` 在 `clawloops_shared` 中可被 `traefik` 与鉴权链路访问
- per-user runtime 在 `clawloops_shared` 中可通过固定 alias `rt-<runtimeId>` 被平台访问

关键约束：

- runtime 配置中的 `OPENAI_BASE_URL` 固定为 `http://litellm:4000`
- 稳定内部地址固定为 `http://rt-<runtimeId>:18789`
- 浏览器正式入口仍然只能走 workspace 子域，不走 `internalEndpoint`

### 6.2 网络核对清单

上线前至少核对一次：

1. `traefik` 已加入 `clawloops_shared`
2. `clawloops-api` 已加入 `clawloops_shared`
3. `runtime-manager` 已加入 `clawloops_shared`
4. `litellm` 已加入 `clawloops_shared`
5. per-user runtime 创建后会加入 `clawloops_shared`
6. `litellm` 服务名在 runtime 容器内可解析
7. `rt-<runtimeId>` alias 在平台侧可解析

---

## 7. Runtime 与浏览器入口前置条件

部署前必须接受以下事实：

- runtime readiness 只看 `18789`
- runtime 默认不作为公网直连服务暴露
- 浏览器正式入口必须经过 Traefik 和平台 session 鉴权
- `ready=true` 只是前端允许跳转的条件，不替代网关鉴权

这意味着：

- 即使 runtime 已经运行，若 cookie、域名或 `ForwardAuth` 没配对，浏览器仍然无法进入
- 即使知道 `browserUrl`，若 session 不合法，仍然必须被挡在网关
- 即使知道 `internalEndpoint`，它也只能用于平台内部诊断，不能作为浏览器入口

---

## 8. 最低验收清单

上线前，至少逐条确认以下事项：

1. 主域与 workspace 子域属于同一域名体系
2. 主域与 workspace 子域都已启用 HTTPS
3. `clawloops_session` 的 `Domain` 覆盖主域与 workspace 子域
4. 登录、invitation 接受、强制改密、logout 使用同一套 cookie 属性
5. 所有 workspace 子域统一经过 Traefik `ForwardAuth`
6. `ForwardAuth` 指向 `GET /internal/auth/workspace-access`
7. Traefik 会透传原始 cookie、`Host` 与约定的 `X-Forwarded-*` 头
8. `workspace-access` 的非 `2xx` 响应不会继续转发到 runtime
9. `traefik`、`clawloops-api`、`runtime-manager`、`litellm`、per-user runtime 全部加入 `clawloops_shared`
10. `litellm` 在共享网络中可通过服务名 `litellm` 访问
11. per-user runtime 在共享网络中可通过固定 alias `rt-<runtimeId>` 访问
12. `browserUrl` 指向的是受保护的 workspace 子域，而不是 runtime 直连端口

---

## 9. 与其他文档的边界

本清单负责回答：

- 部署前必须先配好什么
- 网关和域名层必须满足什么条件
- session 与 workspace 子域为什么能打通

本清单不替代：

- `后端/Architecture_Design.md`：系统整体设计与职责边界
- `后端/API_Spec.md`：接口、状态码与字段冻结
- `docker运行管理/RuntimeManager 开发契约.md`：RM internal API、挂载、drift、删除语义

如与其他文档冲突，应优先回到对应领域的主文档解释：

- cookie、workspace-access、browserUrl 语义：以后端文档为准
- 共享网络、internalEndpoint、runtime contract：以后端与 RM 契约共同冻结口径为准

---

v0.1
2026-03-27
