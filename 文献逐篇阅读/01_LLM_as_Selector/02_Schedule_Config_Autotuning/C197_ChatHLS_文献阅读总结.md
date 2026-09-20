# ChatHLS 文献阅读总结

论文题目：**ChatHLS: Towards Systematic Design Automation and Optimization for High-Level Synthesis**

作者：Runkai Li、Jia Xiong、Xiuyuan He、Jieru Zhao、Jiaqi Lv、Haowen Fang、Lei Qi、Xi Wang

发表时间：2026（ACL 2026；arXiv 首次公开于 2025）

发表平台：Proceedings of the 64th Annual Meeting of the Association for Computational Linguistics（ACL 2026 Long Papers），20996–21015

论文链接或编号：DOI `10.18653/v1/2026.acl-long.962`；arXiv `2507.00642`；官方页面 <https://aclanthology.org/2026.acl-long.962/>

关键词：高层综合（High-Level Synthesis, HLS）、大语言模型（LLM）、HLS directive/pragma、QoR、代码调试、RAG、DPO、VODA、FPGA

> 本文档只依据 A1 staging 中的 ACL 官方 PDF。论文事实与阅读后的研究思考分开描述；未由正文明确给出的信息标记为“论文中未明确说明”。

---

## 1. 研究背景

HLS 将 C/C++ 等高层程序转换为可综合硬件，降低了 FPGA/ASIC 设计的硬件描述语言门槛。但 HLS 设计的性能和资源使用高度依赖循环与数组上的硬件优化指令，例如 `PIPELINE`、`UNROLL` 和 `ARRAY_PARTITION`。论文引言指出，指令组合空间呈组合爆炸，单次综合可能耗时数分钟到数小时；指令之间还有非线性、设计相关的相互作用。

传统方法主要有三类：人工专家调参、启发式/搜索式 DSE，以及预测模型或 DSL。人工方法依赖 HLS 专家；启发式方法需要大量迭代；学习式方法在训练分布外的 kernel 上泛化有限；DSL 方法降低了部分编码风险，但需要额外掌握 DSL，并限制表达能力（第 1、2 节）。

论文认为，通用 LLM 的问题不只是缺少 HLS 文档，还包括不能稳定理解可综合性约束、directive 语义和 directive 对 QoR 的因果影响。因此 ChatHLS 将领域知识、专门微调、HLS 工具反馈和层次化代理组合起来。

## 2. 论文要解决的问题

### 2.1 HLS 数据稀缺

现有 HLS 数据集通常缺少可综合性约束、directive 选择理由及 directive 与 QoR（质量结果，主要是 latency 和资源利用率）的对应关系，导致 LLM 生成的 HLS-C 经常出现兼容性错误。

### 2.2 指令优化效率不足

`PIPELINE`、`UNROLL`、`ARRAY_PARTITION` 等指令的类型、位置和因子共同决定硬件并行度与资源开销。论文要解决在资源约束下快速选择、组合、配置并插入 directive 的问题，而不是对算法级 C/C++ 做任意重写。

### 2.3 HLS-C 调试能力有限

标准 C/C++ 中合法的动态数组、指针或控制结构可能无法综合；directive 误放置、冲突或因子不匹配也会导致仿真/综合失败。论文要将 HLS 工具的错误日志转化为可执行的诊断与修复动作。

> 本文主要研究：如何利用专门训练的 LLM、多代理反馈和 HLS 工具验证，完成 HLS-C 生成、directive 级性能调优以及 HLS 特定错误修复。

## 3. 核心方法概述

ChatHLS 包含 HLS-C 生成和 HLS-C 调试两个阶段。生成阶段由 LLM 1 借助 Vitis HLS 文档 RAG 完成初始 HLS-C 生成，由 HLSTuner（微调后的 LLM 2）选择 directive 组合、配置和插入位置。调试阶段由 HLSFixer 解析 C-Simulation、Synthesis 和 C/RTL Co-simulation 的错误，通过分析代理、修复代理和必要的 LLM-as-a-judge 产生并筛选修复指令。VODA 持续从工具失败中扩展 BugRAG 和验证数据。

