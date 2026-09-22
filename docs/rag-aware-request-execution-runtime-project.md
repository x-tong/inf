> **状态：已被取代，仅作历史记录。**
> 2026-09-23 起，实现以 [Cache-aware Context Execution Runtime v1](./projects/cache-aware-context-runtime-v1.md) 为准。
> 决策见 [ADR 0002](./adr/0002-cache-aware-context-execution.md) 与 [ADR 0003](./adr/0003-decision-backend.md)。
> 本文不再作为 v1 的路径、成功标准或实施计划。

# SLO-aware Context Execution Runtime 项目书

## 1. 项目定位

本项目旨在实现一个面向生产级 RAG 与长上下文 LLM 系统的请求执行层：

> A SLO-aware context execution runtime for retrieval-augmented and long-context LLM serving.

它不是一个 Agent 框架，不是一个向量数据库，也不是一个推理引擎。它位于应用/Agent 和 LLM serving backend 之间，负责在一次请求进入模型之前，根据 latency SLO、context budget、retrieval confidence、rerank cost 和 backend 状态，动态构造高质量、低成本、可观测的 model-ready context。

核心定位：

```text
Agent / App / API Server
        ↓
SLO-aware Context Execution Runtime
        ↓
LLM Backend: vLLM / SGLang / OpenAI API
```

项目第一版聚焦 query 到 context 的执行路径：

```text
query
  -> embedding
  -> retrieval
  -> topK selection
  -> optional rerank
  -> context packing
  -> execution trace
```

项目第二阶段引入 SLO-aware request execution：

```text
query + request budget
  -> request analysis
  -> retrieval confidence estimation
  -> execution route decision
  -> no retrieval / retrieval only / retrieval + rerank / strict packing
  -> LLM backend call
  -> end-to-end trace and evaluation
```

这个项目的核心不是“做一个 RAG 应用”，而是研究和实现：

> 如何把 LLM 请求进入模型之前的 context execution path 做成可测、可控、可优化、可扩展的 infra 层。

## 2. 背景与问题

当前大量 RAG 系统仍然由 Python glue、vector database、reranker、prompt builder 和 LLM API 拼接而成。典型路径如下：

```text
Python App
  -> embedding service
  -> vector database
  -> reranker
  -> prompt builder
  -> LLM backend
```

这类架构的问题不是“不能工作”，而是很难以 infra 的方式管理：

- 请求路径碎片化，跨组件序列化和数据复制成本不透明。
- 难以获得 stage-level latency breakdown。
- retrieval、rerank、context packing、LLM prefill 之间缺少统一执行模型。
- P95/P99 tail latency 难以控制。
- 缺少基于 latency SLO、quality target 和 cost budget 的动态路径选择。
- benchmark 往往只展示平均延迟或主观回答效果，无法解释优化收益。
- 很少把 retrieval 质量、context packing、prompt token 数、TTFT、TPOT 和总成本放在同一个实验框架下分析。

本项目的核心假设是：

> 生产级 LLM 系统的瓶颈不只在模型推理，也在 model input 之前的 context construction 和 request execution path。

因此，本项目聚焦一个底层问题：

> 如何根据请求的质量和延迟目标，动态决定检索、精排、上下文打包和 backend 调用策略，并用严肃 benchmark 量化每个决策的收益和代价？

## 3. 与当前 LLM Infra 趋势的关系

当前 LLM infra 的主线正在从单纯“调用模型”演进到以下方向：

1. **Serving latency and throughput**
   关注 TTFT、TPOT、prefill/decode、continuous batching、prefix caching、KV cache、disaggregated serving。

2. **Request routing and policy**
   根据请求类型、context 长度、cache 命中、backend 负载、成本预算选择执行路径和模型后端。

3. **Context engineering**
   不再只是把 topK chunks 塞进 prompt，而是控制 evidence 质量、token budget、dedup、邻接合并、source attribution 和上下文压缩。

4. **RAG evaluation**
   生产系统需要衡量 retrieval recall、context precision、answer faithfulness、latency、cost，而不是只看单次 demo 输出。

5. **Observability**
   每次请求都应输出 trace：embedding、retrieval、rerank、packing、LLM backend、tokens、cache、cost、route decision。

