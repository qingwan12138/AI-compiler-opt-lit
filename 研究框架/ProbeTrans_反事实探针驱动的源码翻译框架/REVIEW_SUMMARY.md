# ARIS Review Summary — ProbeTrans IR-v2

**Problem**：反事实探针驱动的 LLVM IR→LLVM IR 漏向量化修复
**Rounds**：5/5
**Final Score**：8.8/10
**Verdict**：CONDITIONAL_GO

> 本轮按 ARIS rubric 在当前模型内完成对抗复核；受会话约束，没有调用独立第二模型，不能声称跨模型审稿一致。

## Problem Anchor

在不修改 LLVM 后端、不使用 RL、不依赖 ISA intrinsic 的前提下，利用 LLM 对 pre-LoopVectorize LLVM IR 做语义保持翻译，修复一部分 missed vectorization，并证明收益来自目标 LoopVectorize。

## Round-by-Round Resolution

| Round | 严厉问题 | 修订 | 结果 |
|---|---|---|---|
| 0 | 错写成 source-to-source，偏离用户意图 | 冻结为 LLVM IR→LLVM IR；删除源码 patch/restrict helper 叙事 | 已纠正 |
| 1 | 没有明确在哪篇论文代码上修改 | 选择 IR-OptSet NeurIPS 2025 MIT 工具链；列出复用/新增边界 | 已解决 |
| 2 | `-O0` 太低、`Oz/O3` 太晚，输入 IR 无科学边界 | 定义 frozen canonical pre-LV IR；P0 canonical 与正式 O3-aligned 两阶段 | 部分待实现 |
| 3 | 探针可能只是非法 attributes；最小性夸大 | 探针只供诊断；SOC；限定 registry/budget 内 inclusion-minimal | 已解决 |
| 4 | Benchmark 混用、TSVC 泄漏、IR-OptSet 无 runtime harness | calibration/main/external/static/RVV 五种角色分离；group holdout | 已解决 |
| 5 | LLM 可能完全是装饰；共享 CPU 性能不可靠 | certificate→template kill baseline；mechanism-only gate；CPU noise protocol | 已解决为可证伪问题 |

## Reviewer’s Strongest Rejection Case

> ProbeTrans 可能只是 IR-OptSet 工具链上“更详细的 compiler feedback + LLM IR generation”。所谓 certificate 由人工设计探针决定，template 可能足以实现；如果收益来自新加 attributes 或标量简化，则向量化与 LLM 两个叙事都不成立。

该攻击被转换为四项必做证据：

1. held-out calibration 上 certificate 对 precise remarks 的诊断增量；
2. same-certificate template baseline；
3. semantic-strengthening audit 与运行时 guard/fallback；
4. original/edit × vectorizer on/off attribution。

任一核心项失败均触发降级或停止，而不是追加模型和搜索预算。

## Scores

| Dimension | Score | Reason |
|---|---:|---|
| Problem Fidelity | 9.6 | 已纠正到明确的 LLVM IR Translator |
| Method Specificity | 9.1 | 表示点、输入输出、探针、证书、门控均可实现 |
| Contribution Quality | 8.2 | 最近邻密集；需要实验证明 certificate 不是 renamed remark |
| Frontier Leverage | 8.6 | LLM 角色窄且必要性可测；没有强行加 RL/RAG |
| Feasibility | 8.7 | IR-OptSet 显著降低工程成本；Alive2/CPU 是长尾风险 |
| Validation Focus | 9.3 | benchmark 角色分离、强模板基线和 kill gates 清楚 |
| Venue Readiness | 8.0 | 方法可能成论文，但尚无任何正结果 |
| **Overall** | **8.8** | 纸面已足够进入 P0；9 分只能由数据补齐 |

## Final Status

- Anchor：preserved after correction。
- Focus：tight；单一 LoopVectorize、单一 IR abstraction。
- Modernity：appropriately frontier-aware；零训练优先。
- Remaining weaknesses：pre-LV capture 的工程真实性、证书可安全实现率、LLM 超过模板的幅度、共享 CPU 性能稳定性。