```text
自然语言或 C/C++ 算法
        ↓
LLM 1 + Vitis HLS 文档 RAG
        ↓
初始 HLS-C
        ↓
HLSTuner：分析循环/数组、初始 QoR、目标约束
        ↓
选择 directive 类型、因子、作用位置和组合策略
        ↓
插入代理生成 directive-level HLS-C
        ↓
Vitis HLS：CSIM / CSYN / COSIM + latency/resource QoR
        ↓
HLSTuner 根据反馈调整并重试；失败则 HLSFixer 诊断/修复
```

本次语料库角色判定：ChatHLS 的最终优化输出不是新的编译器 pass，也不是直接替换后的通用源程序；HLSTuner 输出的是由 HLS 工具执行的 directive 选择、组合、配置和插入计划。因此主角色为 `SELECTOR`，二级类为 `S2_Schedule_Config_Autotuning`。HLSFixer 的代码修复是辅助支线，不改变主优化角色。

## 4. 实验框架与训练流程

### 4.1 HLS-C 生成阶段

官方 Vitis HLS User Guide（UG1399）被切分为长度 1000、相邻块重叠 200 的文本块，嵌入知识库。LLM 1 检索与当前代码片段相关的定义、约束和使用模式，生成初始 HLS-C。论文称该阶段主要用于提供 HLS 领域知识。

### 4.2 HLSTuner 训练与运行

HLSTuner 输入 HLS-C、循环/数组元数据（如数组维度和循环 trip count）、初始 QoR（latency cycles、DSP/FF/LUT 利用率）。它输出 directive 类型及因子、目标代码片段、插入动作和优化计划；插入代理再将计划写入 HLS-C。论文限制支持的指令为循环 `PIPELINE`、循环 `UNROLL`、数组 `ARRAY_PARTITION`。

训练数据来自 20 个 Rosetta kernel，共 4,804 个优化样本。NSGA-II 生成多样化的 HLS 设计并收集 QoR；教师模型 DeepSeek-V3.2 根据已经验证的源代码、优化代码、优化前后 QoR 生成优化 CoT，解释 directive 变化与硬件性能/资源变化之间的关系。该 CoT 用于微调 Qwen-2.5-Coder-14B-Instruct。

运行时若初始方案未达性能目标，HLSTuner 根据最新 QoR 增减循环并行度：资源超预算时降低并行度并调整相关 directive；资源有余量时优先增加深层循环并行度。论文强调这是 qualitative trend reasoning，而不是精确预测每个硬件指标。

### 4.3 HLSFixer 训练与运行

HLSFixer 的分析代理读取结构化错误日志，定位错误行、解释根因并输出具体修改指令；修复代理执行修改并重新测试。对于单个分析代理无法解决的错误，多个 LLM 产生候选诊断，评分代理按清晰度、逻辑正确性、与错误日志的一致性和修改范围选择建议。

论文构造 10,878 个覆盖 33 类错误的 buggy code 样本，并构造 3,716 个 CoT/非 CoT preference pairs。HLSFixer 使用 SFT 学习错误诊断/修复，再使用 DPO（Direct Preference Optimization，直接偏好优化）学习偏好。DPO 阶段以 SFT 模型为起点，使用 rank=8 的 LoRA。

### 4.4 VODA 数据扩增

VODA（Verification-Oriented Data Augmentation）从 HLS 工具失败中提取错误类型、示例、错误信息和根因分析；若 BugRAG 中没有匹配项，则登记新的错误切片和 mnemonic。随后插入代理从 BugRAG 检索适用错误，生成 buggy code，形成可验证训练数据。BugRAG 包含 33 个模块化错误切片，使用 `all-MiniLM-L6-v2` 和 Chroma 向量库，top-k=2。

### 4.5 训练阶段配置

实验使用 8× NVIDIA H800-80G、Qwen-2.5-Coder-14B-Instruct、AdamW、bfloat16 和 DeepSpeed ZeRO-3。SFT 使用全参数微调，3 epochs、学习率 1e-5、余弦调度和 0.1 warmup ratio；DPO 使用 2 epochs、学习率 5e-6、LoRA rank=8、`beta=0.1`。论文未使用 PPO/GRPO 等在线强化学习；DPO 是偏好优化训练，不等同于环境交互式 RL。

## 5. 奖励函数、损失函数或关键公式

### 5.1 pass@k

论文使用如下 pass@k 估计在 k 个候选中至少有一个正确结果的概率：

```text
pass@k = E_i [ 1 - C(n-c_i*, k) / C(n, k) ]
```