本项目正好切入这些趋势之间的交叉点：它不深入实现 GPU inference engine，但它理解 LLM backend 的 serving 指标；它不自研 vector database，但它把 retrieval/rerank/packing 纳入一个可控执行层；它不做 Agent 框架，但为 Agent/App 提供低延迟、可观测、可评估的上下文执行能力。

## 4. 目标用户与求职信号

### AI Infra Engineer

展示能力：

- 请求执行层设计
- SLO-aware routing
- stage-level metrics and tracing
- latency / QPS / memory / cost benchmark
- timeout、fallback、cancellation、error handling
- config-driven execution policy

### LLM Systems Engineer

展示能力：

- query 到 model-ready context 的执行路径
- retrieval / rerank / context packing 编排
- TTFT、TPOT、prompt tokens、prefix/cache 命中等 serving 指标意识
- LLM backend adapter：vLLM / SGLang / OpenAI API
- route-aware and budget-aware execution

### Retrieval / Search Engineer

展示能力：

- vector retrieval
- topK selection
- rerank orchestration
- recall quality 与 latency tradeoff
- recall@k / MRR / nDCG / context precision
- 从广告召回/排序系统迁移到 RAG retrieval execution 的能力

### Infra-oriented C++ Engineer 转型信号

展示能力：

- C++ hot path implementation
- memory layout、SIMD、thread-local topK、allocation control
- benchmark discipline
- 从传统低延迟请求系统迁移到 LLM context execution

## 5. 非目标

本项目明确不做以下内容：

- 不做完整 Agent 框架。
- 不做完整 vector database。
- 不做完整 inference engine。
- 不自研 embedding model。
- 不自研 rerank model。
- 不实现分布式一致性、事务、WAL、SQL、多租户权限。
- 不一开始实现复杂 semantic-router clone。
- 不做 GPU kernel、KV cache scheduler、continuous batching 等推理引擎内部能力。
- 不把 UI/chatbot 作为核心展示。

这些能力可以作为未来扩展方向，但不是 v1 的核心。

## 6. 核心价值主张

一句话：

> Not another RAG demo. This project implements the SLO-aware context execution layer before LLM serving.

核心价值：

1. 把 RAG 从 Python glue 变成可分析、可追踪、可 benchmark 的 execution path。
2. 用 C++ 实现 retrieval hot path，展示底层系统能力，但不把项目降级成 vector DB 复刻。
3. 对 retrieval、rerank、context packing、LLM backend call 做统一编排。
4. 内建 stage-level tracing、quality evaluation 和 latency/cost benchmark。
5. 根据 latency SLO、quality target 和 context budget 动态选择执行路径。
6. 连接现代 LLM serving 关注的 TTFT、prompt tokens、prefix cache、backend routing 等指标。

## 7. 总体架构

### 7.1 Offline Path

```text
documents
  -> chunker
  -> embedding provider
  -> vector index builder
  -> local index files + metadata files
  -> evaluation query set
```

### 7.2 Online Path

```text
Request
  - query
  - latency_budget_ms
  - quality_target
  - max_context_tokens
  - cost_budget
        ↓
RequestAnalyzer
        ↓
EmbeddingProvider
        ↓
Retriever
        ↓
RetrievalConfidenceEstimator
        ↓
ExecutionRouter
        ↓
+-----------------------------+
| no retrieval                |
| retrieval only              |
| retrieval + rerank          |
| expanded retrieval + rerank |
| strict context packing      |
+-----------------------------+
        ↓
ContextPacker
        ↓
LLMBackendAdapter
        ↓
ExecutionResult + ExecutionTrace
```

### 7.3 Runtime 边界

本项目实现的是 context execution runtime，不是完整 serving backend。它可以选择是否调用 LLM backend，但重点是把 backend 调用纳入 trace 和 benchmark。

```text
Runtime owns:
  - request analysis
  - retrieval execution
  - rerank orchestration
  - context packing
  - routing policy
  - trace and metrics
  - benchmark harness

Runtime does not own:
  - model weights
  - GPU inference scheduler
  - distributed vector database
  - Agent planning loop
```

## 8. 核心模块

