# ClawLoops × 业务内轻量认证实施文档

这份文档是给你直接在 Cursor 里开干用的，不再停留在“讨论方案”，而是明确到：

- 轻量认证首版该怎么接
- ClawLoops 自己要承担哪些认证能力
- 前后端的登录与 invitation 页面怎么对齐
- session、cookie、权限中间件、部署要点如何安排

---

## 1. 先说结论

### 1.1 为什么这次不继续用外部 IAM

因为你当前 MVP 真正需要的只有：

1. 用户名密码登录
2. 邀请制首次接入
3. 管理员和普通用户分流
4. workspace 子域受保护

这些能力完全可以由平台自己承担，而且会明显减少：

- flow 配置
- 外部 token 概念
- 登录后收口页面
- 网关头透传
- 调试链路长度

### 1.2 首版推荐方案

推荐的首版不是“再找另一个 IAM”，而是下面这套：

- ClawLoops 负责用户、密码哈希、session、invitation
- ClawLoops 负责 invitation 消费与 workspace membership 绑定
- 前端登录页直接调用平台登录接口
- invitation 接入页直接在站内完成首次设密
- workspace 子域通过平台 session 鉴权中间层保护
- 首版只开本地账号密码
- 种子管理员默认密码固定为 `admin`，首次登录必须强制改密
- 首版不做找回密码

---

## 2. 首版边界（必须冻结）

```text
首版使用业务内轻量认证
不依赖外部 IAM
只做本地账号密码
管理员由平台初始化脚本创建
种子管理员初始密码固定为 admin
种子管理员首次登录后必须立刻改密
普通用户通过 invitation 首次设密进入
平台 session 是唯一登录真相
workspace 子域受平台 session 保护
不做通用改密
不做找回密码
RuntimeManager internal API 仍同步执行
Orchestrator 对外仍异步返回 taskId
```

把这几条当成首版硬边界，不要在开发中途漂移。

---

## 3. 认证对象与最小数据模型

### 3.1 User

建议字段：

- `id`
- `subject_id`
- `username`
- `password_hash`
- `role`
- `status`
- `created_at`
- `last_login_at`

推荐口径：

- `subject_id = clawloops:<userId>`
- `role = admin | user`
- `status = active | disabled`

### 3.2 Session

建议字段：

- `id`
- `user_id`
- `issued_at`
- `expires_at`
- `revoked_at`
- `created_by_ip`
- `user_agent`

### 3.3 Invitation

建议字段：

- `id`
- `invite_token_hash`
- `target_email`
- `login_username`
- `workspace_id`
- `role`
- `status`
- `expires_at`
- `consumed_by_user_id`
- `consumed_at`

首版状态只认：

- `pending`
- `consumed`
- `revoked`

---

## 4. 密码与 session 实现建议

### 4.1 密码哈希

建议：

- 优先 `argon2id`
- 若现有栈接入成本高，可退而用 `bcrypt`

硬规则：

- 只存哈希
- 不存明文
- 不把密码写进日志

### 4.2 session 方式

首版建议使用服务端 session：

- 浏览器拿 HttpOnly cookie
- 服务端存 session 记录
- 每次请求由服务端查 session

这样做的优点：

- 简单
- 易撤销
- 易做 disabled 用户失效
- 不要求前端理解 JWT

### 4.3 cookie 建议

- `HttpOnly = true`
- `Secure = true`
- `SameSite = Lax`
- 主域和 workspace 子域按部署方式统一 cookie 可见范围

如果主域与 workspace 子域共享 cookie：

- 需明确 cookie domain
- 仍要由服务端判断用户是否可访问当前 workspace

---

## 5. 登录实现

### 5.1 登录接口

建议接口：

- `POST /api/v1/auth/login`

请求体：

```json
{
  "username": "emp001",
  "password": "secret"
}
```

服务端步骤：

