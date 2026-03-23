# CrewClaw 前端交付文档（中文 / Markdown）

本压缩包包含以下 4 份面向 UI 设计与前端开发的文档：

1. `1_routes_permissions.md`：前端页面清单与路由权限文档
2. `2_page_interaction_states.md`：页面级交互状态文档
3. `3_components_ui_spec.md`：前端组件 / UI 规范文档
4. `4_api_page_data_mapping.md`：接口到页面的数据映射文档

## 适用范围

- 首版身份系统：Authentik
- 首版登录方式：仅本地账号密码
- 首版接入方式：邀请制
- 前端角色：普通用户、管理员
- 前端边界：不自建密码体系，不自行保存密码，不绕开 Authentik 登录

## 依据的后端基线

- `Architecture_Design.md`
- `MVP_Contract.md`
- `API_Spec.md`
- `AUTHENTIK_Implementation_Guide.md`

## 推荐阅读顺序

1. 先看路由与权限，明确页面范围。
2. 再看页面交互状态，明确状态机与异常处理。
3. 再看组件 / UI 规范，统一视觉与交互语言。
4. 最后看接口映射，进入开发联调。