### 8.1 Request Model

职责：

- 定义一次请求的输入、预算和策略约束。
- 支持不同 execution profile：fast、balanced、quality、debug。

示例字段：

```text
Request
  - request_id
  - query
  - latency_budget_ms
  - max_context_tokens
  - quality_target
  - cost_budget
  - execution_profile
```

### 8.2 Vector Store

职责：

- 存储 dense embedding vectors。
- 使用 contiguous row-major memory layout。
- 管理 doc id、chunk id、offset、token count 等 metadata。
- 支持从离线构建文件加载。

设计重点：

- vector payload 与 chunk metadata 分离。
- retrieval hot path 尽量顺序扫描。
- 避免 query path 中频繁对象分配。
- 为 SIMD scoring 提供 aligned memory 选项。

定位说明：

> Vector Store 是 runtime 内部的 retrieval experiment core，不是 Milvus/Weaviate/FAISS 的替代品。

### 8.3 Scorer

职责：

- 计算 query vector 与 document vector 的相似度。
- 支持 dot product / cosine similarity。
- 提供 naive、SIMD 两种实现。

优化方向：

- AVX2 / AVX512 dot product。
- loop unrolling。
- batched query scoring。
- 减少 branch。
- 对不同 embedding dimension 做 benchmark。

### 8.4 TopK Selector

职责：

- 从 N 个候选向量中选出 topK。
- 支持 heap-based topK。
- 支持 per-thread local topK merge。

设计重点：

- 避免全量 sort。
- 控制临时内存分配。
- 支持 retrieval latency 与 K 的 ablation study。

### 8.5 Retriever

职责：

- 组合 VectorStore、Scorer、TopKSelector。
- 支持 single-thread 与 multi-thread scan。
- 输出 CandidateSet。

多线程设计：

```text
VectorStore shards
  -> thread-local scoring
  -> thread-local topK
  -> global topK merge
```

### 8.6 Retrieval Confidence Estimator

职责：

- 根据检索结果估计当前 retrieval 是否足够可信。
- 为 ExecutionRouter 提供决策信号。

输入信号：

- top1 score
- top1/topK score gap
- score distribution
- candidate count
- query length
- keyword pattern
- historical route statistics

输出：

```text
RetrievalConfidence
  - confidence_score
  - ambiguity_score
  - evidence_density
  - suggested_route_hint
```

### 8.7 Rerank Module

职责：

- 对 retriever 返回的 candidate set 做二阶段精排。
- 支持外部 reranker，如 bge-reranker 或 cross-encoder。
- 输出 reranked candidate list。

v1 不自研 rerank model，只实现 orchestration、timeout、fallback 和 metrics。

重点指标：

- rerank latency
- rerank timeout rate
- rerankN 对质量和延迟的影响
- no-rerank vs rerank 的 quality delta

### 8.8 Context Packer

职责：

- 将候选 chunks 组装成 LLM prompt context。
- 支持 token budget。
- 支持 chunk dedup。
- 支持 adjacent chunk merge。
- 支持 source attribution。
- 支持不同 packing policy。

设计重点：

- 在有限 token budget 下优先保留高价值 evidence。
- 控制 prompt token 数，避免无意义 prefill 成本。
- 输出可解释 source attribution，便于 evaluation 和 debug。

Packing policies：

```text
score_first
  - 按 score/rerank score 优先选择 chunks

diversity_first
  - 避免多个高度相似 chunks 占满 context

adjacency_merge
  - 合并相邻 chunks，提升上下文连贯性

strict_budget
  - 在低 latency 或低 cost 预算下 aggressively shrink context
```

### 8.9 Execution Router

职责：

- 根据 request budget、retrieval confidence、context pressure 和 cost signal 选择执行路径。

初版使用 rule-based policy，不训练复杂 classifier。

示例策略：

```text
high confidence + tight latency budget
  -> retrieval only

ambiguous query + enough budget
  -> retrieval + rerank

low confidence + quality profile
  -> expanded retrieval + rerank

short/simple query + no evidence need
  -> no retrieval

long context pressure
  -> strict context packing
```

目标不是追求“智能路由”概念本身，而是量化：

```text
always retrieval + rerank
vs
SLO-aware execution
```

