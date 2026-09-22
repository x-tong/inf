# ADR 0003: DecisionBackend 只提供信号

## 状态

Accepted

## 日期

2026-09-23

## 背景

旧设计把 `ExecutionRouter` 和 `RetrievalConfidenceEstimator` 写成两个会做判断的核心模块。2026-09-15 TypeSafe 公开的 Jev 是 System One 决策模型：输入 state 和封闭问题，输出带概率的类型化答案，不生成文本，也不执行预算、packing 或 trace。它不是 runtime。

若把 Router 本身换成可插拔模型，SLO、前缀不变量和 fallback 会离开本仓库拥有的代码。若在 v1 必经路径上调用 Jev，作品会变成接入演示，并且决策延迟（厂商报告约 70–500ms）会和要测量的 TTFT 混在一起。

## 决策

`ExecutionRouter` 留在 runtime。它的输入是预算、packer 已算出的事实，以及一份 `Decision`。给定这些输入，路由结果必须确定，且单测不联网。

`DecisionBackend` 只回答两个封闭问题，不选路：

- `need_evidence`：这一轮要不要把新 evidence 追加到后缀。
- `tool_result_duplicate`：这段工具结果是否已经在当前上下文中。

`Decision` 使用 runtime 自己的字段，不出现厂商 primitive 名称：

```text
need_evidence
tool_result_duplicate
confidence
backend                 rules | mock | jev
latency_ms
input_tokens
overridden
override_reason         budget | low_confidence | backend_error | prefix_invariant | none
```

默认实现是 `RulesDecisionBackend`，延迟和 input tokens 记 0，置信度记 1。没有外部 API key 也必须跑通全部测试和 mock benchmark。

以下事项不交给 DecisionBackend：

- 前缀字节是否可改。除显式 compaction 外，前缀字节保持不变。
- token、延迟和成本预算是否付得起 compaction 或一次决策调用。
- 超时、错误、置信度处于中间带时退回哪条规则。

Jev 或同类模型只允许作为同接口的可选实现，不在 v1 的 10 周必经路径上。适配器负责把上述两个问题翻译成厂商 API，再映射回 `Decision`。trace 必须钉死模型版本，禁止使用会漂移的 `latest` 别名。第 10 周验收通过之前不实现该适配器。

`RetrievalConfidenceEstimator` 撤销，不保留空接口。检索分差不再是 v1 信号。

## 原因

- 要不要 compact、前缀算不算长，用 token 数和前缀哈希就能决定。交给模型会让实验不可复现。
- 规则实现把「策略」和「信号从哪来」拆开，以后才能在同一 packer 上对照 rules 与 Jev。对照指标必须包含 `decision_latency_ms` 和决策调用的 input tokens；决策变慢可以吃掉前缀省下的 TTFT，这是有效结论。
- 厂商的速度和校准数字不是本项目的前提。

## 后果

- v1 代码包含 DecisionBackend 接口和规则实现，不包含 Jev 客户端。
- 路由单测使用固定 `Decision`，不依赖规则以外的后端。
- 将来若接入决策模型，失败时覆盖为规则结果，并写 `override_reason`。预算不够支付该调用时同样覆盖，不发起调用。
