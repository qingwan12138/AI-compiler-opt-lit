# LIFT 文献阅读总结

论文题目：**LIFT: LLM-Based Pragma Insertion for HLS via GNN Supervised Fine-Tuning**  
作者：Neha Prakriya、Zijian Ding、Yizhou Sun、Jason Cong  
发表时间：2025（arXiv 首次公开，2025-04-29）  
发表平台：arXiv 预印本（2025）
论文链接或编号：arXiv:2504.21187
元数据核验来源：[arXiv](https://arxiv.org/abs/2504.21187)
代码/数据/工件：未找到可确认的作者代码仓库或工件
关键词：大语言模型（LLM）、高层次综合（HLS）、pragma 插入、图神经网络（GNN）、设计空间探索（DSE）、LoRA、结构监督

> 本笔记依据本地完整 PDF（10 页）正文整理；论文事实与阅读后的分析分开描述。本文在 Taxonomy v2 中建议归类为 `SELECTOR / S1_Pass_Phase_Flag_Selection`：LLM 输出的是已有 HLS 工具执行的 pragma 配置，而不是变换后的源程序。

## 1. 研究背景

论文研究 FPGA 高层次综合（High-Level Synthesis, HLS）。HLS 将 C/C++ 提升为硬件设计输入，但高性能仍依赖用户插入 pipeline、loop unroll/parallel、tiling 等 pragma，以影响微架构。手工选择既需要硬件知识，又要反复运行综合；单次评估可能耗时数分钟到数小时。

传统路线包括非线性规划、设计空间探索（DSE）和 GNN 性能代理模型。论文指出，pragma 配置空间可达百万量级；AutoDSE 需要大量真实 HLS 评估，GNN 方法虽可降低评估开销却可能受训练分布限制。LLM 擅长长距离代码上下文，但普通代码模型不一定理解一个很小 pragma 数值变化对并行度、流水化和资源占用的硬件后果。

## 2. 论文要解决的问题

### 2.1 双向上下文问题

pragma 值常依赖插入点前后的代码，例如后面的循环边界和内存访问模式会影响并行或 tiling 因子；标准因果式下一词预测不天然适合此任务。

### 2.2 有效与失败设计的学习问题

HLSyn 同时包含可综合设计和因资源超限、综合超时或工具失败而无效的配置。论文不希望简单丢弃失败样本，而是让模型学习哪些配置不可行。

### 2.3 结构语义问题

预实验显示，仅靠文本微调时，不同 pragma 配置的 LLM 表征更倾向按 kernel 身份聚类，难以区分微架构影响。因此论文研究如何用程序图结构监督 LLM 的 pragma 学习。

> 本文主要研究：如何让 LLM 为给定 HLS C/C++ 代码推断低延迟且可综合的 pragma 配置，并用 GNN 的结构语义信号辅助微调。

## 3. 核心方法概述

LIFT 把 pragma 因子预测建模为代码 infilling：输入含 `auto{}` 占位符的 HLS 代码，LLM 输出 pragma 键值，再交给传统 HLS 工具编译/综合。训练时，预测配置和真实配置分别回填代码，编译为 LLVM IR，再通过 ProGraML 构造程序图，由预训练 HARP GNN 编码；图嵌入均方误差与 pragma token 交叉熵共同反向监督 LLM。

```text
含 auto{} pragma 占位符的 HLS C/C++
        ↓
ILM/infilling 格式输入 DeepSeek-Coder 7B
        ↓
预测 pipeline / parallel / tile 等 pragma 因子
        ↓
回填代码，交给 HLS/LLVM IR/ProGraML 图管线
        ↓
HARP GNN 比较预测图与目标图的结构嵌入
        ↓
CE + 图嵌入 MSE + 延迟权重 → LoRA 更新
        ↓
推理时一次生成 pragma 配置，再由 HLS 工具综合评估
```

LLM 的最终角色是选择/生成已有 pragma 空间中的配置；HLS 工具负责实际硬件变换和 QoR 测量。论文的输出是 pragma 配置，而不是新的 C/C++ 变换器或新的编译器 pass。

## 4. 实验框架与训练流程

### 4.1 数据预处理与 infilling

论文使用 Infilling by Language Modeling（ILM）组织样本：`input_code <sep> target_pragmas`，损失只作用于目标 pragma 部分。这样单向模型可以把占位符前后完整代码作为条件。

### 4.2 延迟加权与重采样

HLSyn 的 cycle latency 经对数稳定化、幂变换和反向 min-max 归一化转为样本权重。权重高的样本过采样，权重低的样本保留一部分；阈值为 `τ=0.5`。无效设计的 `perf` 为 0，获得很低权重但没有被完全丢弃。

### 4.3 预实验

作者先对 DeepSeek-Coder 7B 做 LoRA 微调。仅使用性能加权和重采样时训练损失停滞；t-SNE 显示不同 pragma 配置仍按 kernel 聚类，促使作者加入 GNN 结构监督。

### 4.4 GNN 结构监督微调

LLM 预测 pragma 后，与目标 pragma 一起回填输入代码；两份代码均编译到 LLVM IR，使用 ProGraML 建图并加入 pragma/pseudo 节点和控制、数据、调用流边。预训练 HARP GNN 输出两图嵌入，嵌入距离被纳入训练损失。LLM 使用 DeepSeek-Coder 7B 的 LoRA 更新，论文没有采用 PPO、GRPO 或其他强化学习。

### 4.5 推理与工具版本迁移

推理时，模型直接为未指定 pragma 因子的 kernel 生成配置；生成配置被送入 HLS 工具进行综合。作者还用 Vitis 2021.1 测试在 Vitis 2020.2 上训练的模型，以评估工具版本迁移。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数，核心是监督微调损失和性能权重。

### 5.1 性能样本权重

对 latency `ℓ_i`，论文先计算 `ℓ'_i = log(1 + max(ℓ_i, ε0))`，再计算 `z_i=(ℓ'_i)^p`，最后以反向 min-max 归一化得到 `w_i`。低 latency 获得更高权重；无效设计 latency 为 0，在实现中被赋予很低训练权重。

### 5.2 图结构损失

`L_GNN = || f_GNN(G_pred) - f_GNN(G_tgt) ||²₂`

其中 `G_pred` 和 `G_tgt` 是预测/目标 pragma 回填后构造的程序图，`f_GNN` 是 HARP 的图编码器。该项要求预测配置产生的程序结构表征接近目标设计。

### 5.3 Token 损失与总损失

`L_CE = CrossEntropy(y_hat, y)`  
`L_total = w(x) · L_CE + w(x) · L_GNN`

`y_hat` 是预测 pragma 序列，`y` 是目标序列。论文用同一样本权重同时调节文本正确性和结构监督；没有单独的显式资源利用率损失，也没有直接将真实硬件运行时间作为在线奖励。

## 6. 实验设置

### 6.1 数据集来源

使用 HLSyn，覆盖 42 个 kernel、超过 40,000 个 design points，来源包括 PolyBench 和 MachSuite。每个点包含 `perf`（cycle latency）、pragma 配置、资源利用率和 `valid` 标记；无效条件包括综合超过 200 分钟、超过板卡资源或工具无法实现。训练/验证/测试按 70%/15%/15% 划分，测试 kernel 对训练不可见。训练数据主要使用 latency 权重，预处理没有显式纳入板上资源利用率指标。

### 6.2 模型与工具

基础模型为 DeepSeek-Coder 7B；LoRA rank=8、alpha=16，目标模块为 q/k/v/o projection、gate/up/down projection，共约 1,900 万可训练参数，占完整模型约 0.29%。训练用 PyTorch 和 8 张 AMD MI250（每张 128GB HBM2e）。HLSyn 的性能数据来自 Vitis 2020.2、AMD/Xilinx Alveo U200、250 MHz；背景描述还提到 Merlin 和 Vitis HLS。图构造使用 LLVM IR、ProGraML 和预训练 HARP GNN。

### 6.3 对比方法

AutoDSE 使用 6、10、24 小时预算；HARP 在相同数据集训练后再做 1 小时 DSE；GPT-4o 直接按提示为 pragma 因子填值，并要求资源不超过 75%。这些 baseline 多数通过搜索或代理模型找配置，不是直接配置生成器。

### 6.4 评价指标

主要指标是综合设计 latency（cycles，越低越好）及相对 latency 的 speedup（越高越好）；“Failed”表示在 15 小时内未能完成编译和综合。工具版本迁移实验比较 Vitis 2021.1 下 HARP 与 LIFT 的 latency。论文还报告训练耗时和生成配置的 valid/compilable 情况。

## 7. 实验结果与结论

### 7.1 主要结果

表 I 的 9 个测试 kernel 中，LIFT 在多数 kernel 达到或超过 baseline。论文报告平均相对 AutoDSE-24h 快 3.52×、相对 HARP 快 2.16×、相对 GPT-4o 快 66×。例如 `trmm-opt` 的 latency 为 3,284 cycles，相对 AutoDSE-24h 的 9,387 cycles 为 2.86×，相对 HARP 的 7,395 cycles 为 2.25×；`fdtd-2d-large` 为 249,346 cycles，相对 AutoDSE-24h 的 2,236,378 cycles 为 8.97×。

### 7.2 与传统方法的比较

LIFT 以单次 pragma 预测避免了 AutoDSE 的长时间实际搜索；但并非所有 kernel 都胜过 A24，例如 `atax-medium` 的相对值为 0.88×。因此“平均改善”不能理解为每个任务均改善。

### 7.3 与其他 LLM 方法的比较

GPT-4o 在多项 kernel 上 latency 很高，且作者观察到它有时为同一 directive 输出多个值，违反 HLS 语法；提示中加入“每个 factor 只能一个值”后才进行比较。LIFT 是领域微调模型，不能把结果泛化为通用 LLM 都具备 HLS pragma 能力。

### 7.4 消融与训练诊断

正文主要用“朴素 LoRA 微调损失停滞”和 t-SNE 聚类作为动机诊断；提供了文本损失与图嵌入损失的组合设计。当前 PDF 内容不足以确认一个完整的逐组件数值消融表，因此不能声称每个模块都有独立 ablation 数字。

### 7.5 工具版本迁移

模型在 Vitis 2020.2 训练、Vitis 2021.1 测试时，相对 HARP 平均改善 2.74×。表 II 中 `covariance`、`syr2k`、`jacobi-2d`、`trmm-opt`、`gemm-p-large` 的相对值分别为 7.05×、1.83×、1.05×、1.33×、2.45×。作者据此认为 LIFT 对工具版本变化更稳健，但该结论仍限于给定 HLSyn kernel 和 Vitis 版本设置。

## 8. 主要创新点

### 8.1 创新点一：面向 HLS pragma 的 LLM Selector

论文把 LLM 用作 HLS pragma 配置选择器，直接输出已有配置空间的因子，区别于只预测性能再驱动搜索的代理模型。测试结果支持其在特定 HLSyn 设置下减少搜索依赖。

### 8.2 创新点二：GNN 图级监督 LLM

预测与目标 pragma 都被编译并转成程序图，以 HARP GNN 嵌入距离监督 LLM。价值在于把 pragma 对控制/数据流和微架构相关结构的影响引入语言模型训练，而不只依赖 token 共现。

### 8.3 创新点三：延迟感知且保留失败样本的训练数据策略

论文用 latency 生成训练权重，并保留低权重无效样本，尝试同时学习高性能区域和失败边界。该设计是训练目标/数据处理机制，不等同于在线 RL。

## 9. 局限性

### 9.1 论文明确承认或限定的范围

实验基于 Merlin 生成的 HLSyn 数据，目标硬件主要是 Alveo U200，训练与迁移只覆盖 Vitis 2020.2/2021.1。论文称方法可推广到其他 HLS 工具，但没有在本文中完整验证。模型输出的是预定义 pragma 因子，不能据此声称能自动设计任意硬件微架构。

### 9.2 阅读后发现的潜在局限

第一，随机划分和同一 benchmark 家族内的 held-out kernel 仍可能留下相近结构分布，跨应用、跨厂商工具和跨 FPGA 架构的泛化需要额外验证。第二，结构监督依赖 HLS/LLVM IR/ProGraML/HARP 管线，训练成本高；8 张 MI250、每 epoch 约 3.1 小时并不等于低成本部署。第三，论文报告了 valid/compilable，但没有在正文中提供完整资源、功耗和真实板上吞吐的逐项统计。第四，低 latency 权重与资源约束之间可能存在目标冲突，而预处理没有显式使用资源利用率作为损失。

## 10. 阅读后的研究方向反思

值得借鉴的是“让 LLM 只负责离散配置选择、让传统工具负责语义执行和性能测量”，以及用可编译中间表示把表面 token 与结构后果连接起来。不能直接照搬的是 HLSyn、Merlin、HARP 和 Alveo U200 的组合；只将 FPGA 换成 RISC-V 并不足以形成新贡献。对 LLVM/RISC-V 研究，LIFT 更适合作为 Selector baseline 或配置策略模块，而不是完整 RISC-V 编译器方案。迁移到 RISC-V 时必须重新定义目标配置（如 pass/flag、RVV 向量化参数、后端调度选项）、合法性检查和真实硬件代价模型。

## 11. 可进一步尝试的研究方向

### 11.1 RVV 后端配置的结构监督选择

#### 研究问题

让 LLM 选择 LLVM pass 序列、RVV 向量化参数或后端调度配置，并学习其对 LLVM IR/机器指令图的影响。

#### 与原论文的区别

目标从 HLS pragma/FPGA 综合转为 RV64/RVV 后端决策，同时处理可变向量长度和 ISA 特性约束。

#### 可能的创新点

构造 RVV 语义与硬件计数器联合图监督，显式建模 VL/VTYPE、寄存器压力和访存代价。

#### 实验框架

```text
LLVM IR → LLM 选择 pass/RVV 配置 → LLVM 后端 → Spike/真实 RVV 测量 → 图与性能反馈
```

#### 可行性

需要 LLVM、RISC-V GCC/LLVM、Spike 或真实板卡、性能计数器和可复现 benchmark。

#### 主要风险

RVV 真实性能受微架构、编译器版本和内存系统影响；图嵌入接近不代表语义或性能等价。

### 11.2 显式资源-延迟 Pareto Selector

#### 研究问题

将 latency、LUT/BRAM/DSP 或 RISC-V 代码大小/能耗作为多目标配置选择约束。

#### 与原论文的区别

不把资源仅作为 latency 的隐式信号，而是输出可解释 Pareto 配置并显式验证约束。

#### 可能的创新点

多目标结构监督、约束解码和失败反例分类。

#### 实验框架

```text
程序图 + 目标约束 → LLM 候选配置 → 合法性检查 → 编译/综合/测量 → Pareto 更新
```

#### 可行性

可复用 HLSyn 的 resource 字段或建立 LLVM/RISC-V 多目标测量集。

#### 主要风险

多目标权衡可能导致训练信号稀疏，且不同设备的 Pareto 前沿不稳定。

## 12. 与其他已读文献的关系

本轮只有 LIFT 完成正文阅读，因此不把其他候选论文的未阅读内容作为事实比较。与正式 corpus 中已有 Selector 条目的去重结论是：LIFT 不同于 MailoHLS（本轮明确排除）、AUTOSPARSE、REASONING COMPILER、FlowCompile 和其他已有 HLS/编译优化条目；其独特交集是“LLM 直接选择 HLS pragma 因子 + GNN 结构监督微调”。它可作为后续 pass/phase/config Selector 的 HLS 领域 baseline，但不能与直接输出变换后源码的 Translator 混为一类。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | LLM 选择 HLS pragma 配置 |
| 核心问题 | pragma 依赖双向代码上下文且具有结构/微架构后果 |
| 输入 | 含 `auto{}` pragma 占位符的 HLS C/C++ |
| 输出 | pipeline、parallel、tile 等 pragma 因子 |
| 核心方法 | ILM infilling + 延迟加权重采样 + HARP GNN 图监督 |
| 训练方式 | DeepSeek-Coder 7B 的 LoRA 监督微调；无 RL |
| 关键损失 | 加权 token CE 与图嵌入 MSE |
| 数据集 | HLSyn，42 kernels、超过 40,000 design points |
| 工具与平台 | Merlin/HLSyn、LLVM IR、ProGraML、HARP、Vitis、Alveo U200 |
| 主要 baseline | AutoDSE、HARP、GPT-4o |
| 主要结果 | 平均相对 A24、HARP、GPT-4o 分别 3.52×、2.16×、66× |
| 主要局限 | HLS 工具/硬件范围窄，资源目标不显式，跨平台证据有限 |
| Taxonomy 建议 | `SELECTOR / S1_Pass_Phase_Flag_Selection`，置信度 HIGH |