在质量损失可控的前提下，是否降低 P95/P99 latency、prompt tokens 和 cost。

### 8.10 LLM Backend Adapter

职责：

- 对接不同 LLM backend。
- 将 backend latency 和 token metrics 纳入 trace。

支持对象：

- OpenAI-compatible API
- vLLM HTTP/OpenAI-compatible server
- SGLang HTTP/OpenAI-compatible server

记录指标：

- backend name
- prompt tokens
- completion tokens
- TTFT
- total generation latency
- tokens/sec
- timeout / error
- cache hint / prefix hit signal if available

v1 可以 mock backend 或使用 OpenAI-compatible local endpoint；关键是接口和 trace 设计要像真实 infra。

### 8.11 Execution Trace and Metrics

职责：

- 记录每次请求的完整 stage-level trace。
- 支持 JSONL 输出，便于离线分析。
- 可选支持 Prometheus-style metrics 和 OpenTelemetry trace format。

Trace 字段：

```text
ExecutionTrace
  - request_id
  - route
  - embedding_latency_ms
  - retrieval_latency_ms
  - rerank_latency_ms
  - packing_latency_ms
  - backend_ttft_ms
  - backend_total_latency_ms
  - total_latency_ms
  - prompt_tokens
  - completion_tokens
  - candidate_count
  - selected_chunk_count
  - transient_allocation_bytes
  - timeout_stage
  - fallback_used
  - quality_metrics
```

## 9. 技术栈

### 核心语言

- C++20：retrieval hot path、runtime core、benchmark binary。
- Python：baseline、embedding/rerank/LLM adapter、benchmark harness、report generation。

### 构建与测试

- CMake 或 Bazel 二选一。
- GoogleTest / Catch2 用于 C++ unit tests。
- pytest 用于 Python benchmark scripts。

### 依赖

- FAISS：retrieval baseline。
- sentence-transformers 或 OpenAI embedding API：embedding provider。
- bge-reranker / cross-encoder：rerank baseline。
- pybind11 或 C API：Python adapter。
- 可选：Prometheus client、OpenTelemetry SDK。

### 推荐优先级

```text
Must have:
  - C++ retrieval core
  - Python benchmark harness
  - FAISS baseline
  - trace JSONL
  - context packing
  - rule-based SLO router

Nice to have:
  - HTTP/gRPC server
  - Prometheus metrics
  - OpenTelemetry trace export
  - vLLM/SGLang live backend
```

## 10. Benchmark 设计

Benchmark 是本项目的核心资产。项目是否有说服力，主要取决于 benchmark 是否严肃。

### 10.1 对比对象

系统级 baseline：

- Plain Python RAG pipeline
- LangChain RAG pipeline
- LlamaIndex RAG pipeline

检索核 baseline：

- FAISS Flat index
- naive Python / NumPy baseline
- C++ naive scan
- C++ SIMD scan
- C++ multithread scan

执行策略 baseline：

- always retrieval only
- always retrieval + rerank
- always large topK + rerank
- SLO-aware execution

### 10.2 指标

Latency:

- P50
- P95
- P99
- max latency
- TTFT
- backend total latency

Throughput:

- QPS under fixed concurrency
- QPS under fixed latency SLO

Memory:

- index resident memory
- per-query transient allocation estimate
- metadata memory

Token and cost:

- prompt tokens
- completion tokens
- total tokens
- estimated API cost
- context packing token savings

Retrieval quality:

- recall@k
- MRR
- nDCG
- evidence hit rate
- context precision
- context recall

Answer quality:

- answer correctness
- faithfulness
- citation correctness

Cost decomposition:

- query embedding
- retrieval
- rerank
- context packing
- LLM backend prefill/generation
- total request time

### 10.3 必须产出的图表

- Latency breakdown by stage。
- P50/P95/P99 对比图。
- TTFT vs prompt tokens。
- topK vs latency。
- rerankN vs latency/quality。
- context budget vs quality/cost。
- vector dimension vs retrieval latency。
- concurrency vs QPS。
- naive vs SIMD vs multithread retrieval。
- always-rerank vs SLO-aware execution。
- route distribution under different workloads。

