<div align="center">

# RA-Agent · 科研智能助手

**基于 FastAPI + LangGraph 的全栈 RAG Agent 后端服务**
*A RAG agent backend: streaming chat, hybrid retrieval, long-term memory, MCP tools, multi-provider LLM, and three multi-agent teams (seminar / debate / deep research).*

FastAPI · LangGraph · ChromaDB · PostgreSQL · BM25 · MCP

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](https://www.python.org/) [![FastAPI](https://img.shields.io/badge/FastAPI-0.140-009688)](https://fastapi.tiangolo.com/) [![LangGraph](https://img.shields.io/badge/LangGraph-1.2-1C3C3C)](https://langchain-ai.github.io/langgraph/) [![Tests](https://img.shields.io/badge/tests-260%20cases-brightgreen)](#运行测试) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

</div>

---

## 项目简介

ra-agent 是一个「科研助手」风格的 **RAG Agent 后端服务**（配套单页前端 `ra-web/`，由本服务同源托管），把以下能力组装成一个完整产品：

- **多轮对话**：LangGraph 状态机编排，SSE 流式输出（打字机效果），Postgres checkpoint 持久化会话、跨设备同步；支持分支会话、重新生成（rewind）、**中途停止生成**（部分答复保留）、上下文占用展示与自动压缩。
- **RAG 知识库**：上传 PDF / DOCX / TXT / MD → 解析 → 分块 → 向量化入 Chroma；对话时由 LLM 判断「是否检索、查哪个库」，支持**向量 + BM25 混合检索（RRF 融合）**与**父块聚合返回**；嵌入模型按库可配、可免解析重建。
- **MCP 工具调用**：外部 MCP Server 工具目录**运行时动态发现**（新增工具零代码改动），加进程内「原生工具」（按用户自动隔离），generate ⇄ tool_executor 循环直到不再需要工具。
- **长期记忆**：每轮结束后 LLM 自动抽取「值得记住的用户信息」，core / short 分层注入 prompt；超限触发「主题压缩 → LRU 淘汰」的膨胀控制管线。
- **多厂商 LLM 接入**：OpenAI 兼容协议接入 10+ 厂商（openai / deepseek / qwen / moonshot / zhipu / siliconflow / minimax / openrouter / gemini / ollama 及任意自建端点）；用户级配置落库、api_key AES 加密、指数退避重试、上下文窗口自动探测。
- **多 Agent 团队**：三支可切换的多 Agent 协作团队——「格致会讲」（研讨式：四学者研读汇报、承接式问答、构想工作坊、独立评审打分，产出研究构想组合）、「争鸣社」（辩论式：四辩手立场碰撞、主持人调度多轮辩论、撰稿人收敛科研方案）与「溯源社」（深度调研式：初调 → 分章深研 → 修订 → 产出结构化研究报告）；支持按角色绑定不同模型、**会后追问**（点名角色作答）、成稿自动入库形成跨会话知识飞轮。
- **用户体系与可观测性**：注册 / 登录（bcrypt + JWT）、公共/私人两级知识库可见性与越权防护；调用链追踪、事件流推送、token 用量计量。**

## 功能特性

| 模块         | 能力                                                                                                                            |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------- |
| 对话         | SSE 流式输出、思考过程流式展示、生成中断（部分答复保留）、分支会话、重新生成、上下文占用进度 + 达阈值自动压缩                                                                    |
| 知识库        | PDF/DOCX/TXT/MD 入库、逐文件事务与进度明细、入库/重建/复制后台任务可取消、知识库复制与分类、每库独立嵌入模型                                                               |
| 检索         | 纯向量 / 向量+BM25 混合两种模式、RRF 融合排序、父块聚合返回、检索参数前后端可调                                                                                |
| 记忆         | core/short 分层注入、short 层 TTL 过期、条数上限控制、主题压缩 → LRU 淘汰逐级降级                                                                       |
| 工具         | MCP stdio 多 server 管理、连接失败降级空目录、原生工具按用户隔离（知识库检索 / 列库内文件 / 取完整原文 / 学术检索 / 引文追溯 / GitHub / 数据集 / 文档保存 / BibTeX 导出等）             |
| 多 Agent 团队 | 格致会讲（研讨式：研读笔记 → 议程式汇报问答 → 构想卡谱系 → 独立评审聚合）、争鸣社（辩论式：立场书 → 主持人调度辩论 → 收敛成稿）、溯源社（深度调研式：初调 → 分章深研 → 修订 → 报告产出）三支团队前端一键切换；每角色可绑定不同模型、token 预算硬熔断（撰稿豁免）、工具失败自动降级、成稿自动入库 |
| LLM        | 多厂商目录动态拉取模型列表、用户级 base_url/model/api_key、密钥 AES 落库加密、限流与额度耗尽区分处理、KV prefix cache 命中率工程化                                       |
| 运维         | /health 健康检查、启动孤儿状态自愈（中断任务复位）、幂等建表补列、Alembic 迁移、token 用量报表                                                                    |

## 架构总览

```
            ┌────────────────────────────────────────────┐
            │  API 层  app/api/                           │  ← HTTP 契约：校验 / 状态码 / SSE
            │  auth chat kbs conversations memories       │
            │  llm_config settings traces usage feedbacks │
            │  brainstorm seminar deep_research           │  ← 多 Agent 团队（争鸣社/格致会讲/溯源社）
            └──────────────┬─────────────────────────────┘
            ┌──────────────▼─────────────────────────────┐
            │  编排层  app/graph/                         │  ← LangGraph 状态机
            │  主图：记忆加载→压缩→路由→检索→生成⇄工具循环 │     8 节点 + 条件路由
            │  子图：争鸣社辩论 / 格致会讲研讨 / 溯源社深研 │     议程驱动 + Send fan-out
            │  （agent_runtime 共享运行时：空完成重试/     │
            │   强制收尾轮/每角色独立模型）                │
            │  →记忆抽取→保存；WorkflowContext 依赖注入   │
            └──┬───────────┬───────────┬─────────┬───────┘
     ┌─────────▼───┐ ┌─────▼─────┐ ┌───▼────┐ ┌──▼─────────┐
     │ 服务层       │ │ 抽象层     │ │ MCP 层 │ │ 核心层      │
     │ kb_service   │ │ llm       │ │ host   │ │ db/jwt     │
     │ memory_svc   │ │ embedding │ │ adapter│ │ crypto     │
     │ parsing      │ │ vectorst. │ │        │ │ tracing    │
     │              │ │ bm25      │ │        │ │ events/... │
     └──────┬───────┘ └─────┬─────┘ └────────┘ └─────┬─────┘
     ┌──────▼───────────────▼───────────────────────▼─────┐
     │ 数据层：PostgreSQL(元数据/checkpoint) + Chroma(向量)   │
     │         + 本地磁盘(chunk 文本)                        │
     └─────────────────────────────────────────────────────┘
```

分层依赖单向向下：`api → graph → services / abstractions / mcp → core → models`。所有长生命周期组件在 `main.py` 的 lifespan 里一次性构建并挂到 `app.state`——装配点唯一，测试与生产同构。

## 技术栈

| 类别       | 选型                                        | 用途                         |
| -------- | ----------------------------------------- | -------------------------- |
| Web 框架   | FastAPI + uvicorn                         | REST API + SSE 流式          |
| Agent 编排 | LangGraph + langgraph-checkpoint-postgres | 状态机工作流 + 会话 checkpoint     |
| LLM 接入   | langchain-openai（ChatOpenAI）              | OpenAI 兼容协议多厂商             |
| 向量库      | ChromaDB（PersistentClient）                | 每库一个 collection            |
| 稀疏检索     | rank-bm25                                 | 混合检索的 BM25 腿               |
| 关系库      | PostgreSQL 16 + SQLAlchemy 2 + Alembic    | 业务元数据 + checkpoint 存储      |
| 缓存       | Redis 7                                   | 已预留（当前主链路未强依赖）             |
| 文档解析     | pypdf / python-docx                       | PDF 逐页 / DOCX 段落           |
| 工具协议     | mcp + langchain-mcp-adapters              | MCP Host / Client，stdio 传输 |
| 安全       | bcrypt / PyJWT / cryptography(Fernet)     | 密码哈希 / token / 密钥加密        |
| 配置       | pydantic-settings + PyYAML                | 环境变量 > yaml > 默认值 三级合并     |

## 快速开始

### 前置要求

- Docker & Docker Compose（启动 Postgres 16 + Redis 7）
- Python 3.10+
- 一个 OpenAI 兼容的 LLM API Key（deepseek / qwen / openai 等任意一家即可）

### 启动步骤

```bash
# 1. 克隆并进入项目
git clone https://github.com/all8279/ra-agent.git
cd ra-agent

# 2. 准备环境变量：至少配置一个 LLM_API_KEY
cp .env.example .env && vim .env

# 3. 启动基础设施（PostgreSQL 16 + Redis 7）
docker compose up -d

# 4. 安装依赖（建议放在项目根的 .venv 下，run.sh 默认使用它）
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 5. 启动服务（二选一）
./run.sh                 # 守护进程方式：start|stop|restart|status，日志 /tmp/ra-agent.log
# 或前台直跑：
.venv/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 8000

# 6. 打开应用
#    浏览器访问 http://localhost:8000 —— 注册账号后即可使用
#    健康检查：curl http://localhost:8000/health
```

首次启动会自动幂等建表 / 补列（无需手工跑迁移；Alembic 用于结构演进）。未配置云端嵌入 Key 时会回退本地 ONNX 嵌入模型。

### 可选配置：MCP 学术检索工具

仓库自带示例 MCP Server（`servers/research_server.py`：联网搜索 / 学术检索 / 网页阅读），连接配置见 `config/settings.yaml` 的 `mcp_servers` 段，默认即启用、无额外密钥也能用（Semantic Scholar 免费共享额度，高峰期可能 429）。可选环境变量：

- `S2_API_KEY`：Semantic Scholar 专属 key（免费申请），避免共享限流；
- `OPENALEX_MAILTO`：OpenAlex 礼貌池邮箱，进入更稳定的访问池。

## 配置说明

三级合并，优先级从高到低：**环境变量 / `.env` > `config/settings.yaml` > 代码默认值**。环境已设置的字段不会被 yaml 覆盖——密钥类放 `.env`，行为类参数放 yaml。

关键环境变量：

| 变量                                        | 说明                                                                    |
| ----------------------------------------- | --------------------------------------------------------------------- |
| `DATABASE_URL`                            | Postgres 连接串（默认 `postgresql+psycopg://ra:ra@localhost:5432/ra_agent`） |
| `REDIS_URL`                               | Redis 连接串                                                             |
| `LLM_API_KEY`                             | 系统级默认模型的 API Key                                                      |
| `EMBEDDING_API_KEY`                       | 云端嵌入模型密钥（不配则走本地 / 自建端点）                                               |
| `EMBEDDING_DEFAULT_PROVIDER`              | `local` / `doubao` / `ollama` 等                                       |
| `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` | 出网代理（无代理环境留空即可）                                                       |

完整行为参数（检索 k 值、父块聚合、LLM 重试、上下文窗口、嵌入端点目录等）均在 `config/settings.yaml`，带注释可直接改。

## 目录结构

```
ra-agent/
├── app/                     # 应用主体（分层核心）
│   ├── main.py              # FastAPI 入口 + lifespan 组件装配
│   ├── settings.py          # 三级配置合并加载
│   ├── api/                 # API 层：HTTP 端点与 SSE
│   ├── graph/               # 编排层：主图 + 争鸣社/格致会讲/溯源社子图 + agent_runtime 共享运行时
│   ├── api/brainstorm.py    # 争鸣社 API（SSE/停止/列表/追问）
│   ├── api/seminar.py       # 格致会讲 API（同构）
│   ├── api/deep_research.py # 溯源社 API（同构）
│   ├── services/            # 服务层：kb_service / memory_service / parsing
│   ├── abstractions/        # 抽象层：llm / embedding / vectorstore / bm25 四大接口
│   ├── mcp/                 # MCP 层：host（连接发现）/ adapter（执行适配）
│   ├── core/                # 核心层：db/jwt/security/crypto/tracing/events/errors...
│   └── models/              # SQLAlchemy ORM 实体
├── config/settings.yaml     # 行为参数配置（主题式嵌套）
├── servers/research_server.py   # 示例 MCP Server（联网搜索/学术检索/网页阅读）
├── alembic/                 # 数据库迁移
├── tests/                   # pytest 全量测试（30 个文件、260 个用例）
├── ra-web/index.html        # 单页前端（服务同源托管）：对话/知识库/双团队多 Agent/记忆/追踪
├── docker-compose.yml       # 基础设施：Postgres 16 + Redis 7
├── Dockerfile               # 应用镜像（python:3.10-slim）
├── run.sh                   # 服务启停脚本
└── .env.example             # 环境变量模板
```

## 运行测试

测试全程**离线、确定性**：强制本地嵌入 + 临时数据目录 + 独立测试库（绝不触碰开发库的数据）。

```bash
# 1. 创建一次专用测试数据库（只需做一次）
docker exec ra-postgres psql -U ra -d ra_agent \
  -c "CREATE DATABASE ra_agent_test OWNER ra"

# 2. 运行全部测试
DATABASE_URL=postgresql+psycopg://ra:ra@localhost:5432/ra_agent_test pytest tests/ -q
```

## 实测效果

> 以下数据来自本地实测，测试环境：Windows 11、Python 3.11、PostgreSQL 16 + ChromaDB、DeepSeek-chat、本地 ONNX 嵌入（all-MiniLM-L6-v2）。

### 检索消融实验

为验证「混合检索」相比「纯向量召回」的真实收益，构造标注评测集并对两种模式做对照：

- **评测集**：22 份文档（12 份结构高度相似的服务运维档案 + 10 份噪音），12 个标注问题（均为 paraphrase 问法，答案埋在文档中的错误码 / 工号 / 端口号），判定 top-3 含正确块为命中。
- **结果**：

| 模式 | 命中率 | 检索 P50 |
| ---- | ---- | ------ |
| 纯向量召回 | 8%（1/12） | ~300 ms |
| 向量 + BM25 混合（RRF） | 67%（8/12） | ~300 ms |

- **结论**：在高度相似文档场景下，纯向量因语义等距命中率接近随机；引入 BM25 后命中显著提升，且 BM25 为内存计算、**零额外延迟**。4 个未命中集中在中文分词场景（见 Roadmap）。

### 成本与稳定性

| 指标 | 数值 |
| ---- | ---- |
| LLM 调用成功率 | 100%（重试包装生效） |
| DeepSeek 上下文缓存命中率 | 78.9% |
| 单轮 RAG 对话成本 | 约 ¥0.001 |
| 检索链路 P50 | ~300 ms（瓶颈为本地嵌入推理） |

## Roadmap

- [ ] 检索评估自动化接入 CI
- [ ] 中文检索分词优化（jieba / n-gram，消融实验暴露的短板）
- [ ] 更多向量库后端（Milvus / Qdrant）
- [ ] MCP over HTTP(S) 传输支持
- [ ] 多语言文档界面
