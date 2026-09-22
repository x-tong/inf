> **状态：已被取代，仅作历史记录。**
> 2026-09-23 起，实现以 [Cache-aware Context Execution Runtime v1](./cache-aware-context-runtime-v1.md) 为准。
> 决策见 [ADR 0002](../adr/0002-cache-aware-context-execution.md) 与 [ADR 0003](../adr/0003-decision-backend.md)。
> 本文中的检索 / rerank 路径、fast / balanced / quality profile，以及 `embedding_latency_ms`，都不再作为 v1 规格。

# C++ Runtime Skeleton v1 设计

## 背景

当前主项目是 SLO-aware Context Execution Runtime。项目长期目标是把 RAG 与长上下文 LLM 请求进入模型前的 execution path 做成可测、可控、可降级、可优化的 infra 层。

本阶段优先训练 LLM runtime 架构能力，而不是先做 retrieval 性能优化。因此 v1 采用 C++ first、mock-first 的方式，在 macOS 本地完成一个可配置、可追踪、可降级的 runtime skeleton。

## 目标

- 使用现代 C++ 构建 runtime 核心接口和执行骨架。
- 使用 xmake 作为默认构建、测试和依赖管理入口。
- 建立 request、budget、route、trace、fallback、backend adapter 的稳定边界。
- 所有外部依赖先使用 mock 实现，保证 runtime 行为可复现。
- 输出结构化 trace，后续 benchmark、router 和真实 backend 都复用同一套语义。

## 非目标

- v1 不实现真实 embedding、vector index、reranker 或 LLM backend。
- v1 不追求 SIMD、多线程 topK、内存布局等 retrieval hot path 优化。
- v1 不提供 HTTP/gRPC server。
- v1 不实现复杂 learned routing policy。
- v1 不把 UI 或 chatbot 作为展示入口。

## 技术栈

- 语言标准：C++23。
- 构建系统：xmake。
- 本地平台：macOS + Apple clang。
- 测试框架：GoogleTest。
- 配置格式：YAML。
- trace 输出：JSONL。
- 推荐依赖：`gtest`、`nlohmann_json`、`yaml-cpp`、`cli11`。
- C++ 能力优先使用：`std::expected`、`std::jthread`、`std::stop_token`、`std::span`、`std::chrono`。

## Runtime 边界

v1 的执行路径如下：

```text
Request
  -> ExecutionRuntime
  -> ExecutionRouter
  -> MockRetriever
  -> MockReranker
  -> ContextPacker
  -> MockBackendAdapter
  -> ExecutionResult + ExecutionTrace
```

Runtime owns:

- request budget enforcement。
- route decision。
- stage timeout。
- fallback decision。
- context packing。
- backend adapter interface。
- structured trace。

Runtime does not own:

- model weights。
- 真实 vector database。
- 真实 embedding service。
- 真实 rerank model。
- 真实 inference scheduler。

## 核心模块

### Request and Budget

`Request` 描述一次 runtime 调用：

- `request_id`
- `query`
- `execution_profile`
- `latency_budget_ms`
- `max_context_tokens`
- `cost_budget`
- `quality_target`

`RequestBudget` 负责在各 stage 之间传递约束：

- total latency budget。
- rerank timeout。
- backend timeout。
- context token budget。
- fallback 是否允许。

### ExecutionRouter

v1 使用 rule-based router，不训练 classifier。

支持 route：

- `no_retrieval`
- `retrieval_only`
- `retrieval_rerank`
- `strict_budget`

初始规则：

- `fast` profile 默认避免 rerank。
- `balanced` profile 在预算允许时使用 retrieval + rerank。
- `quality` profile 优先 retrieval + rerank。
- 当 token budget 紧张时进入 strict packing。
- 当剩余 latency budget 不足时跳过 rerank。

### Mock Stage

mock stage 的目标是让 runtime 行为可重复：

- 每个 stage 支持固定延迟。
- 每个 stage 支持注入 timeout/error。
- MockRetriever 返回确定性 candidate list。
- MockReranker 返回确定性 rerank score。
- MockBackendAdapter 返回确定性 token usage 和 latency。

