# Cache-aware Context Execution Runtime v1

## 状态

本文件是实现依据。工作标题，对外名称未最终确认。

配套决策：

- [ADR 0002](../adr/0002-cache-aware-context-execution.md)：v1 主问题和执行路径。
- [ADR 0003](../adr/0003-decision-backend.md)：Router 与 DecisionBackend 的边界。

学习计划在本文末尾，不另建文档。

## 背景

本仓库位于 Agent / App 和 OpenAI-compatible LLM backend 之间。它不拥有推理引擎。

prefix cache 只复用字节级相同的前缀。多轮会话里，系统提示、历史和工具结果若被重写进前缀，backend 上已有的 KV 会失效，uncached tokens 和 TTFT 上升。serving 侧的 prefix-aware 路由解决不了客户端把前缀改坏。

v1 用可复现实验衡量这件事。没有稳定 GPU。TTFT 的绝对毫秒数来自写明常数的 mock；排序关系可以由 mock 单独成立。真后端只核对 cached tokens 字段，不是必跑依赖。

## 目标

- 用 C++23 和 xmake 实现 session 上的 context execution runtime。
- packing 默认保持前缀字节不变，工具结果只进后缀。
- 规则 Router 在 token、延迟和成本预算下决定追加、截断或显式 compaction。
- 每次请求写一条 JSONL trace，能解释路由、前缀是否变化、uncached tokens 和 mock TTFT。
- 用固定 fixture 对照四条策略，并给出一条复现命令。

## 非目标

v1 不做：

- Agent 循环、工具沙箱、记忆系统。
- 检索核、向量库、embedding、rerank、SIMD。
- 与外部 RAG 框架或向量库做对照。
- 推理引擎、Attention kernel、KV scheduler、PD 分离、投机解码、Beam Search。
- 自研决策模型，或在前 10 周调用 Jev。
- 用模型做摘要。compaction 必须是确定性字符串变换。
- HTTP / gRPC server、Prometheus、OpenTelemetry、聊天 UI。
- 把 mock TTFT 写成某张 GPU 的测量结果。

## 系统边界

```text
Runtime owns:
  session 与 turn
  前缀 / 后缀 packing
  预算与规则路由
  DecisionBackend 接口和规则实现
  mock backend 与 OpenAI-compatible adapter 的边界
  JSONL trace

Runtime does not own:
  模型权重和 GPU scheduler
  决策模型的权重或托管服务
  Agent planning loop
  向量数据库
```

## 执行路径

```text
Session + Turn
  -> RequestBudget
  -> DecisionBackend（默认规则，只出信号）
  -> ExecutionRouter（runtime 拥有，结果确定）
  -> ContextPacker
  -> BackendAdapter
  -> ExecutionResult + 一条 JSONL
```

## 数据模型

### Session

- `session_id`
- `system_text`：构造时给定，输入侧不可变。
- `required_fact_ids`：质量检查必须仍能在最终 prompt 中找到的标识。
- 已提交的 prefix 字节和已经打包过的 block 列表，供下一轮比较。

### Turn

- `turn_index`，从 0 开始。
- `kind`：`user` 或 `tool`。
- `text`
- `fact_ids`：这一轮带入的事实标识。没有则为空。fixture 必须显式填写，v1 不做自然语言判断。

### Block

- `id`
- `placement`：`prefix` 或 `suffix`
- `source`：`system` | `user` | `tool` | `digest`
- `text`

Prompt 字节是 prefix blocks 按提交顺序拼接，再拼接 suffix blocks。编码为 UTF-8。哈希是整段 prefix 字节的 SHA-256。

### Budget

- `latency_budget_ms`
- `max_context_tokens`
- `cost_budget`
- `tool_cap_bytes`
- `compaction_latency_ms`：执行一次 compaction 从延迟预算中扣除的常数。

### Decision

见 ADR 0003。Router 不读取厂商原始响应。

## 四条策略

策略由配置选定。Router 只在 `stable_prefix` 上做预算决策。另外三条是对照臂，故意不优化。

token 数在 v1 一律是估计值：`estimated_tokens = ceil(utf8_bytes / 4)`。trace 用 `token_accounting=estimate` 标明。接上真 tokenizer 或厂商 usage 之前，报告不得把该字段称为真实 token。

