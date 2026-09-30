# TimelyHLS 文献阅读总结

论文题目：**TimelyHLS: LLM-Based Timing-Aware and Architecture-Specific FPGA HLS Optimization**
作者：Nowfel Mashnoor、Mohammad Akyash、Hadi Kamali、Kimia Azar
发表时间：2025
发表平台：arXiv cs.CR，v1，2025-07-23
论文链接或编号：[arXiv:2507.17962](https://arxiv.org/abs/2507.17962)；DOI 10.48550/arXiv.2507.17962
关键词：FPGA、LLM、HLS、timing closure、RAG、pragma

## 1. 研究背景

论文研究 FPGA 高层次综合（High-Level Synthesis, HLS）。HLS 将 C/C++ 程序转换为硬件，但流水线、循环展开、数组分区等 pragma 会显著影响时序、吞吐和资源。论文指出，面向不同 FPGA 器件的 pragma 选择仍依赖人工试错；AutoDSE 等启发式搜索需要大量综合运行，传统 ML 方法又可能难以迁移到新设计或新架构。TimelyHLS 引入 LLM、检索增强生成（RAG）和综合工具反馈，目标是自动生成面向器件的 HLS 代码。

## 2. 论文要解决的问题

### 2.1 时序收敛

设计需要满足最大频率约束，避免负 slack；深层逻辑、路由拥塞和循环依赖会使时序闭合困难。

### 2.2 架构特定的 pragma 配置

不同 FPGA 家族的资源数量、工具链和约束不同，同一 pragma 配置不能直接迁移。论文主要研究如何根据器件元数据生成相应的 HLS 代码和指令。

> 本文主要研究：如何利用带 FPGA 知识检索和综合反馈的 LLM，生成可综合、功能正确且满足时序的 HLS 代码。

## 3. 核心方法概述

TimelyHLS 将器件数据表、厂商 HLS 指南和架构参考资料整理为知识库。LLM 根据 kernel、性能目标和 FPGA 元数据生成带 pragma 的 C/C++，随后由 Vitis HLS 和 Vivado 检查并把日志返回模型进行迭代修正。

```text
HLS kernel + testbench + FPGA 元数据
        ↓
RAG 检索架构特征、pragma 和工具规则
        ↓
LLM 生成带 pragma 的 HLS C/C++
        ↓
Vitis HLS 编译、综合和 C/仿真检查
        ↓
导出 RTL，由 Vivado 综合、时序分析和 RTL 仿真
        ↓
日志、slack、资源和正确性反馈
        ↓
LLM 修正代码，直到满足约束或迭代结束
```

LLM 的最终输出是修改后的 HLS 源代码，因此按 Taxonomy v2 更接近 `TRANSLATOR / T1_Source_Optimization_Refactoring`，而不是只输出 pragma 配置的纯 Selector。

## 4. 实验框架与训练流程

### 4.1 数据集准备

作者从 CHStone、LegUp benchmarks 和 MachSuite 等开源仓库选择 10 个 HLS 应用，覆盖矩阵乘、卷积、向量运算、Bitonic Sort、Viterbi、CORDIC、FIR 和 Needleman–Wunsch 等任务，并为每个应用准备 HLS C++、测试平台、综合报告、时序摘要和资源日志。

### 4.2 初始生成

输入 prompt 包含功能、吞吐或循环目标以及目标 FPGA 的器件家族、DSP、BRAM、LUT 数量和时序约束。RAG 知识库来自器件数据表、HLS 用户指南和架构手册。论文举例使用 Code LLaMA 或 GPT-4。

### 4.3 HLS 层修正

Vitis HLS 2024.2 编译和仿真生成结果。遇到语法、资源绑定或流水线深度问题时，系统把日志放回 prompt，要求 LLM 重新生成，直到通过 HLS 综合和功能仿真。

### 4.4 RTL 层修正

通过 HLS 后导出 Verilog，由 Vivado 2024.2 综合并进行 WNS、TNS、资源利用率和 RTL 仿真检查。综合错误、关键路径报告和功能不匹配再次反馈给 LLM。本文没有 SFT、PPO、GRPO 或其他强化学习训练；主要是提示推理和工具反馈迭代。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数，也没有报告模型训练损失。系统的实际停止条件是：HLS 和 RTL 功能仿真通过、Vivado 可综合、目标器件无负 slack。论文没有给出一个统一的数学目标函数来定义多轮反馈中的候选排序。

## 6. 实验设置

### 6.1 数据集来源

数据来自 CHStone、LegUp 和 MachSuite，最终选择 10 个代表性应用；每个应用包含源代码、测试平台和工具日志。论文没有给出训练/验证/测试拆分，也没有明确说明数据泄漏控制。

### 6.2 模型与工具

实验环境为 Ubuntu 24.04.2，13 代 Intel i7-13700、32 GB 内存；Vitis HLS 2024.2 和 Vivado 2024.2。比较的 LLM 为 OpenAI GPT-4 与 Anthropic Claude-3.5-Sonnet，温度为 0.7。目标器件覆盖 Zynq、Zynq UltraScale+、Artix/Kintex-7、Spartan-7、Virtex UltraScale+、Kintex UltraScale+ 和 Versal AI Edge，共 10 个目标 FPGA。

### 6.3 对比方法

论文背景中讨论 OpenTuner、AutoDSE、HARP、LIFT 和 HLSPilot；结果部分主要报告 Base 与 TimelyHLS 的差异，并未给出完整统一的数值对比表。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| Speedup/latency | 相对基线的性能或延迟变化 | speedup 越大、latency 越小越好 |
| WNS/TNS | 最差/总负 slack，用于时序闭合 | 无负 slack 更好 |
| FF/LUT 使用量 | FPGA 资源占用 | 需结合性能权衡 |
| Loop II | 循环 initiation interval | 通常越小越好 |
| 功能仿真 | HLS/RTL 输出是否符合测试预期 | 通过更好 |

## 7. 实验结果与结论

### 7.1 主要结果

论文报告 Artix-7 上最高约 4× speedup；图 2 给出 Matrix Multiplication 3.85×、Bitonic Sort 3.7×、Needleman–Wunsch 2.8×、Viterbi 2.6×等最高值。摘要还报告部分 Viterbi 配置 FF 减少 57%。这些是论文报告的特定 benchmark/器件结果，不是所有任务的平均值。

### 7.2 时序和资源

Matrix Multiplication 的循环 II 从 16 降到 1–2；Bitonic Sort 从非流水线变为 II=1；Matrix-Vector Multiplication 使用流水线并达到 II=1。Vector Addition 和 Matrix-Vector Multiplication 的 LUT 使用量增加约 50–65%，体现吞吐与面积的权衡。Viterbi 在部分器件上同时减少 FF/LUT。

### 7.3 与其他 LLM 方法的比较

论文声称结果与专家优化设计相当或更好，但没有提供完整的 GPT-4、Claude-3.5、LIFT、HLSPilot 逐项结果表。不能据此推导对所有 LLM baseline 的统计优势。

### 7.4 消融实验

论文没有报告独立的 RAG、HLS 反馈、RTL 反馈或器件元数据消融实验。因此各模块的单独因果贡献无法由正文确认。

### 7.5 案例分析

正文以 Matrix Multiplication、Vector Dot Product、Vector Addition、Viterbi 等说明，LLM 通过 pipeline、unroll 和 array partition 改善瓶颈；同时部分任务以增加资源换取更低延迟。

## 8. 主要创新点

### 8.1 架构知识驱动的 HLS 代码生成

作者把 FPGA 特征、综合指令和 pragma 模板组织成 RAG 知识库，使生成结果包含目标器件信息。实验覆盖多个 FPGA 家族，但论文的“首次”表述应视为作者主张，不能据此确认领域首创性。

### 8.2 两级工具反馈闭环

HLS 级编译/仿真和 RTL 级综合/时序/仿真形成两级反馈，日志用于下一轮生成。该闭环比只依赖源代码提示更接近实际综合约束。

### 8.3 时序和面积的联合工程权衡

系统不只追求延迟，还观察 slack、FF、LUT 和 II，并在部分任务用资源增加换取时序收益。论文给出的实验支持这种工程取舍，但未给出统一的多目标优化公式。

## 9. 局限性

### 论文明确或正文可见的局限

论文未来工作提出扩展到多目标 power/area/performance、更多工具链和硬件平台，并提升跨设计泛化；这说明当前实验主要聚焦 FPGA 时序和面积，工具链集中在 Xilinx Vitis HLS/Vivado。论文也未报告正式训练、数据拆分或完整的推理成本。

### 阅读后发现的潜在局限

论文只有 6 页，部分 speedup、latency 表和 baseline 定义不够完整；Table III 的若干 latency 数值与“speedup”叙述难以独立复算。功能仿真通过只能说明测试覆盖范围内的行为一致，不能等同于形式化等价证明。目标器件没有 RISC-V 处理器或 RVV 后端，因此迁移到 RISC-V 需要新的硬件/编译器接口。

## 10. 阅读后的研究方向反思

值得借鉴的是把目标硬件元数据、编译器日志和性能报告作为模型上下文，并把最终输出限制在可由 HLS 工具执行的 pragma/代码接口。核心贡献是其 RAG 加两级反馈组合，不能只把 FPGA 替换成 RISC-V 就视为新方法。对 RISC-V 研究而言，论文更适合作为反馈闭环和实验组织方式的参考；若输出改为 RISC-V 编译器 flags、RVV intrinsic 或 LLVM pass 配置，还需要证明新动作空间、硬件计数器和验证方法带来独立问题。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V/RVV 的约束编译配置选择

#### 研究问题

如何让模型根据 RVV 向量长度、缓存和流水线约束选择 LLVM pass、vectorization 参数或后端配置。

#### 与原论文的区别

输出应限定为现有 LLVM/RISC-V 工具执行的配置或 pass 序列，不让模型直接改写完整程序。

#### 可能的创新点

把硬件计数器、LLVM remark 和可验证的编译配置动作统一进搜索空间。

#### 实验框架

```text
程序/LLVM IR + RISC-V 硬件元数据
        ↓
LLM 选择 pass/config
        ↓
LLVM/RVV 编译与仿真
        ↓
性能计数器和正确性反馈
```

#### 可行性

需要 LLVM、RISC-V 仿真器或真实板卡、CompilerGym 类环境和可重复 benchmark。

#### 主要风险

硬件测量噪声、配置空间过大和跨微架构泛化可能削弱结论。

### 11.2 HLS/RISC-V 跨层反馈的可验证动作接口

#### 研究问题

如何将模型输出限制为已注册、语义边界明确的指令或配置，并在每轮反馈后保留可审计轨迹。

#### 与原论文的区别

增加动作 schema、静态检查和语义验证，而不只把日志拼接到 prompt。

#### 可能的创新点

建立“候选配置—编译产物—验证结果—性能”的因果记录。

#### 实验框架

```text
候选动作 schema → 编译器执行 → 静态/差分验证 → 性能反馈 → 候选更新
```

#### 可行性

可从 LLVM pass 和 RVV 编译选项的小型动作集开始。

#### 主要风险

动作约束过严可能压低优化上限，过松则会产生不可复现输出。

## 12. 与其他已读文献的关系

本轮队列中，LIFT 和 MailoHLS 都把 LLM 用于 HLS pragma/directive 选择，分别侧重 GNN 结构监督和多适配器 Pareto 配置；TimelyHLS 则生成带 pragma 的 HLS 源代码并依赖 HLS/RTL 工具反馈，因此最终角色更接近 Translator。RALAD 也直接生成优化后的 HLS 程序，已在候选缓存中记录为 role mismatch。TimelyHLS 可作为 HLS 反馈闭环的系统参考，但与只输出配置的 LIFT/MailoHLS 不应合并为同一角色。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | FPGA HLS 的架构特定、时序感知代码优化 |
| 核心问题 | pragma 选择依赖人工且跨器件难迁移 |
| 输入 | HLS kernel、测试平台、FPGA 元数据和目标约束 |
| 输出 | 带 pragma 的 HLS C/C++ 代码 |
| 核心方法 | LLM + RAG + Vitis HLS/Vivado 两级反馈 |
| 使用的模型 | GPT-4、Claude-3.5-Sonnet；另举 Code LLaMA |
| 使用的编译器工具 | Vitis HLS 2024.2、Vivado 2024.2 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；使用 HLS/RTL 仿真和综合检查 |
| 数据集规模 | 10 个 HLS 应用、10 个 FPGA 目标；未报告标准拆分 |
| 主要指标 | speedup、latency、WNS/TNS、FF/LUT、Loop II、功能仿真 |
| 最重要实验结果 | Artix-7 最高约 4×；Matrix Multiplication II 16→1–2 |
| 核心创新 | 架构知识 RAG 与 HLS/RTL 反馈闭环结合 |
| 主要局限 | 实验规模和 baseline 细节有限，缺少消融与形式化等价证明 |
| 与 RISC-V 研究的相关性 | 中；反馈闭环可迁移，器件和工具接口不能直接迁移 |
| 最适合作为 | HLS 反馈系统参考、跨硬件优化 baseline |

这篇论文最值得学习的是把硬件知识和综合日志接入生成闭环；最主要的局限是实验与对照信息不足，且验证依赖有限的仿真测试。如果用于后续研究，合理做法是复用其反馈组织方式并为 RISC-V/LLVM 设计新的受限动作空间与验证协议，而不是只替换目标 FPGA。

## 元数据与分类核验

- Paper_ID：C222；Primary_Category：TRANSLATOR；Secondary_Category：T1_Source_Optimization_Refactoring。
- DOI：10.48550/arXiv.2507.17962；arXiv：2507.17962。
- Code_Status：NOT_FOUND；Code_URL：空；Code_Checked_At：2026-10-01。
