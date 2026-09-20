# SalvageIR Research Framework Design

日期：2026-09-20

## 设计结论

建立一条单一 `TRANSLATOR` 研究主线：对可解析但被 Alive2 明确反驳的 LLM LLVM-IR 优化候选求解 Refinement-Constrained Edit Salvage（RCES）。核心系统将源—目标差异构造成结构可重放组件，以 state-conditioned 反例软排序，在多个安全盆地的非支配前沿上回滚/重放，并对最终完整函数执行直接验证。反例贡献必须优于 structured ddmin、无反例 best-first、MaxSAT/ILP 和打乱反例安慰剂，不能把事务回滚本身包装成创新。

## 已确认需求

- 不使用强化学习；
- 不以 RISC-V 为主背景；
- RISC-V 只在方法冻结后验证指标是否仍有改善；
- 尽量只属于 Translator；
- 显式处理不同 LLM 水平造成的公平性问题；
- 完成研究问题、机制、数据、实验、统计、风险和论文定位，而不是只提供概念草图。

## 设计选择比较

### 方案 A：整函数反馈修复

实现简单，但 Compiler Generated Feedback、LLM-Vectorizer、LPO 等已有工作已经覆盖“错误反馈后重新生成”，且难以区分收益来自模型还是机制。

### 方案 B：正确候选的进一步优化

仓库中已有 `Translator_后优化框架` 讨论该问题。它需要证明正确候选仍有足够二次优化空间，与当前用户关注的失败候选来源和公平性问题不完全一致。

### 方案 C：失败候选的结构化部分回收（采用）

研究对象最清楚，能够用相同 `(source,candidate)` 做配对实验；核心可以完全确定性执行，不依赖第二个 LLM；但最大风险是自然候选未必存在可回收子变换。因此设计包含强制 P0 oracle 门。

## 架构

```text
Frozen Candidate
  -> Normalizer/Aligner
  -> Structurally Replayable Component Graph
  -> Stable Counterexample Adapter
  -> Budgeted Safe-Frontier Search
  -> Multi-Basin Profit Recovery
  -> Final Whole-Function Validation
  -> Exact Backend Cost
```

各模块通过内容哈希和显式状态对象交互；任何 unknown 或工具失败不进入 verified 集合。

## 测试与评价

- 单元测试：规范化、对齐、闭包、composer、flag/poison/PHI。
- 机制测试：人工可分/不可分案例、带 provenance 变异、小规模穷举 oracle、真实/静态/打乱反例和强结构搜索。
- 公平性：冻结候选、统一预算、模型分层、模型宏平均、留一模型。
- 正确性：最终 `Alive2(source,final)` 直接验证。
- 性能：主指标冻结为经 `clang -Oz` 校准的 x86-64 目标函数 ELF `ST_Size`；至少减少 `max(2 bytes, 1%)` 且不得增加 allocatable non-BSS 字节才算有益。D-H 再检查链接后二进制 code+rodata。RISC-V 仅使用从同一源代码独立生成的 RV64GC IR。

## 完整规格位置

主规格位于：

- [`研究框架/SalvageIR_验证失败候选部分回收框架/README.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/README.md)
- [`FINAL_PROPOSAL.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/FINAL_PROPOSAL.md)
- [`ALGORITHM_SPEC.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/ALGORITHM_SPEC.md)
- [`DATA_AND_FAIRNESS_PROTOCOL.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/DATA_AND_FAIRNESS_PROTOCOL.md)
- [`PILOT_PROTOCOL.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/PILOT_PROTOCOL.md)
- [`GATE_CARD.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/GATE_CARD.md)
- [`EXPERIMENT_PLAN.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/EXPERIMENT_PLAN.md)
- [`EVIDENCE_MATRIX.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/EVIDENCE_MATRIX.md)
- [`THREATS_AND_REVIEW.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/THREATS_AND_REVIEW.md)
- [`REVIEW_ROUND_1.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/REVIEW_ROUND_1.md)
- [`REVISION_ROUND_1.md`](../../../研究框架/SalvageIR_验证失败候选部分回收框架/REVISION_ROUND_1.md)

## 自审

- 无 `TODO/TBD` 作为设计要求；摘要中的实验结果占位明确标为未来真实结果。
- 主任务不混入 syntax invalid、unknown 或 verified candidates。
- 核心方法不依赖 RL 或额外 LLM。
- RISC-V 未进入训练、方法开发、Prompt、组件规则或主目标；仅在冻结后执行编辑决策投影和冻结算法重跑，且不直接改写 x86 IR triple。
- 事实、推导与候选创新已分开。
- 已设置能真正否定研究主张的停止条件。
- 预实验已冻结测试集选择算法、顺序扩样、`n_oracle=12`、无权重字典序搜索、预算剖面、F/I/S 门和 RISC-V 外部验证门。
