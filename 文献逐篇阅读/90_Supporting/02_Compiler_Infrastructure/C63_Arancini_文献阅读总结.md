# Arancini 文献阅读总结
元数据核验来源：[ACM DOI](https://doi.org/10.1145/3779212.3790127)；[TU Delft 作者机构典藏](https://repository.tudelft.nl/record/uuid:f1956019-43ad-4101-85cc-f3b36e2c9dfc)
代码/数据/工件：作者 artifact：[Arancini Zenodo](https://zenodo.org/records/18257168)；[作者团队仓库](https://github.com/binary-translation/arancini-exploration)

论文题目：**Arancini: A Hybrid Binary Translator for Weak Memory Model Architectures**

作者：Sebastian Reimers, Dennis Sprokholt, Martin Fink, Theofilos Augoustis, Simon Kammermeier, Rodrigo C. O. Rocha, Tom Spink, Redha Gouicem, Soham Chakraborty, Pramod Bhatotia

发表时间：2026；发表平台：ASPLOS 2026，Volume 2，pp. 157–174（18 页）

论文链接或编号：DOI 10.1145/3779212.3790127

关键词：binary translation、weak memory model、RISC-V、LLVM、formal verification

## 1. 研究背景

异构硬件使 x86-64 到 Arm/RISC-V 的二进制迁移更常见。静态二进制翻译通常覆盖不完整，动态翻译则有运行时开销；更关键的是强内存模型 guest 到弱内存模型 host 的并发语义可能被破坏（正文第 1、2 节）。

## 2. 论文要解决的问题

### 2.1 完整性与性能

系统需要同时支持提前翻译和遇到未知代码时的运行时翻译。

### 2.2 并发正确性

需要为 x86-64 到 Arm/RISC-V 的内存访问建立可证明的映射，并处理 mixed-size accesses。

> 本文主要研究：如何构建一个兼顾正确性、完整性和性能的强到弱内存模型混合二进制翻译器。

## 3. 核心方法概述

Arancini 共享一个中间表示 ArancinIR；静态路径经 LLVM 优化生成混合二进制，动态路径用轻量后端生成代码；内存指令映射由形式化模型和已验证规则约束（第 3、5、6 节）。

```text
guest binary
  ↓ lifting + verified memory mappings
ArancinIR
  ↓ static path: LLVM IR + LLVM optimizations
hybrid host binary
  ↓ unknown PC at runtime
dynamic lifting → ArancinIR → lightweight host backend → code cache
```

本文不涉及 LLM、SFT 或强化学习。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用静态翻译、动态翻译和形式化验证的系统执行流程。ArancinIR 将寄存器、内存和控制状态表示为 value/action nodes；静态后端使用 LLVM，动态后端避免重量级 LLVM JIT。实验比较原生执行、QEMU 系统和 Arancini 的静态/动态覆盖与性能（第 7 节）。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数。关键正确性目标是：源架构允许的并发行为在目标架构上不被错误扩大；论文用 axiomatic memory model、mapping schemes 和 Isabelle/HOL 风格证明论证该性质（第 5.1–5.5 节）。

## 6. 实验设置

### 6.1 数据集来源

使用多线程 benchmark suite，覆盖 pthread 和 lock-free/atomic 相关程序；正文还评估静态翻译覆盖、动态触发及不同线程数的扩展性。具体 benchmark 总数和每个程序的完整配置以论文表格/Artifact 为准，当前 PDF 内容不足以确认一个统一总样本数。

### 6.2 模型与工具

目标后端包括 Arm 和 RISC-V；静态路径使用 LLVM，动态路径使用自研轻量 backend，形式化部分建模 x86-64、Arm 和 RISC-V 内存模型。论文不使用机器学习模型。

### 6.3 对比方法

主要与 QEMU-based binary translators、原生编译二进制及禁用 dead-flag/fence-merge 优化的变体比较（图 8–11）。

### 6.4 评价指标

| 指标 | 含义 |
|---|---|
| Coverage | 可成功翻译/执行的 guest 代码覆盖 |
| Relative performance | 相对原生程序的执行性能 |
| Translation distribution | 静态与动态翻译代码的执行占比 |
| Scaling | 线程数增加时的性能变化 |

## 7. 实验结果与结论

### 7.1 主要结果

论文报告 Arancini 在 Arm/RISC-V 后端上可保持正确性与完整性，并在多线程 benchmark 上最高达到约 5× QEMU-based translator 的性能；该数字是相对 QEMU 的系统结果，不是相对原生执行的加速（摘要、第 7 节）。

### 7.2 传统方法比较

混合设计让常见路径由静态翻译承担，未覆盖路径才进入动态翻译；图 8、9 显示静态代码占主要执行部分时性能更接近原生路径。

### 7.3 消融实验

图 10 比较 dead-flag 与 fence-merge 优化的开关，说明减少状态更新和合并 fence 会影响翻译后运行时间。

### 7.4 案例分析

形式化映射覆盖 load/store、fence 和 atomic-update；论文特别讨论 mixed-size concurrency 以及 x86-64 到 Arm/RISC-V 的强到弱映射。

## 8. 主要创新点

### 8.1 统一静态/动态翻译的 ArancinIR

同一 IR 同时服务静态 LLVM 路径和动态低延迟路径，减少两套翻译语义不一致的风险。

### 8.2 形式化指导的跨 ISA 内存映射

论文把内存模型形式化并验证映射规则；价值在于把并发语义约束放到翻译设计中，而不是只做事后测试。

### 8.3 混合二进制翻译系统

静态性能、动态完整性和验证规则被整合到端到端系统；这不是单纯替换目标 ISA。

## 9. 局限性

### 9.1 论文明确承认或体现的局限

系统重点是用户态二进制和弱内存模型映射；目标架构、操作系统边界及更多 ISA 的适用性并未由本文实验全面证明。

### 9.2 阅读后发现的潜在局限

形式化证明覆盖映射规则，不等于整个 LLVM、运行时库和实现都被形式化验证；性能结果依赖 benchmark 与特定硬件，不能直接外推到所有 RISC-V 实现。

## 10. 阅读后的研究方向反思

最值得借鉴的是“共享 IR + 可证明边界 + 静/动态协同”。对 RISC-V 研究而言，直接替换为另一 RV 扩展不足以构成创新；更有价值的是把 RVV、混合宽度原子访问或定制内存扩展纳入可验证映射。本文更适合作为跨 ISA 翻译基础设施和正确性 baseline，而非 LLM 优化框架。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RVV 的可证明二进制翻译

#### 研究问题
如何将向量寄存器状态与内存排序规则统一映射到 ArancinIR。

#### 与原论文的区别
扩展对象是 RVV 数据并行语义，不只是标量 x86-64 映射。

#### 可能的创新点
向量长度无关语义、别名与异常行为的验证规则。

#### 实验框架
guest binary → lifting → RVV-aware IR → verified lowering → Spike/真实硬件对比。

#### 可行性
需要 LLVM/RVV、模拟器和 litmus tests。

#### 主要风险
向量内存和异常语义的形式化成本高。

## 12. 与其他已读文献的关系

与 C56 T2T 都涉及跨硬件/跨 ISA，但 Arancini 翻译已编译二进制并关注内存模型正确性，T2T 面向 CUDA 到 NPU 的源/中间层转换与向量化。与 C61 Cpp2Rust 都是传统翻译器基础设施，但前者关注机器码语义，后者关注源语言内存安全。Arancini 适合作为跨 ISA 正确性与运行时系统 baseline。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 强到弱内存模型的混合二进制翻译 |
| 核心问题 | 正确性、完整性、性能三者兼顾 |
| 输入 | x86-64 guest binary |
| 输出 | Arm/RISC-V host execution |
| 核心方法 | ArancinIR、静态 LLVM、动态 backend、验证映射 |
| 使用的模型 | 无机器学习模型 |
| 使用的编译器工具 | LLVM、自研翻译器 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 是，映射与内存模型 |
| 主要指标 | 覆盖、相对性能、线程扩展 |
| 最重要实验结果 | 最高约 5× QEMU-based translator |
| 核心创新 | 共享 IR 与形式化指导的混合翻译 |
| 主要局限 | 架构与实现边界有限 |
| 与 RISC-V 研究的相关性 | 高；直接包含 RISC-V 后端 |
| 最适合作为 | 跨 ISA 基础设施与正确性 baseline |

> 这篇论文最值得学习的是把跨 ISA 翻译的性能机制和内存模型证明放进同一系统；最主要的局限是证明边界不等于全系统证明；后续应将其作为 RISC-V/RVV 翻译基础，而不是简单更换目标平台。
