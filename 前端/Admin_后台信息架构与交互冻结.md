# ClawLoops Admin 后台信息架构与交互冻结（轻量认证修订）

## 1. 文档目标

本文补足 `/admin` 后台的信息架构、默认首页、导航、跨页跳转与关键交互冻结规则，用于把管理后台推进到“前端可直接开工”的程度。

本文与以下文档共同生效：

- `前端/页面清单与冻结边界.md`
- `前端/页面调用流程_BFF编排.md`
- `前端/UI_状态模型.md`
- `后端/API_Spec.md`

---

## 2. 冻结结论

首版管理后台不是“只有若干散落二级页”的能力集合，而是一套完整控制面：

- `/admin` 是真实默认首页
- `/admin` 不依赖 workspace membership
- `/admin` 首页只做摘要、待办与跳转
- 所有写操作仍在二级页面完成
- 管理后台左侧导航固定，不做自定义排序
- 管理后台认证依赖平台 session，而不是外部 IAM
- 若命中首登强制改密，管理员不得先进入后台壳层
- 管理后台前端不读取、不管理 `clawloops_session`
- 管理后台前端不调用 `/internal/*`，包括 `/internal/auth/workspace-access`

---

## 3. 后台路由结构

| 路由 | 页面定位 | 首版目标 |
| --- | --- | --- |
| `/admin` | 管理后台首页 | 平台治理摘要、待办事项、快捷入口 |
| `/admin/public-area` | 公共区域管理 | 目录浏览、覆盖上传、删除与分页治理 |
| `/admin/users` | 用户列表 | 快速查看用户状态与进入详情 |
| `/admin/users/:userId` | 用户详情 | 查看用户资料、runtime 与启用/禁用 |
| `/admin/invitations` | invitation 列表 | 创建、撤销、重发 invitation |
| `/admin/invitations/:invitationId` | invitation 详情 | 查看单条 invitation 详情，可与列表抽屉合并 |
| `/admin/models` | 模型治理 | 模型开关与单项策略编辑 |
| `/admin/provider-credentials` | provider 凭据治理 | 新增、验证、删除平台凭据 |
| `/admin/user-files/:username` | 用户文件夹列表 | 管理用户的文件夹 |
| `/admin/usage` | usage 汇总 | 全局 usage 只读查询 |

冻结规则：

- `admin` 登录后默认进入 `/admin`
- `/admin` 不是重定向到 `/admin/users` 的跳板页
- 后续即使增加更多后台页，也不能改变上述首版默认首页规则

---

## 4. 后台壳层布局

## 4.1 固定布局

首版采用固定三段式壳层：

- 左侧：一级导航
- 顶部：当前管理员信息、刷新、退出
- 主内容区：当前路由页面

不做：

- 顶部/侧栏双导航同时可配置
- 个性化布局记忆
- 可拖拽卡片面板

## 4.2 左侧一级导航

顺序冻结为：

1. 管理首页
2. 公共区域管理
3. 用户管理
4. 邀请管理
5. 模型治理
6. Provider 凭据
7. 用户文件管理
8. Usage 汇总

导航规则：

- 当前页高亮必须稳定
- 后台内部切换不丢失壳层
- 非 admin 用户误入 `/admin/*` 统一进入 403 无权限页
- 命中 `PASSWORD_CHANGE_REQUIRED` 时，不得渲染后台壳层主内容，必须立即跳 `/force-password-change`

---

## 5. `/admin` 默认首页冻结

## 5.1 页面目标

管理员首页必须在 10 秒内回答三个问题：

1. 现在平台整体状态是否正常？
2. 有没有需要立刻处理的 invitation 或 runtime 异常？
3. 我下一步最常去的后台页是什么？

## 5.2 数据来源

唯一数据源：

- `GET /api/v1/admin/home`

前端禁止做法：

- 不根据其他接口自行拼摘要
- 不把 `/admin` 当成过渡页
- 不读取 session cookie 判断是否登录
- 不通过任何 `/internal/*` 接口预判 workspace 或后台权限

---

## 6. 跨页交互冻结

### 6.1 登录后的后台落点

- 管理员通过 `/api/v1/auth/login` 登录成功后，默认进入 `/admin`
- 若返回 `mustChangePassword=true` 或 `redirectTo=/force-password-change`，必须先进入强制改密页
- 不再经过 `/post-login`
- 已登录管理员再次进入 `/login` 时，应直接回 `/admin`
- 已登录但仍需改密的管理员再次进入 `/login` 时，应直接回 `/force-password-change`

### 6.2 invitation 管理

- 列表页负责高频动作
- 详情页负责查看完整信息
- 创建成功后必须能直接复制 `inviteUrl`
- invitation 列表与详情只把 `pending / consumed / revoked` 视为状态真相
- `expired` 只作为时间语义理解，不要求前端额外造一个库存状态
- 当用户侧 `accept` 命中 `replayed=true` 时，后台仍只把该 invitation 视为已稳定消费，不需要额外“重复提交异常”状态

### 6.3 用户治理

- 用户列表页展示 `username / role / status`
- 用户详情页展示用户基础信息、runtime 信息、启用禁用操作
- disabled 后用户应在最短时间内失去业务访问权限

---

### 6.4 公共区域治理

- 管理员通过 `/admin/public-area` 管理共享文件目录
- 页面初始化与用户侧公共区域共用 `GET /api/v1/public-area/files/list`
- 后台允许执行 `overwrite upload` 与 `delete`
- 删除目录只允许空目录，删除非空目录需给出冲突提示
- 路径栏需展示完整服务器绝对路径
- 回退越界必须阻断并提示“不允许再回退”
- 宿主机公共区影响范围仅限该后台管理入口；admin 在 OpenClaw 中同类操作不影响宿主机

---

## 7. 首版不做

- 后台导航内改密入口
- 后台找回密码入口
- 外部身份源管理页
- 外部 IAM 配置页
- 后台前端自管 session cookie 生命周期
- 后台前端直连 workspace 网关鉴权接口

---

## 8. 最终后台结论

管理后台与轻量认证方案的关系只需要记住两点：

1. 后台完全建立在平台 session 之上
2. 首登强制改密页在后台壳层之外，改完后才进入后台首页
3. 后台首页、用户治理和 invitation 治理不再依赖任何外部认证流程
4. 后台前端不感知 `clawloops_session` 细节，也不调用任何 `/internal/*` 鉴权入口

---

v0.4-轻量认证修订  
2026-03-25
