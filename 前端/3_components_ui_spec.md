# 3. 前端组件 / UI 规范文档

## 3.1 文档目标

本文档用于统一首版前端组件层、页面布局层、状态表达层和交互反馈层的规范，供 UI 设计与前端开发共同使用。

首版重点不是追求组件数量，而是统一：

- 信息层级
- 状态视觉语言
- 按钮优先级
- 表格与详情页结构
- 错误反馈方式
- 与后端枚举的视觉映射

---

## 3.2 设计原则

### 3.2.1 平台型产品优先

首版属于平台控制台，不是营销官网。页面应优先满足：

- 快速识别状态
- 高信息密度但不拥挤
- 危险操作清晰可见
- 异常反馈明确

### 3.2.2 首版风格建议

- 简洁、稳重、偏控制台风格
- 以卡片、表格、抽屉、确认弹窗为主
- 少做强装饰性动效
- 重点强调状态与动作关系

### 3.2.3 组件优先级

优先保证以下组件可用：

1. Layout
2. PageHeader
3. StatusBadge
4. DataTable
5. DetailSection
6. EmptyState / ErrorState / Skeleton
7. Modal / Drawer
8. Toast
9. PollingStatus
10. ConfirmDialog

---

## 3.3 页面布局规范

## 3.3.1 全局布局

建议采用三层结构：

```text
AppShell
├─ GlobalTopBar
├─ SideNav (管理员场景可见)
└─ PageContainer
   ├─ PageHeader
   ├─ PageToolbar (可选)
   └─ PageContent
```

### 3.3.2 用户区布局

用户区页面建议：

- 顶部保留品牌、当前用户、退出入口
- 左侧导航可简化为 3 ~ 4 项
- 工作台首页尽量卡片化

### 3.3.3 管理区布局

管理区建议：

- 左侧导航固定
- 右侧为内容区
- 列表页统一“筛选 + 表格 + 分页 / 滚动”模式

---

## 3.4 页面头部规范

### 3.4.1 `PageHeader`

建议包含：

- 页面标题
- 页面说明
- 主操作区
- 次级操作区
- 返回按钮（详情页）

### 3.4.2 标题层级

- 一级标题：页面名
- 二级说明：页面职责说明，不超过两行
- 操作按钮放右上角

### 3.4.3 按钮摆放规则

- 主要动作：最右侧，主按钮样式
- 次级动作：主按钮左侧，次按钮样式
- 危险动作：放入危险区或详情页底部，不与主要动作并列主视觉竞争

---

## 3.5 状态视觉规范

## 3.5.1 通用状态组件：`StatusBadge`

必须统一用于表达后端枚举状态。

### 3.5.2 用户状态映射

| 后端值 | 文案 | 视觉语义 |
| --- | --- | --- |
| `active` | 正常 | 成功态 |
| `disabled` | 已禁用 | 危险态 |

### 3.5.3 invitation 状态映射

| 后端值 | 文案 | 视觉语义 |
| --- | --- | --- |
| `pending` | 待接入 | 处理中 / 默认态 |
| `consumed` | 已使用 | 成功态 |
| `revoked` | 已撤销 | 危险态 |
| `expired` | 已过期 | 警告态 |

### 3.5.4 runtime 状态映射

| 后端值 | 文案 | 视觉语义 |
| --- | --- | --- |
| `creating` | 启动中 | 处理中 |
| `running` | 运行中 | 成功态 |
| `stopped` | 已停止 | 中性态 |
| `error` | 异常 | 危险态 |
| `deleted` | 已删除 | 弱化态 |

### 3.5.5 task 状态映射

| 后端值 | 文案 | 视觉语义 |
| --- | --- | --- |
| `pending` | 排队中 | 中性态 |
| `running` | 执行中 | 处理中 |
| `succeeded` | 已完成 | 成功态 |
| `failed` | 失败 | 危险态 |
| `canceled` | 已取消 | 弱化态 |

