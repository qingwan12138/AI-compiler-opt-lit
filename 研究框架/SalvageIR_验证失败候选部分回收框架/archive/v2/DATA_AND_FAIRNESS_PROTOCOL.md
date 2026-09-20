# 历史归档：v2，不得用于新实验执行

原文件：DATA_AND_FAIRNESS_PROTOCOL.md。2026-09-21 被 v3 替代；保留原文用于追溯，不代表当前约束。

# 候选数据与多模型公平性协议

## 1. 为什么模型来源必须成为实验设计的一部分

不同模型产生的失败候选分布不同：弱模型可能大量语法失败，强模型可能失败较少但编辑更复杂。若直接合并候选，SalvageIR 的效果可能只是某个模型的错误模式造成的。

因此数据单位定义为：

```text
candidate_id = hash(source_id, model_id, model_revision,
                    prompt_version, decoding_config,
                    generation_index, raw_output)
```

任何方法都从同一个冻结 `candidate_id` 开始。模型不参与 SalvageIR 内部决策，只作为数据生成器和分层因素。

## 2. 两条候选来源

### Track A：公开 artifact 重放

优先清点 LLM-VeriOpt CGO 2026 artifact：

- 论文测试集包含 4,386 个评价样本；
- 公开仓库声明保存模型输出、Alive2 日志和详细指标；
- 配置目录含 Qwen、Llama、CodeLlama 等 SFT/基础模型配置；
- 完整 Zenodo 档案为 1.1 GB。

主实验只纳入：

- base/zero-shot；
- 监督微调模型；
- 公开、版本可固定的非 RL 模型。

LLM-VeriOpt 的 GRPO/模型正确性/模型延迟输出不进入主候选池，因为本研究明确不做 RL。若保留，单列为“高正确率模型压力测试”，不与非 RL 主结果合并。

该 artifact 的公开通用 Prompt 以最小化延迟为目标。因此它只服务 P0 的“失败候选是否可分解、是否可语义回收”现象研究，并沿用/复核其声明的延迟口径。论文主目标已经冻结为 code size，这些 latency-oriented 输出不能进入主要盈利性数据；正式主数据必须由 Track B 按代码大小目标重新生成。

### Track B：受控重新生成

为避免完全依赖一个 artifact，使用项目级隔离的源 IR 重新生成候选：

- IR-OptSet 的 160 函数项目隔离发现集与 160 函数顺序扩样保留池；
- LLVM Opt Benchmark/nightly 的 24 项目 × 10 函数未触碰确认集；
- LLVM test-suite 的 24 个端到端和跨后端保留程序。

先冻结源函数，再调用模型。不得看到某模型输出后换函数。

Track B 的 Prompt、搜索和最终评价都明确优化 code size。不得用 latency-oriented 候选在事后切换目标并声称方法发现了原任务收益。确定性选择算法、配额和扩样门见 [`PILOT_PROTOCOL.md`](PILOT_PROTOCOL.md)。

## 3. 模型矩阵

受控 Track B 不再使用 artifact 的任务特定 SFT adapter。统一改用可固定 revision、未针对 LLM-VeriOpt latency 目标微调的 instruct checkpoint：

| 家族 | 配置 | 用途 |
|---|---|---|
| CodeLlama | `meta-llama/CodeLlama-7b-Instruct-hf` | 老一代代码模型错误分布 |
| Llama 3 | `meta-llama/Llama-3.2-3B-Instruct` | 较新通用模型家族 |
| Qwen 2.5 | `Qwen/Qwen2.5-Coder-3B-Instruct` | 较新代码模型家族 |

同一家族不同参数规模作为亚组，不冒充独立家族。模型 ID、revision、权重 SHA-256、许可证、官方训练/对齐披露、量化、推理引擎和 chat template 写入 `models.lock.json`。本研究不进行 SFT、DPO、PPO、GRPO 或其他任务特定训练；上游 checkpoint 的对齐史作为候选来源属性披露，不能含糊写成“模型从未用过 RL”。若某 checkpoint 无法合法取得，只能在第一条生成前发布新协议版本；不能根据失败率替换模型。artifact SFT 输出只留在 Track A，不能与 Track B 主结果合并。IR 专用模型和更强模型只可作为 P2 外部有效性补充，不能替换这三个预注册家族或用于 P0 调门。

这一变化牺牲了“直接复用 artifact adapter”的便利，但消除了用 latency-oriented SFT 生成 code-size 候选的核心混淆。论文不比较哪个 LLM 更强；模型只是冻结的候选生成层。

## 4. 统一生成协议

所有模型获得语义等价的任务说明：

```text
Given the LLVM IR function below, produce a complete optimized LLVM IR
function with the same signature and observable behavior. Optimize for the
declared objective. Do not return explanations. Do not add stronger
preconditions, undefined behavior, or unsupported external dependencies.
```

控制项：

- 相同源 IR；
- 相同目标：代码尺寸或延迟，不混用；
- 相同示例数量和信息内容；
- greedy decoding 为主，或统一 temperature/seed 的重复采样；
- 相同最大输出 Token；
- 相同最大请求数；
- 原始响应完整保存；
- API 不支持固定 seed 时记录不可复现性。

不同 tokenizer 造成 Token 数不可直接相等，因此同时限制：每函数最大请求、字符/输出上限、总费用和墙钟；Token 成本按模型单列。

## 5. 候选漏斗

每次生成进入且只能进入一个状态：

