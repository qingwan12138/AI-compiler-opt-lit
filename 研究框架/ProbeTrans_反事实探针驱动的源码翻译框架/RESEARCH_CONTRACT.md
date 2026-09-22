# ProbeTrans Research Contract — LLVM IR Translator

**日期**：2026-09-22
**类别**：TRANSLATOR
**输入/输出**：canonical pre-LoopVectorize LLVM IR function → semantically refining LLVM IR replacement function
**当前状态**：设计已冻结到 M0 可实施；实现未开始；实验未运行
**裁决**：`conditional_go`，仅允许进入 P0 M0

## 1. 不可变问题锚点

ProbeTrans 研究：主动、可撤销的编译器干预能否产生比 standard remarks 更有用的翻译条件，使 LLM 在函数级 LLVM IR 内安全修复部分 LoopVectorize 漏优化。

不研究通用 `-O3` 替代、多 Pass 协调、Pass ordering、C/C++ patch、显式 SIMD、RL 或以 RISC-V 为主要背景。P0 优先零训练。

## 2. 表示与目标边界

输入不是 `-O0` 原始 IR，也不是完整 `Oz/O3` 的末端 IR，而是运行冻结 canonical prefix 后、目标 LoopVectorize 前的 snapshot。P0 必须明确它是实验 pipeline，而非冒充 `default<O3>` 的精确中间态。

表示是 **target-aware but ISA-intrinsic-free LLVM IR**：保留 triple、data layout、CPU features；不手写 x86/RVV intrinsic。x86 用于证书生成与主评测，RV64GCV 只评估冻结 IR 的 portability/robustness。

模型只能输出一个同名同签名 replacement function。任何需要修改 signature、callee/declaration、global/module metadata、vector-function mapping、新 helper 或跨函数证明的方案标为 `MODULE_CHANGE_REQUIRED`，不进入当前 Translator 的安全成功分子。

## 3. 唯一主机制

Intervention-Certified IR Translation（ICIT）在冻结、有限的 probe-instance registry 中，使用最多 16 次 compiler query 搜索能稳定翻转同一目标 loop 决策的证书。证书必须完成全部删除检查和两次无缓存 clean replay，才可标为 registry/budget 内 `inclusion-minimal`。

P0 只启用 profitability、alias/dependence、trip-count。Profitability probe 是 `DECISION_ONLY`，只说明成本/启发式决策可被改变，不代表缺少语义事实。Alignment 延后或严格限域；control/call 不进入 P0。

完整 probe schema、11+3+2 查询算法、certificate schema、终态和错误码以 [P0 实现规范](P0_IMPLEMENTATION_SPEC.md) 为唯一来源。

## 4. SOC 与 runtime versioning

Semantic-Obligation Contract（SOC）禁止无依据新增 `noalias`、`nonnull`、`dereferenceable`、更强 alignment、`nsw/nuw/exact/inbounds`、fast-math、memory effects 或 `llvm.assume`。

允许的条件只有两类：原 IR 可证明；或通过函数内 runtime guard + fast path + 原语义 fallback 实现。P0 排除 EH/invoke、convergent、deopt、不可建模 side effects、irreducible CFG 和跨函数事实。详细 guard 边界见实现规范。

## 5. Loop lineage 与 effect

原始 loop 到 fast path、fallback、post-LV vector body 和 epilogue 是一对多关系。metadata、源码行、block name 和 header hash只作提示；主匹配依靠 induction/SCEV、memory-access multiset、exit/live-out 和 CFG neighborhood。

歧义返回 `AMBIGUOUS_LOOP_LINEAGE`，不得计入 CVUR/APIR。目标 effect 必须发生在目标 fast-path lineage，不能用其他 loop、fallback 或 remainder 的向量化冒充。

## 6. 语义验证合同

Alive2 方向固定为原函数 `source` → replacement `target` 的 refinement。状态单独报告：`FORMALLY_PROVED`、`DISPROVED`、`TIMEOUT`、`UNSUPPORTED`、`INTERNAL_ERROR`、`DIFFERENTIAL_ONLY_SUPPORTED`。

Strict CVUR/APIR 主表只接受 `FORMALLY_PROVED`；差分通过但形式验证未解决的候选只进入补充 coverage 表，并做排除后的敏感性分析。Sanitizer 和固定数量随机测试不是等价证明。

## 7. D0 合同

所有满足 hidden missed、exposed vectorized、oracle equivalent、可复现的 pair 都保留；registry 不覆盖时标 `OUT_OF_REGISTRY`，不能删除。Decision-flip task 与 blocker-classification task 分开；blocker ground truth 只能来自原始 transformation、冻结规则或不知道 probe 结果的双人标注。

分别报告 coverage、conditional accuracy 和 overall accuracy。Probe 搜索结果不得充当自身标签。

## 8. 公平性与成本

- generation-budget matched：同模型、token、候选数、repair 和 seed 策略。
- natural-cost：Direct/Remark 不跑 probe；ProbeTrans/Template 共享 certificate。
- query-matched feedback：Iterative Remark/Analysis 使用相同 charged query 上限，但不做语义干预。
- 不主张等总成本优越，只检验 certificate 信息增量。
- 每系统报告 model calls、tokens、compiler queries、cache hits、verifier/Alive2 calls、wall time，以及 certificate 的 amortized/non-amortized 成本。

## 9. VAG 合同

主 VAG 四格 `O_on/O_off/E_on/E_off` 只改变目标 LoopVectorize 是否运行；SLP 和其余 pipeline、features、PGO、codegen 保持一致。全局同时关闭 SLP 只能是敏感性分析。VAG 是机制归因证据，不是完整因果证明。

## 10. Benchmark 合同

- D0：现代 LLVM 重认证的 autovec hidden/exposed pairs，按原始 kernel 分组切分。
- 主测试：TSVC 自动筛选的 held-out eligible missed loops。
- 外部：冻结的 PolyBench/C 与少量公开应用热点。
- 静态压力：IR-OptSet 子集，只报告工具稳定性。
- RV64GCV：x86 阶段冻结 IR，不重新 prompt；QEMU 不作性能结论。

## 11. 实施顺序与停止权

顺序固定为 M0 环境/gate fixtures → M1 capture/lineage/effect → M2a registry/search → M2b 6–12 loop vertical slice → M3 calibration/D0 → M4 splicer/SOC/Template → M5 LLM systems → M6 performance/VAG。

M2b 通过前不得部署或调用 LLM，不得运行 24-case E0。任何基础设施失真、预算超限、replay 不稳定、标签循环、lineage 歧义或 template 追平等 kill condition 都必须导致停止/降级，而非增加模型或搜索预算。

## 12. 开源基座

基座为 IR-OptSet。只复用其公开仓库实际提供的 extraction/preprocessing、LLVM wrapper、verify、Alive2 adapter、静态分析、LLM 和日志设施；ProbeTrans 的 probe registry、bounded search、lineage、function splicer、SOC 和 VAG 均需新实现。服务器运行只依赖项目内相对路径和冻结 commit。