---

## 3.6 按钮规范

## 3.6.1 按钮等级

| 等级 | 用途 | 示例 |
| --- | --- | --- |
| Primary | 当前页面唯一主动作 | 前往登录、继续接入、启动 runtime、进入工作区、创建邀请 |
| Secondary | 补充动作 | 重试、查看详情、返回列表 |
| Tertiary / Text | 弱操作 | 复制链接、展开详情 |
| Danger | 不可逆或高风险操作 | 删除 runtime、撤销 invitation、禁用用户、删除凭据 |

### 3.6.2 loading 规则

当按钮发起请求时：

- 当前按钮进入 loading
- 文案可变为“提交中 / 启动中 / 撤销中”
- 防止重复点击

### 3.6.3 禁用规则

按钮禁用必须给出明确原因，优先使用：

- tooltip
- 辅助文案
- 次级说明文字

例如：

- runtime 启动中，不可重复操作
- invitation 已使用，不能撤销

---

## 3.7 列表与表格规范

## 3.7.1 `DataTable` 基线

首版管理后台尽量用统一表格组件。

### 3.7.2 表格列设计原则

- 第一列放识别信息（如邮箱 / 用户名 / workspace）
- 状态字段用 `StatusBadge`
- 时间字段统一格式化
- 行尾放操作按钮

### 3.7.3 推荐表格页面

- 用户列表
- invitation 列表
- 模型列表
- provider 凭据列表
- usage 明细

### 3.7.4 表格状态

统一支持：

- loading skeleton
- empty state
- error state
- row action loading

---

## 3.8 详情区块规范

## 3.8.1 `DetailSection`

用于详情页和工作台卡片。

建议结构：

- section title
- section description（可选）
- key-value list 或子卡片
- footer actions（可选）

### 3.8.2 Key-Value 显示规则

适用于：

- 用户详情
- invitation 详情
- runtime 详情
- 凭据详情

显示建议：

- label 左，value 右
- 长文本支持复制
- URL 支持复制或跳转

---

## 3.9 卡片规范

## 3.9.1 首页卡片建议

工作台首页建议至少包含：

- 身份卡片
- runtime 卡片
- 模型摘要卡片
- quota 卡片

### 3.9.2 runtime 状态卡

runtime 卡必须展示：

- observedState
- desiredState
- browserUrl（仅 ready 时）
- lastError（有则展示）
- 主操作按钮

### 3.9.3 invitation 结果卡

创建 invitation 成功后，建议以结果卡或成功弹层展示：

- invitationId
- targetEmail
- workspace
- role
- expiresAt
- inviteUrl
- 复制按钮

---

## 3.10 表单规范

## 3.10.1 首版重点表单

- 创建 invitation 表单
- 用户状态修改确认表单（可选）
- provider 凭据新增表单
- 模型策略编辑表单

### 3.10.2 表单项排列

建议规则：

- 关键字段单列布局
- 中短字段可双列
- 危险说明放提交区上方

### 3.10.3 字段错误反馈

- 优先行内错误
- 表单级失败用顶部错误块
- 服务端校验失败要保留用户已填写内容

### 3.10.4 创建 invitation 表单字段

建议按以下顺序：

1. targetEmail
2. workspaceId / workspace 选择器
3. role
4. expiresInHours 或 expiresAt

---

## 3.11 弹窗 / 抽屉规范

## 3.11.1 何时用弹窗

适合：

- 删除 runtime
- 撤销 invitation
- 禁用用户
- 删除 provider 凭据

### 3.11.2 何时用抽屉

适合：

- 创建 invitation
- 新增 provider 凭据
- 轻量编辑模型策略

### 3.11.3 危险确认弹窗必须包含

- 明确动作名称
- 影响说明
- 是否可恢复
- 确认按钮
- 取消按钮

---

## 3.12 空态规范

## 3.12.1 `EmptyState` 使用场景

