# ProbeTrans 实现前终审摘要

**裁决**：`READY_FOR_M0`
**含义**：文档足以开始 M0；不代表实现或实验完成。
**审查方式**：当前模型内的对抗式协议审查；没有独立第二模型复核。

## Readiness checklist

- [x] LLVM IR→LLVM IR Translator 的输入、输出和禁止范围明确。
- [x] Probe family 已展开为有限 instance schema，P0 收缩为三族。
- [x] 11+3+2 搜索算法最坏路径不超过 16；minimality 声明有硬条件。
- [x] D0 不按 registry coverage 筛样；标签不由搜索结果生成。
- [x] Generation-matched、query-matched 与成本报告口径分开。
- [x] Module-level 与跨函数证书退出当前安全成功分子。
- [x] Loop lineage 支持 fast/fallback/vector/epilogue 一对多，并有歧义失败态。
- [x] Alive2 refinement 方向、未决状态、差分边界和 strict 主口径明确。
- [x] VAG 主协议只开关目标 LoopVectorize。
- [x] M2b 前禁止 LLM，run ID 与 tracker 已统一。
- [ ] 实现与 fixtures 尚未开始。
- [ ] 任何 coverage、正确性、性能和泛化结论均待实验。

## 最强拒稿攻击与对应防线

1. **证书只是更多 queries**：增加同 charged-query 上限、但不做语义干预的 Iterative Remark/Analysis。
2. **D0 只挑方法能覆盖的样本**：OUT_OF_REGISTRY 保留在总分母，分别报告 coverage/conditional/overall。
3. **所谓最小证书预算不闭合**：固定 baseline-inclusive 11 discovery +3 deletion +2 replay；删除检查不全即 non-minimal。
4. **成功其实来自非法 attribute 或 module edit**：SOC、realization class 和 replacement-function 边界共同拒绝。
5. **追踪错 loop 或加速来自标量改写**：一对多 lineage + target-fast-path G4 + target-LV-only VAG。
6. **差分测试被包装成证明**：strict 主表只接受 Alive2 proved，未决/差分候选单列。

## 尚存拒稿风险

- Registry 可能覆盖率太低，尤其 alias/trip facts 难以安全实现。
- Alive2 对循环、memory 和 versioning CFG 的 unresolved 比例可能过高。
- Template 可能足以完成所有可安全实现证书。
- Canonical prefix 的结果可能不能迁移到真实 O3-aligned capture。
- AutoDL CPU 可能无法支持可靠性能归因。

## 审稿结论

可开始实现 M0，但不得跳过 M2b，也不得把 `conditional_go`、文档完成度或 seeded fixture 通过写成方法有效。下一次审查应基于 M0–M2b 的真实 artifacts，而不是再增加纸面模块。
