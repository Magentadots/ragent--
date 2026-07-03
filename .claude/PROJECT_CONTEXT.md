# Ragent 项目背景与架构说明 (Project Context)

这份文档为 AI Agent 提供本项目的全貌与核心上下文信息，在理解业务逻辑、修改代码和规划任务时应以此为参考。

## 1. 项目简介 (Project Overview)
* **项目名称**：Ragent AI (Enterprise-grade Agentic RAG Platform)
* **核心目标**：构建一个企业级、高可扩展的 Agentic RAG 平台，涵盖从多格式文档摄取管道（Ingestion Pipeline）到流式对话交互的完整链路，提供多模型健康路由、多级意图识别树、多通道并行检索、歧义引导和 MCP 工具集成等核心生产特性。
* **技术栈**：
  * **语言/版本**：Java 17
  * **主框架**：Spring Boot 3.5.7
  * **AI/大模型**：Spring AI (用于某些原生集成)，infra-ai 自研模型代理与降级路由层
  * **数据存储**：Milvus 2.6.6 (向量存储) / PostgreSQL pgvector + MyBatis-Plus (持久化与备用向量)
  * **消息队列**：RocketMQ (异步摄取通知)
  * **安全认证**：Sa-Token
  * **前端**：React 18 + TypeScript

---

## 2. 核心架构与模块组织 (Core Architecture & Modules)
项目采用 Maven 多模块架构，业务逻辑与基础设施严格解耦：

### 2.1 模块依赖关系
```
frontend (React) ─ ─ ─ (HTTP/SSE/WS) ─ ─ ─ ┐
                                           ▼
                                       bootstrap (主应用)
                                       ├── framework (共享基础设施)
                                       ├── infra-ai (AI 客户端抽象与路由)
                                       └── mcp-server (工具暴露服务)
```

### 2.2 模块职责与边界
* **`framework/` (共享基础设施)**
  * 封装统一的三级异常体系与全局拦截、分布式幂等锁、Snowflake ID 生成器。
  * 提供 `SseEmitterSender` 对线程安全 SSE 的封装，及支持 TraceId 和 UserContext 跨线程传递的包装线程池。
* **`infra-ai/` (AI 基础设施层)**
  * 提供 `LLMService`、`EmbeddingService` 和 `RerankService` 接口，解耦具体模型供应商。
  * 内置健康检测、首包探测（Probe）、自动熔断降级链（三态熔断器 CLOSED ➜ OPEN ➜ HALF_OPEN）。
* **`mcp-server/` (MCP 独立工具服务)**
  * 独立的 Spring Boot 应用，通过 Model Context Protocol 暴露具体业务工具（如 Sales, Ticket 等数据查询）。
* **`bootstrap/` (业务核心逻辑)**
  * 包含文档解析分块（`core`）、知识库任务调度与 Ingestion 节点编排（Fetcher -> Parser -> Chunker -> Enhancer -> Enricher -> Indexer 6 节点 DAG 管道）。
  * 包含 RAG 完整管道逻辑（`rag` 包）：意图分类器、查询改写、检索通道、歧义检测、MCP 客户端集成、JDBC 对话记忆及 StreamChatPipeline。

---

## 3. RAG 对话完整管道 (StreamChatPipeline)
用户请求流式问答的核心执行流程位于 `StreamChatPipeline.execute()`，包括 8 个步骤 and 3 个短路分支：

```
                              [用户输入 question]
                                      │
                                      ▼
                        Stage 1: loadMemory()
                                      │
                                      ▼
                        Stage 2: rewriteQuery()
                                      │
                                      ▼
                        Stage 3: resolveIntents()
                                      │
           ┌──────────────────────────┼──────────────────────────┐
           ▼ (短路1)                  ▼ (短路2)                  ▼
      [检测到歧义意图]           [仅系统对话意图]            [正常流程]
    handleGuidance()           handleSystemOnly()                │
    返回选项卡以引导澄清            直接返回系统消息                     │
           │                          │                          ▼
           └─────────── 退出 ─────────┘                 Stage 6: retrieve()
                                                   (多通道检索 + MCP 工具调用)
                                                                 │
                                       ┌─────────────────────────┴─────────┐
                                       ▼ (短路3)                           ▼
                                 [检索上下文全空]                     [得到证据数据]
                              handleEmptyRetrieval()                      │
                                 返回默认无匹配提示                         │
                                       │                                   ▼
                                       └─────────── 退出 ─────────Stage 8: streamRag()
                                                                  (Prompt 路由 + SSE 流)
```