其中 `n` 是每个测试实例生成的总候选数，`c_i*` 是其中通过验证的候选数，`k` 是考虑的候选数。对 debugging，论文通常设 n=5；对 HLS-C generation，分阶段评估时设 n=20。正确性要求通过 HLS toolchain 的 CSIM、CSYN 和 COSIM。

### 5.2 Speedup 与资源约束

```text
Speedup = Lat(H, λ(θbaseline)) / Lat(H, λ(θoptimized))
约束：Util_r(λ) ≤ 80%，r ∈ {DSP, FF, LUT}
```

`H` 是 Vitis HLS 工具，`λ` 是设计，`θ` 是插入的 directive 配置，`Lat` 是综合时序分析得到的 latency cycles，`Util` 是资源利用率。Speedup 越高越好，但优化设计必须满足 DSP、FF、LUT 的资源上限。

论文没有定义单独的环境奖励函数，也没有进行 PPO/GRPO。DPO 的偏好损失是训练实现的一部分；PDF 没有给出完整展开的 DPO 数学式，只明确给出 `beta=0.1` 和 preference pairs。

## 6. 实验设置

### 6.1 数据集来源

- HLSFixer 训练：35 个 Kernel（PolyBench/C）和 Vitis 来源基础设计，自动注入 10,878 个 buggy samples，覆盖 33 类错误；另有 3,716 个 DPO preference pairs。
- HLSTuner 训练：20 个 Rosetta kernels，4,804 个优化样本；样本通过 NSGA-II 生成设计并收集 QoR。
- HLS-C generation 测试：108 个自然语言到 HLS-C 任务，包括 HLS-Eval 的 85 个设计和 23 个自定义设计，覆盖 PolyBench、MachSuite 和 CHStone。
- HLSFixer 测试：591 个测试案例，来自 Kernel、Vitis 和 15 个手工设计；由 32 个正确 HLS 设计注入 34 类错误，保留 HLS 工具明确报告注入错误类型的样本。
- HLSTuner 性能测试：线性代数、密码算法和神经网络加速器 kernel；另测试 MobileNet 和 Transformer。

论文附录报告了训练/测试相似度分析，但没有在正文中给出完整的去重比例表。训练数据由 DeepSeek-V3.2 生成或辅助生成；论文未证明不存在预训练语料泄漏，且附录 G 明确讨论了 DeepSeek-V3.2 在部分 kernel 上可能存在记忆/数据污染风险。

### 6.2 模型与工具

- 基础模型：Qwen-2.5-Coder-14B-Instruct；对照包括 DeepSeek-V3.2、Gemini-3-pro、Claude-opus-4.5、Qwen2.5-Coder-7B 和 Qwen3 系列。
- 编译/验证工具：Vitis HLS 2022.1；CSIM、CSYN、C/RTL COSIM；后续实现验证包含 Place-and-Route。
- 硬件平台：Xilinx ZCU106 MPSoC，目标频率 100 MHz；训练服务器为 8× NVIDIA H800-80G，另有 2× Intel Xeon Platinum 8480+ CPU。
- 检索与训练：官方 Vitis HLS UG1399、BugRAG、Chroma、`all-MiniLM-L6-v2`、DeepSpeed ZeRO-3、AdamW、bfloat16。

### 6.3 对比方法

- HLS-C 生成/调试：DeepSeek-V3.2、Gemini-3-pro、Claude-opus-4.5、Qwen2.5-Coder-7B；另比较 C2HLSC、HLSRewriter。
- 检索对照：通用 LLM 加 Vitis 文档 RAG；调试还比较 RAG-1（Vitis 文档+BugRAG）与 RAG-2（完整训练集检索）。
- HLS 优化：Vitis HLS auto-pipeline optimization baseline、DeepSeek-V3.2、Gemini-3-pro、RALAD；另与 Dahlia、HeteroCL、Allo 和 HGBO-DSE 比较。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| CSIM/CSYN/COSIM pass@k | C 仿真、综合、C/RTL 协同仿真通过率 | 越大越好 |
| Latency | 综合时序分析得到的周期数 | 越小越好 |
| DSP/FF/LUT utilization | 目标 FPGA 资源利用率 | 需不超过 80%约束 |
| Speedup | 基线 latency 除以优化后 latency | 越大越好 |
| Critical path | Place-and-Route 后关键路径 ns | 越小越好，需满足频率 |
| Total on-chip power | 实现后的片上功耗 | 越小越好 |