### 10.4 Benchmark 可信度要求

- 固定 corpus、query set、hardware config。
- 输出 raw result JSONL/CSV。
- 区分 warm run 和 cold run。
- 区分 online latency 和 offline index build cost。
- 报告 P50/P95/P99，而不是只报告平均值。
- 每个优化必须对应 ablation study。
- 明确说明 mock backend 和 real backend 的区别。

## 11. 8 周实施路线

### Week 1: Baseline, Dataset, and Cost Model

目标：

- 搭建普通 Python RAG pipeline。
- 准备 corpus、query set 和 ground truth。
- 得到第一版 stage-level latency breakdown。

交付物：

- baseline Python pipeline。
- docs chunking script。
- embedding generation script。
- FAISS baseline index。
- 初版 benchmark result。
- 初版 cost model：embedding / retrieval / rerank / packing / backend。

### Week 2: Benchmark and Evaluation Harness

目标：

- 把 benchmark 做成项目核心资产。
- 固化实验配置、指标输出和图表生成。

交付物：

- benchmark runner。
- result JSONL/CSV。
- latency histogram。
- quality metric scripts。
- topK / rerankN / chunk size ablation。
- 第一版 benchmark report。

### Week 3: C++ Vector Store and Brute-force Retrieval Core

目标：

- 实现可独立运行的 C++ retrieval core。
- 建立与 FAISS/Python baseline 的对照。

交付物：

- VectorStore。
- Scorer naive implementation。
- TopKSelector。
- Retriever。
- C++ benchmark binary。
- 与 Python/FAISS baseline 的初版对比。

### Week 4: SIMD, Multithreading, and Allocation Control

目标：

- 展示底层性能优化能力。
- 明确这些优化对端到端路径的实际影响。

交付物：

- AVX2 / AVX512 dot product。
- aligned vector storage。
- shard-level parallel scan。
- per-thread topK merge。
- transient allocation estimate。
- naive vs SIMD vs multithread benchmark。
- retrieval latency / QPS / memory report。

### Week 5: Rerank Orchestration and Context Packing

目标：

- 从 vector search 升级为 context execution。

交付物：

- CandidateSet。
- RerankClient。
- rerank timeout and fallback。
- ContextPacker。
- source attribution。
- no-rerank vs rerank benchmark。
- context budget ablation。

### Week 6: Execution Runtime, Trace, and Backend Adapter

目标：

- 串联完整 query -> context -> LLM backend execution path。
- 把 TTFT、tokens、backend latency 纳入 trace。

交付物：

- ExecutionRuntime。
- ExecutionResult。
- ExecutionTrace。
- config-driven execution。
- OpenAI-compatible backend adapter。
- mock backend for deterministic benchmark。
- stage-level trace output。
- end-to-end benchmark。

### Week 7: SLO-aware Execution Router

目标：

- 引入基于 latency/quality/cost budget 的动态执行策略。

交付物：

- RequestAnalyzer。
- RetrievalConfidenceEstimator。
- ExecutionRouter。
- rule-based route policy。
- always-rerank vs SLO-aware benchmark。
- latency/quality/cost tradeoff report。

### Week 8: Packaging, README, and Career Material

目标：

- 把项目包装成求职资产和后续 side project 主干。

交付物：

- GitHub README。
- architecture diagram。
- benchmark report。
- quickstart。
- design doc。
- side project roadmap。
- resume bullets。
- interview talking points。

## 12. 推荐目录结构

```text
slo-context-runtime/
  README.md
  CMakeLists.txt
  configs/
    fast.yaml
    balanced.yaml
    quality.yaml
    route_aware.yaml
  cpp/
    include/
      context_runtime/
        request.h
        vector_store.h
        scorer.h
        topk.h
        retriever.h
        confidence.h
        reranker.h
        context_packer.h
        router.h
        backend_adapter.h
        runtime.h
        trace.h
    src/
      vector_store.cc
      scorer_naive.cc
      scorer_simd.cc
      topk.cc
      retriever.cc
      confidence.cc
      context_packer.cc
      router.cc
      runtime.cc
    tests/
      vector_store_test.cc
      scorer_test.cc
      topk_test.cc
      retriever_test.cc
      context_packer_test.cc
      router_test.cc
  python/
    context_runtime/
      __init__.py
      baseline.py
      embeddings.py
      rerank.py
      llm_backend.py
      benchmark.py
      evaluation.py
      reports.py
    scripts/
      build_corpus.py
      run_baseline.py
      run_benchmark.py
      generate_report.py
  benchmarks/
    datasets/
    results/
    reports/
  docs/
    architecture.md
    benchmark_methodology.md
    routing_policy.md
    project_writeup.md
    side_project_roadmap.md
```