| 状态 | 判定 | SalvageIR 主任务 |
|---|---|---|
| `FORMAT_INVALID` | 无法提取完整 IR | 否，报告模型可用性 |
| `IR_INVALID` | LLVM parser/verifier 失败 | 否，可作未来语法修复任务 |
| `REFUTED_ELIGIBLE` | 可解析且 Alive2 明确反驳，范围支持 | 是 |
| `REFUTED_UNSUPPORTED_SCOPE` | 明确反驳但方法范围不支持 | 否，报告覆盖率 |
| `UNKNOWN_TIMEOUT` | Alive2 超时 | 否 |
| `UNKNOWN_UNSUPPORTED` | Alive2 不支持 | 否 |
| `VERIFIED_COPY` | 与源相同或仅非语义变化 | 否 |
| `VERIFIED_NO_GAIN` | 正确但没有主目标收益 | 否，不混入失败恢复 |
| `VERIFIED_PROFITABLE` | 正确且有收益 | 否，计入端到端基线 |

论文必须展示每个模型的完整漏斗，防止只抽取“容易恢复的失败”。

## 5.1 非 LLM 匹配负对照

为证明研究对象确实具有 LLM Translator 特征，从已经 verified 的自然候选或传统 `-Oz` 输出构造非 LLM 负对照：在不查看恢复结果的前提下，按每个 LLM 候选的编辑原子数、组件数、CFG 是否变化和语义 flag 类别做 1:1 匹配变异。负对照只回答：

- LLM 失败是否更常呈现“正确编辑与错误编辑混合”的结构；
- SalvageIR 的相对优势是否与候选来源存在交互。

负对照不混入 LLM 主成功率，也不能用人为可分样例替代自然候选。若方法对两类输入同样有效，论文必须改写为通用 RCES/IR repair，不把 LLM 特异性列为贡献。

## 6. 三层评测分母

均衡域/复杂度抽样估计的是 **balanced benchmark estimand**，不是现实软件世界的自然发生率。另在冻结语料全集上按原始项目/函数权重给出 `corpus-weighted descriptive estimate`；后者只作描述性结果，不宣称代表未知总体。

### 6.1 候选条件恢复

```text
UsefulSalvageRate_m =
  profitable verified recovered candidates from model m
  / all frozen REFUTED_ELIGIBLE candidates from model m
```

### 6.2 模型宏平均

先对每个模型计算恢复率，再对模型等权平均。不能让失败数量最多的模型主导结论。

### 6.3 端到端结果

```text
EndToEndUsefulRate_m =
  source functions ending with verified profitable IR
  / all preregistered source functions queried for model m
```

端到端流程中，初始 verified-profitable 候选直接成功，refuted 候选才交给 SalvageIR，其他状态安全回退但收益为零。

## 7. 配对比较

对每个 `candidate_id`，执行完全相同的一组方法：

```text
B0 discard/fallback
B1 same-model whole-function resampling
B2 same-model whole-function repair with Alive2 feedback
B3 text-hunk ddmin rollback
B4 random dependency-closed rollback
B5 dependency-closed rollback without counterexample guidance
B6 hierarchical ddmin on the same component graph
B7 counterexample-slice hitting-set MaxSAT/ILP diagnosis
M  SalvageIR
O  exhaustive oracle on small-component subset
```

B1/B2 若调用 LLM，SalvageIR 核心版不调用 LLM。主要提供两种公平视图：

1. 相同 wall-clock/总成本预算；
2. 相同验证调用预算，并单列 LLM Token；
3. `16/32/64/128/256` 次验证预算的完整性能剖面和曲线面积。

这样既不会因 SalvageIR 无模型费用而人为吃亏，也不会让整函数修复获得无限请求。

## 8. 训练/开发/测试隔离

- 先按上游项目划分，再抽取函数和生成候选；
- 近重复 IR 通过规范化哈希聚类后放在同一划分；
- 开发模型用于调对齐阈值和搜索顺序；
- 留一模型家族只在方法冻结后运行；
- 保留项目在方法冻结前不查看候选和结果；
- 调整方法后已查看的保留数据自动转为开发数据。

不训练模型并不意味着不存在数据泄漏；Prompt、阈值、组件闭包和停止规则同样可能过拟合。

## 9. 数据表

### `sources.csv`

```text
source_id,project,commit,module,function,license,split,
raw_sha256,normalized_sha256,llvm_version,eligible,exclusion_reason
```

### `candidates.csv`

```text
candidate_id,source_id,model_id,model_revision,prompt_version,
decode_config,generation_index,raw_sha256,ir_sha256,
parse_status,alive_status,alive_log_sha256,candidate_class,
scope_status,component_count,split,frozen
```

### `runs.csv`

```text
run_id,candidate_id,method,method_version,seed,
verify_budget,compile_budget,wall_budget,llm_request_budget,
visited_states,alive_calls,compile_calls,input_tokens,output_tokens,
final_status,final_ir_sha256,exact_cost_source,exact_cost_final,
salvage_success,stop_reason,total_ms
```

### `components.jsonl`

每行保存组件 ID、原子编辑、闭包原因、依赖、源/目标范围、对齐置信度、所命中的反例切片和每个状态中的选择。

## 10. 防止不公平的审计清单

- [ ] 所有方法是否共享完全相同的初始候选？
- [ ] 是否在看恢复结果之前冻结候选清单？
- [ ] 是否分模型报告而不是只给总数？
- [ ] 是否把语法失败、refuted 和 unknown 分开？
- [ ] 是否把返回源程序排除出成功？
- [ ] 是否匹配验证次数、墙钟和 LLM 请求预算？
- [ ] 是否存在某一方法额外获得反例、测试或目标后端信息？
- [ ] 是否按源函数/项目聚类而不是把多个模型输出当独立程序？
- [ ] 是否包含留一模型和留一项目？
- [ ] 是否同时报告条件恢复率和端到端率？