## 7. 实验结果与结论

### 7.1 主要结果

HLSFixer 在综合测试集上达到 93.4% 的整体 debugging pass@1；论文称相对 Claude-opus-4.5 提高 36.8%，相对 Gemini-3-pro 提高 32.6%。在 HLS-C generation 的 Table 2 中，ChatHLS w/HLSFixer 的 CSIM/CSYN/COSIM pass@1 分别为 82.1%、81.2%、77.2%，pass@5 分别为 90.1%、90.0%、87.6%。这些数字是 108 个生成任务/对应验证流程下的通过率，不是源代码语法通过率。

HLSTuner 在 15 次以内的优化尝试中，相对 Vitis HLS baseline 的几何平均 speedup 为 18.1；相对 DeepSeek-V3.2、Gemini-3-pro 和 RALAD 分别为 4.0×、1.5×和 3.3×。论文同时要求资源利用率保持在 80%以下。

在 MobileNet 上，Table 3 给出的 latency 为 HLSTuner 5.88M、RALAD 13.33M、baseline 13.14M，speedup 分别为 2.233×、0.986×、1.000×；在 Transformer 上，latency 为 HLSTuner 68.51K、RALAD 88.34K、baseline 83.30K，speedup 为 1.216×、0.943×、1.000×。DSP/FF/LUT 利用率随工作负载不同而变化。

### 7.2 与传统方法的比较

在相同 HLS kernel 上，HLSTuner 的几何平均 speedup 相对 Dahlia、HeteroCL、Allo、HGBO-DSE 分别为 19.4×、4.0×、2.3×、1.6×。论文把优势归因于 directive 语义理解、QoR-aware 迭代和较少的无效组合尝试。HGBO-DSE 需要约 100 次搜索，HLSTuner 评估限制在少于五次优化迭代的运行轨迹展示中，但最终 Figure 9 的对比还使用最多 15 次尝试，两个口径不能混写。

### 7.3 与其他 LLM 方法的比较

仅增加 Vitis 文档 RAG 并不能弥合差距：对 DeepSeek-V3.2，RAG 使 CSIM 从 47.0% 降至 42.6%、CSYN 从 43.2% 降至 35.8%，COSIM 从 31.5% 小幅升至 32.1%；对 Gemini-3-pro 也只有有限变化。调试场景中 RAG-1 能改善通用模型，但 HLSFixer 仍达到 93.4%，高于 Gemini+RAG-1 的 84.5%。

### 7.4 消融实验

HLSFixer 的分析代理使修复 pass rate 相比单一修复代理增加 16.6%；加入 DPO 后再增加 3.7%；无法单次解决的错误通过多模型候选和评分代理得到额外 16.5% 改善。该结果来自 Figure 7 的 HLSFixer 消融，不应解释为所有任务的统一绝对增益。

### 7.5 案例与失败分析

论文展示了动态内存分配、DATAFLOW 冲突、PIPELINE-UNROLL 冲突、数组分割维度/因子错误等 HLS 特定错误。HLSTuner 的优化轨迹显示其会根据 QoR 增加或降低并行度。附录 G 指出 DeepSeek-V3.2 在 `kernel_2mm`、`kernel_symm` 等 kernel 上多次生成过度并行化或综合失败方案，作者认为这可能与记忆化/数据污染有关；该判断是作者的分析，不是独立泄漏证明。

## 8. 主要创新点

### 8.1 创新点一：QoR-aware directive 选择器

HLSTuner 不仅从代码到优化代码做模仿学习，还用已验证的 QoR 变化生成 CoT，使模型学习 directive 对 latency、并行度、存储带宽和资源的定性因果关系。实验中它可以根据反馈动态调整 directive 组合与因子，并在资源约束下获得较高 speedup。

### 8.2 创新点二：面向 HLS 的层次化调试

HLSFixer 将错误日志解析、根因分析、具体修改指令、修复执行和再验证拆开；复杂错误再引入多模型候选与评分代理。该设计把 HLS 工具的长日志压缩为可供 LLM 使用的结构化证据。

### 8.3 创新点三：VODA/BugRAG 的验证导向数据扩增

VODA 将工具失败转化为可检索错误切片和新的训练案例，覆盖 HLS-C 兼容性、仿真、编译、功能和 directive 五类错误。其价值在于把失败样本和 HLS 工具证据纳入数据构造，而不是只扩充成功代码。