1. 按 `username` 查用户
2. 校验 `status`
3. 校验 `password_hash`
4. 创建 session
5. 若 `mustChangePassword=true`，返回 `redirectTo=/force-password-change`
6. 否则返回当前用户信息与正常 `redirectTo`

### 5.2 首登强制改密接口

建议接口：

- `POST /api/v1/auth/password/change`

建议请求体：

```json
{
  "currentPassword": "admin",
  "newPassword": "admin#2026!new",
  "newPasswordConfirm": "admin#2026!new"
}
```

服务端步骤：

1. 校验当前 session
2. 只允许修改当前登录用户自己的密码
3. 校验 `currentPassword`
4. 校验 `newPassword` 与 `newPasswordConfirm`
5. 拒绝与当前密码相同的新密码
6. 更新 `password_hash`
7. 清除 `mustChangePassword`
8. 返回 `redirectTo=/admin`

### 5.3 退出接口

建议接口：

- `POST /api/v1/auth/logout`

服务端步骤：

1. 定位当前 session
2. 标记 `revoked_at`
3. 清理 cookie

### 5.4 当前用户接口

建议接口：

- `GET /api/v1/auth/me`
- `GET /api/v1/auth/access`

作用：

- `/auth/me` 给前端当前登录身份
- `/auth/access` 给前端当前用户是否还能进入业务
- 若 `mustChangePassword=true`，前端必须把用户收口到 `/force-password-change`

---

## 6. invitation 首次接入实现

### 6.1 为什么首版改成单层 invitation

因为你真正需要的是：

- invitation 预览
- 用户首次设置密码
- 平台完成 membership 绑定

这些都可以在一个请求内完成，没必要再拆成：

- 业务 invitation
- 身份 invitation
- 登录后再收口

### 6.2 推荐接口

- `GET /api/v1/public/invitations/{token}`
- `POST /api/v1/public/invitations/{token}/accept`

### 6.3 `accept` 的服务端步骤

1. 校验 token
2. 校验 `status`
3. 校验 `expiresAt`
4. 校验 `username` 与 `loginUsername`
5. 校验 `password` 与 `passwordConfirm`
6. 创建用户或激活预创建用户
7. 写入 `password_hash`
8. 创建 `workspace membership`
9. 标记 invitation 为 `consumed`
10. 创建 session
11. 返回 `redirectTo=/app`

### 6.4 幂等设计

最小要求：

- 同一 invitation 重复提交不能创建多个用户
- 同一 invitation 重复提交不能创建多条 membership
- 重复成功请求返回稳定结构

推荐做法：

- 数据库唯一约束保证 `invitationId + consumedByUserId`
- membership 写入放事务里
- `consumed` 后再提交时返回稳定结果或明确错误

---

## 7. 权限中间件与 workspace 子域保护

### 7.1 平台控制面中间件

至少要有两层：

1. `requireSession`
2. `requireAdmin`

语义冻结：

- `requireSession` 只判断 session 是否有效
- `requireAdmin` 在 session 有效后判断 `role=admin`

### 7.2 workspace 鉴权中间层

你需要一个面向 workspace 子域的轻量鉴权入口。

可选实现：

1. Traefik `ForwardAuth` 指向平台 `/internal/auth/workspace-access`
2. workspace 前单独放一个轻量代理，回源平台鉴权接口

无论哪种实现，平台鉴权接口都至少要判断：

- 当前 session 是否有效
- 当前用户是否绑定目标 workspace
- 当前用户是否被 disabled
- 当前 runtime 是否允许进入

### 7.3 为什么不能只靠前端拦截

因为：

- `browserUrl` 会泄露给浏览器
- 用户可直接访问子域
- 真正的权限边界必须在服务端或网关

---

## 8. 前端页面要怎么改

### 8.1 `/login`

旧方案：

- 点按钮跳外部登录页

新方案：

- 直接展示用户名和密码输入框
- 提交到 `POST /api/v1/auth/login`
- 若命中种子管理员首登，成功后先按 `redirectTo` 进 `/force-password-change`
- 其他场景再进入 `/app` 或 `/admin`

