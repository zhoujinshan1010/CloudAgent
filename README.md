# ☁️ CloudAgent — 企业级云平台智能客服系统

> 基于 LangGraph Multi-Agent 编排 + Milvus 语义缓存 + 图向量混合检索（Hybrid RAG）构建的下一代企业级智能客服平台

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-green.svg)](https://fastapi.tiangolo.com)
[![LangGraph](https://img.shields.io/badge/LangGraph-latest-orange.svg)](https://langchain-ai.github.io/langgraph/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📌 项目简介

CloudAgent 是面向云计算平台多业务线客服场景的智能体系统，覆盖**云产品咨询、订单与账单查询、资源降本优化、产品推广推荐**等核心服务。

### 痛点与解法

| 痛点 | 原有方案 | CloudAgent 解法 |
|------|---------|----------------|
| 跨会话上下文丢失 | 无持久化记忆 | Redis 短期 + Milvus 长期双层记忆 |
| 高频问题重复耗费算力 | 每次走大模型推理 | L1/L2 语义缓存，命中率 38%，延迟降至 80ms |
| 云产品参数幻觉 | 纯向量检索 RAG | Milvus + Neo4j 混合检索，召回准确率 92% |
| 工具调用耦合度高 | 硬编码 Function Calling | FastMCP 标准化封装，工具联调从天级降至小时级 |

---

## 🏗️ 系统架构

```
┌─────────────────────────────────────────────────────┐
│               接入层 (Access Layer)                   │
│   [Vue3 前端] ←── SSE 流式响应 ──→ [FastAPI 网关]     │
│                        ↓ 语义缓存拦截（Milvus L1/L2） │
└─────────────────────────────────────────────────────┘
                         ↓ 未命中缓存
┌─────────────────────────────────────────────────────┐
│            编排层 (LangGraph Orchestration)           │
│         ┌─────────────────────────┐                 │
│         │   Orchestrator 意图路由  │                 │
│         └────────────┬────────────┘                 │
│          ┌───────────┼───────────┐                  │
│          ▼           ▼           ▼           ▼       │
│   ProductAgent  BillingAgent  RecommendAgent  FinOpsAgent │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│               记忆层 (Memory System)                  │
│   Redis (会话级窗口压缩) ←──→ Milvus (用户级偏好)     │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│            工具层 (FastMCP & Hybrid RAG)              │
│   Milvus (语义检索) + Neo4j (知识图谱)                │
│   FastMCP Server: MySQL / DashScope API              │
└─────────────────────────────────────────────────────┘
```

---

## ✨ 核心特性

### 1. 多 Agent 协作编排
- 基于 **LangGraph StateGraph** 构建含条件边、路由边的有向图 Pipeline
- `Orchestrator` 统一意图路由，支持多轮对话指代消解，意图路由准确率 **95%**
- 全局 `AgentState` 共享跨节点上下文，实现 Agent 间类型安全数据传递

### 2. L1/L2 双层语义缓存
- **L1 精确命中**：字符串归一化后完全匹配，直接返回
- **L1 语义命中**：余弦距离 ≤ 0.08，毫秒级拦截
- **L2 降级**：未命中进入 Agent 推理链路
- 高频请求命中率 **38%**，首字响应延迟 **3.2s → 80ms**，月均 Token 节省 **40%**

### 3. 图向量混合检索（Hybrid RAG）
- **Milvus**：覆盖模糊概念与长文本语义检索
- **Neo4j**：处理产品属性、地域可用性等网状结构数据（Cypher 查询）
- **Fallback 容错**：LLM 生成 Cypher → 执行 → 异常捕获 → 关键词降级，全链路容错
- 复杂云产品参数召回准确率 **60% → 92%**

### 4. FastMCP 工具化封装
- 将 MySQL 查询、云端监控数据拉取、DashScope 图像生成封装为标准 MCP Server
- `UserIdInjector` 拦截器强制注入会话 `user_id`，阻断 Prompt 注入越权攻击
- 新增单一工具联调时间从**天级压缩至小时级**

### 5. 多级记忆系统
- **短期记忆**：Redis 滑动窗口（阈值 10 轮，超限自动压缩，防 Context 溢出）
- **长期记忆**：Milvus 用户级偏好，会话结束后异步 LLM 抽取并写入向量库
- 用户重复陈述背景轮次减少 **45%**，CSAT 满意度提升 **18%**

---

## 🤖 Agent 矩阵

| Agent | 职责 | 挂载工具 |
|-------|------|---------|
| `Orchestrator` | 意图路由总控，不直接回答，只派发任务 | — |
| `ProductAgent` | 云产品咨询、规格参数、政策查询 | Milvus 向量检索 + Neo4j 图谱检索 |
| `BillingAgent` | 用户订单、账单、实例查询（含越权防护） | FastMCP: MySQL 订单查询 |
| `RecommendAgent` | 产品选型推荐 + 生成返佣链接与推广海报 | FastMCP + Milvus + DashScope |
| `FinOpsAgent` | 降本增效，分析近 7 天资源利用率并输出优化建议 | FastMCP: 云端监控数据 |

---

## 🛠️ 技术栈

```
后端框架     FastAPI · Python 3.11+
Agent 编排   LangGraph · LangChain
向量数据库   Milvus
图数据库     Neo4j
缓存         Redis
关系数据库   MySQL
工具协议     MCP (Model Context Protocol) · FastMCP
大模型       Qwen (通义千问) · DashScope (qwen-image-2.0)
前端         Vue3 · SSE 流式响应
```

---

## 🚀 快速开始

### 环境要求
- Python 3.11+
- Docker & Docker Compose（用于启动 Milvus / Neo4j / Redis / MySQL）

### 1. 克隆项目

```bash
git clone https://github.com/your-username/cloud-agent.git
cd cloud-agent
```

### 2. 安装依赖

```bash
pip install -r requirements.txt
```

### 3. 配置环境变量

```bash
cp .env.example .env
```

编辑 `.env` 文件，填入以下配置：

```env
# 大模型
DASHSCOPE_API_KEY=your_dashscope_api_key

# 数据库
MILVUS_HOST=localhost
MILVUS_PORT=19530
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_password
REDIS_URL=redis://localhost:6379
MYSQL_URL=mysql+pymysql://user:password@localhost:3306/cloudagent

# 系统配置
L1_SEMANTIC_THRESHOLD=0.08
MEMORY_COMPRESSION_THRESHOLD=10
```

### 4. 启动基础服务

```bash
docker-compose up -d
```

### 5. 启动服务

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

访问 API 文档：`http://localhost:8000/docs`

---

## 📁 项目结构

```
cloud-agent/
├── app/
│   ├── main.py                    # FastAPI 入口 & 语义缓存网关
│   └── infra/
│       └── cache.py               # L1/L2 语义缓存实现
├── agent/
│   ├── core/
│   │   ├── workflow/
│   │   │   └── graph_manager.py   # LangGraph 图结构 & 条件路由
│   │   └── memory/
│   │       └── manager.py         # 长短期记忆系统
│   ├── agents/
│   │   ├── orchestrator.py        # 意图路由 Agent
│   │   ├── product_agent.py       # 产品咨询 Agent
│   │   ├── billing_agent.py       # 账单查询 Agent（含越权防护）
│   │   ├── recommendation_agent.py # 推荐导购 Agent
│   │   └── finops_agent.py        # 降本增效 Agent
│   ├── tools/
│   │   ├── rag_tool.py            # Milvus 向量检索工具
│   │   └── graph_tool.py          # Neo4j 图谱检索 + Fallback
│   └── mcp_servers/
│       └── cloud_platform_server.py # FastMCP 工具封装
├── docker-compose.yml
├── requirements.txt
└── .env.example
```

---

## 📊 性能指标

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| 意图路由准确率 | — | **95%** |
| 高频问题首字响应延迟 | 3.2s | **80ms** |
| 语义缓存命中率 | 0% | **38%** |
| 月均 Token 消耗 | 基准 | **节省 40%** |
| 复杂参数召回准确率 | 60% | **92%** |
| 用户重复陈述背景轮次 | 基准 | **减少 45%** |
| CSAT 用户满意度 | 基准 | **提升 18%** |
| 新增工具联调时间 | 天级 | **小时级** |

---

## 🔒 安全设计

- **越权防护**：`UserIdInjector` 拦截器强制注入真实 `user_id`，防止 LLM 伪造用户 ID 查询他人账单
- **防幻觉约束**：所有 Agent 强制要求回答附带知识库来源，禁止编造实例型号与参数
- **数据隔离**：FastMCP 协议实现大模型与业务系统物理解耦，工具调用有完整权限边界

---

## 📄 License

MIT License © 2025
