
# 📘 基于LeetCode和知识库的RAG知识问答以及自动出题系统  

> 一个垂直领域的多模态检索增强生成 (Multimodal RAG) 系统，专注于 LeetCode 算法题目与计算机学术论文的智能解析与辅导。

![Docker](https://img.shields.io/badge/Docker-Enabled-blue?logo=docker)
![React](https://img.shields.io/badge/Frontend-React_18-61DAFB?logo=react)
![FastAPI](https://img.shields.io/badge/Backend-FastAPI-009688?logo=fastapi)
![Neo4j](https://img.shields.io/badge/GraphDB-Neo4j-008CC1?logo=neo4j)
![Ollama](https://img.shields.io/badge/LLM-Ollama-000000)

## 📖 项目简介 (Overview)

本项目旨在解决传统算法刷题过程中**题目知识点碎片化**以及**通用大模型易产生幻觉**的痛点。

通过构建**双路知识库**（向量库 + 图数据库），系统不仅能基于语义检索 LeetCode 题目和 PDF 论文，还能通过**知识图谱**可视化展示算法知识点（如：困难度 -> 标签 -> 题目）的层级关系。同时，引入了 **Check Layer (容错校验层)**，通过多重校验机制大幅提升了回答的准确性与可信度。

### 🚀 核心能力
*   **多模态 RAG**: 支持让 AI “看见” PDF 中的图表和 Markdown 中的本地图片引用。
*   **双库协同**: ChromaDB 处理非结构化文本检索，Neo4j 处理结构化知识推理。
*   **防幻觉校验**: 独创的三级校验机制（检索熔断、引用核查、质量过滤）。
*   **沉浸式交互**: 支持 Markdown 表格、LaTeX 公式 ($O(n \log n)$) 及代码高亮的流式对话。
*   **云原生架构**: 全系统容器化，支持 Docker Compose 一键部署。

---

## 🛠 技术栈 (Tech Stack)

| 模块 | 技术选型 | 说明 |
| :--- | :--- | :--- |
| **前端** | **React 18 + Vite** | 构建工具与框架 |
| **UI/可视化** | Material UI / React Force Graph | 沉浸式对话与 2D 力导向图渲染 |
| **后端** | **Python 3.10 + FastAPI** | 高性能异步 Web 微服务 |
| **LLM 服务** | **Ollama** | 本地托管 Qwen2.5:7b-instruct 模型 |
| **向量库** | **ChromaDB** | 本地持久化向量存储 |
| **图数据库** | **Neo4j Desktop** | 存储算法知识图谱关系 |
| **基础设施** | **Docker & Docker Compose** | 服务编排与容器化部署 |

---

## 🏗 系统架构 (Architecture)

系统采用 **云原生分层架构**，所有模块均封装为独立 Docker 容器。

```mermaid
graph TD
    %% =======================
    %% 0. 容器编排层 (新增)
    %% =======================
    Orch[容器编排层]
    Orch -->|一键启动| DC[docker-compose.yml]
    DC -.->|编排| B
    DC -.->|编排| C
    DC -.->|编排| D
    DC -.->|编排| E
    DC -.->|编排| F
    DC -.->|编排| G

    %% =======================
    %% 1. 客户端层
    %% =======================
    A[客户端层 User/Browser] -->|HTTP/WS| C

    %% =======================
    %% 2. 前端服务层
    %% =======================
    subgraph Frontend_Layer [前端服务层]
        direction TB
        Conf_Front[frontend.Dockerfile] -.->|Build| B[前端容器 React App]
        B -->|React 18 + Vite + MUI| B1[Chat组件 & GraphView组件]
    end

    %% =======================
    %% 3. 网关层
    %% =======================
    subgraph Gateway_Layer [网关层]
        Conf_Nginx[nginx.conf] -.->|Config| C[反向代理容器 Nginx]
        C -->|静态资源托管| B
        C -->|API 转发 :8088| D
    end

    %% =======================
    %% 4. 后端业务层 (核心)
    %% =======================
    subgraph Backend_Layer [后端业务层]
        direction TB
        Conf_Back[backend.Dockerfile] -.->|Build| D[后端容器 FastAPI + Uvicorn]
        
        D --> D_Logic[RAG Agent 核心逻辑]
        
        subgraph Agent_Module [Agentic Workflow]
            D_Logic --> D1[Prompt工程]
            D1 -->|CoT 推理| D2[LLM 交互]
            
            %% 新增 checker.py 位置
            D2 --> D3[容错校验层 Guardrails]
            D3 -->|核心实现| D3_Code[agent/core/checker.py]
            D3_Code -->|1.检索熔断| Check1[无上下文拦截]
            D3_Code -->|2.引用核查| Check2[Evidence ID正则匹配]
            D3_Code -->|3.质量过滤| Check3[回复长度/拒答检测]
        end
    end

    %% =======================
    %% 5. 模型与存储层 (Infra)
    %% =======================
    D -->|API Call| E[LLM服务容器 Ollama]
    E --> E1[Qwen2.5:7b-instruct 推理]
    E --> E2[Nomic-embed-text 向量化]

    D_Logic -->|Query| F[向量库容器 ChromaDB]
    D_Logic -->|Cypher| G[图数据库容器 Neo4j]
    
    F --> F1[非结构化数据<br/>PDF/MD/图片描述]
    G --> G1[结构化知识图谱<br/>难度-标签-题目关系]

    %% =======================
    %% 6. 数据处理流水线
    %% =======================
    subgraph Data_Pipeline [离线数据处理层]
        H[Python 脚本集 scripts/]
        H --> H1[爬虫: leetcode-crawler.py]
        H --> H2[多模态解析: parse_pdf_multimodal.py]
        H --> H3[向量建库: build_index_ollama.py]
        H --> H4[图谱构建: build_leetcode_graph.py]
    end
    
    H -.->|写入| F
    H -.->|写入| G

    %% =======================
    %% 7. 数据源
    %% =======================
    H --> I[原始数据源 data/ & docs/]
    I --> I1[LeetCode HTML]
    I --> I2[学术论文 PDF]
    I --> I3[本地图片资源 assets]

    %% =======================
    %% 样式定义
    %% =======================
    classDef orchestration fill:#212121,stroke:#000,stroke-width:2px,color:#fff;
    classDef config fill:#ffecb3,stroke:#ff6f00,stroke-width:2px,stroke-dasharray: 5 5;
    classDef container fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef logic fill:#f3e5f5,stroke:#4a148c,stroke-width:2px;
    classDef storage fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px;
    classDef codeFile fill:#fff3e0,stroke:#bf360c,stroke-width:2px;

    class DC orchestration;
    class Conf_Front,Conf_Nginx,Conf_Back config;
    class B,C,D,E,F,G container;
    class D1,D2,D3 logic;
    class D3_Code codeFile;
    class F1,G1 storage;

````

### 🛡️ 容错校验层 (Check Layer)

为了解决幻觉问题，系统在 `checker.py` 中实现了三级护栏：

1. **检索层熔断**: 若无向量召回，直接拦截，拒绝编造。
2. **引用合规性检查**: 强制检查回答中是否包含 `[evidence_id]` 证据链。
3. **启发式质量过滤**: 过滤过短或复读式回答。

---

## 📂 项目结构 (Directory Structure)

```text
rag/
├── deploy/                     # [云原生] 容器化部署配置
│   ├── docker-compose.yml      # 服务编排
│   ├── backend.Dockerfile      # 后端镜像构建
│   └── frontend.Dockerfile     # 前端镜像构建
├── data/                       # [ETL数据] 清洗后的 JSONL 与提取的图片
├── docs/                       # [源数据] LeetCode Markdown 题目库
├── frontend/                   # [前端源码] React + MUI + ForceGraph
├── index/                      # [索引] Chroma 向量索引文件
├── scripts/                    # [流水线] 爬虫、多模态解析、建库脚本
└── services/
    └── agent/
        ├── core/checker.py     # [核心] 幻觉校验模块
        └── rag_demo.py         # [入口] FastAPI 主程序
```

---

## 🚀 快速开始 (Getting Started)

### 前置要求

* Docker & Docker Desktop

### 1. 启动服务

在项目根目录下，使用 Docker Compose 一键启动所有服务（前端、后端、数据库、模型服务）：

```bash
docker-compose -f deploy/docker-compose.yml up -d --build
```

### 2. 下载模型

容器启动后，需要进入 Ollama 容器下载 Qwen2.5 模型：

```bash
docker exec -it rag_ollama ollama run qwen2.5:7b-instruct
```

### 3. 访问系统

* **Web 界面**: [http://localhost:5173](http://localhost:5173)
* **后端 API 文档**: [http://localhost:8088/docs](http://localhost:8088/docs)
* **Neo4j 控制台**: [http://localhost:7474](http://localhost:7474) (账号: `neo4j` / 密码: `password123`)

---

## 📊 数据工程 (Data Engineering)

本项目包含完整的数据处理流水线：

1. **爬虫**: `leetcode-crawler.py` 爬取近 10000 条题目并转为 Markdown。
2. **多模态解析**: `parse_pdf_multimodal.py` 解析 PDF 表格及 Markdown 图片引用。
3. **向量化**: `build_index_ollama.py` 使用 `nomic-embed-text` 构建索引。
4. **图谱构建**: `build_leetcode_graph.py` 提取元数据构建知识图谱。

---

## 👥 作者与贡献 (Author)


本项目工作量涵盖以下四个维度：

* **云原生架构设计**: Docker 容器化配置、Docker Compose 服务编排、K8s 适配性方案预研。
* **智能体策略开发**: Prompt 工程设计、RAG 推理链路 (CoT) 优化、容错校验层 (Checker) 开发。
* **全栈工程开发**: React 可视化前端开发、FastAPI 后端微服务开发。
* **数据流水线建设**: 爬虫工程化、多模态数据清洗、知识库构建脚本编写。

---

## 📜 致谢 (Acknowledgments)

本项目在开发过程中使用了以下开源项目：

* [Ollama](https://ollama.com/)
* [LangChain](https://www.langchain.com/)
* [React Force Graph](https://github.com/vasturiano/react-force-graph)
* [ChromaDB](https://www.trychroma.com/)
* [LeetCode](https://leetcode.com/) (数据来源)

```