### ContextPacker

v1 只实现最小 packing 语义：

- 按 route 结果选择 candidate。
- 遵守 `max_context_tokens`。
- 输出 selected chunks 和 prompt token estimate。
- 当 token budget 不足时记录 `strict_budget` 或 `truncated` 标记。

### BackendAdapter

Backend 使用接口隔离：

```text
BackendAdapter
  -> MockBackendAdapter
  -> future: OpenAICompatibleBackendAdapter
  -> future: vLLMBackendAdapter
  -> future: SGLangBackendAdapter
```

v1 只实现 mock backend，但接口要保留真实 backend 需要的指标：

- prompt tokens。
- completion tokens。
- TTFT。
- total latency。
- timeout。
- backend error。

## Trace Schema

每次请求输出一条 JSONL trace。字段应覆盖：

```text
request_id
profile
route
fallback_used
fallback_reason
timeout_stage
error_code
embedding_latency_ms
retrieval_latency_ms
rerank_latency_ms
packing_latency_ms
backend_ttft_ms
backend_total_latency_ms
total_latency_ms
candidate_count
selected_chunk_count
prompt_tokens
completion_tokens
max_context_tokens
latency_budget_ms
```

v1 即使没有真实 embedding，也保留 `embedding_latency_ms` 字段，便于后续替换真实 stage 时保持 trace schema 稳定。

## Error and Fallback

错误使用结构化 `RuntimeError` 表示，调用边界优先返回 `std::expected<T, RuntimeError>`。

基础错误类型：

- `invalid_request`
- `config_error`
- `stage_timeout`
- `stage_error`
- `budget_exceeded`
- `backend_error`

Fallback 规则：

- rerank timeout：降级为 `retrieval_only`，并记录 fallback。
- backend timeout：返回 structured error，并完整输出已完成 stage trace。
- retrieval error：如果允许 fallback，则降级为 `no_retrieval`；否则返回 error。
- packing 超预算：进入 strict packing；仍失败则返回 `budget_exceeded`。

## 配置 Profile

v1 提供三个 profile：

- `configs/fast.yaml`
- `configs/balanced.yaml`
- `configs/quality.yaml`

配置至少包含：

- route policy。
- stage timeout。
- mock stage latency。
- max context tokens。
- fallback 开关。
- trace output path。

## Target 划分

推荐 xmake target：

```text
context_runtime          static library
runtime_cli              command line smoke test
context_runtime_tests    unit tests
```

推荐目录：

```text
cpp/
  include/context_runtime/
  src/
  tools/
  tests/
configs/
benchmarks/
```

## 验证方式

基础验证命令：

```bash
xmake f -m debug
xmake
xmake run context_runtime_tests
xmake run runtime_cli --config configs/fast.yaml --query "test query"
```

v1 验收标准：

- 三个 profile 都能运行。
- 每次 CLI 调用输出一条 JSONL trace。
- rerank timeout 能降级到 retrieval only。
- backend timeout 能返回 structured error。
- token budget 紧张时能触发 strict packing。
- 单元测试覆盖 router、budget、fallback、trace serialization、context packing。

## 风险

- 过早实现真实 retrieval 会分散 runtime 架构训练重点。规避方式：v1 保持 mock-first。
- 过度抽象会让小项目变复杂。规避方式：接口只围绕 v1 stage 和后续真实 backend 替换点设计。
- trace schema 后续频繁变化会影响 benchmark。规避方式：v1 先固定核心字段，扩展字段只追加不重命名。
- xmake 包依赖可能在本机首次安装时耗时或失败。规避方式：依赖保持少量且常见，先用基础包跑通。

## 后续路线

1. 实现 xmake 工程骨架和 mock runtime。
2. 加入 OpenAI-compatible mock/live backend adapter。
3. 加入真实 retrieval core 接口。
4. 引入 benchmark harness 和 trace report。
5. 在有 baseline 后实现 SLO-aware router 对比实验。
