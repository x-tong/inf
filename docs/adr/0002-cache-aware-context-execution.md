# ADR 0002: 主问题改为 cache-aware context execution

## 状态

Accepted

## 日期

2026-09-23

## 背景

仓库里有两份互相冲突、且都没有实现的设计：

- `docs/rag-aware-request-execution-runtime-project.md` 把主路径定义成单次 RAG：`query → embed → retrieve → rerank → pack → trace`，成功标准包含 C++ 暴力检索核、SIMD、FAISS / LangChain 对照。
- `docs/projects/cpp-runtime-skeleton-v1.md` 改成 mock-first 的 runtime 骨架，但执行路径仍是检索和 rerank。

2026 年 9 月的实现依据不再是这条路径。模型路由、费用和 guardrail 已由 AI Gateway 覆盖；serving 侧已有 prefix-aware / KV-aware 路由。本仓库不拥有推理引擎，也没有稳定 GPU。单次 RAG 编排加一个本地暴力扫描核，不能证明进引擎之前的执行层是否破坏了 prefix cache。

## 决策

主问题改为：多轮请求进入 LLM backend 之前，context 如何组装，决定 prefix cache 还能不能用。

v1 要证明的是：在同一信息保留约束下，前缀字节稳定的 packing 加规则路由，相对三条对照（整段重写、无上限追加、默认 compaction），降低 uncached tokens 和 mock TTFT；每次决策一条 JSONL trace。绝对毫秒数不声称等于某张 GPU。

v1 执行路径：

```text
session + 新的一轮（用户消息或工具结果）
  -> budget
  -> pack（稳定前缀 | 可变后缀，默认只追加）
  -> rule router
  -> backend（mock TTFT，可选 OpenAI-compatible cached tokens）
  -> JSONL trace
```

以下各项退出 v1 成功标准，不再作为实施计划：

- 自研向量核、SIMD、多线程 topK
- FAISS、LangChain、LlamaIndex 对照
- embedding、rerank 模型与 RetrievalConfidenceEstimator
- Beam Search
- HTTP / gRPC server、Prometheus、OpenTelemetry

学习计划写在 v1 规格内，不另建课程文档。规格路径：

`docs/projects/cache-aware-context-runtime-v1.md`

旧项目书和 C++ 骨架设计降为历史记录，正文保留，不再作为实现依据。

## 原因

- 前缀改一个字节，已缓存的 KV 后缀就会失效。这个约束发生在进引擎之前，引擎侧的 prefix-aware 路由补不回来。
- 检索延迟通常不是这条链路的端到端瓶颈；在 macOS 上把 AVX2 / AVX512 当标题也与机器不符。
- mock 按未命中前缀的 token 数计算 TTFT，无 GPU 也能复现排序关系。真后端只用于核对 cached tokens，不作为本地必跑测试。
- 尚未发出过 trace，不存在需要兼容的旧 schema。骨架设计里的 `embedding_latency_ms` 不进入 v1 必填字段。

## 后果

- 实现、测试和 benchmark 以 v1 规格为准。与旧项目书冲突时，以本 ADR 和规格为准。
- v1 的对照对象是四条 packing 策略，不是 RAG 框架。
- 检索若以后出现，只作为追加到后缀的 evidence，不恢复为系统主路径。
- 对外名称仍使用工作标题 Cache-aware Context Execution Runtime。仓库名 `inf` 不变。对外名称另行确认前，不把该标题写成已经公开定名。