### 8.2 `/force-password-change`

新页面职责：

- 只给已登录且 `mustChangePassword=true` 的用户访问
- 强制提交当前密码与新密码
- 成功后进入 `/admin`

### 8.3 `/invite/:token`

旧方案：

- 点击继续接入
- 拿 `redirectUrl`
- 跳外部 enrollment flow

新方案：

- 先读 invitation 预览
- 站内填写初始密码
- 提交 `POST /api/v1/public/invitations/{token}/accept`
- 成功后直接进入 `/app`

### 8.4 删除 `/post-login`

旧方案有它，是因为登录与业务绑定分两段。

新方案里：

- invitation 接受本身就完成登录和绑定
- 普通登录也直接建立 session

所以不再需要：

- `post-login`
- pending invitation session
- 登录后业务收口页

---

## 9. 后端代码落点建议

如果你按常见服务拆分实现，建议最少有：

- `auth service`
- `session repository`
- `invitation service`
- `workspace access guard`

职责建议：

| 模块 | 职责 |
| --- | --- |
| `auth service` | 登录、logout、密码校验、session 创建 |
| `invitation service` | invitation 预览、accept、幂等与事务 |
| `access guard` | `/auth/access`、workspace 鉴权判定 |
| `admin user service` | 用户禁用、列表查询、详情查询 |

---

## 10. 迁移现有文档和接口时要做的删减

删掉这些概念：

- 外部 IAM provider
- enrollment flow
- `itoken`
- `post-login`
- 外部组映射
- 外部鉴权头
- 外部管理员初始化

替换成这些概念：

- `POST /api/v1/auth/login`
- `POST /api/v1/auth/logout`
- `GET /api/v1/auth/me`
- `GET /api/v1/auth/access`
- `POST /api/v1/public/invitations/{token}/accept`
- 平台 session
- workspace 鉴权中间层

---

## 11. 首版明确不做

- 通用改密
- 找回密码
- 邮箱验证码
- 多因素认证
- 第三方登录
- 企业 SSO
- SCIM / 目录同步

不要把这些能力偷偷夹带进首版。

---

## 12. 常见误区

### 误区 1：既然自己做认证，就顺手把所有身份能力做完

不对。首版只做：

- 登录
- logout
- invitation 首设密码
- session

### 误区 2：workspace 访问只靠前端是否已登录

不对。真正的权限边界必须放在网关或服务端。

### 误区 3：既然不做找回密码，就可以弱化密码哈希和 session 管理

不对。安全边界依然要完整。

### 误区 4：普通用户接入后先跳一个中转页再说

不推荐。成功后直接进 `/app`，并由工作台承接“已加入 workspace”的确认信息。

---

## 13. 最后的正式建议

如果目标是：

- 尽快可落地
- 降低接入复杂度
- 文档能直接指导开发

那么首版最稳的方案就是：

1. **平台自己负责身份、密码哈希、会话**
2. **平台自己负责 invitation 业务真相和 workspace/workspaceRole 绑定**
3. **Traefik + 平台自有鉴权中间层负责统一前置鉴权**
4. **首版只做本地密码**
5. **种子管理员默认密码为 `admin`，首次登录必须先改密**
6. **首版不做找回密码**
7. **把 invitation 链接定义成一次性首设密码入口**
8. **把管理员首页与工作区跳转分开：`admin` 默认进 `/admin`，非管理员用户默认进 `/app`**
9. **给 `/admin` 一个真正可用的首页，并用聚合接口返回摘要与待办**
10. **把 runtime V1 明确定成 `clawloops_shared + 18789 + rt-<runtimeId> + compat 必填`**
11. **让 Orchestrator 负责异步任务，让 RuntimeManager 只做同步执行器**

这套方案更贴近你当前 MVP 的真实需求，也更容易被前后端直接实现。

---

v0.12-轻量认证修订
reno  
2026-03-25
