# CrewClaw 测试执行记录

## 使用说明

每次发布前复制以下模板，记录一次完整测试执行结果。

若本次只做小版本修复，也建议至少填写：

- 发布信息
- 执行范围
- 结果摘要
- 风险接受
- 最终结论

---

## 模板

### 1. 发布信息

| 项目 | 内容 |
| --- | --- |
| 版本 / Tag |  |
| 分支 / Commit |  |
| 执行环境 |  |
| 执行日期 |  |
| 执行人 |  |
| 复核人 |  |

### 2. 参考文件

- 发布门禁：`docs/部署/Release_Gates.md`
- 发布清单：`docs/部署/Release_Checklist.md`
- 环境矩阵：`docs/部署/Environment_Matrix.md`
- 后端测试说明：`apps/clawloops-api/TESTING.md`

### 3. 本次测试范围

| 类别 | 是否执行 | 备注 |
| --- | --- | --- |
| 安装与启动便捷性检查 |  |  |
| 环境与网关检查 |  |  |
| 管理员主链路 |  |  |
| 普通用户主链路 |  |  |
| 负向与回归 |  |  |
| 后端 `pytest` |  |  |

### 4. 环境摘要

| 项 | 实际值 |
| --- | --- |
| `CLAWLOOPS_DOMAIN` |  |
| `RUNTIME_MANAGER_DOMAIN` |  |
| `CLAWLOOPS_MODEL_GATEWAY_BASE_URL` |  |
| 默认模型 |  |
| 是否启用 HTTPS |  |
| Cookie `Domain` |  |
| Traefik 生效 provider |  |

### 5. 关键结果摘要

| 检查项 | 结果 | 备注 |
| --- | --- | --- |
| 默认服务成功启动 |  |  |
| 管理员首次登录与强制改密通过 |  |  |
| invitation 创建与接入通过 |  |  |
| 用户工作台与 runtime 状态通过 |  |  |
| workspace 入口通过 |  |  |
| disabled / 403 / invitation 负向分支通过 |  |  |
| 后端 `pytest` 通过 |  |  |

### 6. 自动化测试记录

```bash
pytest apps/clawloops-api/tests -vv
```

| 项 | 内容 |
| --- | --- |
| 执行命令 |  |
| 通过数 |  |
| 失败数 |  |
| 跳过数 |  |
| 失败文件 / 用例 |  |
| 是否阻断发布 |  |

### 7. 问题清单

| 编号 | 问题描述 | 严重级别 | 是否阻断 | 负责人 | 状态 |
| --- | --- | --- | --- | --- | --- |
| 1 |  |  |  |  |  |

### 8. 已知偏差接受记录

| 偏差项 | 本次是否接受 | 风险说明 | 补救措施 | 回收时间 | 负责人 |
| --- | --- | --- | --- | --- | --- |
| `browserUrl` 仍可能为 hostPort 直连 |  |  |  |  |  |
| OpenClaw token 仍以过渡方案参与访问 |  |  |  |  |  |
| `allowInsecureAuth` 等危险开关仍存在 |  |  |  |  |  |
| LiteLLM 密钥仍采用单密钥对齐策略 |  |  |  |  |  |

### 9. 最终结论

| 结论 | 勾选 | 说明 |
| --- | --- | --- |
| `Go` |  |  |
| `Go with Risk` |  |  |
| `No-Go` |  |  |

结论说明：

```text
在这里写最终判断、未解决问题、是否允许发布。
```

---

## 示例

```text
版本: v0.14.1
环境: staging
结果: Go with Risk
原因: 主链路通过，pytest 通过；但 browserUrl 仍为 hostPort 过渡方案，本次接受并计划在下个版本回收。
```
