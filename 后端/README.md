# CrewClaw × Authentik 实施文档包

本压缩包包含 4 份可直接在 Cursor 中使用和继续迭代的 Markdown 文档：

1. `Architecture_Design.md`：架构设计文档（已整合官方 Authentik 首版接入方案）
2. `MVP_Contract.md`：MVP 模块契约文档（已补齐认证、邀请、首登、强制改密规则）
3. `API_Spec.md`：API 接口基线（已补齐 invitation / auth / post-login / 管理员治理接口）
4. `AUTHENTIK_Implementation_Guide.md`：面向落地实施的详细操作文档，含架构、流程、配置建议、Cursor 开发任务清单

## 本包采用的落地结论

- 首版直接使用**官方 Authentik**。
- **不改 OpenClaw / CrewClaw 上游源码**，只改你自己的平台接入层、网关和管理逻辑。
- Docker 采用**分容器、同网络**。
- 首版只开放**本地账号密码**。
- 后续再逐步接入 **Google / GitHub / 企业 SSO / 微信 / 钉钉 / 飞书**。

## 关于“首次启动就是 admin 登录”这件事

官方 Authentik 在首次安装时，默认的初始化管理员用户是 **`akadmin`**，首次密码设置走官方 `initial-setup` 流程。

因此：

- 如果你接受“首个登录的是**管理员角色**”而不是“用户名字面必须是 `admin`”，那么**官方能力直接满足**。
- 如果你要求“用户名字面必须叫 `admin`”，又**不改官方源码**，那么建议采用：
  1. 首次用 `akadmin` 完成初始设置；
  2. 立刻创建一个本地管理员 `admin`；
  3. 之后把 `akadmin` 只保留为 break-glass 应急账号，或停用日常使用。

## 推荐首版实现方式

推荐把“邀请链接免密码进入”理解为：

- 用户第一次不需要事先知道密码；
- 通过**一次性 invitation token** 打开接入页；
- 在 Authentik 的 enrollment flow 中完成用户名/资料/密码设置；
- 完成后由 Authentik 自动登录；
- CrewClaw 再根据 invitation 元数据绑定 workspace / role。

这样最贴近你的业务目标，也最符合官方 Authentik 的能力边界。