### 3.1 Pipeline 各阶段主要细节
* **Stage 1 (loadMemory)**: 从持久化存储加载最近 N 条历史消息，并拉取长对话自动生成的 System 级摘要消息，拼接当前问题。
* **Stage 2 (rewriteQuery)**: 对用户问题执行同义词/缩写标准化，再经由 LLM 进行多轮对话消歧改写与子问题拆分。
* **Stage 3 (resolveIntents)**: 并行调用 LLM 意图分类器。单次查询意图阈值分数限制为 `score >= 0.35`，子问题最多分配 3 个意图。
* **Stage 6 (retrieve)**:
  * **知识库检索**：交由 `MultiChannelRetrievalEngine` 执行多通道（VectorGlobal / IntentDirected）检索，并经过 Deduplication (去重) 与 Rerank (重排序) 后置处理器。
  * **工具调用**：对于 MCP 意图，执行 LLM 参数提取，构造 JSON Schema，再通过 MCP Client 同步调用远程 MCP 服务，渲染为结构化上下文。
* **Stage 8 (streamRagResponse)**: 根据检索源动态路由 Prompt 场景（`KB_ONLY`、`MCP_ONLY` 或 `MIXED`），拼装完整 Prompt，由降级路由模型层流式吐出答案。

---

## 4. 核心组件与设计规范

### 4.1 意图识别系统 (Intent Tree & Classifier)
意图的静态定义保存在 MySQL 表 `t_intent_node` 中，系统缓存至 Redis (`ragent:intent:tree`, 7天 TTL)。
* **层次结构**：DOMAIN (领域) ➜ CATEGORY (分类) ➜ TOPIC (主题，叶子节点)。
* **分类机制**：仅叶子节点参与 LLM 分类。当用户子问题 top-2 意图评分比值大于等于 0.8 时，判定存在严重歧义，自动进入歧义引导阶段；若在 `[0.65, 0.8)` 灰色区间内，调用 LLM 再次进行歧义确认。

### 4.2 多通道检索与后处理规范 (Multi-Channel Retrieval)
为保证高覆盖率与低延迟的平衡，检索模块严格基于责任链与策略模式：
1. **通道定义 (`SearchChannel`)**：
   * `IntentDirectedSearchChannel` (优先级高)：当意图识别明确且置信度高 (`>=0.4`) 时激活，针对意图关联的特定 Collection 执行密集召回。
   * `VectorGlobalSearchChannel` (全局兜底)：当意图置信度低 (`<0.6`) 或需要补充召回时条件激活，在所有 Collection 执行全局搜索。
   * *扩展规范*：新检索通道（如 ES 关键词通道）只需实现 `SearchChannel` 并注册为 `@Component`。
2. **后处理器 (`SearchResultPostProcessor`)**：
   * `DeduplicationPostProcessor`：对多通道召回结果进行主键/段落去重，合并结果并保留高优先级通道分数。
   * `RerankPostProcessor`：启用外部 Rerank 模型进行精准的二次算分与截断。
   * *扩展规范*：实现该接口并配置 `@Order` 即可无缝加入后处理链。

### 4.3 并发控制与上下文传播规范 (Concurrency & Context)
项目中包含大量高并发异步调用，必须严格遵循以下线程池与上下文规范：
* **8个独立线程池**：系统按负载特征划分为：MCP批量调用、RAG上下文组装、多路检索、内部检索、意图分类、记忆摘要、模型流式输出、入口队列等线程池。
* **跨线程透传**：所有异步线程池必须用阿里巴巴的 `TransmittableThreadLocal` (`TTL`) 包装执行器，确保 `TraceContext` (全链路追踪 ID) 和 `UserContext` 在多层异步嵌套下不丢失。
* **并发限流**：Ragent 采用基于 Redis 信号量 + ZSET + Pub/Sub 的分布式公平队列限流器，限流状态实时通过 SSE 触达用户。
