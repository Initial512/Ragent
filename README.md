<div align="center">

# Ragent AI

**企业级多智能体 GraphRAG 知识库助手**

[中文](README.md) · [English](README.en.md)

基于 LangGraph Supervisor-Workers 架构构建的全栈检索增强生成平台，具备 GraphRAG 语义网络能力。支持多智能体协作、混合（向量 + 图）检索、人工介入（HITL）中断/恢复，以及实时流式回答。

![Python](https://img.shields.io/badge/Python-3.12+-3776AB?style=flat&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115+-009688?style=flat&logo=fastapi&logoColor=white)
![Vue.js](https://img.shields.io/badge/Vue.js-3-4FC08D?style=flat&logo=vuedotjs&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-1.1-1C3C3C?style=flat&logo=langchain&logoColor=white)
![Milvus](https://img.shields.io/badge/Milvus-2.5-00A1E0?style=flat&logo=milvus&logoColor=white)
![Neo4j](https://img.shields.io/badge/Neo4j-5.26-4581C3?style=flat&logo=neo4j&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat&logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

<br/>

![Ragent AI 界面](docs/img.png)

</div>

---

## 目录

- [项目概览](#项目概览)
- [架构与流程](#架构与流程)
- [核心能力](#核心能力)
- [技术栈](#技术栈)
- [项目结构](#项目结构)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [API 参考](#api-参考)
- [版本路线图](#版本路线图)

---

## 项目概览

Ragent AI 是一套可用于生产环境的**多智能体 GraphRAG 平台**。它将私有文档检索、网络搜索、结构化数据分析和**知识图谱遍历**组合起来，由专用 AI Worker 回答用户问题。平台采用 **LangGraph Supervisor-Workers** 架构，输出可溯源、可审计的答案。

主要能力包括：

- **多智能体协作**：Supervisor 负责意图识别与调度；支持 RAG、局部图搜索、全局图搜索、网络搜索、数据分析和直接回答等专用智能体，并可并行分发任务。
- **GraphRAG 语义网络**：文档入库时由 LLM 抽取实体与关系，使用 Neo4j 存储图谱，借助 Leiden 社区聚类与分层摘要形成全局语义视图。
- **图向量混合检索**：稠密向量（Qwen text-embedding-v1，1536 维）、BM25 稀疏向量和图三元组通过三通道 RRF 融合；支持局部搜索（向量检索后图扩展）与全局搜索（社区摘要匹配）。
- **三级分层切块**：L1/L2/L3 分别为 1200/600/300 字符，使用滑动窗口和自动合并检索器；L2 用于图谱抽取，L3 用于向量索引。
- **人工介入（HITL）**：低置信度 RAG 检索和高风险 SQL 均可通过 LangGraph `interrupt()` 暂停，待人工确认后恢复执行。
- **状态持久化**：基于 MySQL 的 LangGraph checkpointer 跨会话保存状态，支持长时间中断/恢复。
- **实时流式输出**：基于 SSE 的 token 流和 Trace Canvas，实时展示智能体状态及 RAG/图检索步骤。
- **双主题界面**：Gemini 风格的明暗主题、实时多智能体追踪面板和 HITL 审批弹窗。

---

## 架构与流程

### 总体架构


<!-- 保留旧版图示源文本供变更记录使用；不在 Markdown 中渲染。
┌────────────────────────────────────────────────────────────────────────┐
│                         前端（Vue 3 SPA）                               │
│     聊天 UI · 会话管理 · 知识库上传                                    │
│     Trace Canvas（智能体状态 + 时间线）· HITL 审批弹窗                  │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │ SSE / HTTP
┌──────────────────────────────▼─────────────────────────────────────────┐
│                         FastAPI 应用层                                  │
│  ┌────────────────────────┐  ┌─────────────────────────────────────┐  │
│  │  api/routes.py          │  │  schemas.py                         │  │
│  │  REST + SSE 接口        │  │  Pydantic 请求/响应模型             │  │
│  └───────────┬─────────────┘  └─────────────────────────────────────┘  │
│              │                                                          │
│  ┌───────────▼──────────────────────────────────────────────────────┐  │
│  │        LangGraph Supervisor-Workers 编排器（v8）                 │  │
│  │                                                                   │  │
│  │                    ┌──────────────┐                               │  │
│  │                    │ Supervisor   │（意图路由）                   │  │
│  │                    └──────┬───────┘                               │  │
│  │        ┌──────────────────┼─────────────────────────────────┐    │  │
│  │        │                  │                                 │    │  │
│  │ ┌──────▼──────┐ ┌────────▼───────┐ ┌─────────▼─────────┐   │    │  │
│  │ │ RAG 专家    │ │ 局部图搜索      │ │ 全局图搜索         │   │    │  │
│  │ └──────┬──────┘ └────────┬───────┘ └─────────┬─────────┘   │    │  │
│  │        │                  │                   │             │    │  │
│  │ ┌──────▼──────┐ ┌────────▼───────┐ ┌─────────▼─────────┐   │    │  │
│  │ │ 网络搜索器   │ │ 数据分析师      │ │ 直接回答 → END    │   │    │  │
│  │ │（Tavily）   │ │（Text-to-SQL） │ │（跳过 Critique）   │   │    │  │
│  │ └──────┬──────┘ └────────┬───────┘ └───────────────────┘   │    │  │
│  │        └──────────────────┼─────────────────────────────┘   │    │  │
│  │                           │                                  │    │  │
│  │                    ┌──────▼───────┐                          │    │  │
│  │                    │ Synthesize    │ ← 多 Worker 合并         │    │  │
│  │                    └──────┬───────┘                          │    │  │
│  │                           │                                  │    │  │
│  │                    ┌──────▼───────┐                          │    │  │
│  │                    │ Critique      │ ← 事实核查（v8）          │    │  │
│  │                    └──────┬───────┘                          │    │  │
│  │                  有效 │   │ 无效，重试 < 2                    │    │  │
│  │                   ┌────┘   └────┐                            │    │  │
│  │                   ▼             ▼                            │    │  │
│  │                  END        ┌─────────┐                       │    │  │
│  │                             │ Replan  │ → Supervisor（v8）     │    │  │
│  │                             └─────────┘                       │    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                             │                                            │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │                       RAG 管线（LangGraph）                       │  │
│  │ retrieve → grade → [rewrite → retrieve_expanded → grade_v2]      │  │
│  │                     ↑ 第 2 次失败时 force_interrupt               │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                            │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │                            检索引擎                                │  │
│  │ 混合向量检索 · 重排 · 自动合并 · 查询改写                          │  │
│  │ 局部图搜索（向量 → 一跳扩展）· 全局图搜索（社区摘要匹配）          │  │
│  │ 三通道 RRF 融合（Dense + Sparse + Graph）                         │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┬──────────────┐
        │                      │                      │              │
┌───────▼───────┐  ┌───────────▼──────────┐  ┌───────▼───────┐  ┌──▼─────┐
│    Milvus     │  │       MySQL          │  │     Redis     │  │ Neo4j  │
│   向量数据库  │  │ 会话 · 消息          │  │   热缓存      │  │ 图存储 │
│ HNSW + Sparse │  │ 块 · 摘要            │  │  HITL 锁     │  │        │
│   + 摘要      │  │ 图检查点              │  │              │  │        │
└───────────────┘  └──────────────────────┘  └───────────────┘  └────────┘
```

```text
Vue 3 单页应用
  └─ 聊天界面、会话管理、知识库上传、Trace Canvas、HITL 审批
                         │ SSE / HTTP
FastAPI 应用层
  └─ REST/SSE 接口、Pydantic 请求与响应模型
                         │
LangGraph Supervisor-Workers 编排器
  ├─ Supervisor：意图路由与并行派发
  ├─ Planner：复杂问题拆解（v8）
  ├─ RAG Specialist：文档检索
  ├─ Local / Global Graph Search：图谱检索
  ├─ Web Searcher：Tavily 网络搜索
  ├─ Data Analyst：Text-to-SQL
  ├─ Direct Answer：闲聊与通用问题
  ├─ Synthesize：汇总多个 Worker 的结果
  └─ Critique → Replan：事实核查与最多两次自我纠正
                         │
检索引擎
  └─ 混合向量检索、重排、自动合并、查询改写、局部/全局图检索、RRF 融合
                         │
Milvus（向量库） · MySQL（会话、文档、检查点） · Redis（缓存、HITL 锁） · Neo4j（图谱）
```
-->

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Frontend (Vue 3 SPA)                                                 │
│ Chat UI · Sessions · Knowledge Upload · Trace Canvas · HITL Modal    │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │ SSE / HTTP
┌──────────────────────────────────▼───────────────────────────────────┐
│ FastAPI Application Layer                                            │
│ api/routes.py: REST + SSE · schemas.py: Pydantic Models              │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
┌──────────────────────────────────▼───────────────────────────────────┐
│ LangGraph Supervisor-Workers Orchestrator (v8)                       │
│ Supervisor → [RAG | Local Graph | Global Graph]                      │
│            → [Web Search | Data Analyst | Direct Answer → END]       │
│ Workers → Synthesize → Critique → valid: END | retry < 2: Replan     │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
┌──────────────────────────────────▼───────────────────────────────────┐
│ RAG Pipeline (LangGraph)                                             │
│ retrieve → grade → rewrite → retrieve_expanded → grade_v2            │
│ force_interrupt on second failure                                    │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
┌──────────────────────────────────▼───────────────────────────────────┐
│ Retrieval Engine                                                     │
│ Hybrid Search · Reranking · Auto-Merge · Local/Global Graph · RRF    │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
                  ┬────────────────┬────────────────┬────────────────┬
                  │                │                │                │
                  ▼                ▼                ▼                ▼
           ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
           │ Milvus      │  │ MySQL       │  │ Redis       │  │ Neo4j       │
           │ Vector DB   │  │ State Store │  │ Cache/HITL  │  │ Graph DB    │
           └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
```

### 智能体路由流程（v8）

用户问题先由 Supervisor 判断意图：复杂问题会先交给 Planner 生成执行计划；随后可路由至一个或多个检索/分析 Worker。结果由 Synthesize 汇总，Critique 使用已检索上下文交叉核查草稿答案。核查失败时，Replan 将缺失信息作为补充查询重新交由 Supervisor 执行，最多重试两次。无需检索的直接回答会跳过核查。

<!-- 保留旧版 ASCII 图示源文本供变更记录使用；不在 Markdown 中渲染。
                         ┌──────────────┐
                         │  用户问题    │
                         └──────┬───────┘
                                │
                     ┌──────────▼──────────┐
                     │    Supervisor       │
                     │    意图路由器       │
                     └──────────┬──────────┘
                                │
          ┌─────────────────────┼─────────────────────────────┐
          │                     │                             │
          ▼                     ▼                             ▼
 ┌─────────────────┐  ┌────────▼────────┐  ┌─────────────────▼──┐
 │  Planner（v8）  │  │ RAG Specialist  │  │ 局部/全局图搜索     │
 │  复杂查询拆解   │  │  文档检索       │  │                    │
 └────────┬────────┘  └────────┬────────┘  └────────┬───────────┘
          │                    │                     │
          └────────────────────┼─────────────────────┘
                               │
          ┌────────────────────┼─────────────────────┐
          │                    │                     │
 ┌────────▼────────┐  ┌────────▼────────┐  ┌─────────▼─────────┐
 │  网络搜索器      │  │  数据分析师      │  │  直接回答          │
 │ （Tavily API）   │  │ （Text-to-SQL） │  │  → END（跳过核查） │
 └────────┬────────┘  └────────┬────────┘  └───────────────────┘
          │                    │
          └────────────────────┼──────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Synthesize       │ ← 多 Worker 结果聚合
                    │     合并答案         │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Critique（v8）    │ ← 事实核查
                    │ 与检索上下文交叉验证 │
                    └──────────┬──────────┘
                               │
                    ┌──────────┼──────────┐
                    │                     │
                  有效              无效，重试 < 2
                    │                     │
                    ▼                     ▼
               ┌─────────┐         ┌───────────┐
               │  答案   │         │  Replan   │ → Supervisor（自我纠正）
               └─────────┘         └───────────┘
```
-->

```mermaid
flowchart TB
    Q[用户问题] --> S[Supervisor：意图路由]
    S --> P[Planner v8：复杂查询拆解]
    S --> R[RAG Specialist：文档检索]
    S --> G[局部 / 全局图搜索]
    S --> W[网络搜索器：Tavily]
    S --> D[数据分析师：Text-to-SQL]
    S --> A[直接回答：跳过核查]
    P --> M[Synthesize：合并答案]
    R --> M
    G --> M
    W --> M[Synthesize：合并答案]
    D --> M
    A --> ANS[答案]
    M --> C[Critique v8：事实核查]
    C -->|有效| ANS
    C -->|无效，重试少于 2 次| RP[Replan：补充检索]
    RP --> S
```

### 文档入库流程

```text
文档上传
  ├─ L1/L2/L3 三级分层切块
  ├─ L3 → Milvus 向量索引
  ├─ L1/L2 → MySQL 父块存储
  └─ L2 文本 → LLM 实体/关系抽取 → Neo4j
       ├─ MERGE 实体节点（名称、类型、描述）
       ├─ MERGE RELATES_TO 边（谓词、权重）
       └─ 将来源 L3 块 ID 绑定到关系边
```

### GraphRAG 离线流程

```text
Neo4j 全量图谱
  ├─ 拉取 Entity 与 RELATES_TO，构建 NetworkX DiGraph
  ├─ Leiden（Louvain）社区发现
  ├─ 将 community_id 写回 Neo4j 实体
  ├─ 按社区收集实体与关系
  ├─ LLM 生成 200–400 词的社区摘要
  └─ 摘要向量化并写入 Milvus + MySQL
```

---

## 核心能力

### 多智能体与检索

| 能力 | 说明 |
|---|---|
| Supervisor 路由 | 基于 LLM 的意图分析，可单路或通过 LangGraph `Send` 并行派发。 |
| Planner（v8） | 将多跳复杂问题拆成可由不同智能体处理的分步计划。 |
| RAG Specialist | 混合检索、重排、自动合并、相关性评分、改写与扩展检索的完整 RAG 流程。 |
| 局部图搜索 | 向量搜索 → Neo4j 实体定位 → 一跳邻居扩展，适合多跳推理。 |
| 全局图搜索 | 在 Milvus 中匹配社区摘要，适合全景与概览类问题。 |
| Web Searcher | 集成 Tavily 实时搜索；API 失败时自动回退至 RAG。 |
| Data Analyst | 探测 schema、生成只读 SQL、执行并呈现数据洞察。 |
| Direct Answer | 无检索开销地处理问候、闲聊和通用知识问题。 |
| Critique / Replan | 将草稿与检索上下文交叉核验；发现幻觉时补充查询并重新路由。 |

### GraphRAG 引擎

| 能力 | 说明 |
|---|---|
| 实体关系抽取 | 入库时从 L2 块中以结构化形式提取主语、谓词、宾语三元组。 |
| 实体去重 | Neo4j `MERGE` 配合实体名唯一约束，避免跨文档重复节点。 |
| 来源溯源 | 每条图关系保存 `source_chunks`（来源 L3 块 ID），可完整追踪证据。 |
| Leiden 聚类 | 将相关实体划分为主题社区，并把社区 ID 回写至 Neo4j。 |
| 社区摘要 | LLM 为每个社区生成摘要，向量化后用于全局搜索。 |
| 三通道 RRF | `RRF_Score = w1/(k+rank_dense) + w2/(k+rank_sparse) + w3/(k+rank_graph)`。 |

### 查询智能与 HITL

| 能力 | 说明 |
|---|---|
| Step-Back Prompting | 对细节问题生成更高层问题，以扩大检索范围。 |
| HyDE | 对模糊或概念性问题生成假设文档，用于语义检索。 |
| 复杂查询扩展 | 组合两种策略处理多步骤问题，并进行去重。 |
| 相关性评分 | LLM 评估召回文档；不相关时触发改写，连续两次失败时触发 HITL。 |
| 低置信度防护 | RAG 评分两次失败后暂停图执行，人工可修改问题或补充上下文。 |
| SQL 安全审查 | Data Analyst 生成非 SELECT SQL 时，须经人工批准或拒绝才会执行。 |
| 会话锁 | Redis 分布式锁避免 HITL 待处理期间的并发消息（HTTP 423）。 |

### 平台与治理能力

- **应用层**：SSE 流式输出、Trace Canvas、MySQL 会话持久化与 Redis 缓存、50 轮后自动压缩历史、可审计的 RAG trace、生成中止、双主题及死循环检测。
- **知识治理（v4）**：跨 MySQL/Milvus/Neo4j 的级联软删除，文档版本与哈希索引，两阶段实体消歧，以及支持 `valid_from`/`valid_to` 的时态 GraphRAG。
- **评测与 CI/CD（v4）**：覆盖 7 种查询类型的 80 条 Golden QA 数据集；支持 `retrieval`、`pipeline`、`e2e`、`graph`、`graph_compare` 评测模式、RRF 网格搜索、A/B 报告和 GitHub Actions 流水线。
- **可观测性与高可用（v5）**：OpenTelemetry 链路、Prometheus 指标、structlog JSON 日志、LLM/Tavily 熔断、Neo4j 超时降级、指数退避重试和 Jaeger/Prometheus/Grafana 监控栈。
- **成本优化（v6）**：Milvus + MySQL 语义缓存、动态模型路由、Redis Singleflight 防击穿、文档删除驱动的缓存失效与并发基准测试。
- **多模态（v7）**：PDF 版面分析、图片/表格抽取与 MinIO 存储、Qwen-VL 描述、视觉检索第四 RRF 通道和多模态专用智能体。
- **MCP 集成（v9）**：支持 SSE/stdio MCP Server、动态 `StructuredTool` 注册、工具语义召回、多数据源分析与 ECharts 图表生成。
- **本体约束（v10）**：11 类实体、12 类关系、70+ 三元组规则；受限提示词、Pydantic 校验及按类型实体消歧。
- **增量与流式图引擎（v11–v13）**：SHA-256 文档指纹、arq 异步入库、Redis Streams 三阶段管线、增量社区重聚类及脏标记摘要重生成。
- **多租户与 SaaS（v14–v15）**：JWT/RBAC、MySQL/Milvus/Neo4j 租户隔离、租户级限流、SLA 感知降级、Token 计量、审计日志与 Alembic 迁移。
- **工作流、推理、记忆、研究（v16–v21）**：DAG 工作流、6 类自适应检索、图推理路径发现与核验、用户记忆图谱、证据驱动的深度研究，以及动态假设/冲突检测循环。

---

## 技术栈

| 层级 | 组件 |
|---|---|
| 后端 | FastAPI、Uvicorn、LangChain、LangGraph、Pydantic、SQLAlchemy |
| 前端 | Vue 3（CDN）、marked.js、highlight.js、Font Awesome |
| 向量库 | Milvus 2.5（HNSW + SPARSE_INVERTED_INDEX） |
| 图数据库 | Neo4j 5.26 Community Edition |
| 嵌入模型 | Qwen text-embedding-v1（1536 维）、BM25（自定义实现） |
| 大模型 | 通过 DashScope OpenAI 兼容 API 调用 Qwen-Plus / Qwen3.6-Plus |
| 数据与缓存 | MySQL 8.0、Redis 7.0 |
| 图算法 | NetworkX、python-louvain（Leiden/Louvain 社区发现） |
| 搜索与评测 | Tavily、Gaode/Amap、Ragas、matplotlib、pytest |
| 可观测性 | OpenTelemetry、Prometheus、Grafana、Jaeger、structlog |
| 基础设施 | Docker Compose、GitHub Actions CI、Dockerfile、arq 异步任务队列 |

---

## 项目结构

```text
Ragent-AI/
├── backend/
│   ├── api/              # FastAPI 应用工厂与 REST/SSE 路由
│   ├── auth/             # 多租户 RBAC、JWT、认证接口
│   ├── agent/            # 编排器、智能体工具、网络搜索、数据分析、MCP
│   ├── rag/              # RAG 工作流、混合检索、图检索、视觉检索
│   ├── documents/        # 文档加载、三级切块、图谱抽取、指纹
│   ├── ontology/         # 本体约束层
│   ├── pipeline/         # 异步入库与 Redis Streams 管线
│   ├── embedding/        # 稠密/稀疏嵌入服务
│   ├── milvus/           # 向量库客户端与批量写入
│   ├── storage/          # MySQL、Redis、检查点、Neo4j 客户端及治理
│   ├── graph/            # 社区聚类与实体消歧
│   ├── evaluation/       # 自动化 RAG 评测
│   ├── observability/    # 链路、指标与结构化日志
│   ├── ha/               # 熔断、重试和降级
│   ├── cache/            # 语义缓存与防击穿
│   ├── memory/           # 记忆图谱
│   └── research/         # 深度研究引擎
├── scripts/              # 聚类、实体消歧、评测、报告与基准脚本
├── frontend/             # Vue 3 聊天界面、SSE 逻辑和主题样式
├── tests/                # 单元/集成测试与 Golden 数据集
├── data/documents/       # 上传文档存储
├── docs/                 # 功能规格、实现计划与界面截图
├── docker-compose.yml    # 全栈 Docker 服务
├── pyproject.toml        # Python 依赖和项目元数据
├── start.py              # 应用启动脚本
├── start_worker.py       # arq 异步入库 Worker 入口
└── .env.example          # 环境变量模板
```

---

## 快速开始

### 前置条件

- Python 3.12+
- Docker 与 Docker Compose
- MySQL 8.0+
- Redis 7.0+
- [uv](https://docs.astral.sh/uv/)（推荐）或 pip

### 1. 克隆并安装依赖

```bash
git clone https://github.com/your-username/Ragent-AI.git
cd Ragent-AI

# 方案 A：uv（推荐）
uv sync

# 方案 B：pip
python -m venv .venv
source .venv/bin/activate        # Windows：.venv\Scripts\activate
pip install -e .
```

### 2. 配置环境变量

将 `.env.example` 复制为 `.env`，然后填入密钥：

```env
# ===== LLM（DashScope / Qwen）=====
ARK_API_KEY=your_dashscope_api_key
MODEL=qwen-plus
BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
EMBEDDER=text-embedding-v1
GRADE_MODEL=qwen-plus
SUPERVISOR_MODEL=qwen-plus
MAX_TOKENS=8192

# ===== 数据库 =====
DATABASE_URL=mysql+pymysql://root:password@localhost:3306/agent_chat
REDIS_URL=redis://localhost:6379/0

# ===== Milvus =====
MILVUS_HOST=127.0.0.1
MILVUS_PORT=19530
MILVUS_VECTOR_DIM=1536
MILVUS_SEARCH_TOP_K=20

# ===== Neo4j（GraphRAG）=====
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password

# ===== 重排（可选；不可用时会优雅降级）=====
RERANK_MODEL=qwen3-rerank
RERANK_BINDING_HOST=https://dashscope.aliyuncs.com/compatible-mode/v1
RERANK_API_KEY=your_dashscope_api_key
RERANK_TOP_K=10

# ===== 网络搜索（可选）=====
TAVILY_API_KEY=your_tavily_api_key

# ===== 工具（可选）=====
AMAP_WEATHER_API=https://restapi.amap.com/v3/weather/weatherInfo
AMAP_API_KEY=your_amap_key
```

### 3. 启动基础设施

```bash
# 启动完整服务栈（Milvus + Neo4j 等）
docker compose up -d

# 检查服务健康状态
docker compose ps
```

| 服务 | 端口 | 说明 |
|---|---:|---|
| Milvus | 19530 | 向量数据库（gRPC） |
| Milvus Health | 9091 | 健康检查接口 |
| MinIO | 9000/9001 | 对象存储 / 控制台 |
| Attu | 8080 | Milvus Web 管理界面 |
| Neo4j | 7474 | Neo4j Browser（HTTP） |
| Neo4j Bolt | 7687 | Neo4j 驱动协议 |

### 4. 创建数据库

```sql
CREATE DATABASE IF NOT EXISTS agent_chat CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

首次启动时，SQLAlchemy 的 `Base.metadata.create_all()` 会自动创建 `chat_sessions`、`chat_messages`、`parent_chunks`、`community_summaries`、`graph_checkpoints` 和 `graph_checkpoint_writes` 等表。

### 5. 启动应用

```bash
# 方案 A：uv
uv run python start.py

# 方案 B：python
python start.py
```

在浏览器中打开：

- **前端**：http://127.0.0.1:8000/
- **API 文档**：http://127.0.0.1:8000/docs
- **Neo4j Browser**：http://localhost:7474

### 6. 执行 GraphRAG 离线流程

上传文档后，运行社区聚类脚本：

```bash
uv run python scripts/run_community_clustering.py
```

该脚本会：

1. 从 Neo4j 拉取完整知识图谱；
2. 执行 Leiden 社区发现；
3. 使用 LLM 生成社区摘要；
4. 将摘要索引至 Milvus，以支持全局搜索。

---

## 配置说明

### 主要环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `ARK_API_KEY` | — | DashScope API 密钥（必填） |
| `MODEL` | `qwen-plus` | Worker 智能体聊天模型 |
| `SUPERVISOR_MODEL` | `qwen-plus` | Supervisor 路由模型；未设置时回退至 `MODEL` |
| `BASE_URL` | — | OpenAI 兼容 LLM API 地址 |
| `EMBEDDER` | `text-embedding-v1` | 嵌入模型名称 |
| `GRADE_MODEL` | `qwen-plus` | 文档相关性评分模型 |
| `MAX_TOKENS` | `8192` | 最大输出 token 数 |
| `DATABASE_URL` | `mysql+pymysql://...` | MySQL 连接字符串 |
| `REDIS_URL` | `redis://localhost:6379/0` | Redis 连接字符串 |
| `MILVUS_HOST` / `MILVUS_PORT` | `127.0.0.1` / `19530` | Milvus 服务地址与端口 |
| `MILVUS_COLLECTION` | `embeddings_collection` | Milvus collection 名称 |
| `MILVUS_VECTOR_DIM` | `1536` | 嵌入向量维度 |
| `MILVUS_SEARCH_TOP_K` | `20` | 初始召回候选数 |
| `NEO4J_URI` | `bolt://localhost:7687` | Neo4j Bolt URI |
| `NEO4J_USER` / `NEO4J_PASSWORD` | `neo4j` / `password` | Neo4j 凭据 |
| `RERANK_MODEL` / `RERANK_TOP_K` | `qwen3-rerank` / `10` | 重排模型与输出候选数 |
| `TAVILY_API_KEY` | — | Tavily 网络搜索 API 密钥（可选） |
| `AMAP_API_KEY` | — | 高德天气 API 密钥（可选） |
| `AUTO_MERGE_ENABLED` / `AUTO_MERGE_THRESHOLD` | `true` / `2` | 是否启用及触发分层自动合并的最小同级块数 |
| `LEAF_RETRIEVE_LEVEL` | `3` | 用于检索的叶子块层级 |
| `WEB_SEARCH_MAX_RESULTS` | `5` | 网络搜索最大结果数 |
| `RRF_WEIGHT_DENSE` / `SPARSE` / `GRAPH` | `0.4` / `0.3` / `0.3` | 三个 RRF 通道的权重（v4） |
| `ENTITY_SIM_THRESHOLD` | `0.75` | 实体去重编辑距离阈值（v4） |
| `OTEL_ENABLED` | `false` | 是否启用 OpenTelemetry（v5） |
| `METRICS_ENABLED` | `true` | 是否暴露 Prometheus `/metrics`（v5） |
| `LOG_LEVEL` / `LOG_FORMAT` | `INFO` / `json` | 日志级别与格式（v5） |
| `NEO4J_QUERY_TIMEOUT` | `1.5` | Neo4j Cypher 查询超时秒数（v5） |
| `CACHE_SIMILARITY_THRESHOLD` | `0.95` | 语义缓存余弦相似度阈值（v6） |
| `CACHE_TTL_SECONDS` | `86400` | 缓存条目 TTL（秒，v6） |
| `MODEL_TURBO` / `MODEL_MAX` | `qwen-turbo` / `qwen-max` | 轻量与重推理任务模型（v6） |

### 切块参数

| 层级 | 块大小 | 重叠长度 | 用途 |
|---|---:|---:|---|
| L1（根） | 1200 字符 | 240 字符 | 主题上下文单元、图谱抽取来源 |
| L2（中间） | 600 字符 | 120 字符 | 中间聚合单元、图谱抽取来源 |
| L3（叶） | 300 字符 | 60 字符 | 向量化检索单元 |

---

## API 参考

### 聊天

| 方法 | 接口 | 说明 |
|---|---|---|
| `POST` | `/chat` | 同步聊天，返回完整响应 |
| `POST` | `/chat/stream` | 携带智能体状态事件的 SSE 流式聊天 |

### HITL（人工介入）

| 方法 | 接口 | 说明 |
|---|---|---|
| `POST` | `/chat/hitl/resume` | 人工干预后恢复暂停的图执行 |

`/chat/stream` 会推送以下 SSE 事件：

| 事件 | 说明 |
|---|---|
| `routing` | Supervisor 的路由决定（智能体与原因） |
| `agent_start` / `agent_done` | 智能体节点开始/结束执行 |
| `rag_step` | RAG/图检索步骤：检索、评分、改写、图扩展等 |
| `graph_expand` | 局部图搜索：实体定位与跳数扩展 |
| `community_match` | 全局图搜索：社区摘要匹配 |
| `content` / `worker_content` | 最终答案文本块 / Trace 面板中的 Worker 答案 |
| `trace` / `agent_trace` | 完整 RAG 审计轨迹 / 智能体级追踪数据 |
| `hitl_interrupt` | 触发 HITL 中断，图已暂停且已获得锁 |
| `error` | 错误信息 |

### 会话、指标、研究与文档

| 方法 | 接口 | 说明 |
|---|---|---|
| `GET` | `/sessions` | 列出全部会话 |
| `GET` | `/sessions/{id}` | 获取会话消息 |
| `DELETE` | `/sessions/{id}` | 删除会话 |
| `GET` | `/metrics` | Prometheus 指标：Token 用量、延迟、路由、熔断器等 |
| `POST` | `/research/create` | 创建并启动研究任务 |
| `GET` | `/research/list` | 列出用户的研究执行记录 |
| `GET` | `/research/{id}` | 获取研究状态和进度 |
| `GET` | `/research/{id}/evidence` | 列出已收集的证据 |
| `GET` | `/research/{id}/report` | 获取生成的研究报告 |
| `POST` | `/research/{id}/cancel` | 取消运行中的研究 |
| `GET` | `/documents` | 列出文档及其块数量 |
| `POST` | `/documents/upload` | 上传并向量化文档（SSE 进度）；同时触发图谱抽取 |
| `DELETE` | `/documents/{filename}` | 删除文档及其向量 |

应用启动后，可访问 `/docs`（Swagger UI）查看完整交互式 API 文档。

---

## 版本路线图

| 版本 | 已完成能力 |
|---|---|
| v2.0 | 多智能体 Supervisor-Workers、并行派发、Text-to-SQL、HITL、MySQL 状态持久化、Redis 锁、Trace Canvas 与双主题界面。 |
| v3.0 | Neo4j GraphRAG、实体/关系抽取、局部与全局图搜索、Leiden 聚类、社区摘要、三通道 RRF。 |
| v4.0 | 知识治理：软删除、文档生命周期、实体消歧、时态图谱、Golden 评测集、Ragas、A/B 报告与 CI/CD。 |
| v5.0 | OpenTelemetry、Prometheus、结构化日志、熔断、超时降级、重试及完整可观测性服务栈。 |
| v6.0 | 语义缓存、动态模型路由、Singleflight、防陈旧缓存和并发成本/延迟基准。 |
| v7.0 | PDF 多模态解析、图像/表格抽取、Qwen-VL 描述、视觉检索和多模态智能体。 |
| v8.0 | Planner、Critique、Replan 和最多两次的自我纠正循环。 |
| v9.0 | MCP 连接管理、动态工具注册、工具语义检索、多源数据分析与 ECharts。 |
| v10.0 | 受领域本体约束的图谱抽取、类型校验与图拓扑评测。 |
| v11–v13 | 文档指纹、异步增量入库、负载感知自适应检索、Redis Streams 与增量图聚类。 |
| v14–v15 | 多租户 RBAC、数据隔离、SaaS 计量、限流、审计与迁移。 |
| v16–v21 | Agent 工作流、自适应 GraphRAG、图推理、记忆图谱、深度研究、假设驱动的证据图谱与冲突检测。 |
| 后续计划 | 侧栏会话名称编辑、刷新页面后的 HITL 状态恢复、Markdown/PDF 对话导出、D3 图谱可视化、实体级引用链接。 |

### 逐项交付清单

#### v2.0 — 多智能体与 HITL ✓

- [x] 四个智能体的 Supervisor-Workers 架构与 LangGraph `Send` 并行派发。
- [x] Data Analyst（Text-to-SQL）、网络搜索失败回退 RAG、`recursion_limit=15` 死循环防护。
- [x] RAG 低置信度与 SQL 审批的 HITL 中断/恢复；MySQL checkpointer 状态持久化与 Redis 分布式锁。
- [x] 前端 Trace Canvas、HITL 审批弹窗及 Gemini 风格明暗主题。

#### v3.0 — GraphRAG 语义网络 ✓

- [x] Docker Compose 中部署 Neo4j，初始化图谱 schema、约束与索引。
- [x] 文档入库时进行 LLM 实体/关系抽取，实体去重并绑定来源 chunk。
- [x] 局部图搜索（向量 → 实体 → 一跳扩展）与全局图搜索（社区摘要匹配）。
- [x] Leiden/Louvain 社区发现与分层摘要；社区摘要写入 Milvus 和 MySQL。
- [x] Dense、Sparse、Graph 三通道 RRF 融合，以及图搜索 SSE 事件和前端追踪展示。

#### v4.0 — 知识治理与评测 ✓

- [x] MySQL → Milvus → Neo4j 跨库级联软删除、DocumentIndex 生命周期状态机、孤儿节点/边清理。
- [x] 编辑距离召回、LLM 确认、Cypher 合并组成的两阶段实体消歧；实体与关系支持 `valid_from` / `valid_to`。
- [x] 80 条 Golden QA、7 类查询、路由准确率评测、Ragas 评测和 retrieval/pipeline/e2e 三种模式。
- [x] RRF 权重网格搜索、A/B 差异报告、HTML 雷达/柱状/路由矩阵/延迟报告、CI 阈值脚本。
- [x] GitHub Actions CI/CD、Dockerfile、实体消歧 CLI。

#### v5.0 — 可观测性与高可用 ✓

- [x] Agent 节点、Milvus、Neo4j 的 OpenTelemetry 手工 Span；Prometheus `/metrics` 六项自定义指标。
- [x] structlog JSON 日志；LLM 与 Tavily 三次失败熔断及恢复。
- [x] Neo4j 查询超时后降级至 Dense+Sparse；LLM 生成和数据库写入使用指数退避重试。
- [x] Docker Compose 监控栈：Jaeger、Prometheus、Grafana 与预置数据源。

#### v6.0 — 成本与延迟优化 ✓

- [x] Milvus ANN + MySQL 的语义缓存（余弦相似度阈值 0.95）。
- [x] 轻量任务使用 qwen-turbo、重推理任务使用 qwen-plus/max 的动态模型路由。
- [x] Redis Singleflight 防缓存击穿、文档软删除驱动缓存失效、TTL 过期与并发基准测试。

#### v7.0 — 多模态升级 ✓

- [x] PyMuPDF 版面分析，区分 PDF 文本、图像、表格；媒体上传 MinIO 并与 chunk 关联。
- [x] Qwen-VL 生成图表/表格中文描述，视觉通道加入四通道 RRF。
- [x] Neo4j ImageNode/TableNode、关键词触发的 Multimodal Specialist 与视觉检索回答。

#### v8.0 — 自适应推理与自我纠正 ✓

- [x] GraphState 新增 `query_plan`、`critique_result`、`retry_count`、`draft_answer`。
- [x] Planner 拆解复杂问题；Critique 对草稿与检索上下文交叉验证；Replan 注入补充信息。
- [x] Critique → Replan → Supervisor 最多两次自我纠正；新增相关 SSE 事件与 Trace Canvas 样式。
- [x] 修复 multimodal_specialist 缺失的 synthesize 边；直接回答绕过无效事实核查。

#### v9.0 — MCP 集成与数据联邦 ✓

- [x] MCP 连接管理器支持 SSE/stdio、`tools/list`、`tools/call`。
- [x] MCP 工具自动转换为 LangChain `StructuredTool`，Milvus top-k 工具召回避免上下文膨胀。
- [x] Data Analyst 同时支持本地 MySQL 与外部 MCP 数据源；ECharts 自动检测图表类型并渲染。
- [x] Planner 支持带依赖和 `input_mapping` 的 DAG，`tool_outputs` 支持跨步骤传值。

#### v10.0 — 本体控制的图谱抽取 ✓

- [x] 领域本体包含 11 类实体、12 类关系谓词、70+ 三元组合法规则。
- [x] 显式白名单抽取提示词、Pydantic 规范化校验器和 `_validate_extraction()` 后处理拦截。
- [x] 支持 DashScope 字段映射、按类型消歧、图拓扑统计和 `graph` / `graph_compare` 评测。

#### v11.0 — 增量管线与 DevOps ✓

- [x] 上传计算 SHA-256 指纹；不变文档跳过完整管线；DocumentIndex 记录哈希、块数与版本。
- [x] `cleanup_by_filename()` 先剥离边上的块 ID，再清理空边与孤儿实体。
- [x] 基于 Redis 的 arq 异步任务队列和不可用时的同步回退；完整 Docker 服务栈与资源限制。

#### v12.0 — 自适应检索与负载降级 ✓

- [x] Query Profiler 使用关键词与嵌入相似度分类 L1/L2/L3 查询；YAML 意图权重矩阵取代静态权重。
- [x] Redis 滑动窗口 QPS 监控 NORMAL/WARNING/CRITICAL 三态；高负载时跳过 Critique/Replan 或熔断 Neo4j/Tavily。
- [x] 推送 `query_profiler`、`system_state` SSE 事件，新增 Prometheus 负载指标、Locust 和 A/B 测试。

#### v13.0 — 流式增量图引擎 ✓

- [x] 邻居共识局部修补与受影响社区的子图重聚类，避免每次重算全图。
- [x] `CommunitySummary.is_dirty` 只重生成脏社区摘要，降低 Token 开销。
- [x] Redis Streams 三阶段消息总线（`doc_ingest → graph_extract → vector_sync`）与消费者组、死信处理。

#### v14.0 — 多租户 RBAC 与数据隔离 ✓

- [x] Tenant/User/Role SQLAlchemy 模型、JWT、注册和 OAuth2 密码授权接口。
- [x] MySQL `tenant_id` 外键、Milvus 检索表达式预过滤、Neo4j 子图范围约束。
- [x] `tenant_id` 与 `access_level` 贯穿上传、队列、入库、SupervisorState、RAG/Graph/Data Analyst。
- [x] SQL 强制租户过滤、租户级会话和缓存隔离，以及四个权限升级红队测试。

#### v15.0 — SaaS 计量、限流与审计 ✓

- [x] `token_usage_logs` 记录请求 token；按租户、时段汇总用量。
- [x] 基于 Redis 滑动窗口的租户限流、429/`Retry-After` 中间件和分级 SLA 降级策略。
- [x] 不可变审计日志记录 MCP、SQL、HITL 事件；`AuditContext` 自动划分风险等级。
- [x] 租户范围的 Billing API、HITL Webhook、前端 JWT 认证、配置校验、SQL/上传安全和 Alembic 迁移。

---

<div align="center">

**基于 LangGraph · Milvus · Neo4j · FastAPI · Vue 3 构建**

</div>
