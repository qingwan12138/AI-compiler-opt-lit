# Towards Autonomous Accelerator Design 文献阅读总结

论文题目：**Towards Autonomous Accelerator Design: FPGA Accelerator Generation with SECDA**
作者：Vinamra Sharma、Xingjian Fu、Jude Haris、José Cano
发表时间：2026
发表平台：Machine Learning for Architecture and Systems Workshop（MLArchSys），与 ISCA 2026 联合举办；正文页眉为 arXiv:2606.11117v1。
论文链接或编号：[arXiv:2606.11117](https://arxiv.org/abs/2606.11117)；DOI：未找到。
版本族：SECDA-DSE 前作 arXiv:2605.05920；本稿明确称其为扩展评估版本，采用本稿作为唯一候选正文。
PDF：`Towards_Autonomous_SECDA_2606.11117.pdf`，6 页，`%PDF-` 签名有效，pypdf 可解析。

> 本文档用于候选阶段复核。论文事实来自本地 PDF 正文；分类和后续方向属于阅读后的分析。

## 1. 研究背景

FPGA 加速器设计需要同时决定计算并行度、分块、存储层次、数据流和片上资源。传统 DSE 通过穷举、启发式、贝叶斯优化或遗传算法搜索，但设计组合多且每次 HLS、综合和硬件执行成本高。SECDA 提供 SystemC 模板、仿真和 FPGA 执行流，降低了硬件开发门槛，但配置探索仍依赖专家。本文研究把 LLM 的推理和检索能力放入 SECDA-DSE 闭环，并用真实 FPGA 执行验证生成结果。

## 2. 论文要解决的问题

### 2.1 配置空间过大

需要在 workload、FPGA 设备和架构约束下选择合法的计算、缓冲、数据流与 DMA 配置。

### 2.2 生成结果必须可综合、可执行

自由生成硬件代码容易产生不符合 SECDA 模板、无法 HLS 或无法下板执行的结果，因此需要模板约束和逐级反馈。

> 本文主要研究：如何利用 LLM 引导 SECDA 中的 FPGA 加速器设计生成与 DSE，并通过 HLS、逻辑综合和真实 FPGA 执行验证设计。

## 3. 核心方法概述

SECDA-DSE 由结构化 DSE Explorer、LLM Stack 和 Evaluation Module 组成。输入是自然语言 workload、目标 FPGA 和架构 directives；输出是 SECDA 原生 SystemC 加速器设计及其集成文件，而不是仅一个 pragma 列表。

```text
workload + FPGA + 架构 directives
        ↓
RAG 检索 SECDA 代码、模板和历史硬件 datapoints
        ↓
LLM 通过 CoT/LoRA 生成或修正候选架构与 SystemC 文件
        ↓
SECDA SystemC 仿真 → Vivado HLS → 逻辑综合 → FPGA 执行
        ↓
延迟、资源、DMA、执行正确性与失败原因
        ↓
写入 hardware datapoints，作为后续 refinement 输入
```

## 4. 实验框架与训练流程

### 4.1 检索与提示

RAG 知识库包含 SECDA-TFLite 源码、架构模板、API 上下文和历史硬件 datapoints。图式检索使用代码注释的模糊匹配缩小上下文。

### 4.2 参数高效适配

论文使用 LoRA。数据由历史配置、仿真/执行成功状态、延迟和资源利用率组成，并由人工参与保证正确性。正文没有给出完整训练集规模。

### 4.3 生成与评估循环

LLM 生成 SECDA 原生设计；候选先做 SystemC/HLS 验证，成功后进入逻辑综合和 FPGA 执行。失败配置作为负 datapoint，用于下一轮修正。当前流程仍有人参与文件传递、路径检查和脚本执行，但不手工修改生成的加速器逻辑。

## 5. 奖励函数、损失函数或关键公式

本文没有使用 PPO、GRPO 等强化学习奖励函数。论文把失败结果称为 negative reinforcement，但实现上是反馈 datapoint 和 LoRA/refinement 数据，不是形式化 RL 奖励。搜索关注延迟、资源利用率、DMA 和执行成功状态；正文未给出统一的显式损失函数。

## 6. 实验设置

### 6.1 数据集来源

评估 workload 为 element-wise vector multiplication、2D convolution 和 matrix transpose。LoRA 初始 datapoints 来自 matrix addition 和 matrix multiplication。不是公开标准数据集，且规模未明确报告。

### 6.2 模型与工具

- 模型：TinyLlama，约 1.1B 参数，通过 Ollama 本地运行。
- 工具：SECDA、SystemC 仿真、Vivado HLS 2019.2、逻辑综合流程。
- 硬件：Xilinx Zynq-7000，器件型号 `xc7z020-clg400-1`。

### 6.3 对比方法

正文主要比较 SECDA-DSE 生成的三个 workload 设计及其执行/资源结果；没有提供完整的统一 DSE baseline 数值表，也没有与 LLM4HLS 等方法做同条件比较。

### 6.4 评价指标

| 指标 | 含义 |
|---|---|
| Validation | FPGA 输出与 CPU 参考结果逐元素比较 |
| Latency | FPGA 执行延迟，单位 ms |
| HWC cycles | 数据加载、计算、写回三个硬件计数器周期 |
| DMA metrics | 收发大小、速度和等待时间 |
| Resource utilization | BRAM、DSP、LUT、FF 利用率 |

## 7. 实验结果与结论

### 7.1 主要结果

三个生成设计均通过 FPGA 功能验证。正文表 I 报告：2D Conv 延迟 163 ms、VMUL 135 ms、Transpose 238 ms；对应 DSP 利用率分别为 1.36%、21.82%、5.45%，LUT 利用率分别为 6.64%、8.23%、8.85%。

### 7.2 迭代过程

VMUL 经过 4 个 refinement cycle 才得到可执行设计；2D Conv 一轮成功；Transpose 需要 9 轮，其中前两轮通过 HLS，之后还需 7 轮完成逻辑综合和 FPGA 执行。

### 7.3 结果解释

VMUL 偏计算密集，因此 DSP 利用率最高；Transpose 偏数据移动，DMA 规模和 LUT 利用率更高。结果支持 workload-specific configuration 的可行性，但不等于达到性能最优。

### 7.4 消融实验

正文没有提供独立的 RAG、CoT、LoRA 或反馈闭环消融表。

### 7.5 失败案例

论文记录了迭代次数和 HLS/综合失败后的修正过程，但没有给出所有失败候选的完整日志。

## 8. 主要创新点

### 8.1 SECDA 约束下的 LLM 引导 DSE

创新不只是使用 LLM，而是把检索上下文、架构模板、硬件反馈和候选生成接入已有 SECDA 流程。

### 8.2 真实 FPGA 端到端验证

相较前作只验证单个仿真设计，本稿验证三个 workload，并完成 HLS、综合、FPGA 执行。

### 8.3 面向 workload 的反馈 refinement

系统把延迟、DMA、资源和执行失败原因存成 datapoint，使后续候选能针对计算、存储或数据移动瓶颈修正。

## 9. 局限性

### 论文明确承认的局限

- 仍有人参与评估流程。
- 只有三个 workload 和一个 FPGA 平台。
- 需要较丰富的初始 hardware datapoints。
- 当前重点是生成可执行设计，不是全面证明性能最优。

### 阅读后的潜在局限

最终 LM 输出是 SystemC/SECDA 设计与集成文件，实际角色不是纯 Selector 的 pragma/config 选择，而更接近硬件后端组件生成。因此建议分类为 `GENERATOR / G3_Backend_Compiler_Component_Generation`，置信度 MEDIUM，Needs_Review=YES。

## 10. 阅读后的研究方向反思

值得借鉴的是“LM 提议—确定性工具验证—结构化反馈”的边界。不能把 SECDA 模板约束下的硬件设计生成简单改名为 HLS pragma Selector；若迁移到 RISC-V，必须重新定义合法配置、后端计数器和跨微架构反馈，而不是只替换 FPGA 名称。

## 11. 可进一步尝试的研究方向

### 11.1 RISC-V/RVV 配置生成器

研究问题：让 LM 在固定合法动作空间中生成 RVV 向量长度、tile、unroll 和内存布局配置。与本文区别是输出配置/动作而非 SystemC 设计，编译器执行 LLVM/MLIR 变换。风险是硬件反馈和合法性约束复杂。

### 11.2 失败边界记忆

研究问题：把编译失败、资源超限和真实性能退化分开编码为可检索证据。与本文区别是显式建模失败类型，风险是跨平台迁移时 datapoint 分布变化。

## 12. 与其他已读文献的关系

与本轮 LUMINA、Beacon 都使用 LLM 引导硬件 DSE，但本文输出完整 SystemC/SECDA accelerator design；LUMINA 输出 GPU 架构动作，Beacon 输出合法硬件候选。与正式库中的 LIFT、LLM4HLS、iDSE、ChatHLS 相比，本文更靠近硬件设计生成而非源码 pragma 选择。适合作为 Generator/硬件生成对照，不宜作为纯 Selector baseline。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | SECDA 中的 LLM 引导 FPGA 加速器设计生成与 DSE |
| 核心问题 | 大型硬件配置空间与可综合/可执行性 |
| 输入 | workload、FPGA、架构 directives、SECDA 上下文 |
| 输出 | SystemC 加速器设计和 SECDA 集成文件 |
| 核心方法 | RAG、CoT、LoRA、HLS/FPGA 反馈闭环 |
| 使用的模型 | TinyLlama 1.1B |
| 使用的工具 | SECDA、Vivado HLS 2019.2、Ollama |
| 是否使用强化学习 | 否；仅有反馈式 refinement |
| 是否使用形式化验证 | 否；使用仿真、HLS、综合和逐元素功能验证 |
| 数据集规模 | 论文未明确给出 |
| 最重要实验结果 | 三个 workload 均通过 FPGA 验证 |
| 核心创新 | 结构化 SECDA 模板与 LLM 反馈生成结合 |
| 主要局限 | 单平台、小 workload、仍有人参与 |
| 与 RISC-V 研究的相关性 | 中；可借鉴反馈闭环，但不是 RISC-V 论文 |
| 最适合作为 | GENERATOR/G3 对照与硬件反馈工具参考 |

这篇论文最值得学习的是让 LLM 的生成结果始终经过 SECDA/HLS/FPGA 验证；最主要的局限是输出已经越过纯配置选择，且评估规模有限；后续应把其反馈契约迁移为明确的 RVV 配置 Selector，而不是简单替换目标平台。
