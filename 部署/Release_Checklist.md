# CrewClaw 发布前自测清单

## 1. 使用方式

本清单用于发布前人工自测。

建议按顺序执行：

1. 先做安装与启动便捷性检查。
2. 再做环境与网关检查。
3. 然后跑管理员与普通用户主链路。
4. 最后做负向回归、记录结果并给出发布结论。

配套文件：

- 发布门禁：`docs/部署/Release_Gates.md`
- 环境矩阵：`docs/部署/Environment_Matrix.md`
- 测试记录：`docs/测试/Test_Run_Log.md`

---

## 2. 发布基本信息

| 项目 | 填写 |
| --- | --- |
| 版本 / 分支 |  |
| 执行环境 |  |
| 执行日期 |  |
| 执行人 |  |
| 复核人 |  |

---

## 3. 安装与启动便捷性清单

| 检查项 | 结果 | 备注 |
| --- | --- | --- |
| 新成员仅根据 `README.md` 与 `infra/compose/README.md` 就能找到启动入口 |  |  |
| 能明确知道需要复制 `infra/compose/.env.example` 为 `.env` |  |  |
| `CLAWLOOPS_DOMAIN`、`RUNTIME_MANAGER_DOMAIN`、`DASHSCOPE_API_KEY` 等关键变量说明清楚 |  |  |
| 本机 hosts 或域名准备步骤清楚，不需要口头补充 |  |  |
| `docker compose up -d --build` 可以成功执行 |  |  |
| 默认服务启动后，主站入口、API 入口、Runtime Manager 入口都能访问 |  |  |
| 遇到启动失败时，文档给出的日志查看方式足够排障 |  |  |
| 本次安装过程中是否出现“必须询问老成员才知道”的隐藏步骤 |  | 若有，必须补文档 |

---

## 4. 环境与网关前提清单

| 检查项 | 结果 | 备注 |
| --- | --- | --- |
| 主域与 workspace 子域属于同一域名体系 |  |  |
| `clawloops_session` cookie 可以覆盖主域与 workspace 子域 |  |  |
| 主域与 workspace 子域访问方式一致，不混用多套域名口径 |  |  |
| 所有 workspace 子域统一经过 `ForwardAuth` |  |  |
| `ForwardAuth` 指向 `GET /internal/auth/workspace-access` |  |  |
| Traefik 会透传原始 cookie、`Host` 与约定的 `X-Forwarded-*` 头 |  |  |
| `traefik`、`clawloops-api`、`runtime-manager`、`litellm`、per-user runtime 都在 `clawloops_shared` |  |  |
| runtime 可通过 `rt-<runtimeId>` 在共享网络中解析 |  |  |
| `litellm` 可通过 `http://litellm:4000` 访问 |  |  |
| 当前 Traefik 生效路由与 `infra/compose/.env` 口径一致 |  | 若不一致，阻断发布 |

---

## 5. 管理员主链路清单

| 检查项 | 结果 | 备注 |
| --- | --- | --- |
| 种子管理员使用默认密码 `admin` 首次登录后，会立即进入 `/force-password-change` |  |  |
| 强制改密前，管理员不能进入 `/admin` 业务壳层 |  |  |
| 改密成功后自动进入 `/admin` |  |  |
| `/admin` 首页能正常加载摘要信息 |  |  |
| `/admin/users` 能查看用户列表 |  |  |
| `/admin/users/:userId` 能查看用户详情 |  |  |
| 管理员可以启用 / 禁用用户 |  |  |
| `/admin/invitations` 能创建 invitation |  |  |
| invitation 创建成功后可看到 `inviteUrl`、`loginUsername`、`workspaceName`、`expiresAt` |  |  |
| invitation 可以撤销、重发 |  |  |
| `/admin/models` 能查看模型治理信息 |  |  |
| `/admin/provider-credentials` 能新增 / 验证 / 删除凭据 |  |  |
| `/admin/usage` 能查看 usage 汇总 |  |  |

