# Research Proposal：ProbeTrans

**全称**：Intervention-Certified LLVM IR Translation for Missed Auto-Vectorization
**最终定位**：LLVM IR → LLVM IR Translator
**ARIS 裁决**：CONDITIONAL_GO，先通过 P0，再决定是否扩成论文工程。

## Problem Anchor

- **Bottom-line problem**：让 LLM 在 LLVM IR 层修复一部分 `LoopVectorize` 漏优化，同时保持语义、目标无关性和可归因性。
- **Must-solve bottleneck**：remark 不足以确定哪组 IR 条件足以改变编译器决策；自由生成又容易产生错误或偷偷强化语义。
- **Non-goals**：源码翻译、通用 IR 优化器、显式 SIMD、Pass ordering、RL、RISC-V 主场景。
- **Constraints**：IR-OptSet 开源基座、单 RTX 4090、P0 不训练、LLVM 19.1.x、公开 benchmark。
- **Success condition**：在 held-out TSVC 上，同模型同预算时，ProbeTrans 获得比 Direct/Remark-only/template 更高的正确向量化解锁率，并在稳定 CPU 上产生可由 LoopVectorize 归因的真实收益。

## Technical Gap

现有工作分别覆盖了若干相邻问题：LLM-Vectorizer 生成显式 SIMD 并验证；VecTrans 在源码层重构以触发 auto-vectorization；IR-OptSet 提供 IR 数据与验证工具链；IntOpt 以高层 intent 指导一般 IR 优化；精确 compiler remarks 改善 agent 优化成功率。因而以下主张已不新：LLM 生成 IR、给 LLM 编译器反馈、增加 Alive2、或让代码最终向量化。

剩余的窄问题是：

> 在目标 Pass 的输入 IR 上，能否通过主动、可撤销的编译干预得到一个操作性证书，说明“哪些条件的联合改变足以翻转该循环的向量化决策”，并让 LLM 只负责把该证书安全地实现为 IR，而不是同时猜诊断与修复？

## Method Thesis

在固定 LLVM 工具链和有限探针空间内，先搜索能翻转同一循环向量化决策的 inclusion-minimal intervention certificate，再以禁止无依据语义强化的 contract 约束函数级 IR 翻译，能够比 remark-only 或自由 IR 生成产生更多正确且可归因的向量化解锁。

## Contribution Focus

### Dominant Contribution：Intervention-Certified IR Translation

核心不是“探针”“LLM”“验证”三个平行模块，而是一个新接口：**将编译器行为实验得到的最小充分条件作为 IR Translator 的翻译条件**。该接口把诊断从模型生成中剥离，并产生可复查、可重放的中间证据。

### Supporting Mechanism：Semantic obligation + attribution

SOC 与 VAG 合并为一个可信验收支撑：前者阻止非法语义强化，后者阻止错误机制归因。它们不单独争夺论文贡献位。

### Explicit Non-contributions

- 不声称创建 IR 数据集或新的形式验证器；
- 不声称首次用 LLM 优化 LLVM IR/向量化；
- 不声称探针揭示唯一真实根因；
- 不声称跨 ISA 必然加速。

## Base System and Modification Boundary

### Reused from IR-OptSet

IR extraction/preprocessing、Clang/`opt` wrapper、`opt -verify`、Alive2 adapter、`llvm-mca`、model adapter、parallel runner、logging 和 dataset schema。

### New in ProbeTrans

1. **Pre-LV Capture**：固定 canonicalization prefix，保存 target-independent IR 与稳定 loop ID。
2. **Probe Registry**：以 IR attribute、metadata、assumption 或局部临时改写测试 profitability、alias/dependence、alignment/memory、trip count 和 control/call 障碍。
3. **Certificate Search**：预算 16 次查询，搜索可重放的 inclusion-minimal probe set。
4. **Function Translator**：模型读取 target function + certificate + contract，输出 replacement function。
5. **Semantic Audit**：比较 attributes、flags、metadata 与 assumptions，拦截无依据强化。
6. **Validation and Attribution**：verifier → Alive2/differential → exact-loop effect → performance → vectorizer on/off interaction。

## System Overview

```text
Benchmark source + harness
          │
          ▼
IR-OptSet extraction + frozen canonical prefix
          │
          ▼
pre-LoopVectorize IR + stable loop identity
          │
          ├── original LoopVectorize → confirmed missed
          │
          ▼
diagnostic-only IR probes ── budgeted search/minimization
          │
          ▼
intervention certificate
          │
          ▼
semantic-obligation prompt → LLM replacement function
          │
          ▼
splice → verifier → semantic validation → vectorization effect
          │
          ▼
performance + vectorizer on/off attribution
          │
     accept / original fallback
```

## Core Mechanism

### 1. Stable IR Boundary

P0 以 LLVM 19.1.x 为唯一主工具链。先从原始程序产生未优化 IR，再运行固定 canonicalization prefix，得到进入向量化研究阶段前的 IR。该 IR 同时包含 `target triple`、`data layout` 和必要 declarations，但不包含目标 ISA intrinsic。

若无法稳定截取 `default<O3>` 中 LoopVectorize 之前的精确状态，P0 使用显式声明的 canonical prefix；不得把它描述为完整 O3 的中间状态。正式实验再用 PassBuilder instrumentation 对齐真实 O3 pipeline。

### 2. Counterfactual Probe Registry

探针是对编译器可见事实的临时增强，不是候选补丁。例如：

- 添加诊断性 `noalias` 或 alias-scope metadata；
- 临时加强 alignment；
- 加入 trip-count/倍数 assumption；
- 强制 vectorization metadata，以区分 profitability 与 legality；
- 临时暴露调用的 memory effects 或控制不变量。

