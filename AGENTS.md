# Ragent Agent工作指令

这是本项目的核心控制文件。作为开发 Agent，在开始任何工作前必须严格遵循本指南。

> 🧭 **文档框架导航（短入口，深链接）**
> 本仓库采用分层文档治理结构，深层规则拆分至 `docs/` 与 `.claude/` 下分类维护，避免根指令膨胀：
> - 架构与设计事实来源：[`docs/ragent-architecture.md`](./docs/ragent-architecture.md)、[`docs/multi-channel-retrieval.md`](./docs/multi-channel-retrieval.md)
> - 项目背景与全貌事实来源：[`.claude/PROJECT_CONTEXT.md`](./.claude/PROJECT_CONTEXT.md)
> - 快速启动与重构总结：[`docs/quick-start.md`](./docs/quick-start.md)、[`docs/refactoring-summary.md`](./docs/refactoring-summary.md)
> - 执行计划与状态：[`.claude/claude-progress.md`](./.claude/claude-progress.md)、[`.claude/feature_list.json`](./.claude/feature_list.json)

---

## 1. 核心工作流
* **开工必读**：依次读取 `.claude/PROJECT_CONTEXT.md` (架构与业务全貌)、`.claude/claude-progress.md` (当前进度)、`.claude/feature_list.json` (获取最高优先级 `not_started` 任务)。
* **单线处理**：每次仅处理单一任务，严禁多线并行。遇 Blocker 在 `.claude/feature_list.json` 记录原因。
* **代码修改**：优先使用 `codegraph` 工具定位功能或获取代码结构，再读取详细代码。
* **收尾闭环**：会话结束前必须更新 `.claude/claude-progress.md` 记录工作，并在 `.claude/feature_list.json` 中标记状态与附带运行证据，确保代码库整洁可运行。

---

## 2. 架构与修改约束
* **分层边界**：
  * **`framework/`**：仅包含与具体 AI 业务完全解耦的通用基础设施代码。严禁在此处添加大模型、向量库或业务 DTO。
  * **`infra-ai/`**：封装多供应商大模型/向量/Rerank 底层客户端，只做协议接入与健康探测路由。严禁在此处放置业务 RAG 管道代码。
  * **`bootstrap/`**：核心业务功能及 RAG Pipeline 实现地。
* **设计模式扩展纪律**：
  * **新增检索通道**：实现 `SearchChannel` 接口，用 `@Component` 注册。系统自动感知并将其加入多通道检索引擎。
  * **新增后处理器**：实现 `SearchResultPostProcessor` 接口，定义 `@Order` 并在 `process` 中处理段落合并或精排。
  * **新增 MCP 工具**：实现 `MCPToolExecutor` 接口，定义工具参数 schema 且无需修改注册表。
* **异步与并发规范**：
  * **跨线程传播**：异步调用或使用线程池时，必须用 `TtlExecutors` 或 `TransmittableThreadLocal` 包装，确保 `UserContext` 和 `TraceContext` 不在异步线程中丢失。
  * **SSE 安全性**：流式推送必须使用框架预置的 `SseEmitterSender` 对原生 `SseEmitter` 进行多线程并发安全控制，避免产生频繁的 ConcurrentModificationException 或 Emitter 泄露。
* **修改须授权**：修改方案与分析结果必须先向用户展示，**获明确许可后**才可修改代码。

---

## 3. 测试与验收标准 (DoD)
* **环境匹配**：使用 Java 17 + Maven 进行编译构建。本地开发建议配合 MySQL, Redis, Milvus, RocketMQ 环境。
* **测试与编译执行**：
  * **代码编译核查**：运行 `./mvnw clean compile`，确保没有引入语法、引用或 Maven 构建报错。
  * **测试套件验证**：运行 `./mvnw clean test`（或使用特定测试类命令，如 `./mvnw test -Dtest=PgVectorStoreServiceTest`），确保所有单元测试和集成测试顺利通过（Exit Code 0）。
* **DoD 强制要求**：
  * 1. 编译无任何报错且测试用例 100% 通过。
  * 2. 在 `.claude/feature_list.json` 标记任务为 `passing`，并**强制填入**实际运行命令及结果控制台输出作为证据 (Evidence)。
  * 3. **无实质运行证据，严禁标记完成**。

---

<claude-mem-context>
# Memory Context

# [Ragent] recent context, 2026-07-03 20:00 GMT+8
- Established agent governance and structured documentation (`AGENTS.md`, `.claude/PROJECT_CONTEXT.md`).
- Multi-channel retrieval and Intent Recognition systems mapped.
</claude-mem-context>
