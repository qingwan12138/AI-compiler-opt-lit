# ProbeTrans Pipeline Summary

**研究对象**：target-aware but ISA-intrinsic-free pre-LoopVectorize LLVM IR → LLVM IR
**主机制**：Intervention-Certified IR Translation
**当前裁决**：`READY_FOR_M0`；`conditional_go` 仅授权 P0 M0
**实验状态**：未运行

## 交付物

- [研究契约](RESEARCH_CONTRACT.md)
- [最终提案](FINAL_PROPOSAL.md)
- [P0 实现规范](P0_IMPLEMENTATION_SPEC.md)
- [P0 预实验协议](PREEXPERIMENT_PROTOCOL.md)
- [实验计划](EXPERIMENT_PLAN.md)
- [实验追踪表](EXPERIMENT_TRACKER.md)
- [终审摘要](REVIEW_SUMMARY.md)
- [终审报告](REFINEMENT_REPORT.md)
- [变更记录](REFINEMENT_CHANGELOG.md)
- [证据矩阵](EVIDENCE_MATRIX.md)

## 唯一执行顺序

M0 环境/fixtures → M1 capture/lineage/effect → M2a registry/search → M2b vertical slice → M3 D0 → M4 splicer/SOC/Template → M5 LLM → M6 performance/VAG。

M2b 前不部署模型、不调用 LLM、不运行 E0。Run ID 只以 tracker 为准；schema、错误码和预算只以 P0 spec 为准。

## 必须证明而非假定

1. 全样本口径下 certificate 相对 remark 有信息增量。
2. 增量不能被相同 compiler-query 数解释。
3. LLM 在 strict semantic 口径下超过共享 certificate 的 Template。
4. 目标 fast-path lineage 确实被 LoopVectorize 转换。
5. 性能收益在只关闭目标 LoopVectorize 后显著减弱。

## 下一工程任务

创建 M0 的环境锁与 gate fixtures，并生成 `artifacts/p0/M0_DECISION.json`。本轮没有产生任何实验结果。