## 13. README 叙事结构

README 首页建议结构：

```text
# SLO-aware Context Execution Runtime

Not another RAG demo.

This project implements the request/context execution layer before LLM serving.
It dynamically chooses retrieval, reranking, context packing, and backend execution
strategies based on latency SLO, quality target, context budget, and cost.

## Why
## Architecture
## Features
## Benchmark
## SLO-aware Routing
## Trace and Observability
## Quickstart
## Design
## Roadmap
```

第一屏必须让招聘方立刻看到：

- 这是 infra 项目，不是应用 demo。
- 核心在 request/context execution path。
- 有 C++ performance work。
- 有 benchmark 和 trace。
- 理解 LLM serving 指标：TTFT、tokens、backend latency、cache。
- 与 RAG / long-context / LLM serving 相关。

## 14. 可衍生 Side Projects

这个项目适合作为主干，后续可以衍生出多个 side project。

### 14.1 LLM Context Gateway

定位：

> A gateway that manages context construction, routing, tracing, and backend selection for LLM requests.

能力：

- HTTP/gRPC API。
- OpenAI-compatible request/response。
- context runtime integration。
- request ID propagation。
- backend adapter：OpenAI / vLLM / SGLang。
- timeout、fallback、retry。

求职信号：

- 服务治理。
- LLM gateway。
- backend routing。
- production infra thinking。

### 14.2 RAGBench: RAG Execution Benchmark Platform

定位：

> A benchmark suite for measuring retrieval, reranking, context packing, latency, quality, and cost tradeoffs.

能力：

- 固定数据集和 query set。
- 自动跑 ablation。
- 输出图表和报告。
- 支持不同 pipeline 对比。

求职信号：

- evaluation discipline。
- benchmark methodology。
- data-driven optimization。

### 14.3 SLO Router

定位：

> A policy engine that chooses execution routes based on latency, quality, and cost budgets.

能力：

- rule-based routing。
- profile-based execution。
- latency budget enforcement。
- route-level metrics。
- future: learned policy。

求职信号：

- request routing。
- SLO-aware systems。
- LLM infra policy layer。

### 14.4 Context Cache Lab

定位：

> A research playground for context reuse, prompt prefix caching, and repeated evidence block caching.

能力：

- context block fingerprint。
- repeated query/context analysis。
- cache hit simulation。
- TTFT/token cost impact analysis。

求职信号：

- cache-aware LLM serving。
- long-context cost optimization。
- prefix/cache thinking。

### 14.5 Production RAG Observability

定位：

> A trace and metrics layer for production RAG request execution.

能力：

- per-stage trace。
- route distribution。
- retrieval confidence distribution。
- quality regression dashboard。
- failure reason analysis。

求职信号：

- observability。
- production debugging。
- offline/online quality monitoring。

## 15. 简历表述

英文简历：

```text
Designed and implemented an SLO-aware context execution runtime for RAG and long-context LLM serving,
covering vector retrieval, reranking orchestration, context packing, backend adapters, route-aware execution,
and stage-level observability.

Optimized the retrieval hot path in C++ with cache-aware vector layout, SIMD scoring, multithreaded topK
selection, and allocation control, benchmarking against Python RAG pipelines and FAISS baselines.

Built an end-to-end benchmark suite decomposing LLM request latency into embedding, retrieval, reranking,
context packing, TTFT, and generation stages, enabling quality-latency-cost tradeoff analysis under
different topK, rerankN, context budget, and routing policies.
```

中文面试介绍：

