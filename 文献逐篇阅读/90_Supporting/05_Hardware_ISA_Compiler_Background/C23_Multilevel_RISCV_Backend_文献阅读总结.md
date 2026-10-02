# Multi-level RISC-V Backend 文献阅读总结
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

论文题目：**A Multi-level Compiler Backend for Accelerated Micro-kernels Targeting RISC-V ISA Extensions**

作者：Alexandre Lopoukhine、Federico Ficarelli、Christos Vasiladiotis、Anton Lydike、Josse Van Delm、Alban Dutilleul、Luca Benini、Marian Verhelst、Tobias Grosser

发表时间：2025

发表平台：CGO 2025；DOI: 10.1145/3696443.3708952；arXiv:2502.04063
元数据核验来源：[ACM DOI](https://doi.org/10.1145/3696443.3708952)；[作者公开论文 PDF（含 Artifact Appendix）](https://www.pure.ed.ac.uk/ws/portalfiles/portal/558506860/LopoukhineEtalCGO25AMultiLevelCompilerBackendforAcceleratedMicrokernels.pdf)

关键词：RISC-V、MLIR、xDSL、多层编译后端、渐进 lowering、Snitch、寄存器分配、微内核

> 本文档基于 16 页 CGO 2025 出版 PDF（含工件附录）整理。论文以周期精确 RTL 仿真评测 Snitch，不应误写为量产 RISC-V 真机结果。

## 1. 研究背景

通用编译器通常采用“多前端—窄中层 IR—通用后端”的沙漏结构。LLVM IR 适合 load/store、算术、分支和常规 SIMD，但会丢失高层领域信息；MLIR/TVM/OpenXLA 等虽能在高层逐级优化 DNN，最终常把代码交给通用 LLVM 后端。对于流式寄存器、硬件循环等定制 ISA 扩展，低层后端很难重建高层访问模式和循环保证，于是高性能微内核往往绕过编译器，转向手写汇编或专用代码生成器（第 1–2 节）。

论文以 Snitch 为实例。Snitch 是开源、顺序执行的 RISC-V 核，拥有 Stream Semantic Registers（SSR，流语义寄存器）和 Floating-Point Repetition（FREP，浮点指令重复）扩展：SSR 隐式完成仿射内存访问，FREP 消除循环控制，使 FPU 可持续工作。传统 LLVM RISC-V 后端不能有效表达这些语义。

## 2. 论文要解决的问题

### 2.1 如何打破后端的语义瓶颈

后端需要保留 Linalg 的循环、访问映射和归约语义，同时逐渐引入 RISC-V/Snitch 约束，而不是一次降到扁平 LLVM IR 后再做困难的模式恢复。

### 2.2 如何在结构化后端中完成寄存器分配

目标是利用 SSA 与 region 的显式 use-def、作用域和循环结构，以简单多阶段算法完成寄存器分配，并避免微内核中代价高的 spill。

### 2.3 高层 DSL 能否自动产生接近峰值的目标代码

论文分别检验低层 dialect 表达力、spill-free allocator 的适用性，以及从 Linalg 自动 lowering 后是否仍能达到高 FPU 利用率。

> 本文主要研究：如何用多层 SSA dialect 扩宽 RISC-V 定制加速器后端，使领域语义逐级降到 Snitch ISA 扩展，并自动生成接近手写微内核性能的代码。

## 3. 核心方法概述

```text
MLIR linalg.generic 微内核
        ↓
高层调度：scalar replacement、fill fusion、unroll-and-jam
        ↓
memref_stream：显式迭代边界、访问模式与流区域
        ↓
scf / func / memref 等通用结构化 dialect
        ↓
rv_scf / rv_func：保留循环与 ABI 语义
        ↓
snitch_stream / rv_snitch：SSR、FREP、packed SIMD
        ↓
三阶段、基于 SSA region 的 spill-free 寄存器分配
        ↓
rv / rv_cf：贴近 RISC-V 指令与非结构化控制流
        ↓
顺序打印汇编 → bare-metal ELF → Snitch RTL 周期精确仿真
```

核心思想是让目标硬件能力在多个合适抽象层都有对应 operation/dialect。`memref_stream` 在仍可看到 Linalg 访问映射时确定调度并分离 access/execute；低层 `rv` 把指令源/目的寄存器建模为 SSA operand/result；`rv_scf`、`rv_func` 和 Snitch region 保留循环、ABI 与流寄存器作用域（第 3 节、图 5–7）。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用确定性的编译 pass、寄存器分配和代码生成。

### 4.1 多层目标表示

`rv`/`rv_cf` 表示 RISC-V 指令与跳转；较高层的 `rv_func` 和 `rv_scf` 表示调用约定与结构化循环。`rv_snitch` 表示 packed SIMD 和 FREP，`snitch_stream.streaming_region` 显式圈定 SSR 配置生效范围。编译期常量形式的 stride/bound 让编译器用简单 rewrite 消除连续访问配置和重复访问压力（第 3.1–3.2 节）。

### 4.2 三阶段寄存器分配

第一遍从 RISC-V ABI 的 15 个 caller-saved 整数寄存器和 20 个浮点寄存器中排除 IR 已占用者；第二遍记录在 region 外定义、在 region 内使用的值；第三遍逆向遍历 SSA，在首次逆向遇到 use 时分配、到 definition 时释放。循环结果、block argument 和 loop-carried 值先统一寄存器，再递归处理嵌套 region。Snitch streaming 保留寄存器通过 operation 定义声明（第 3.3 节）。

### 4.3 渐进 lowering 与调度

`memref_stream` 把 Linalg 的访问与计算分离，先确定 Snitch 可执行的访问顺序。归约结果在寄存器中累积；unroll-and-jam 交错多个内层迭代，增加代码和寄存器压力以隐藏三阶段 FPU pipeline 的 RAW 延迟。论文按 pipeline depth 自动选择至少 4 的展开因子，而不是进行昂贵的 schedule space 搜索（第 3.4 节）。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习，因此不存在强化学习奖励函数。

实验核心指标为：

```text
Throughput = kernel FLOPs / execution cycles
FPU utilization = 执行算术指令的 FPU cycles / kernel 总执行 cycles
```

FMA 计为 2 FLOPs。理论最小周期由循环迭代数、loop body FLOPs 和峰值吞吐量估计；周期计数包含函数调用、整数算术、显式 load/store 和加速器设置开销（第 4.1 节）。这不是学习代价模型，也不是形式化性能上界证明。

## 6. 实验设置

### 6.1 数据集来源

作者从 NSNet2 和 AlexNet 选择代表性 DNN 微内核：Sum、Fill、ReLU、3×3 Conv、3×3 Max Pool、3×3 Sum Pool、MatMul、MatMulT，覆盖线性/非线性/稀疏访问、归约、嵌套循环和并行模式。Softmax 与 Sigmoid 依赖指数/对数函数，不在范围内。输入 shape 被选择为可放入 128 KiB TCDM，避免其余内存层级影响（表 1、第 4.1 节）。

工件使用随机输入和预计算输出校验 kernel，正文未给出随机样本条目数或独立训练/测试划分，因为本文不是数据驱动训练任务。

### 6.2 模型与工具

后端使用 xDSL 0.21.1 实现，借助文本 IR 与 MLIR 16.0.6 互操作；工件清单记录 xDSL 0.23.0、MLIR 16.0.6 和基于 LLVM 12.0.1 的 PULP LLVM 0.12.0。目标是 Snitch 开源 SystemVerilog，在 Verilator 上生成周期精确 RTL simulator；运行 bare-metal、无 OS，因此测量确定性。工件 DOI 为 10.5281/zenodo.14052014，约需 46 GB，完整复现于 Ryzen 9 5950X/62 GiB 主机约 1 小时 45 分钟。

### 6.3 对比方法

- Ours low-level：手工以论文的 RISC-V/Snitch dialect 表示，用于检验低层表达力。
- Ours end-to-end：从 Linalg 经完整渐进 lowering 自动生成。
- MLIR：现有 MLIR pass 降到 LLVM dialect/IR，再用 LLVM RISC-V 后端。
- Clang：朴素 C 实现经同一 LLVM RISC-V 后端。
- 最佳手写汇编：用于低层表示结果的参照。

MLIR/Clang 不支持 Snitch 扩展，因此论文明确说它们用于讨论，不是完全同能力的公平 baseline。

### 6.4 评价指标

CPU cycle count、FLOPs/cycle throughput、FPU utilization、整数/浮点寄存器分配数、load/store/FMA/FREP 指令数。周期来自 Snitch performance counter 与 trace 后处理，而非墙钟时间。

## 7. 实验结果与结论

### 7.1 RQ1：低层表示能力

低层 RISC-V/Snitch dialect 能表达选定微内核，并与最佳手写汇编匹配。Sum 与 ReLU 达到 95% FPU utilization；MatMulT 为 74%，吞吐为 2.45 FLOPs/cycle，受额外 vector packing 指令影响（第 4.2 节、图 9）。论文概括最高达到理论吞吐的 94%。

### 7.2 RQ2：spill-free 寄存器分配

所有测试 kernel 均无需 spill。64-bit kernel 最多使用 8/20 个 FP 和 8/15 个整数寄存器；压力最大的 32-bit MatMulT 使用 11/20 FP 与 12/15 整数寄存器。作者承认更深循环和复杂 kernel 可能暴露保守排除 ABI 寄存器的限制（表 2、第 4.3 节）。

### 7.3 RQ3：高层到后端的端到端效果

Clang 与通用 MLIR 流程共享不理解 Snitch 扩展的 LLVM RISC-V 后端，峰值约 42% FPU utilization，多数更低。本文端到端 compiler 在不同 kernel 上达到 73%–90%，并随输入变大逐渐摊薄固定设置开销（图 10–11）。

### 7.4 优化消融

对 `1×200 · 200×5` 的 64-bit MatMul，baseline 为 40,161 cycles、2.49% occupancy、3,000 loads/1,005 stores。依次加入 Streams、Scalar Replacement、FREP、Fuse Fill、Unroll-and-Jam 后，最终为 1,115 cycles、90.67% occupancy、0 loads/0 stores；FP/整数寄存器为 8/20 与 7/15。Streams 约减半周期并把显式 load 减少 66%；Scalar Replacement 再带来超过 4 倍改善；单独 FREP 在该阶段收益小；最后 unroll-and-jam 通过交错 5 个结果消除 RAW pipeline stall（表 3、第 4.4 节）。

## 8. 主要创新点

### 8.1 “宽后端”的多层 SSA 设计

论文不是新增一个孤立 RISC-V pass，而是让领域、流访问、结构化控制、ABI、扩展 ISA 和具体指令分别在合适 dialect 中存在，缓解 LLVM IR 窄接口造成的语义丢失。

### 8.2 结构化、可扩展的 RISC-V/Snitch IR

用 SSA value、region 和 custom operation 表达标准 RISC-V、SSR、FREP 与 packed SIMD，使普通 use-def/作用域分析可直接用于目标后端，并让扩展 ISA 以模块方式加入。

### 8.3 基于 region 的多阶段 spill-free allocator

利用微内核结构化控制流的输入约束，放弃通用程序所需的复杂 CFG 重建和 spill heuristic，以三个线性阶段完成嵌套循环寄存器分配；表 2 给出无 spill 的直接证据。

### 8.4 从 Linalg 直接映射目标能力的渐进 lowering

`memref_stream` 在高层信息仍存在时确定 stream、FREP、scalar replacement 和 unroll-and-jam，最终自动生成 73%–90% FPU utilization 的代码；表 3 清楚分离各 pass 贡献。

## 9. 局限性

### 9.1 论文明确承认的局限

系统只针对结构化线性代数微内核，非结构化控制流不在 allocator 范围；完全不支持 spill，复杂 kernel、深循环或更高寄存器压力可能失败。当前仅评估 Snitch 及其 SSR/FREP/非标准 packed SIMD，Softmax/Sigmoid 和通用 exp/log 实现未覆盖。MLIR/Clang 对比不支持同一扩展能力，因此不是严格等价 baseline。

### 9.2 阅读后发现的潜在局限

实验是周期精确 RTL simulator，不是硅后 RISC-V 芯片；输入刻意放入 TCDM，未评估多核调度、缓存/外存、OS 和端到端 DNN。高性能依赖 Snitch 顺序核、软件管理 L1、固定三阶段 FPU 等特性，迁移到 RVV 或乱序核不能直接复用固定 schedule。正确性依赖 compiler implementation 与随机/预计算输出测试，没有形式化 translation validation。46 GB 工件和特定工具链版本提高复现成本。

## 10. 阅读后的研究方向反思

这篇论文对 RISC-V 研究的关键启发是：目标扩展语义应该尽早进入 IR，而不是期待 LLVM IR 降低后由后端恢复。它适合作为“多层架构特化后端”基线。只把 Snitch 指令替换为 RVV intrinsic 不足以形成创新；RVV 的向量长度无关语义、LMUL/SEW、mask/tail policy 与 `vsetvli` 状态需要不同的多层表示、合法化和代价模型。

论文的 spill-free allocator、`memref_stream` 和固定调度是针对可预测微内核的核心贡献，不能不加区别地照搬到通用程序。更有价值的后续问题是：如何让通用 LLVM/MLIR 后端在保留领域信息的同时，按 kernel 特征选择“结构化专用路径”或“通用 fallback”，并用真机证据决定边界。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RVV 的多层可伸缩向量后端

#### 研究问题

如何在 MLIR 中跨 Linalg、Vector、RVV 配置状态和机器指令层保留可伸缩向量语义，并自动选择 LMUL/SEW/tail/mask 与 `vsetvli` 放置。

#### 与原论文的区别

原文面向固定 Snitch SSR/FREP 和可预测 schedule；新方案面向 VLEN-agnostic RVV、动态 VL 和配置切换成本。

#### 可能的创新点

状态化 RVV dialect、配置区域、跨层合法性证明、VLEN 鲁棒代价模型、真机 PMU 校准。

#### 实验框架

```text
Linalg/Vector kernel → RVV 状态化多层 IR
→ 候选 lowering/配置区域 → LLVM/RVV 汇编
→ QEMU 语义差分 → 多 VLEN 真机 PMU
→ 选择或校准配置 → 与 LLVM baseline 比较
```

#### 可行性与主要风险

可从 MatMul/Conv/Reduction 十余个 kernel 开始；风险是 scalable vector 语义、`vsetvli` 全局状态和后端寄存器压力相互耦合。

### 11.2 结构化专用后端与通用 fallback 的边界学习

#### 研究问题

何时应进入 spill-free 结构化微内核路径，何时应回退到 LLVM 通用寄存器分配和后端优化。

#### 与原论文的区别

原文预设输入都适合专用路径；新方案把适用边界、失败反例和收益判定作为研究对象。

#### 可能的创新点

静态可行性证书、寄存器压力预测、反例驱动边界更新、可审计 fallback 原因。

#### 实验框架

```text
MLIR kernel → 结构/压力特征
→ 专用后端可行性检查
→ 专用与通用两路编译
→ 真机正确性/性能比较
→ 更新边界模型与失败规则
```

#### 可行性与主要风险

可复用公开工件并逐步增加深循环、分支和跨函数 kernel；风险是双路工具链维护和训练数据偏向易例。

## 12. 与其他已读文献的关系

与 C20 的指令选择后端综合互补：C20 从 RISC-V SAIL/gMIR 语义自动生成 GlobalISel 规则，主攻低层指令选择；本文手写多层 dialect 与渐进 lowering，主攻把高层领域语义保留到定制扩展。与 C21 OML-vect 相比，C21 激活既有 MLIR/LLVM 向量器并生成 RVV，本文则绕过窄 LLVM 后端、直接为 Snitch 扩展建后端。与 C22 MLIR RL 相比，C22 学习高层循环变换但依赖通用 CPU lowering；本文说明若后端丢失目标语义，高层选择仍可能受最终代码生成瓶颈限制。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 为 RISC-V 定制加速器构建保留领域语义的多层编译后端 |
| 核心问题 | 通用 LLVM 后端形成语义瓶颈，难以利用 SSR/FREP 等扩展 |
| 输入 | MLIR Linalg 线性代数微内核 |
| 输出 | Snitch RISC-V 扩展汇编与 bare-metal ELF |
| 核心方法 | 多层 SSA dialect、memref_stream、渐进 lowering、spill-free allocator |
| 使用的模型 | 无机器学习模型 |
| 使用的编译器工具 | xDSL、MLIR 16、PULP LLVM、Verilator |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；用随机输入与预计算输出测试正确性 |
| 数据集规模 | 8 类 DNN 微内核，多组可放入 TCDM 的 shape；具体样本总数未明确说明 |
| 主要指标 | cycles、FLOPs/cycle、FPU utilization、寄存器与指令计数 |
| 最重要实验结果 | 低层最高 95% FPU utilization；端到端 73%–90%；MatMul 2.49%→90.67% |
| 核心创新 | 把 RISC-V/加速器后端扩宽为多个保留结构和领域语义的 SSA 层 |
| 主要局限 | Snitch/微内核专用、RTL 仿真、无 spill/非结构化控制流、无外存/端到端 DNN |
| 与 RISC-V 研究的相关性 | 很高；直接实现 RISC-V 与定制 ISA 扩展后端 |
| 最适合作为 | 多层 RISC-V 后端、架构特化 lowering 与寄存器分配基线 |

> 这篇论文最值得学习的是让硬件语义在合适抽象层持续存在，并用逐 pass 消融说明性能来自何处；最主要的局限是对 Snitch 结构化微内核的强假设。后续最合理的使用方式是把它作为 RVV 多层后端和专用/通用边界研究的基础，而不是把固定 Snitch 调度直接套到所有 RISC-V 处理器。
