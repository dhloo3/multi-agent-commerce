# 多 Agent 商品选购与交易协同系统

基于 Python、AgentScope、FastAPI、AG-UI 和 React 的商品选购演示系统。买家输入用途、预算和收货地后，Agent 检索样例商品、比较候选，并在人工确认后操作本地订单账本。

## 技术栈

Python 3.11–3.13、AgentScope 2.x、FastAPI、AG-UI、React、TypeScript、Qdrant、SQLite；Redis 和 OpenTelemetry 按配置启用。

## 核心链路

1. **按需协作**：MainAgent 可直接调用商品、知识及交易工具；复杂任务通过 `task_dispatch` 派发给新建的 SearchAgent 或 TradeAgent，子 Agent 使用独立上下文并返回结论与证据。
2. **商品检索**：Embedding → Qdrant 向量召回 → Reranker 重排 → 预算、收货地、库存等结构化约束过滤。BM25＋向量混合召回可配置，默认关闭。精确商品或 SKU ID 直接查样例目录。
3. **上下文与记忆**：已读取的历史工具结果可归档并按引用回查；长期偏好按买家持久化到 SQLite，正向偏好通过 Embedding 语义召回，负向约束保留。
4. **网页交互**：React 通过 AG-UI 的 POST＋SSE 接入 FastAPI，展示流式回复、工具进度、商品卡及确认操作。运行事件写入 SQLite，支持按游标重放和刷新后恢复已保存的会话状态。
5. **可靠性与观测**：订单、库存及确认状态使用 SQLite 事务；写操作配合确认、幂等和会话版本控制。工程包含 Harness 评测入口及可选 OpenTelemetry 追踪。

网页 AG-UI 请求在 API 进程中执行。Redis 队列用于旧意图接口和异步任务。

## 目录

| 路径 | 用途 |
| --- | --- |
| `app/` | Agent、业务用例、FastAPI 与基础设施适配 |
| `frontend/` | React 页面及 AG-UI 客户端 |
| `data/catalog-v3.jsonl`、`knowledge/` | 样例商品目录与品类知识 |
| `configs/`、`docker/` | 模型配置与容器编排 |
| `tests/`、`eval/`、`scripts/` | 自动化测试、固定评测数据与运行脚本 |

## 本机启动

需要 Python 3.11–3.13、[uv](https://docs.astral.sh/uv/)、Node.js 22、npm，以及支持 OpenAI 兼容接口和工具调用的模型服务。Embedding 服务也必须可用；不配置独立 Embedding 网关时复用 LLM 网关。Reranker 服务未配置或不可用时会降级为向量排序。

```powershell
uv sync --frozen
npm --prefix frontend ci
Copy-Item .env.example .env
```

仅在首次配置时复制 `.env.example`；已有 `.env` 请直接编辑。至少设置实际可用的 `LLM_BASE_URL`、`LLM_API_KEY`、`LLM_MODEL` 和 `EMBEDDING_MODEL`。不要提交 `.env` 或真实数据库。未设置 `QDRANT_URL` 时使用本地嵌入式 Qdrant，无需另起向量服务。

分别在两个终端执行：

```powershell
uv run python -m uvicorn app.presentation.server:app --host 127.0.0.1 --port 8000 --workers 1
```

```powershell
npm --prefix frontend run dev -- --host 127.0.0.1 --port 5173 --strictPort
```

打开 `http://127.0.0.1:5173`。先访问 `http://127.0.0.1:8000/health` 检查后端，再用页面发送一次真实选购请求，核对回复、商品卡和刷新恢复。健康检查不等于模型或 Reranker 已可用。

如需容器环境，参照 `docker/docker-compose.yaml`。Compose 会启动 API、前端、Qdrant、Redis 和 worker；外部模型服务及凭据仍需自行配置。

## 验证

```powershell
$env:LLM_API_KEY = "unit-test-only-key"
$env:LLM_BASE_URL = "http://127.0.0.1:9/v1"
uv run python -X utf8 -m pytest -q tests/test_ag_ui.py tests/test_ag_ui_journal.py tests/test_ag_ui_confirmations.py tests/test_context_governance.py tests/test_memory_persistence.py tests/test_subagent_preference_inject.py tests/test_trade_store.py
npm --prefix frontend test
npm --prefix frontend run build
```

上述后端命令使用不可达的本地模型地址，只验证核心代码，不会执行真实模型链路；运行后请新开终端启动服务，避免假凭据覆盖 `.env`。Harness 评测使用 `make eval-harness`，需要可用的模型凭据。