### 8.4 创新点四：约束 directive-level 变换

论文明确不在 HLSTuner 优化阶段做算法级 C/C++ 重构，而只调整 directive。这样可把优化搜索限制在 pragma 选择、插入与因子配置，降低源代码语义偏移风险；这也是系统边界，而不是普遍解决所有 HLS 优化问题。

## 9. 局限性

### 9.1 论文明确承认的局限

1. 重点是循环并行化，尚不支持复杂 `DATAFLOW` producer-consumer 控制，也不支持插入 AXI interface directive 以完成 FPGA IP 部署。
2. 目前主要在 Xilinx ZCU106 MPSoC 上验证；不同 FPGA 资源约束下的可迁移性仍需更广泛研究。
3. 优化对象限于 `PIPELINE`、`UNROLL`、`ARRAY_PARTITION`，覆盖不了全部 HLS directive。

### 9.2 阅读后发现的潜在局限

- 训练样本和验证流程与 Vitis/Xilinx 生态绑定，其他 HLS 编译器、ASIC flow 或 MLIR-based HLS 的 directive 语义未验证。
- 论文将 HLSFixer 的代码修复和 HLSTuner 的 directive 选择放在一个框架中，主角色边界需要在跨论文比较时保持清晰。
- speedup 主要基于综合时序分析；只有 Table 4 的部分结果经过 Place-and-Route，不能把全部 latency 结果等同于部署后真实端到端运行时间。
- 训练集由 DeepSeek-V3.2 辅助生成，且作者自己指出 DeepSeek-V3.2 在部分 kernel 上可能存在记忆化风险；数据污染控制仍是重要复现问题。
- `PIPELINE` 与 `UNROLL` 的冲突、数组端口和资源利用率约束需要 HLS 工具反复验证，模型输出即使语法正确也不能被视为语义正确或硬件可实现。

## 10. 阅读后的研究方向反思

ChatHLS 最适合作为 HLS directive 选择与工具反馈闭环的 Selector baseline，而不是直接作为 LLVM IR 或 RISC-V 编译器方案。值得借鉴的是把“候选配置—真实编译器验证—QoR 反馈—再选择”作为可审计闭环，并把错误类型、日志和根因结构化。

不能简单照搬为“把 Xilinx 换成 RISC-V”。RISC-V 版本需要重新定义可选择动作、后端代价、合法性检查和真实硬件反馈；仅替换目标硬件不足以形成方法创新。更有价值的关系是：ChatHLS 的 directive Selector 可作为上层策略，LLVM/RISC-V 后端的 pass/flag selector 可作为另一种动作空间，并通过跨平台效果差异做迁移研究。

## 11. 可进一步尝试的研究方向

### 11.1 跨 HLS 编译器的 directive 语义对齐选择器

#### 研究问题

同一循环/数组结构在 Vitis HLS、MLIR-based HLS 和其他后端上的 directive 选择是否能够共享可迁移表示？

#### 与原论文的区别

不只在 ZCU106/Vitis HLS 内调优，而是显式建模不同工具的 directive 语义和资源约束。

#### 可能的创新点

引入 directive effect schema、工具能力掩码和跨编译器反馈校准；将不可支持的动作在候选生成前过滤。

#### 实验框架

```text
程序结构/硬件约束 → 共享 Selector → 工具特定 directive 映射
→ 各 HLS 编译器综合 → QoR/失败类型 → 迁移与校准
```

#### 可行性

需要至少两个可获得的 HLS/MLIR 流程、统一 kernel 集和可比较的 latency/resource 指标。

#### 主要风险

不同工具的 latency、资源报告和合法性定义可能不可直接比较。

### 11.2 面向 LLVM/RISC-V 的 QoR 与真实后端双反馈 Selector

#### 研究问题

如何让 LLM 同时利用静态 LLVM IR 统计和 RISC-V 真实硬件计数器，选择 pass/flag/configuration 而不被单一代理指标误导？

#### 与原论文的区别

ChatHLS 主要使用 HLS 综合 QoR；该方向将动作空间换成 LLVM pass/flag，并加入真实 RISC-V 后端反馈。

#### 可能的创新点

用错误/回归证据和真实性能指纹约束候选选择，区分编译时间、代码尺寸、周期数和能耗的冲突。

#### 实验框架

