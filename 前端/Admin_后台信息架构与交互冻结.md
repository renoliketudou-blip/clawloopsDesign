# ClawLoops Admin 后台信息架构与交互冻结

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

---

## 3. 后台路由结构

| 路由 | 页面定位 | 首版目标 |
| --- | --- | --- |
| `/admin` | 管理后台首页 | 平台治理摘要、待办事项、快捷入口 |
| `/admin/users` | 用户列表 | 快速查看用户状态与进入详情 |
| `/admin/users/:userId` | 用户详情 | 查看用户资料、runtime 与启用/禁用 |
| `/admin/invitations` | invitation 列表 | 创建、撤销、重发 invitation |
| `/admin/invitations/:invitationId` | invitation 详情 | 查看单条 invitation 详情，可与列表抽屉合并 |
| `/admin/models` | 模型治理 | 模型开关与单项策略编辑 |
| `/admin/provider-credentials` | provider 凭据治理 | 新增、验证、删除平台凭据 |
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
2. 用户管理
3. 邀请管理
4. 模型治理
5. Provider 凭据
6. Usage 汇总

导航规则：

- 当前页高亮必须稳定
- 后台内部切换不丢失壳层
- 非 admin 用户误入 `/admin/*` 统一进入 403 无权限页

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

- 首屏并发多个后台接口再自行聚合摘要
- 用用户列表数量近似代替真实摘要计数
- 用 invitation 列表第一页代替待处理事项

## 5.3 首页信息块

页面从上到下固定为 3 个区块：

### A. 摘要卡片区

最小卡片：

- 总用户数
- 活跃用户数
- 已禁用用户数
- 待处理邀请数
- 24 小时内到期邀请数
- 运行中 runtime 数
- runtime 异常数

交互规则：

- 卡片可点击跳转对应二级页
- 卡片不直接执行写操作

### B. 待处理 invitation 区

最小列：

- `targetEmail`
- `workspaceId`
- `role`
- `expiresAt`
- `status`

最小动作：

- 进入邀请列表
- 进入 invitation 详情或打开详情抽屉

### C. runtime 异常区

最小列：

- `userId`
- `runtimeId`
- `observedState`
- `lastError`
- `updatedAt`

最小动作：

- 进入对应用户详情页

## 5.4 首页空态

允许出现以下空态：

- 暂无待处理 invitation
- 暂无 runtime 异常

空态要求：

- 仍保留摘要卡片
- 仍保留快捷入口
- 不把整页渲染成“空白后台”

## 5.5 首页 Done Definition

- 没有 workspace membership 的 admin 也能正常使用
- 首屏可见平台摘要、待办与快捷入口
- 首页不会承担任何复杂表单
- 首页刷新不会影响壳层和当前路由稳定性

---

## 6. 二级页职责冻结

## 6.1 用户管理

`/admin/users` 负责：

- 展示最小用户列表字段
- 承接用户状态修改入口
- 进入用户详情

`/admin/users/:userId` 负责：

- 查看用户基础信息
- 查看 runtime 详情
- 启用 / 禁用用户

## 6.2 邀请管理

`/admin/invitations` 负责：

- 创建 invitation
- 查看列表
- 撤销 invitation
- 重发 invitation

`/admin/invitations/:invitationId` 负责：

- 查看单条 invitation 详情
- 撤销 / 重发

## 6.3 平台治理页

`/admin/models`、`/admin/provider-credentials`、`/admin/usage` 的定位分别是：

- 模型治理：改模型策略
- Provider 凭据：管平台级凭据
- Usage 汇总：看平台级统计

首版要求：

- 都必须可独立直达
- 都必须保留统一后台壳层
- 都采用“简单读写 + 成功后回刷”的交互

---

## 7. 跨页跳转冻结

固定跳转关系：

- `/admin` 卡片 -> 对应二级页
- `/admin` 的 invitation 待办 -> `/admin/invitations` 或 invitation 详情
- `/admin` 的 runtime 异常 -> `/admin/users/:userId`
- `/admin/users` -> `/admin/users/:userId`
- `/admin/invitations` -> `/admin/invitations/:invitationId` 或抽屉详情

禁止：

- 首页直接弹出复杂编辑表单
- 首页直接做启用/禁用用户
- runtime 异常从首页直接做容器治理写操作

---

## 8. 权限与异常收口

## 8.1 权限

- 只有 `admin` 能进入 `/admin/*`
- `USER_DISABLED` 优先级高于 admin 身份，命中后进入禁用页
- 普通用户访问 `/admin/*` 返回 403 页，不跳回工作台伪装成成功

## 8.2 页面错误态

后台各页至少要有：

- 首屏加载失败态
- 空列表态
- 提交中态
- 提交失败态

首页额外要求：

- 即使待办区失败，只要摘要成功，也应保留可用骨架
- 若 `GET /api/v1/admin/home` 整体失败，显示整页重试态

---

## 9. 冻结边界

本文冻结到可开发层级，但首版明确不做：

- 自定义仪表盘配置
- 高级筛选保存
- 跨页批量操作中心
- usage 可视化大屏
- 多维度拖拽报表

---

## 10. 最终可执行结论

前端可以据此直接冻结管理后台实现：

1. `/admin` 是真实首页，不是重定向壳。
2. 首页只读，唯一数据源是 `GET /api/v1/admin/home`。
3. 二级页负责写操作，首页负责摘要、待办和跳转。
4. 左侧导航、路由结构、跨页跳转与权限收口都已冻结。
5. 前端无需再等待额外 admin 产品定义，即可开始页面实现。


v0.2 前端
reno
2026-03-25