### rewrite_all

每一轮用新模板重写前缀，模板至少包含 `turn_index` 和到本轮为止的全部 turn 文本。因此除第 0 轮外 `prefix_changed=true`。不截断、不 compact。可以超出 `max_context_tokens`，trace 记录 `budget_exceeded=true`，请求仍返回 prompt。

### append_uncapped

第 0 轮前缀是 `system_text` 的原始字节。之后前缀字节原样复制，新内容只进后缀，不截断、不 compact。超预算时同样只记录 `budget_exceeded=true`。

### compact_by_default

第 0 轮与 `append_uncapped` 相同。从第 1 轮起每轮都 compaction。这是「默认摘要」的确定性替身，用来制造前缀失效，不是摘要质量方案。

### stable_prefix

治疗臂，必须遵守预算。

1. 第 0 轮前缀等于 `system_text` 原始字节。之后默认复制上一轮前缀字节。
2. `tool` 且 `tool_result_duplicate=true`：不追加该工具结果。
3. `user` 且 `need_evidence=false`：只追加用户文本，不追加 evidence。
4. `user` 且 `need_evidence=true`：在后缀追加用户文本；若 turn 带有尚未出现的 `fact_ids`，再追加一个 evidence block，内容为这些 id 的确定格式列表，不调用检索。
5. 工具结果超过 `tool_cap_bytes` 时，只截断将要放入后缀的那一段。
6. 估计 token 仍超过 `max_context_tokens` 时：剩余延迟预算够支付 `compaction_latency_ms` 则 compaction，并记 `compaction_applied=true`；否则失败，错误码 `budget_exceeded`。失败也要写 trace。

除 compaction 外，任何路径都不得改写已提交的前缀字节。

### compaction 的定义

compaction 把整个 prompt 换成一个新的 prefix block，并清空后缀：

```text
digest:<按字典序排列、且仍存在于 compact 前 prompt 中的 required_fact_ids，以逗号连接>
```

因此前缀哈希必然变化。不在该列表中的工具文本会被丢掉。质量检查只要求 `required_fact_ids` 仍出现；做不到就记质量失败，不换一套更聪明的摘要。

## Router

`stable_prefix` 的路由顺序就是上一节的 1–6。给定 Session、Turn、Budget、上一轮 prefix 字节和 Decision，输出必须唯一。

对照臂不经过这 6 步，直接执行该臂的定义。

预算覆盖决策调用：若某个非规则 DecisionBackend 的预估延迟或成本超过剩余预算，不发起调用，改用规则 Decision，`overridden=true`，`override_reason=budget`。v1 没有这种后端，这条规则先写成测试可注入的行为。

## DecisionBackend

接口只产生 ADR 0003 的 `Decision`。

### RulesDecisionBackend

v1 唯一实现。

- `tool_result_duplicate`：`kind=tool` 且该 `text` 的 SHA-256 已经出现在本 session 已打包 block 中。否则 false。
- `need_evidence`：`kind=user` 且 `fact_ids` 中至少有一个尚未出现在已打包文本中。`fact_ids` 为空则为 false。
- `confidence=1`，`backend=rules`，`latency_ms=0`，`input_tokens=0`，`overridden=false`，`override_reason=none`。

禁止用分词、检索分数或网络请求实现这两个信号。

### Jev

不在前 10 周实现。将来的适配器把上面两个布尔问题映射到厂商的封闭问题，trace 记录具体模型版本、`latency_ms` 和 `input_tokens`。问题文本和阈值到那时再写，不在本规格里预留厂商 schema。

## BackendAdapter

```text
BackendAdapter
  -> MockBackendAdapter          v1 必做
  -> OpenAICompatibleBackendAdapter   第 7 周，无 key 不得阻塞
```

### Mock TTFT

```text
uncached = prefix_changed 或没有上一轮 ? prompt_tokens : suffix 的 estimated_tokens
cached   = prompt_tokens - uncached
ttft_ms  = base_ms + per_uncached_token_ms * uncached
```

`prefix_changed` 的定义：存在上一轮，且本轮 prefix 字节与上一轮 prefix 字节不同。第 0 轮没有上一轮，`prefix_changed=false`，但 cache 为空，因此 uncached 等于全部 prompt tokens。