- 没有可见模型
- 没有 invitation 数据
- 没有用户数据（极少）
- quota 数据为空

### 3.12.2 空态结构

- 标题
- 说明文字
- 推荐下一步动作

### 3.12.3 页面化示例

| 页面 | 空态文案方向 |
| --- | --- |
| 模型列表 | 当前暂无可用模型 |
| invitation 列表 | 还没有邀请记录 |
| 用户列表 | 当前没有可管理用户 |
| quota 页 | 暂无额度数据 |

---

## 3.13 错误态规范

## 3.13.1 `ErrorState`

适用于整页主请求失败。

建议包含：

- 错误标题
- 简短说明
- 重试按钮
- 返回入口（可选）

### 3.13.2 局部错误块

适用于：

- 用户详情成功，但 runtime 详情失败
- invitation 列表成功，但复制链接失败
- runtime task 失败但页面主体可展示

### 3.13.3 错误文案原则

- 面向用户，不暴露技术细节堆栈
- 若存在 `code`，可在次级位置展示，便于排查

---

## 3.14 加载态规范

## 3.14.1 页面级 loading

用于首屏关键请求。

建议：

- 列表页用表格 skeleton
- 详情页用 key-value skeleton
- 首页用卡片 skeleton

### 3.14.2 局部 loading

适用于：

- 行内按钮操作
- 复制链接
- provider 凭据校验
- invitation 重发

---

## 3.15 Toast 规范

## 3.15.1 Toast 分类

- success
- warning
- error
- info

### 3.15.2 使用建议

适合 toast 的动作：

- invitation 创建成功
- 链接已复制
- runtime 启动任务已提交
- 用户状态修改成功
- provider 凭据校验成功

不适合只用 toast 的场景：

- 首屏加载失败
- invitation 失效页
- disabled 用户状态

---

## 3.16 轮询状态组件规范

## 3.16.1 `PollingStatus`

用于 runtime 相关异步任务。

建议内容：

- 当前 task 状态
- action 名称
- message
- 最近刷新时间（可选）

### 3.16.2 可视化建议

- 文案 + 小型加载指示即可
- 不建议首版做复杂步骤条

---

## 3.17 复制能力规范

首版以下字段建议统一支持复制：

- inviteUrl
- browserUrl
- internalEndpoint
- subjectId
- userId
- invitationId
- runtimeId

复制成功统一 toast：

- 已复制

---

## 3.18 时间与格式化规范

### 3.18.1 时间字段

统一展示：

- 日期时间
- 与当前时间的相对语义可作为辅助展示

适用字段：

- `expiresAt`
- `consumedAt`
- `createdAt`
- `updatedAt`
- `lastLoginAt`

### 3.18.2 枚举显示

前端不要直接把原始枚举裸露给用户，应统一转为中文文案。

---

## 3.19 组件命名建议

```text
AppShell
TopBar
SideNav
PageHeader
StatusBadge
StateCard
DataTable
DetailSection
KeyValueList
EmptyState
ErrorState
LoadingSkeleton
ConfirmDialog
FormDrawer
ActionToolbar
PollingStatus
CopyButton
```

---

## 3.20 首版不建议过度设计的部分

首版不建议投入太多设计资源在以下方向：

- 复杂图表库整合
- 自定义大段动效
- 可视化流程编排
- 拖拽式管理后台
- 高复杂度多主题体系

优先级应放在：

- 状态表达一致
- 表单体验可靠
- 错误反馈清晰
- 表格与详情页统一

---

## 3.21 最终结论

首版组件 / UI 规范的核心不是“多”，而是“统一”。

最应该统一的是：

1. **状态 Badge 语义**
2. **危险操作确认方式**
3. **列表 / 详情 / 卡片三类基础骨架**
4. **加载、空态、错误态表达**
5. **runtime 异步任务的反馈组件**

只要这五类组件稳定，首版 UI 和前端就能在较少返工下完成交付。
