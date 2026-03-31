# CrewClaw 发布门禁

## 1. 目的

本文件定义 CrewClaw 当前版本的最小发布通过标准。

目标不是建立完整 QA 体系，而是在没有专职测试团队的情况下，先把“什么情况允许发布、什么情况必须阻断”固定下来。

---

## 2. 发布结论

每次发布只能有以下 3 种结论：

- `Go`：满足全部必过项，可发布。
- `Go with Risk`：存在已知偏差，但已经过负责人确认、写明影响面和回收时间，可带风险发布。
- `No-Go`：存在阻断项，不允许发布。

---

## 3. 最小发布门禁

当前版本一共设置 5 层门禁，必须按顺序通过。

### Gate 1. 安装与启动便捷性

必须至少完成一次从零到可访问的安装验证：

- 按 `infra/compose/README.md` 完成 `.env` 准备。
- `docker compose up -d --build` 能成功拉起默认服务。
- 主站、API、Runtime Manager 的入口都能访问。
- 新成员按文档操作时，不需要额外口头补充隐藏步骤。

阻断条件：

- 缺少关键环境变量说明。
- 文档步骤无法复现。
- 默认启动命令失败且无明确排障路径。

### Gate 2. 环境与网关前提

必须满足 [Deployment_Prerequisites.md](Deployment_Prerequisites.md) 的核心前提：

- 主域与 workspace 子域属于同一域名体系。
- `clawloops_session` 的 cookie 可覆盖主域与 workspace 子域。
- 所有 workspace 子域统一经过 `ForwardAuth`。
- `traefik`、`clawloops-api`、`runtime-manager`、`litellm`、per-user runtime 都加入 `clawloops_shared`。
- `litellm` 能以 `litellm:4000` 被访问。

阻断条件：

- Cookie 只在主域可见。
- workspace 子域绕过平台鉴权。
- 共享网络或模型网关连通性不成立。

### Gate 3. 核心业务链路冒烟

以下链路必须全部通过：

- 种子管理员首次登录并被强制跳转 `/force-password-change`。
- 管理员改密成功后进入 `/admin`。
- 管理员可以创建 invitation，并在列表中看到结果。
- 普通用户可通过 invitation 完成首次设密并进入 `/app`。
- 普通用户可看到 runtime 状态，并可进入 `/workspace-entry`。
- disabled 用户无法继续进入业务页面。

阻断条件：

- 登录、邀请、runtime、workspace 入口任一主链路失败。
- 角色路由错误。
- 错误分支没有稳定收口。

### Gate 4. 页面/API 回归

以下内容至少完成一次人工回归：

- `login`、`invite`、`app`、`workspace-entry`、`admin`、`admin/users`、`admin/invitations`、`admin/models`、`admin/provider-credentials`、`admin/usage`。
- 认证、授权、错误码、runtime 状态、workspace-entry 语义、admin 权限。
- invitation 的过期、撤销、重复消费、用户名不匹配等负向分支。

阻断条件：

- 页面能打开但关键动作失败。
- 页面行为与冻结文档冲突。
- 错误码或收口页错误，导致用户无法判断下一步。

### Gate 5. 自动化与已知偏差确认

必须同时满足：

- 后端 `pytest` 至少执行一次并记录结果。
- 本次发布涉及的失败测试必须被修复，或被明确判定为既有问题且写入风险说明。
- [Architecture_Design.md](../后端/Architecture_Design.md) 第 15 节的临时偏差逐项确认是否接受。

必须显式确认的当前偏差：

- `browserUrl` 仍可能是 hostPort 直连。
- OpenClaw token 仍在过渡方案中参与访问。
- `allowInsecureAuth` 等危险开关仍存在。
- LiteLLM 密钥仍采用先跑通的单密钥对齐策略。

阻断条件：

- 自动化测试未执行。
- 已知偏差未评审、未记录、无人认领。
- 新增风险没有负责人确认。

---

## 4. 最小通过标准

若团队当前资源有限，至少满足以下标准才可发版：

- 安装/启动便捷性检查通过。
- 环境前提检查通过。
- 关键冒烟链路全部通过。
- 至少一轮页面/API 回归通过。
- 后端 `pytest` 已运行并记录。
- 已知偏差已写入发布记录。

这 6 项中任何 1 项缺失，都应判定为 `No-Go`。

---

## 5. 角色分工建议

- 平台/后端负责人：环境前提、管理员链路、runtime、模型链路、自动化测试结果。
- 前端/产品负责人：登录页、邀请页、`/app`、`/workspace-entry`、后台页面回归。
- 发布负责人：汇总测试记录，给出 `Go / Go with Risk / No-Go` 结论。

建议关键链路至少双人交叉验证：

- 管理员首次登录与改密。
- invitation 首次接入。
- 普通用户进入 workspace。

---

## 6. 与其他文件的关系

- 执行清单：`docs/部署/Release_Checklist.md`
- 环境矩阵：`docs/部署/Environment_Matrix.md`
- 测试记录：`docs/测试/Test_Run_Log.md`
- 部署前提：`docs/部署/Deployment_Prerequisites.md`
- 后端测试说明：`docs/部署/TESTING.md`

---

## 7. 发布前最终签字

每次发布前，至少要明确以下信息：

- 发布版本或分支
- 执行环境
- 测试执行人
- 发布负责人
- 最终结论：`Go` / `Go with Risk` / `No-Go`
- 若为 `Go with Risk`，需附风险说明、影响面、补救措施、预计回收时间