`base_ms` 和 `per_uncached_token_ms` 放在配置里。下面的数是占位，保证单测能比较大小，不是 GPU 标定。第 3 周的笔记确认或修订这两常数，并写明修订理由。在那之前 benchmark 报告不得引用具体毫秒数作为性能结论。

```text
base_ms: 10
per_uncached_token_ms: 0.05
```

mock 不产生随机抖动。跨请求的 P95 只来自 fixture 里不同轮次的混合。报告必须写明这一点。

### OpenAI-compatible

第 7 周把厂商 usage 归一为 `cached_prompt_tokens` 和 `uncached_prompt_tokens`，原始 usage 原样放进 trace。`token_accounting=provider`。默认测试使用录制响应，不访问网络。活体调用需要的前缀长度以该厂商当时的 cache 门槛为准，写进当次报告，不写死在本规格。

## Trace

每次调用一行 JSONL。必填：

```text
request_id
session_id
turn_index
policy                         rewrite_all | append_uncapped | compact_by_default | stable_prefix
profile                        tight | default | loose
decision_backend
need_evidence
tool_result_duplicate
decision_confidence
decision_latency_ms
decision_input_tokens
decision_overridden
override_reason
prefix_sha256
prefix_changed
compaction_applied
tool_result_bytes
prompt_tokens
cached_prompt_tokens
uncached_prompt_tokens
token_accounting               estimate | provider
completion_tokens
backend_ttft_ms
backend_total_latency_ms
total_latency_ms
budget_exceeded
fallback_used
fallback_reason
timeout_stage
error_code
max_context_tokens
latency_budget_ms
```

## 错误与 fallback

错误用 `std::expected<T, RuntimeError>` 穿出 runtime 边界。

错误码：

- `invalid_request`
- `config_error`
- `stage_timeout`
- `budget_exceeded`
- `backend_error`

行为：

- backend 超时：返回 `backend_error` 或 `stage_timeout`，trace 保留已经完成的字段。
- `stable_prefix` 无法在预算内放下 prompt：返回 `budget_exceeded`，trace 完整。
- 对照臂超出 token 预算：不失败，`budget_exceeded=true`。
- 决策后端失败：使用规则 Decision，`decision_overridden=true`，`override_reason=backend_error`。

## 配置

三个预算档：

- `configs/tight.yaml`
- `configs/default.yaml`
- `configs/loose.yaml`

每份至少包含：`policy`、`latency_budget_ms`、`max_context_tokens`、`cost_budget`、`tool_cap_bytes`、`compaction_latency_ms`、mock TTFT 两常数、trace 输出路径。

第 2 周再落文件。在那之前以下面这组 `default` 为讨论基准：

```text
policy: stable_prefix
latency_budget_ms: 200
max_context_tokens: 1024
cost_budget: 1.0
tool_cap_bytes: 256
compaction_latency_ms: 50
mock_ttft_base_ms: 10
mock_ttft_per_uncached_token_ms: 0.05
```

`tight` 把 `max_context_tokens` 和 `latency_budget_ms` 降到会触发预算覆盖的水平。`loose` 提高到四条策略都不触顶。具体数字在第 8 周的测试里固定，并写进配置，不在本文件猜一套最终值。

## 质量约束

fixture 给出 `required_fact_ids`。一次请求通过质量检查，当且仅当每个 id 都作为子串出现在最终 prompt 中。

不使用 LLM-as-judge，不计算 recall@k、MRR、nDCG、faithfulness。

## 验证与 benchmark

实现开始后，每次代码变更至少：

```bash
xmake f -m debug
xmake
xmake run context_runtime_tests
```

CLI 冒烟：

```bash
xmake run runtime_cli --config configs/default.yaml --query "test query"
```

第 2 周的 CLI 可以先接受单轮 query。多轮 session 由第 4 周起的测试入口覆盖。CLI 的最终参数在实现该 target 时写成 `--help` 能解释的形式，不在本周提交代码。

第 6 周起，benchmark 必须包含：

- 固定 session fixture 和四条策略。
- 原始 JSONL 路径。
- 一条复现命令。
- 运行环境：操作系统、无 GPU、mock 常数、`token_accounting`。
- 指标：prompt tokens、uncached tokens、mock TTFT、fixture 混合上的 P95、`prefix_changed` 次数、compaction 次数、质量检查通过与否。
- 明确写出 P95 来自负载混合，不是运行时抖动。

