# ProbeTrans 实验计划

**状态**：预注册计划；没有实验结果。
**P0 操作协议**：[PREEXPERIMENT_PROTOCOL.md](PREEXPERIMENT_PROTOCOL.md)
**schema/算法**：[P0_IMPLEMENTATION_SPEC.md](P0_IMPLEMENTATION_SPEC.md)

## Claim map

| Claim | 最小证据 | 否定条件 |
|---|---|---|
| C1 certificate 有信息增量 | 同模型同生成预算下 strict CVUR 高于 Remark；且高于 query-matched Iterative Remark | paired difference 不为正 |
| C2 LLM 有必要 | ProbeTrans 高于共享 certificate 的 Template，并解决预注册的非机械函数内 restructuring | Template 追平 |
| C3 接受结果可信 | SOC、strict refinement、target lineage 和 VAG 均通过 | 依赖语义强化、lineage 歧义或 LV-off 仍有同等收益 |
| Anti-claim | 成本、查询、缓存、模型、数据泄漏与静态代理分别报告/控制 | 收益可由额外预算或筛样偏差解释 |

## Benchmark 角色

| 数据 | 角色 | 纳入/切分 | 指标 |
|---|---|---|---|
| autovec re-certified pairs | D0 诊断 | 所有有效 pair；按 original kernel 分组；OUT_OF_REGISTRY 保留 | coverage、conditional/overall accuracy、replay、queries |
| TSVC eligible missed loops | 主端到端 | dev family 与 held-out family 隔离 | strict CVUR、补充 CVUR、APIR、漏斗 |
| PolyBench/C + 3–5 apps | P0 后外部 | prompt/registry/阈值冻结 | 同上 + distribution shift |
| IR-OptSet fixed-hash subset | 静态压力 | 不要求 runtime harness | parse/verify/search/codegen coverage |
| frozen IR on RV64GCV | 跨 ISA 外部 | 不重新 prompt/调参 | compile、correctness、effect、真机 performance（若有） |

## 实验块

### B0 基础设施有效性（M0–M2b）

验证 verifier/Alive2/detector fixtures、canonical capture、lineage、effect detector、bounded search 与 clean replay。M2b 任何硬条件失败即停止，不接 LLM。

### B1 D0 certificate diagnosis（M3）

系统：raw remark parser、precise-remark classifier、冻结 certificate search。Decision-flip 不需要 blocker 标签；classification 只用独立 ground truth。主要报告全样本 coverage/overall accuracy，conditional accuracy 仅解释 registry 内表现。

### B2 Translator necessity（M4–M5）

系统：LLVM、Direct、Remark-only、Iterative Remark/Analysis、Template、ProbeTrans。主结果用 strict semantics；differential-only 仅为补充敏感性分析。

### B3 Trust isolation（M4–M6）

比较 SOC 前后接受集合、strict 与 differential-only 口径、naive performance 与 target-LV-only VAG。Seeded controls 只证明 gate sensitivity，不混入自然样本成功率。

### B4 P0 后泛化

冻结 ProbeTrans 后在外部 benchmark、第二模型、LLVM 次版本与 RV64GCV 上评估边界。若外部 eligible loops 太少，只作案例分析，不作普遍结论。

## Baseline 公平性

所有 LLM 系统匹配 model、context/output token cap、K、repair 和 seed。分两条轨：

- Natural-cost：Direct/Remark 为 0 probe queries；Template/ProbeTrans 共享 certificate。
- Query-matched：Iterative Remark/Analysis 使用与该 case ProbeTrans 相同的 charged query 上限，不做语义干预。

分别报告 model calls、generated tokens、compiler queries、cache hits、verifier/Alive2 calls、wall time。Certificate 成本同时用 non-amortized 和 amortized 口径。研究主张是额外 certificate 信息的效用，不是等总成本优越性。

## Correctness 与 effect

- Alive2：原始 source → replacement target refinement；proved/disproved/timeout/unsupported/internal error 分开。
- Differential：覆盖 pointer overlap、misalignment、trip 边界、overflow、poison/undef、null/object-size/dereferenceability、guard fast/fallback 和浮点策略。
- Strict 主表：只接受 `FORMALLY_PROVED`。
- Supplementary：`DIFFERENTIAL_ONLY_SUPPORTED` 单列；排除它后结论必须重算。
- G4：只接受目标 fast-path lineage 的 vector body；歧义或其他 loop 成功不计。

## Performance 与 VAG

四格 `O_on/O_off/E_on/E_off` 只改变目标 LoopVectorize。SLP、其他 pass、PGO、features、codegen 不变；pipeline manifest 检查 drift。全局关闭 SLP 是敏感性分析。

保存全部 raw timing；按预注册 warm-up、随机交错、median/MAD、bootstrap CI 报告。共享 CPU 噪声未过 gate 时，只能判 `PERFORMANCE_ENV_BLOCKED` 或 mechanism-only，不能声称 APIR。

## 统计与失败

- 二元 case-level outcome：成功数、paired difference、exact McNemar 或 paired bootstrap CI；小 P0 不以单一 p-value 决策。
- 多候选每 case 只计一次成功；candidate-level failure 另表。
- 超时、崩溃、无证书、module-required、unresolved 和 zero-success 全保留在分母。
- 任何数字必须标为 plan/threshold/result；当前文件只有计划与 gate。

## 里程碑

| Milestone | 内容 | LLM | 决策 |
|---|---|---:|---|
| M0 | 环境与 gate fixtures | 禁止 | infrastructure PASS/FAIL |
| M1 | capture、lineage、effect | 禁止 | stable/ambiguous/drift |
| M2a | registry 与 bounded search | 禁止 | worst path ≤16 |
| M2b | 6–12 loop vertical slice | 禁止 | 硬门 PASS/FAIL |
| M3 | full calibration 与 D0 | 禁止 | D0 GO/NO_GO/LABEL_INSUFFICIENT |
| M4 | splicer、SOC、Template | 禁止 | implementation gate |
| M5 | Direct/Remark/Iterative/ProbeTrans | 允许 | translator gate |
| M6 | performance/VAG | 无新增生成 | P0 verdict |

Run ID 与 artifact 以 [实验追踪表](EXPERIMENT_TRACKER.md) 为准，不在本文复制。

## 论文级 readiness checklist

- [ ] D0 全样本口径显示 certificate 对最强 remark baseline 有增量。
- [ ] Query-matched baseline 未解释全部收益。
- [ ] Strict CVUR 中 ProbeTrans 高于 Direct/Remark/Template。
- [ ] SOC 和 lineage 没有安全/归因漏计。
- [ ] 稳定 CPU 上 VAG 支持目标 LoopVectorize 归因。
- [ ] 外部集趋势不反转。
- [ ] 冻结 IR 至少在 RV64GCV 上有可复现 compile/correctness/effect 报告。

未满足 checklist 前，不使用“方法有效”“跨 ISA 加速”或论文完成表述。