---

## 6. 普通用户主链路清单

| 检查项 | 结果 | 备注 |
| --- | --- | --- |
| 用户打开 `/invite/:token` 能看到 workspace、role、有效期、推荐用户名 |  |  |
| invitation 接入页密码规则与 `/auth/options.passwordPolicy` 一致 |  |  |
| 有效 invitation 可以成功完成首次设密并进入 `/app` |  |  |
| 首次接入后，`/app` 能展示“已成功加入 workspace”的承接信息 |  |  |
| 用户工作台能看到 runtime 状态 |  |  |
| 用户可以执行启动 runtime 动作 |  |  |
| runtime 准备中、可进入、失败态的页面提示正确 |  |  |
| `/workspace-entry` 仅在 `ready=true` 时整页跳转 |  |  |
| 跳转到 workspace 时没有由前端拼接额外鉴权参数 |  |  |

---

## 7. 负向与回归清单

| 检查项 | 结果 | 备注 |
| --- | --- | --- |
| 未登录访问受保护页面或接口时得到 `401 UNAUTHENTICATED` |  |  |
| disabled 用户访问业务页面时进入 `/disabled` 或得到 `403 USER_DISABLED` |  |  |
| 非 admin 访问 `/admin/*` 时进入 `/403` |  |  |
| invitation 过期时进入正确失效页 |  |  |
| invitation 已撤销时进入正确失效页 |  |  |
| invitation 已消费时进入正确失效页 |  |  |
| invitation 用户名不匹配时，页面给出明确提示 |  |  |
| `accept` 重试命中 `replayed=true` 时，前端仍按成功接入处理 |  |  |
| runtime 不存在、未运行、启动中、错误态时，`workspace-entry` 的 `reason` 正确 |  |  |

---

## 8. 页面级最小回归清单

至少逐页确认一遍：

| 页面 | 能打开 | 能加载核心数据 | 主动作可用 | 错误态正确 | 备注 |
| --- | --- | --- | --- | --- | --- |
| `/login` |  |  |  |  |  |
| `/force-password-change` |  |  |  |  |  |
| `/invite/:token` |  |  |  |  |  |
| `/app` |  |  |  |  |  |
| `/workspace-entry` |  |  |  |  |  |
| `/admin` |  |  |  |  |  |
| `/admin/users` |  |  |  |  |  |
| `/admin/invitations` |  |  |  |  |  |
| `/admin/models` |  |  |  |  |  |
| `/admin/provider-credentials` |  |  |  |  |  |
| `/admin/usage` |  |  |  |  |  |

---

## 9. 自动化测试核对

| 检查项 | 结果 | 备注 |
| --- | --- | --- |
| 已执行 `pytest apps/clawloops-api/tests -vv` |  |  |
| 核心 smoke / integration / auth / admin / workspace-entry 测试已通过 |  |  |
| 若存在失败项，已区分为新问题或既有问题 |  |  |
| 失败项已经写入发布记录并明确是否阻断 |  |  |

---

## 10. 已知偏差确认

每次发布必须确认以下偏差是否接受：

| 偏差项 | 接受 / 不接受 | 影响面 | 回收时间 | 备注 |
| --- | --- | --- | --- | --- |
| `browserUrl` 仍可能为 hostPort 直连 |  |  |  |  |
| OpenClaw token 仍以过渡方案参与访问 |  |  |  |  |
| `allowInsecureAuth` 等危险开关仍存在 |  |  |  |  |
| LiteLLM 密钥仍采用单密钥对齐策略 |  |  |  |  |

---

## 11. 最终结论

| 结论 | 勾选 | 说明 |
| --- | --- | --- |
| `Go` |  | 全部必过项通过 |
| `Go with Risk` |  | 已记录风险与回收时间 |
| `No-Go` |  | 有阻断项未解决 |