第 7 周若没有 API key，交付录制响应的解析测试，不交付活体数字。

## 实现布局

第 2 周才创建代码。目标布局：

```text
cpp/include/context_runtime/
cpp/src/
cpp/tests/
cpp/tools/
configs/
benchmarks/
xmake.lua
```

xmake target：

```text
context_runtime
context_runtime_tests
runtime_cli
```

不创建检索、精排或向量类型。

## 风险

| 风险 | 规避 |
|---|---|
| mock 毫秒数被当成 GPU 性能 | 笔记和报告写明常数来源；没有 `token_accounting=provider` 的对照时，不引用绝对 TTFT |
| compaction 做成模型摘要，实验不可复现 | v1 只允许本文件定义的 digest 替换 |
| 对照臂被悄悄加上截断，拉平差异 | 对照臂允许超预算并必须留下 `budget_exceeded` |
| Jev 提前进入必经路径 | 前 10 周不添加客户端；接口测试用注入的 Decision |
| 估计 token 与厂商 token 不一致 | 字段区分 `estimate` 和 `provider`；结论写明用的是哪一种 |

## 学习计划

学习不另开文档。每一周以本节为准。仓库里没有对应证据，这一周不算完成。时间不够时砍第 7、8 周，不砍第 4–6 周。

| 周 | 要搞懂什么 | 学会的证据 | 不学什么 |
|---|---|---|---|
| 1 | 边界：拥有进模型前的组装、预算和 trace；不拥有引擎、向量库、Agent 循环、Jev | 本文件、ADR 0002、ADR 0003。能讲清 v1 路径 | 不读推理引擎源码 |
| 2 | 请求如何穿过 Runtime：Budget、规则 Decision、MockBackend、一条 JSONL | `xmake` 与测试、CLI 在无 key、无 GPU 下跑通 | 不学 packing 策略 |
| 3 | Prefill 算力密集，Decode 带宽密集；KV cache；prefix cache 要求前缀字节不变。mock 公式只保证排序 | `docs/notes/serving-literacy.md`，以及改写前缀比追加更慢的单测 | 不读 nano-vLLM，不写 kernel，不做 PD 分离和投机解码 |
| 4 | 前缀稳定是字节级的。同批共享前缀在这里就是：追加不改前缀，改一个字节就整段失效 | Packer 单测：哈希不变，或 `prefix_changed` | 不学检索和 SIMD |
| 5 | 四条策略的代价。规则路由只消费两个信号 | 同一 fixture 的四条 trace，数字分开 | 不学 Jev，不学 learned router |
| 6 | 实验怎样才算数：固定 fixture、原始 JSONL、复现命令；P95 来自负载混合 | `benchmarks/` 中一条命令打出报告 | 不做外部框架对照，不做 LLM-as-judge |
| 7 | 厂商 cached tokens 如何归一；前缀要多长才会命中 | 录制响应的解析测试。有 key 才补一小段真跑 | 不部署 vLLM，不实现服务端路由 |
| 8 | SLO 是约束。预算不够就覆盖路由，并写 `override_reason` | 紧预算 / 松预算两臂的测试 | 不做 HTTP 服务，不接 Jev |
| 9 | 用代码里存在的数字说明为何停在这一层 | README、简历三条、口述稿与 benchmark 一致 | 不做新功能 |
| 10 | 声称不能超出 trace 能支持的范围 | 若真后端推翻 mock 排序，报告里有订正 | 不开新模块 |

第 3 周是唯一集中识字的一周。笔记提纲：Prefill 与 Decode、KV cache、前缀字节稳定、这对 packing 的惩罚、mock 公式及其不代表什么、本仓库不实现的引擎题。

第 10 周通过之前，不排 Jev。通过之后若要做对照，使用同一 fixture 和同一组问题，trace 包含决策延迟和决策 input tokens。决策延迟吃掉前缀收益时，报告照实写。

## 后续路线

前 10 周的日历就是上一节。不提前实现：

- Jev DecisionBackend
- 活体多厂商对照
- 后缀中的真实检索 evidence
- HTTP 服务
