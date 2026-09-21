# ARIS Pipeline Summary — ProbeTrans IR-v2

**Problem**：LLVM IR 层漏自动向量化修复
**Final Method Thesis**：用最小反事实干预证书条件化函数级 LLVM IR 翻译，并以语义义务和 vectorizer on/off 交互验收。
**Final Verdict**：CONDITIONAL_GO
**Date**：2026-09-22

## Final Deliverables

- Proposal：`FINAL_PROPOSAL.md`
- Research contract：`RESEARCH_CONTRACT.md`
- Detailed P0：`PREEXPERIMENT_PROTOCOL.md`
- Formal experiment plan：`EXPERIMENT_PLAN.md`
- Tracker：`EXPERIMENT_TRACKER.md`
- Review：`REVIEW_SUMMARY.md`
- Evidence：`EVIDENCE_MATRIX.md`

## Contribution Snapshot

- **Dominant contribution**：Intervention-Certified IR Translation。
- **Supporting trust mechanism**：SOC + VAG。
- **Base code**：IR-OptSet, NeurIPS 2025, MIT。
- **Main benchmark**：held-out TSVC eligible missed loops。
- **Explicitly rejected**：source-to-source、RL、RAG、多 Agent、多 Pass、intrinsics、O0/Oz 端点输入。

## Must-Prove Claims

1. Certificate 比 precise remarks 提供可测的信息增量。
2. LLM 比读取同一 certificate 的 template 多解决非机械 IR restructuring。
3. SOC/VAG 实际防止安全泄漏和错误归因。

## First Runs

1. R001–R006：环境、verifier、Alive2、CPU noise；
2. R010–R014：冻结 pre-LV pipeline 与 loop identity；
3. R020–R036：构造 calibration/TSVC registry 并决定 D0 Go/No-Go；
4. 只有 D0 Go 后运行 LLM。

## Main Risks

- certificate 不可安全实现；
- template 追平；
- pre-LV snapshot 与真实 O3 pipeline 不一致；
- AutoDL CPU 不适合性能结论；
- IR-OptSet 静态样本被误当 runtime benchmark。

## Next Action

在服务器基于 IR-OptSet fork 实现 M0–M3；不要先做模型微调或全量 benchmark。
