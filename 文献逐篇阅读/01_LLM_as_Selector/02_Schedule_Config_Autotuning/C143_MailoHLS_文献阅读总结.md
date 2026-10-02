# MailoHLS 文献阅读总结

论文题目：**MailoHLS: Multi-Adapter Structure-Aware Learning for Pareto-Driven HLS Pragma Optimization**

作者：Elena Vouvali、Dimosthenis Masouros、Aggelos Ferikoglou、Dimitrios Soudris、Sotirios Xydis

发表时间：2026 年 6 月 5 日（arXiv:2606.07246v1）

发表平台：arXiv 预印本（2026）

论文链接或编号：arXiv:2606.07246
元数据核验来源：[arXiv](https://arxiv.org/abs/2606.07246)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：FPGAs、HLS、设计空间探索（DSE）、大语言模型（LLM）、LoRA、结构感知优化、目标感知适配

> 本笔记只依据本地核验的 14 页 PDF 正文；论文事实、阅读分析和后续建议分开表述。

## 1. 研究背景

高层次综合（High-Level Synthesis，HLS）把 C/C++ 类高层描述转换为 FPGA 加速器，编译器负责调度、资源分配和 RTL 生成。设计者通过 pragma（编译指示）控制流水线、循环展开、数组分区等微架构决策，因此最终延迟和资源占用高度依赖 pragma 的组合与参数。

论文指出，pragma 之间存在耦合：数组分区决定可用内存带宽，内层循环展开又决定所需并行读写；孤立地选一个 pragma 可能导致资源浪费、带宽不足或综合失败。HLS 设计空间巨大，而每个配置通常需要耗时的综合评估。既有综合驱动、模型驱动和数据驱动方法分别受到评估昂贵、应用依赖或特征表达不足的限制。

论文进一步认为，通用 decoder-only LLM 的序列 token 表示不显式表达循环层次、数据依赖和内存共享关系，因而容易生成放置错误、资源超限或性能不佳的 pragma。MailoHLS 的目标是把 LLM 的语义/优化意图推理与 GNN（图神经网络，显式建模程序结构）结合起来。

## 2. 论文要解决的问题

### 2.1 复杂 pragma 组合的结构依赖

给定 HLS kernel，如何协调循环变换、流水线、数组分区和参数，使配置能够满足硬件约束并达到目标 QoR（Quality of Results，结果质量，如延迟和资源利用率）。

### 2.2 LLM 的结构盲区与非法输出

如何避免让 LLM 直接改写完整代码或自由生成 pragma，从而减少 pragma 位置错误、无效配置和无法综合的问题。

### 2.3 多目标之间的冲突

如何用同一框架分别支持低延迟、低资源和折中目标。论文明确指出，低延迟往往需要更强并行性和更高资源，而资源目标倾向于抑制硬件复制。

> 本文主要研究：如何结合程序结构表示、LLM 语义表示和目标条件适配器，预测预先固定位置上的 HLS directive 配置。

## 3. 核心方法概述

MailoHLS 将 HLS 优化重新表述为 directive-level structured prediction：解析器先找到合法 action point（如循环和数组），插入固定的 directive placeholder，模型只预测 placeholder 的值，不预测 pragma 应放在哪里。一个独立 GNN 从 LLVM IR 构造的扩展层次图中提取结构 embedding；decoder-only LLM 编码源码语义；cross-attention（交叉注意力）把与 placeholder 对齐的结构信息注入 LLM。不同优化目标使用不同 LoRA（Low-Rank Adaptation，低秩参数高效微调）适配器。

```text
C/C++ HLS kernel + 目标（延迟/资源/折中）
        ↓
+ 合法 action point + directive placeholders
        ↓
LLVM IR → ProGraML 风格图 + pragma/辅助/层次节点 → GNN 结构 embedding
        ↓                                  ↘
结构化 prompt → LLM 语义 embedding → cross-attention + gating
                                      ↓
目标专用 LoRA 适配器预测 placeholder 值
        ↓
Vitis HLS 综合 → 延迟/资源 QoR → 与 Pareto 参考点比较
```

LLM 的角色是选择和参数化预定义 directive；编译器/HLS 工具负责解析、综合和测量。论文的方法不是让 LLM 直接生成 RTL，也不是让 LLM 自由决定 pragma 的源代码位置。

## 4. 实验框架与训练流程

本文涉及模型训练，并采用三个阶段；共享的基础模型是 4-bit 量化的 DeepSeek-Coder-7B，每个优化目标各有一个 LoRA adapter。

### 4.1 第一阶段：语义对齐

对 objective-conditioned directive completion 做 SFT（Supervised Fine-Tuning，有监督微调），输入带 placeholder 的 kernel 和目标，输出已评估的合法 directive assignment。cross-attention 暂时关闭，只更新 LoRA；每个 kernel/目标保留得分最高的 6 个 design point，并进行 score-based weighting。

### 4.2 第二阶段：结构对齐

开启 cross-attention，注入 GNN 结构 embedding；冻结 LoRA 和 embedding，仅训练 cross-attention 与 gating 参数，使模型利用循环层次、依赖、并行度和内存交互。

### 4.3 第三阶段：目标偏好对齐

使用 DPO（Direct Preference Optimization，直接偏好优化）训练 Pareto-ranked preference pairs：较优配置作为 chosen，支配或较差配置作为 rejected。LoRA 保持冻结，仅更新 cross-attention 和 gating，使输出靠近当前目标的 Pareto 区域。

推理时根据目标启用 latency、balanced 或 resource adapter，然后预测各 placeholder 的 directive 值，再由 Vitis HLS 综合。论文没有使用 PPO 或 GRPO；它使用 SFT 与 DPO，而不是在线强化学习。

## 5. 奖励函数、损失函数或关键公式

论文没有给出一个用于在线强化学习的奖励函数，因此本文没有强化学习奖励函数。关键优化目标是多目标 Pareto 排序和几何平均 speedup；正文没有给出完整的 SFT 交叉熵公式或 DPO 展开公式。

| 机制 | 正文中的作用 |
|---|---|
| Pareto ranking | 按延迟、资源等目标筛选/排序 design points，并构造偏好对 |
| SFT | 模仿合法、已综合评估的 directive assignment |
| DPO | 让模型偏好目标更合适的配置，不需要单独训练奖励模型 |
| Speedup | 相对于无 pragma 的 Vitis 基线的加速比，越大越好 |
| Resource utilization | BRAM、DSP、FF、LUT 等资源占用，通常越低越好，但需与延迟联合考虑 |

需要注意：论文的“Pareto 参考点”来自 GNΩSIS 数据集中已探索的设计点，不能直接等同于全局最优。

## 6. 实验设置

### 6.1 数据集来源

训练和评估使用 GNΩSIS 数据集。论文称该数据集包含约 219K 个设计点，覆盖两个 FPGA 架构和三个目标时钟频率；本文聚焦 AMD/Xilinx UltraScale+ ZCU104、100 MHz 子集，约 26,500 个设计点。该数据集没有功耗测量，因此功耗不是实验目标。

数据按 80%/10%/10% 划分训练、验证和测试；另有 leave-one-family-out 设置，将整个 MachSuite kernel family 留作测试，并测试完全未见应用。论文没有明确说明完整的去重流程或数据泄漏审计，因此这些风险不能由正文排除。

### 6.2 模型与工具

模型为 4-bit NF4、bfloat16 的 DeepSeek-Coder-7B；对比 backbone sensitivity 时还使用 DeepSeek-Coder-1.3B。训练采用 PagedAdamW8bit、cosine schedule、3% warmup、gradient checkpointing 和 4K context。LoRA 使用 r=8、alpha=16；cross-attention 默认每 8 个 Transformer 层插入一次，结构 memory 为 64 slots、维度 32。

编译/评估使用 LLVM IR、Vitis HLS 2021.1 和目标 ZCU104@100 MHz。图结构以 ProGraML 风格扩展，加入 pragma nodes 和 auxiliary nodes，并用 edge-aware TransformerConv、Jumping Knowledge、多层聚合和 global attention pooling 产生结构表示。正文没有报告 RISC-V 硬件实验。

### 6.3 对比方法

主要对比包括无 directive 的 Vitis 基线、GNΩSIS Pareto 中的 latency-optimal/resource-optimal/Pareto-knee 参考点、CollectiveHLS、LIFT，以及直接提示 GPT-4o、Claude Haiku 4.5 和 Gemini 2.5 Pro。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| Speedup | 相对无 directive Vitis 基线的延迟加速比 | 越大越好 |
| BRAM/DSP/FF/LUT utilization | FPGA 资源占用及其平均/归一化比例 | 通常越低越好，但必须满足容量 |
| Valid/synthesizable design | 配置能否通过工具并得到可用设计 | 越高越好 |
| Pareto proximity/dominance | 与目标 Pareto 参考点的接近或支配关系 | 越接近目标越好 |

## 7. 实验结果与结论

### 7.1 主要结果

在 seen kernels 上，latency、balanced、resource 三个 adapter 的几何平均 speedup 分别为 9.48x、7.95x、2.10x；对应 GNΩSIS 参考值为 12.42x、5.65x、2.07x。latency adapter 平均资源利用率 9.35%，略低于参考点的 9.75%；balanced adapter 为 7.01%，高于 Pareto knee 的 3.79%；resource adapter 为 3.82%，略高于参考点的 2.94%。这些是 ZCU104 数据集与 Vitis HLS 评估口径下的结果。

在 MachSuite 未见 kernel family 上，三个 adapter 的几何平均 speedup 分别为 4.97x、5.21x、2.45x；相应 Pareto 参考值为 20x、5.82x、4.74x。论文将此解读为：跨分布迁移后仍优于无 directive 基线，但达到已探索 Pareto 参考点更困难。

### 7.2 与传统方法的比较

在 Kalman Filter 上，MailoHLS 的 latency、balanced、resource 配置分别达到 8.6x、6.2x、1.9x speedup，并满足 FPGA 资源限制；CollectiveHLS 虽有较高 speedup，但资源过量而不可行。对 GAN，MailoHLS 三个目标点为 4.7x、4.9x、2.0x，均在容量内；CollectiveHLS 和 Claude Haiku 4.5 分别达到 186.8x 和 72.9x，但其策略更激进。MailoHLS 的优势是一次框架提供显式目标条件的多个可行 operating point，而不是在所有 workload 上取得最高峰值。

### 7.3 与其他 LLM 方法的比较

在结构复杂的 Kalman 设置中，GPT-4o 未生成可用设计，Claude Haiku 4.5 与 Gemini 2.5 Pro 的配置超过 FPGA 容量。LIFT 具有结构信息但仍因数组分区处理不完整而超用 BRAM。论文因此把 MailoHLS 的收益归因于固定合法位置、结构 embedding 和目标适配的组合，而非单纯增加 LLM 参数量。

### 7.4 消融实验

在 Kalman kernel 上，只有 objective-conditioned SFT 时，latency、balanced、resource adapter 的 speedup 为 2.0x、1.7x、1.2x；加入 GNN cross-attention 后为 6.3x、6.2x、1.15x；再加入 DPO 后为 8.6x、6.2x、1.9x。该消融支持结构对齐是主要增益来源，DPO 进一步改变目标偏好。

将 backbone 从 1.3B 换为 7B，平均 speedup 从 4.13x 增至 4.89x，平均资源占用从 93% 降至 75%。cross-attention 更密时平均 speedup 从 3.58x 增至 4.89x，但资源占用从 68.0% 增至 93.3%。

### 7.5 案例分析

论文的 GEMV 例子说明数组 partition 与 loop unroll 必须配套：内存带宽不足会使展开无法兑现。模型在 latency 目标下更常使用 PIPELINE II=1、激进展开和较大 partition factor；在 resource 目标下显著减少这些选择。GAN 案例显示保守策略牺牲峰值速度，但能避免资源不可行。

## 8. 主要创新点

### 8.1 创新点一：结构—语义联合选择

论文不是简单把源码塞给 LLM，而是构造包含控制流、数据依赖、循环层次、共享数组及 pragma 节点的图，并通过 cross-attention 对齐到 directive placeholder。消融中结构阶段带来明显提升，实验支持该机制的有效性。

### 8.2 创新点二：directive-level 决策抽象

将合法 action point 和输出 schema 固定下来，LLM 只预测值。这区别于自由代码/pragma 生成，缩小了搜索空间并降低语法和位置错误；但这依赖解析器和预定义 directive 集合，并不意味着所有 HLS 优化都能表达。

### 8.3 创新点三：目标条件的多 LoRA + Pareto 偏好

每个目标独立适配，SFT 学习可行配置，DPO 使用 Pareto-ranked preference pairs 学习折中关系。结果表明 balanced adapter 和 resource adapter 的行为不同，说明目标专用策略不是仅更换 prompt 文本。

## 9. 局限性

### 9.1 论文明确承认的局限

论文基于 ZCU104、Vitis HLS 2021.1 和 GNΩSIS，且不包含功耗目标。模型主要由 pragma placeholder 表达，搜索能力受预定义 directive 和 action point 限制。完全未见 kernel 的性能仍低于已见数据的 Pareto 参考。论文还指出，通用 LLM 的激进策略会超资源，MailoHLS 为保证可行性采用了更保守的配置。

### 9.2 阅读后发现的潜在局限

1. 正文称方法是“LLM-based”，但核心实验使用 DeepSeek-Coder-7B 微调模型；未报告与同规模纯 GNN 或无 cross-attention 的完整等预算比较，增益拆分仍可能受训练预算影响。
2. 论文只给出约 26,500 个目标平台设计点和有限应用，跨 FPGA 架构、ASIC、CPU、GPU 或 RISC-V 的迁移没有实验证据。
3. Pareto 参考来自已有 GNΩSIS 探索集，参考点不等于真实全局最优；论文自己也观察到模型能支配部分参考点，说明参考集可能不完整。
4. DPO 偏好对依赖综合数据和 Pareto 排序，若综合噪声、配置泄漏或训练/测试 kernel 相似度处理不充分，泛化数字需要额外复核。
5. 结构图从 LLVM IR 推导，但 pragma 位置、数组声明与 HLS 前端语义之间的映射仍是工程依赖；正文没有给出完整 parser 失败率和跨工具兼容性。

## 10. 阅读后的研究方向反思

MailoHLS 对本仓库 SELECTOR 方向的直接价值在于：把“选择优化策略”从自由文本生成转成受约束的结构化决策。值得借鉴的是 placeholder/action point 抽象、结构 embedding 与目标条件 adapter 的组合，以及把“可行性”作为选择空间的一部分。

不能直接照搬的部分包括：将 ZCU104 的 pragma 选择原样移植到 RISC-V LLVM pass 选择；HLS pragma 与 LLVM pass 的合法性、反馈信号和硬件约束不同。仅把目标平台替换为 RISC-V 不足以构成新贡献。更有研究价值的是把 LLVM IR 图、pass 前置条件/后置状态、真实 RISC-V 硬件计数器和语义验证联合起来，验证选择策略是否跨程序、跨微架构稳定。

从角色看，MailoHLS 最适合作为 SELECTOR 的结构化策略预测 baseline，也可作为“目标条件选择器 + 约束编译器”的方法参考，而不是完整的 RISC-V 优化系统。

## 11. 可进一步尝试的研究方向

### 11.1 方向一：面向 RISC-V LLVM 的结构化 pass 配置选择

#### 研究问题

能否用 CFG/数据依赖/循环层次图和 pass 状态，预测满足前置条件的 LLVM pass 序列或参数。

#### 与原论文的区别

对象从 HLS pragma 配置变为 LLVM pass/phase 状态，并引入 RISC-V 后端反馈，不是替换硬件名称。

#### 可能的创新点

把 pass 依赖、已执行 pass 历史、RVV 向量化可行性和语义验证结果作为结构化决策上下文。

#### 实验框架

```text
LLVM IR + RISC-V 配置 → 图与 pass 状态 → 约束选择器 → LLVM pass pipeline
        → Alive2/测试 + Spike/真实板卡性能 → 记录可行性与 speedup
```

#### 可行性

需要 LLVM、RISC-V 工具链、CompilerGym/自建 pass 数据、Alive2 或等价验证及 RVV 硬件/模拟器。

#### 主要风险

pass 状态空间和测量噪声很大；语义等价验证可能不覆盖后端性能错误。

### 11.2 方向二：目标条件的延迟—代码尺寸—能耗选择器

#### 研究问题

一个选择器能否根据不同部署目标切换 pass 强度，同时保持语义正确和资源约束可行。

#### 与原论文的区别

将 HLS 的 latency/resource Pareto 扩展为 RISC-V 真实二进制的运行时间、代码尺寸与能耗，并比较不同微架构。

#### 可能的创新点

用多目标偏好数据和硬件计数器校准目标 adapter，分析同一 pass 在 RV64GC 与 RVV 平台上的策略迁移。

#### 实验框架

```text
多目标编译样本 → Pareto/偏好对 → 目标适配器 → RISC-V 编译
        → 功能测试 + perf/功耗测量 → 更新偏好与跨平台评估
```

#### 可行性

需要多目标编译样本、RISC-V 开发板或可重复模拟环境，以及统一的 benchmark harness。

#### 主要风险

功耗和性能测量方差可能掩盖真实收益；目标之间也可能产生不可比的 Pareto 前沿。

### 11.3 方向三：验证门控的 pass 选择

#### 研究问题

能否在每一步选择 pass 前，利用语义验证和编译器诊断拒绝高风险动作，减少探索浪费。

#### 与原论文的区别

MailoHLS 通过预定义 placeholder 限制配置；此方向把验证反馈作为动态门控状态，处理 LLVM pass 间的状态变化。

#### 可能的创新点

将 pass 前置条件、失败诊断、IR 差分和 Alive2 结果融合为可审计的选择证据。

#### 实验框架

```text
候选 pass → 静态前置条件筛选 → 执行/验证 → 性能测量
        → 失败记忆与风险更新 → 下一步 pass 选择
```

#### 可行性

适合在现有 LLVM/Alive2/CompilerGym 基础上逐步实现，先做离线数据集再做在线 agent。

#### 主要风险

验证成本可能超过优化收益；有限测试不能被误写为形式化证明。

## 12. 与其他已读文献的关系

本轮只完成 MailoHLS 一篇论文，因此没有当前批次内可作事实横向比较的其他新论文。与仓库既有条目的关系只能作分类层面的建议：MailoHLS 的核心角色是 SELECTOR，具体更接近 `S2_Schedule_Config_Autotuning`，因为它选择并参数化 HLS directive 配置，而不是直接生成源代码、LLVM IR 或新 compiler pass。它可与已有的 LLM pass/phase 选择论文作 baseline 对照，但不能把 HLS pragma 决策结果直接当作 LLVM pass ordering 结果。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 目标条件的 HLS pragma 配置选择 |
| 核心问题 | 结构耦合、多目标冲突和 LLM 非法/低效 directive |
| 输入 | C/C++ HLS kernel、目标、固定 action point/placeholders |
| 输出 | 每个 directive placeholder 的合法值 |
| 核心方法 | LLVM IR 扩展层次图 + GNN + LLM cross-attention + 多 LoRA + DPO |
| 使用的模型 | 4-bit DeepSeek-Coder-7B；敏感性实验含 1.3B |
| 使用的编译器工具 | LLVM IR、Vitis HLS 2021.1 |
| 是否使用强化学习 | 否；使用 SFT 和 DPO |
| 是否使用形式化验证 | 否；依赖 HLS 可综合性/资源可行性，正文未报告形式化证明 |
| 数据集规模 | GNΩSIS 约 219K 设计点；目标子集约 26,500 |
| 主要指标 | speedup、BRAM/DSP/FF/LUT、设计有效性、Pareto 接近度 |
| 最重要实验结果 | seen kernels 几何平均 speedup：latency 9.48x、balanced 7.95x、resource 2.10x；MachSuite holdout：4.97x、5.21x、2.45x |
| 核心创新 | 结构—语义融合、directive-level 决策、目标专用 Pareto 偏好适配 |
| 主要局限 | 单一 FPGA/HLS 生态，跨平台与真实 RISC-V 证据缺失，Pareto 参考非全局最优 |
| 与 RISC-V 研究的相关性 | 中：选择器抽象和结构融合可迁移，但 HLS pragma 与 RISC-V LLVM pass 语义不同 |
| 最适合作为 | SELECTOR/S2 的方法参考与 baseline |

> 这篇论文最值得学习的是把 LLM 的优化决策限制在合法、可对齐的 directive 槽位，并用结构表示和目标适配器改善选择；最主要的局限是实验集中在单一 HLS/FPGA 环境，不能直接声称适用于 RISC-V LLVM pass。若用于后续研究，最合理的使用方式是借鉴其结构化选择与 Pareto 偏好机制，而不是简单替换硬件平台。
