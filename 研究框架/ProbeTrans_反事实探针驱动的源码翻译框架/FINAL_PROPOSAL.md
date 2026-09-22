# ProbeTrans：Intervention-Certified LLVM IR Translation

**定位**：pre-LoopVectorize LLVM IR → LLVM IR Translator
**当前裁决**：`READY_FOR_M0`；尚未实现，尚无实验结果

## 研究问题

在固定 LLVM 工具链、有限 probe registry 和严格函数级边界内，可重放的最小反事实干预证书是否比 standard compiler remarks 更能帮助 LLM 安全地修复 LoopVectorize 漏优化？

已有研究已覆盖 LLM 生成 IR、编译器 feedback、显式 SIMD 生成、源码层重构和多层验证。因此 ProbeTrans 不把“使用 LLM/Alive2/remarks/向量化”本身作为创新。唯一主线是：**把主动编译器行为实验得到的、预算内可重放 certificate 作为 IR Translator 的条件，并显式分离诊断假设与最终语义义务。**

## 方法

```text
benchmark + harness
→ frozen canonical pre-LV IR + original loop identity
→ finite probe instances + deterministic ≤16-query search
→ replayed inclusion-minimal certificate
→ realizability audit
→ certificate-conditioned replacement function
→ splice / verify / SOC / source→target refinement
→ target fast-path lineage effect
→ x86 timing + target-LoopVectorize-only VAG
```

### 1. Counterfactual probe

P0 只研究 profitability、alias/dependence、trip-count。每个 probe instance 都有精确 target、IR edit、参数、适用谓词、语义要求、realization class、hash、成本和组合约束。Profitability probe 仅为 `DECISION_ONLY`，不会被当成程序事实。

### 2. Certificate

查询预算固定为 discovery 11（含 baseline）+ deletion 3 + clean replay 2。最大集合大小为 3。只有完成所有删除检查并在两个新进程中重放成功，才能称为冻结 registry/预算内的 inclusion-minimal certificate。算法与 schema 见 [P0 实现规范](P0_IMPLEMENTATION_SPEC.md)。

### 3. Function translation

LLM 只能输出同名同签名 replacement function。函数内允许 CFG/loop restructuring 及合规的 runtime guard + fast/fallback；禁止修改 module、declaration、callee、global 或新增 helper。超界证书标 `MODULE_CHANGE_REQUIRED`，不计安全可实现成功。

### 4. Trust boundary

SOC 拦截无依据 alias/alignment/overflow/poison/FP/memory-effect 强化。Alive2 检查原函数 source 到候选 target 的 refinement；timeout、unsupported、internal error 均是 unresolved。Strict 主结果只接受 formally proved；differential-only 另表报告。

Loop lineage 显式表示 original、fast path、fallback、vector body 和 epilogue 的一对多关系；歧义直接失败。VAG 主协议只开关目标 LoopVectorize，其他 pass 与 SLP 不变；其结论是归因证据，不是完整因果证明。

## D0 与 benchmark

D0 保留全部有效 autovec hidden/exposed pairs，包括 `OUT_OF_REGISTRY`。Decision-flip 与 blocker classification 分开，ground truth 来源独立于搜索结果，并同时报告 coverage、conditional accuracy、overall accuracy。

TSVC 是 held-out 主测试；PolyBench/C 和公开应用只在 P0 Go 后做冻结外部集；IR-OptSet 只做静态压力；RV64GCV 只测试冻结 IR。IR 表示是 target-aware but ISA-intrinsic-free，不声称跨 target 决策相同或普遍加速。

## Baselines

- LLVM canonical baseline；
- Direct Translator；
- Remark-only Translator；
- Iterative Remark/Analysis query-matched feedback；
- Certificate→Template；
- ProbeTrans。

LLM 系统匹配 generation budget。Natural-cost 与 query-matched 两条轨分开；ProbeTrans 不主张等计算成本更优。Template/ProbeTrans 的共享 certificate 同时报告 amortized 与 non-amortized 成本。

## 可检验假设

- **H1 信息增量**：在同模型同生成预算下，ProbeTrans 的 strict CVUR 高于 Remark-only；若 paired difference 不为正，不支持 certificate 主张。
- **H2 非纯查询收益**：ProbeTrans 高于 query-matched Iterative Remark；否则收益可能只是更多 compiler interaction。
- **H3 LLM 必要性**：ProbeTrans 高于共享 certificate 的 Template，且成功包含非机械函数内 restructuring；否则移除 LLM 叙事。
- **H4 可信验收**：SOC/VAG 会拒绝真实或 seeded 的安全泄漏/错误归因；若不改变任何判定且 fixtures 也不敏感，则删除相应机制主张。

## P0 顺序

M0 环境与 fixtures → M1 capture/lineage/effect → M2a registry/search → M2b 6–12 loop vertical slice → M3 D0 → M4 splicer/SOC/Template → M5 LLM → M6 performance/VAG。

M2b 是硬门：在其通过前不部署模型、不调用 LLM、不运行 24-case E0。

## 可证伪与降级

- 16 次内无法稳定搜索/replay：停止 ICIT。
- OUT_OF_REGISTRY 占多数或证书多为 module-required/unrealizable：收窄 probe 问题或停止 Translator。
- Remark/query-matched baseline 追平：certificate 没有独立信息价值。
- Template 追平：降级为确定性 compiler repair，不保留 LLM 主张。
- Strict semantic 口径没有收益：不以 differential-only 包装成功。
- VAG 表明收益在目标 LV off 时仍存在：不能归因为目标向量化。

## 代码基座和事实边界

IR-OptSet 是唯一工程基座，但公开仓库只提供通用 LLVM IR 工具链；ProbeTrans 专用模块尚待实现。LLVM/Alive2/IR-OptSet 的官方事实核查链接和日期记录在实现规范。本提案不宣称任何实验成功率、性能或覆盖率。
