# ADR 0002: v1 主问题是 cache-aware context execution

## 状态

Accepted

## 日期

2026-09-23

## 背景

本仓库位于 Agent / App 和 OpenAI-compatible LLM backend 之间，不拥有推理引擎，也没有稳定 GPU。

多轮会话进模型之前，context 如何组装，决定 prefix cache 还能不能用。前缀改一个字节，已缓存的 KV 后缀就会失效。serving 侧已有 prefix-aware / KV-aware 路由，补不了客户端把前缀改坏。模型路由、费用和 guardrail 已由 AI Gateway 覆盖，不在本仓库范围内。

## 决策

v1 证明：在同一信息保留约束下，前缀字节稳定的 packing 加规则路由，相对三条对照（整段重写、无上限追加、默认 compaction），降低 uncached tokens 和 mock TTFT。每次决策一条 JSONL trace。绝对毫秒数不声称等于某张 GPU。

执行路径：

```text
session + 新的一轮（用户消息或工具结果）
  -> budget
  -> pack（稳定前缀 | 可变后缀，默认只追加）
  -> rule router
  -> backend（mock TTFT，可选 OpenAI-compatible cached tokens）
  -> JSONL trace
```

v1 不包含：

- 检索核、向量库、embedding、rerank、SIMD
- 与外部 RAG 框架或向量库的对照
- Beam Search、推理引擎、Attention kernel、KV scheduler
- HTTP / gRPC server、Prometheus、OpenTelemetry
- 以模型摘要充当 compaction

学习计划写在 v1 规格内，不另建课程文档。规格路径：

`docs/projects/cache-aware-context-runtime-v1.md`

trace 没有 embedding 阶段，不设对应字段。

## 原因

- 要测量的失败发生在进引擎之前：packing 是否保持前缀字节稳定。
- mock 按未命中前缀的 token 数计算 TTFT，无 GPU 也能复现排序。真后端只核对 cached tokens，不是本地必跑测试。
- 检索延迟不是这条链路要证明的瓶颈。

## 后果

- 实现、测试和 benchmark 以 v1 规格为准。
- 对照对象是四条 packing 策略。
- 若以后加入检索结果，只作为追加到后缀的 evidence，不成为主路径。
- 对外名称使用工作标题 Cache-aware Context Execution Runtime。仓库名 `inf` 不变。对外名称另行确认前，不把该标题写成已经公开定名。