每个探针必须记录其作用对象、IR diff、预计检查项和安全实现要求。探针生成文件永不进入模型可复制的候选区。

### 3. Inclusion-Minimal Certificate

对有限探针集合 `P`，寻找集合 `S⊆P`，使冻结 pipeline 下：

- 原始 IR 的目标 loop 为 missed；
- 应用 `S` 后目标 loop 在两次 clean process 中均 vectorized；
- 对任一 `p∈S`，删除 `p` 后不再满足上述条件。

该定义只保证 registry 与预算内的包含最小性。

### 4. Contract-Guided Function Translation

模型获得完整目标函数，而不是整 module。输出必须为同名同签名 replacement function。SOC 明确列出：

- certificate 中每个条件；
- 哪些条件已由原 IR 证明；
- 哪些必须 runtime check + fast/fallback versioning；
- 禁止新增的 poison/UB/alias/alignment/FP 语义；
- required compiler effect。

相比 unified diff，完整函数输出允许模型重新组织 SSA 和 basic blocks，又把影响范围限制在函数内。

### 5. Verification and Attribution

候选依次通过：

1. `llvm-as`/`opt -verify`；
2. semantic-strengthening audit；
3. Alive2；若 `unknown/timeout`，进入 differential testing，但不得标为 formally proved；
4. exact-loop vectorization evidence：remark + vector IR structure；
5. x86 真实运行；
6. VAG 四格：原始/改写 × vectorizer on/off。

性能交互定义为：

`A = [log T(O,on)-log T(E,on)] - [log T(O,off)-log T(E,off)]`

`A>0` 且置信区间为正，表示改写的相对收益在开启 vectorizer 时更大。它不是完整因果证明，但能排除明显的“标量改写本身更快”。

## Model and Training Plan

P0 不训练：通过 OpenAI-compatible endpoint 接一个冻结 coder model。服务器默认选择能在 24GB 4090 稳定运行的 14B 级 instruct/coder 模型或等价量化模型，精确 model ID、revision、quantization 写入配置。`temperature=0`，每样本 `K=3`，最大输出 4096 tokens，一次 verifier-error repair。

只有 P0 证明 certificate 有信息增量、但模型经常不会实现安全版本化时，才考虑用自动生成的证书—LLVM pass realization pairs 做 SFT；这不是首版贡献。

## Benchmark Design

### Diagnostic Calibration

现代 LLVM 重认证 `autovec-benchmark` hidden/exposed pairs。按原始 kernel group split，测证书诊断准确性、查询成本和可实现性。

### Main End-to-End Benchmark

完整 TSVC 经自动规则筛出 eligible missed loops；主结果只用未参与 prompt/probe 调整的 held-out families。TSVC 提供与 LLM-Vectorizer、VecTrans 和 CoV 的可比锚点。

### Generalization

PolyBench/C 全量扫描得到 hot missed loops，加 Blackscholes、Kmeans、LBM 和公开 LLVM test-suite applications 中的热点。IR-OptSet 的 200–500 函数只做静态压力测试。

### RISC-V

冻结 x86 阶段接受的 IR，不再调用模型，转向 RV64GCV；QEMU 测 correctness，真机才测性能。

## Claims

### C1：Certificate Information Gain

在相同模型与生成预算下，certificate-conditioned IR translation 的 CVUR/APIR 高于 Direct 和 Remark-only。

### C2：LLM and Trustworthiness Are Necessary

确定性 template 不能覆盖 LLM 的非机械 IR restructuring；同时 SOC/VAG 会实际拒绝一部分看似成功但不安全或归因错误的候选。

## Claim-Driven Validation

1. **诊断实验**：autovec held-out pairs；比较 remarks、单探针贪心和 minimal certificate search；指标为 blocker set-F1、flip precision、queries。
2. **24-loop P0**：held-out TSVC；Direct、Remark-only、Template、ProbeTrans；指标为 verifier/semantic/effect funnel、CVUR、APIR（若 CPU 稳定）。
3. **删除实验**：w/o certificate、w/o SOC、w/o VAG；回答主机制和可信支撑是否改变结果。

## Failure Modes

- **捕获位置不真实**：P0 显式命名为 canonical pipeline；正式实验实现 PassBuilder capture。
- **证书只会加入非法假设**：标记 `UNREALIZABLE_CERTIFICATE`，不调用模型或计失败。
- **Alive2 不支持复杂 IR**：保留 proved/unknown/failed 三态，并用差分测试补充，不混报。
- **共享 AutoDL CPU 噪声高**：先测 noise floor；P0 可用 CVUR 作机制 gate，APIR 等稳定 CPU 后确认。
- **模板追平**：移除 LLM 叙事，转成确定性 compiler repair 工具或停止。
- **TSVC 模式记忆**：使用 kernel-group holdout 和 PolyBench/real-app frozen external set。

## Complexity Intentionally Rejected

- 不加入 RAG、多 Agent、memory bank、RL、MCTS；
- 不做多个 LLVM Pass；
- 不在线反馈性能给 LLM；
- 不修改 LLVM 后端；
- 不在 P0 微调；
- 不为 RVV 单独生成候选。

## Final Assessment

修订后的 ProbeTrans 不再是源码重构系统，也不是 IntOpt 的缩小版。它依托 IR-OptSet 的开源 LLVM IR 工具链，把研究变量压缩为一个问题：**反事实 compiler interventions 产生的最小证书，是否是比 remarks 更有效、更安全的 IR translation condition。** 该问题具备清晰实现边界和强失败条件，但尚无数据，因此只能有条件推进。
