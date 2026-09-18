# DataMind AI

DataMind AI 是一个自然语言数据分析 Demo。用户上传 CSV 或 XLSX 后，可以直接用中文提问；后端让 DeepSeek 生成受限的 MySQL 查询，再返回可核对的结论、结果表和 ECharts 图表。

## 当前状态

当前版本为可运行的 `v0.1 Demo`，已经完成：

- CSV、XLSX 上传、字段识别和前 20 行预览；
- 上传数据写入 Docker 中的 MySQL 8.4；
- DeepSeek Tool Calling 生成单表只读 SQL；
- SQL 白名单、数据表隔离、10 秒查询限制和 200 行结果限制；
- 中文结论、柱状图、折线图、结果表及 SQL 依据；
- 内置合成销售数据，可完成比赛演示闭环。

当前边界：单机、单用户、单数据集问答；暂不支持登录、历史记录、跨表分析、复杂数据清洗和预测。

## 架构

```text
Vue 3 + Vite
      │ /api
      ▼
FastAPI ── DeepSeek API
      │
      ▼
Docker MySQL 8.4
```

| 层 | 技术 | 运行方式 |
| --- | --- | --- |
| 前端 | Vue 3、TypeScript、ECharts、Vite | 本机进程 |
| 后端 | Python、FastAPI、Polars、PyMySQL | 本机虚拟环境 |
| 数据库 | MySQL 8.4 | Docker |
| 模型 | DeepSeek Chat Completions / Tool Calling | 云端 API |

## 首次启动

要求：Docker、Python 3.11+、Node.js 22+、[uv](https://docs.astral.sh/uv/) 和 Corepack。

### 1. 配置

```bash
cp .env.example .env
```

编辑 `.env`，填写自己的 `DEEPSEEK_API_KEY`。不要把 `.env` 或密钥发到群里、截图或提交到 Git。

### 2. 启动 MySQL

```bash
docker compose up -d
```

### 3. 启动后端

```bash
uv venv .venv --python 3.11
uv pip install --python .venv/bin/python -r backend/requirements.txt
.venv/bin/uvicorn app.main:app --app-dir backend --reload --port 8000
```

后端接口文档：<http://localhost:8000/docs>

### 4. 启动前端

另开终端：

```bash
cd frontend
mkdir -p ~/.cache/node/corepack
corepack pnpm install
corepack pnpm dev
```

前端页面：<http://localhost:5173>

### 5. 停止

前后端终端按 `Ctrl+C`，数据库执行：

```bash
docker compose down
```

MySQL 数据保存在 Docker 卷 `datamind-ai_mysql_data`。需要连数据一起清空时才执行 `docker compose down -v`。

## 演示流程

1. 点击“使用示例数据”。
2. 提问“哪个地区销售额最高？按销售额排序展示各地区”。
3. 核对华东合计 1,280,000 元，并展开 SQL。
4. 提问“2026 年每个月的总销售额趋势如何？”。
5. 提问不存在的“客户年龄”，验证系统会说明字段不足。

## 当前 API

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| GET | `/api/health` | 数据库与模型配置状态 |
| POST | `/api/datasets` | 上传 CSV / XLSX |
| POST | `/api/datasets/sample` | 导入内置销售数据 |
| POST | `/api/analyses` | 根据数据集和问题执行分析 |

## 目录

```text
backend/app/main.py       当前 FastAPI、数据导入、SQL 与模型链路
frontend/src/App.vue      当前工作台页面
datasets/sales.csv        合成演示数据
compose.yaml              MySQL
docs/demo-design.md       Demo 设计与验收标准
docs/team-development.md  三人分工、Git 规则与路线图
docs/feature-collaboration-demo.md  新功能三人协作演练
docs/member-handoff.md    可直接转发的新成员交接说明
```

## 协作

开始开发前先阅读 [三人协作与开发边界](docs/team-development.md)。接口或共享数据结构的修改必须先在 PR 中说明影响，不能直接提交到 `main`。
