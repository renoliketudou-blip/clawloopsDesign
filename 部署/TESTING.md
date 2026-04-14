## 概览

本文件说明如何运行当前仓库下 `apps/clawloops-api` 的自动化测试，并把测试结果接入发布流程。

适用范围：

- 后端自动化测试：`apps/clawloops-api/tests`
- 发布人工自测：`docs/部署/Release_Checklist.md`
- 发布门禁：`docs/部署/Release_Gates.md`
- 测试记录：`docs/测试/Test_Run_Log.md`

---

## 一、目录与依赖

当前后端目录如下：

- 代码根目录：`apps/clawloops-api/`
- 测试目录：`apps/clawloops-api/tests/`
  - `tests/api/`：HTTP API、smoke、integration
  - `tests/services/`：领域服务与编排服务
  - `tests/core/`：认证等核心逻辑
  - `tests/repositories/`：仓储层

环境要求：

- Python `>=3.11`
- 推荐在虚拟环境中执行
- 若本机没有 `python` 命令，请使用 `python3`

---

## 二、如何运行后端自动化测试

### 1. 安装依赖

在仓库根目录执行：

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
pip install -e apps/clawloops-api
pip install pytest
```

### 2. 跑完整后端测试

在仓库根目录执行：

```bash
pytest apps/clawloops-api/tests -vv
```

### 3. 跑局部测试

只跑 API 目录：

```bash
pytest apps/clawloops-api/tests/api -vv
```

只跑单个文件：

```bash
pytest apps/clawloops-api/tests/api/test_workspace_entry_api.py -vv
```

失败后快速重试：

```bash
pytest apps/clawloops-api/tests --last-failed
```

若自行安装 `pytest-xdist`，可并行执行：

```bash
pip install pytest-xdist
pytest apps/clawloops-api/tests -n auto
```

---

## 三、当前测试的依赖边界

- 默认情况下，多数测试可在本地直接运行，不依赖完整 Docker 栈。
- 当前测试主要依赖内存仓储、测试夹具与本地 SQLite 回退能力。
- 若未来引入真实外部服务，可通过环境变量覆盖：
  - `CLAWLOOPS_DATABASE_URL`
  - `CLAWLOOPS_RUNTIME_MANAGER_BASE_URL`
  - `CLAWLOOPS_MODEL_GATEWAY_BASE_URL`

注意：

- 自动化测试覆盖的是后端契约、编排与错误码语义。
- Traefik、cookie、workspace 子域、浏览器跳转等仍需要人工验收。

---

## 四、推荐作为发布底线的测试范围

每次发布至少执行一次：

```bash
pytest apps/clawloops-api/tests -vv
```

如果时间不足，至少覆盖以下文件：

- `apps/clawloops-api/tests/test_healthcheck.py`
- `apps/clawloops-api/tests/api/test_auth.py`
- `apps/clawloops-api/tests/api/test_workspace_entry_api.py`
- `apps/clawloops-api/tests/api/test_module6_full_smoke_flow.py`
- `apps/clawloops-api/tests/api/test_runtime_integration.py`
- `apps/clawloops-api/tests/api/test_admin_smoke_governance.py`
- `apps/clawloops-api/tests/api/test_disabled_user_access.py`

这些文件分别覆盖：

- 基础健康检查
- 登录与认证
- workspace-entry 语义
- 用户主链路 smoke
- runtime 编排集成
- 管理端治理 smoke
- disabled 用户错误收口

---

## 五、自动化测试与人工自测的映射

### 1. 自动化测试优先覆盖

- 认证错误码
- admin 权限控制
- runtime 状态与任务轮询
- workspace-entry 的 `ready / reason`
- 模型、凭据、usage 相关后端契约

### 2. 人工自测必须补充

- `docker compose up -d --build` 是否真的能启动
- 主站、API、Runtime Manager 是否可访问
- Traefik 当前真实生效路由是否正确
- 主域与 workspace 子域的 cookie 共享是否正确
- invitation 页面、`/app`、`/admin` 等页面是否按冻结流程工作
- workspace 最终跳转是否符合当前版本预期
- 公共区域页面是否符合角色差异：
  - `user` 仅可列表/新建目录/上传（不覆盖）/下载，且不允许删除
  - `admin` 可覆盖上传、可删除文件与空目录
  - 仅工作台管理员入口操作可影响宿主机公共区
  - OpenClaw 中无论 `user/admin` 对公共文件操作都不应影响宿主机
  - 路径回退越界必须被阻断并提示“不允许再回退”
  - 列表分页固定每页 10 条且目录优先排序

建议执行顺序：

1. 先跑 `pytest`
2. 再执行 `docs/部署/Release_Checklist.md`
3. 最后把结果写入 `docs/测试/Test_Run_Log.md`

---

## 六、常见问题

### 1. `ModuleNotFoundError`

请确认已经执行：

```bash
pip install -e apps/clawloops-api
```

### 2. SQLite 文件或权限问题

默认 SQLite 文件会落在当前项目目录附近。若出现历史脏数据或权限问题，可删除旧文件后重试。

### 3. 测试通过但页面仍异常

这通常说明问题在 Docker、Traefik、Cookie、域名或浏览器入口层，而不是后端单元/集成测试本身。此时应回到：

- `docs/部署/Environment_Matrix.md`
- `docs/部署/Release_Checklist.md`
- `docs/部署/Deployment_Prerequisites.md`

---

## 七、发布记录要求

发布前至少记录以下信息：

- 执行命令
- 通过数 / 失败数
- 失败测试是否阻断
- 本次是否接受已知偏差

建议直接写入：

- `docs/测试/Test_Run_Log.md`