```text
这个项目不是做一个 RAG 应用，而是实现 LLM 请求进入模型之前的 context execution layer。
我重点关注 retrieval、rerank、context packing、backend call 这条路径的延迟、质量、成本和可观测性。
第一版用 Python/FAISS 建 baseline，再用 C++ 实现 retrieval hot path，并做 SIMD、多线程 topK 和内存布局优化。
后续加入 SLO-aware router，根据 latency budget、retrieval confidence 和 context budget 决定是否检索、是否精排、topK 多大、context 怎么打包。
最终用 benchmark 量化不同策略对 P95/P99、TTFT、prompt tokens 和回答质量的影响。
```

## 16. 面试讲述主线

建议按这个顺序讲：

1. RAG 系统常见问题不是功能缺失，而是 context execution path 碎片化。
2. 我把 query 到 model-ready context 的路径拆成 embedding、retrieval、rerank、packing、backend call。
3. 我先搭了 Python/FAISS baseline，得到 stage-level cost model。
4. 然后用 C++ 实现 retrieval hot path，优化 vector layout、SIMD dot product、多线程 topK。
5. 再把 retrieval 扩展为完整 execution runtime，加入 rerank orchestration、context packing、trace。
6. 最后加入 SLO-aware routing，在质量损失可控的前提下降低 P95/P99、TTFT 或 token cost。

一句话：

> I am not building another AI demo. I am building the SLO-aware context execution layer before LLM serving.

## 17. 风险与规避

### 风险 1: 做成普通 RAG demo

规避：

- README 第一屏强调 execution layer。
- 重点展示 benchmark、trace、C++ core、routing policy。
- 不把 UI / chat app 作为核心展示。

### 风险 2: 做成半个 vector database

规避：

- 不做事务、WAL、SQL、分布式、多租户。
- Vector store 只服务 retrieval execution 和性能实验。
- 明确与 FAISS 做 baseline 对比，而不是正面对标完整 vector DB。

### 风险 3: C++ 优化脱离端到端收益

规避：

- 每个 C++ 优化都同时报告 retrieval-only 和 end-to-end impact。
- 如果 retrieval 不是瓶颈，诚实展示 bottleneck 转移到 rerank/backend/prompt tokens。
- 用这个结果引出 SLO-aware routing 和 context packing 的价值。

### 风险 4: 过早追 semantic-router

规避：

- v1 只做 retrieval execution 和 trace。
- v2 做 rule-based SLO router。
- 不做复杂 policy engine、guardrail、multi-agent routing。

### 风险 5: benchmark 不可信

规避：

- 固定 corpus、query set、hardware config。
- 输出 raw result。
- 区分 warm/cold run。
- 报告 P50/P95/P99，而不是只报告平均值。
- 做 ablation study。
- 明确 mock backend 与真实 backend 的差异。

## 18. 项目成功标准

8 周结束时，项目应满足：

- 有一个可运行的 C++ retrieval core。
- 有 Python baseline 和 FAISS baseline。
- 有完整 query -> context -> backend execution path。
- 有 stage-level tracing。
- 有 benchmark report。
- 有 naive/SIMD/multithread 对比。
- 有 no-rerank/rerank 对比。
- 有 context budget ablation。
- 有 always-rerank/SLO-aware 对比。
- 有 TTFT、prompt tokens、backend latency 指标。
- README 能在 30 秒内传达项目价值。
- 简历能自然指向 AI Infra / LLM Systems / Retrieval Engineer。

## 19. 最终项目叙事

最终对外叙事：

> This project explores the SLO-aware context execution layer before LLM serving.
> Instead of building another RAG application, it focuses on the low-latency, measurable, and policy-driven path
> from user query to model-ready context, including retrieval, reranking, context packing, backend execution,
> routing decisions, and stage-level benchmark.

中文版本：

> 这个项目研究 LLM 请求进入模型之前的 context execution path。
> 它不是一个 RAG 应用，而是一个低延迟、可观测、可 benchmark、可按 SLO 动态决策的请求执行层，
> 重点展示 retrieval、rerank、context packing、backend adapter、routing decision 和系统性能优化能力。

对转型的核心意义：

> 它把传统广告基础架构里的召回、排序、低延迟、实验、可观测性经验，迁移到 LLM infra 的 context execution 和 serving-adjacent 问题上。