```text
LLVM IR → Selector 产生 pass/flag 配置 → RISC-V 编译/运行
→ 静态指标+硬件计数器 → 回归检测 → 更新选择策略
```

#### 可行性

需要 LLVM pass pipeline、RISC-V 交叉编译器、仿真器或开发板以及可重复 benchmark。

#### 主要风险

真实硬件噪声、编译时间和后端可用性会增加反馈成本；不能把 HLS 的 directive 经验直接当作 LLVM pass 经验。

### 11.3 错误类型驱动的保守配置搜索

#### 研究问题

能否将编译失败、资源超限、时序失败和功能错误作为独立的错误类型，限制 Selector 的后续动作空间？

#### 与原论文的区别

沿用 VODA 的失败样本思想，但将失败类型直接用于候选配置的风险门控，而不是主要用于训练修复代理。

#### 可能的创新点

形成“错误类型—禁用动作—可恢复动作”的版本化策略约束，并测量安全性与探索效率的权衡。

#### 实验框架

```text
配置候选 → 编译/测试 → 错误分类 → 风险门控
→ 保守候选池 → QoR 选择 → 下一轮配置
```

#### 可行性

可复用 ChatHLS 的错误切片设计，但需要独立验证分类准确性和门控误伤率。

#### 主要风险

过强门控可能排除真正高收益配置；错误日志分类器自身也可能产生偏差。

## 12. 与其他已读文献的关系

本次阶段 2 队列当前只完成 ChatHLS 一篇正文，因此没有第二篇已完成正文可做事实级横向比较。与 A1 阶段发现报告中的 TimelyHLS、HLSPilot 仅存在候选元数据，尚未完成正文核验，不能在本节把它们当作已读证据。

从本篇内部角色划分看：LLM 1 负责 HLS-C 生成，HLSTuner 负责 directive 选择/配置/插入，HLSFixer 负责工具反馈驱动的错误分析与修复，VODA/BugRAG 提供验证数据和错误知识。因此本篇最适合作为 `SELECTOR/S2` 的 HLS baseline，同时可作为工具反馈与错误数据构造的辅助参考。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | HLS-C 生成、directive 级性能优化和 HLS 错误修复 |
| 核心问题 | directive 组合空间大、QoR 非线性、HLS 特定错误难诊断 |
| 输入 | C/C++ 或自然语言、HLS-C、循环/数组元数据、初始 QoR、错误日志 |
| 输出 | HLS-C、directive 选择/配置/插入计划、修复指令 |
| 核心方法 | HLSTuner + HLSFixer + VODA/BugRAG + HLS 工具闭环 |
| 使用的模型 | Qwen-2.5-Coder-14B-Instruct；DeepSeek-V3.2 教师/插入代理；多模型对照 |
| 使用的编译器工具 | Vitis HLS 2022.1、CSIM、CSYN、C/RTL COSIM |
| 是否使用强化学习 | 否；使用 SFT 和 DPO，不是 PPO/GRPO 在线 RL |
| 是否使用形式化验证 | 否；使用测试、综合和协同仿真验证，不是形式化证明 |
| 数据集规模 | HLSTuner 4,804；HLSFixer 10,878 buggy samples、3,716 preference pairs；测试 108/591 |
| 主要指标 | pass@k、latency cycles、DSP/FF/LUT、speedup、critical path、功耗 |
| 最重要实验结果 | HLSFixer debugging pass@1 93.4%；HLSTuner 相对 baseline 几何平均 speedup 18.1× |
| 核心创新 | QoR-aware directive Selector、层次化 HLSFixer、VODA 验证数据扩增 |
| 主要局限 | 仅覆盖有限 directive，偏 Vitis/ZCU106，不支持复杂 DATAFLOW/AXI interface |
| 与 RISC-V 研究的相关性 | 中：可借鉴反馈闭环和保守选择，但平台与动作空间不同，不能直接替换目标硬件 |
| 最适合作为 | HLS Selector baseline、工具反馈模块、错误数据构造参考 |

这篇论文最值得学习的是把 directive 选择、真实 HLS 验证和 QoR 反馈组织成可迭代闭环；最主要的局限是工具、平台和 directive 范围较窄，并且存在数据污染与跨后端泛化风险。如果用于后续研究，最合理的使用方式是作为 HLS Selector 与验证闭环 baseline，而不是简单把 Xilinx 平台替换成 RISC-V 或把 HLS directive 直接改名为 LLVM pass。
