# C71 An MLIR Lowering Pipeline for Stencils at Wafer-Scale 文献阅读总结

论文题目：**An MLIR Lowering Pipeline for Stencils at Wafer-Scale**

作者：Nicolai Stawinoga、David Katz、Anton Lydike、Justs Zarins、Nick Brown、George Bisbas、Tobias Grosser

发表时间：2026

发表平台：ASPLOS 2026，Proceedings of the 31st ACM International Conference on Architectural Support for Programming Languages and Operating Systems，Volume 2，pp. 94–109，16 页
元数据核验来源：[ASPLOS 2026 ACM DOI](https://doi.org/10.1145/3779212.3790124)；[arXiv:2601.17754](https://arxiv.org/abs/2601.17754)

论文链接或编号：DOI [10.1145/3779212.3790124](https://doi.org/10.1145/3779212.3790124)；arXiv [2601.17754](https://arxiv.org/abs/2601.17754)
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

关键词：Cerebras WSE、MLIR、xDSL、stencil、领域特定语言、异步 actor 执行、代码生成、硬件感知编译

> 本文档依据论文正文整理。论文事实、阅读后的分析和后续建议分开描述。正文来源为 arXiv v1 的 16 页 PDF；PDF 首页同时给出 ASPLOS ’26、DOI 和正式论文格式信息。

## 1. 研究背景

Cerebras Wafer-Scale Engine（WSE）把大量处理单元放在一整片晶圆上。论文第 1—2 节介绍，WSE3 包含超过 900,000 个相互连接的 Processing Elements（PE），每个 PE 有独立计算核心、路由器、程序计数器、时钟和 48 KB SRAM；PE 之间通过邻居通信和 32 位 wavelet 传递消息。它最初面向 AI，也适合一部分高性能计算（HPC）工作负载，但其无缓存、分布式本地内存和异步执行模型与 CPU/GPU 上常见的顺序或批同步程序差异很大。

WSE 的官方 Cerebras Software Language（CSL）直接暴露 task、wavelet、color、Data Structure Descriptor（DSD）和布局文件等硬件概念。一个在 Fortran、Python DSL 或其他高层表示中看似简单的时间步循环，若包含跨 PE 数据交换，就必须被改写成由任务和回调组成的控制流。程序员不仅要重写语法，还要把同步算法重新组织为适合异步消息到达、局部内存和邻居路由的形式。

论文将这一困难归纳为两个层面的语义鸿沟：高层 stencil（网格模板）表达描述数学计算和邻域依赖；WSE 则要求显式的数据分块、PE 放置、分块通信、局部内存管理和 actor-like 异步任务。既有方法常要求人工重写代码或使用专门 DSL，因而难以让已有 HPC 前端直接迁移到 WSE。

论文选择 MLIR 及其 Python 兼容工具 xDSL 作为共享编译基础设施。MLIR 的多层 IR（Intermediate Representation，中间表示）和 dialect（方言）机制允许在保留高层 stencil 语义的同时，逐步加入 WSE 特定的信息；xDSL 用 Python 类定义 IRDL 方言，便于快速开发和复用 MLIR 生态中的变换。

## 2. 论文要解决的问题

### 2.1 不修改应用代码地面向 WSE

如何让来自 Fortran/Flang、Devito Python DSL 和 PSyclone 的 stencil 代码，在不要求应用层改写的情况下，经过编译器生成可以在 WSE2/WSE3 执行的 CSL 代码。

### 2.2 表达从数学 stencil 到异步硬件的逐步转换

如何把架构无关的 stencil 表示转换为 WSE 所需的二维 PE 布局、远程数据依赖、分块通信、局部内存和 actor-like task/callback 控制流，同时保持足够的高层信息供后续优化使用。

### 2.3 复用编译器生态并获得实际性能

如何以 MLIR/xDSL dialect 和 lowering pass 的组合复用公共编译器能力，并把 WSE 的硬件知识固化进通信库、DSD 生成、融合、广播、系数下推和 fused multiply-add（FMAC，乘加融合）等优化中，使生成代码至少接近或超过手写 WSE 代码。

> 本文主要研究：如何利用 MLIR/xDSL 的多层表示和领域特定 stencil 信息，把未修改的高层 HPC stencil 程序自动降低为适配 WSE 异步执行模型的高性能 CSL 代码。

## 3. 核心方法概述

论文提出一个面向 WSE 的 MLIR lowering pipeline。输入可以来自 Flang、Devito 或 PSyclone，先进入 xDSL/MLIR 的通用 stencil dialect；随后依次经过 WSE 特定的 `csl-stencil`、`csl-wrapper` 和 `csl-ir` dialect，最终打印为 CSL，再交给 Cerebras SDK 编译器生成 WSE 程序。

```text
Fortran/Devito/PSyclone 前端
        ↓
MLIR/xDSL stencil dialect：数学 stencil 与邻域访问
        ↓
Group 1：分解三维网格、确定数据依赖、沿 Z 维 tensorize
        ↓
Group 2：生成 WSE PE 布局、分块通信与 csl-stencil
        ↓
Group 3：bufferization、memref、PE 内存实现
        ↓
Group 4：把同步控制流映射为 actor/task/callback 图
        ↓
Group 5：linalg → csl-ir、memref → DSD、布局/程序模块生成
        ↓
CSL 通信库 + 优化 pass + CSL printer
        ↓
Cerebras SDK / WSE2 或 WSE3
```

`csl-stencil` 显式描述远程数据预取、分块到达后的处理、局部计算和 stencil 访问；`csl-wrapper` 把 CSL 程序和布局 metaprogram 包装在一起；`csl-ir` 近似复现 CSL 的一大部分构造，使编译器能够通过 printer 生成 CSL。

WSE 的每个 PE 被视为硬件 actor，PE 内的 CSL task 被视为软件 actor。远程数据块到达时触发对应处理 task；所有邻居数据到达后再触发局部计算和后续控制流。通信库使用可分块的四方向异步交换，并调用用户提供的回调。论文没有使用 LLM、SFT、强化学习、工具调用 agent 或形式化验证器；这是传统的领域编译器与硬件协同设计系统。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用静态编译器执行流程。

### 4.1 前端与高层表示

三类前端分别是：从 Fortran 提取 stencil 的 Flang；面向有限差分 PDE 的 Python Devito DSL；以及面向天气/海洋模型的 Fortran/PSyclone DSL。已有工作把这些前端的计算转换到 xDSL 的 stencil dialect。该表示使用值语义表达网格字段、邻域访问和计算，不立即暴露物理内存或通信细节。

### 4.2 Group 1：分解与数据依赖

编译器将三维 `x × y × z` stencil 分解到 WSE 的二维 PE 网格，使每个 PE 保存一个 `z` 方向列。`dmp.swap` 表达 PE 间的数据交换；随后沿 Z 维 tensorize，把标量网格转换为每个 PE 上的 FP32 tensor，从而利用 WSE 的大量 PE 和有限局部 SRAM。

### 4.3 Group 2：布局与通信

`dmp.swap` 被转换为 `csl-stencil.prefetch`。`csl-stencil.apply` 被拆为远程数据块处理区域和局部数据计算区域，通信按可配置块数进行，使接收、归约和计算能够交错。`wrap csl-stencil in csl-wrapper` 生成布局 metaprogram，将 kernel 放置到 PE，并提取程序范围的常量。

### 4.4 Group 3：PE 内存实现

编译器从 tensor 的值语义转向 memref 的引用语义，执行部分 bufferization，并复用 MLIR 的 bufferization 能力。由于 CSL 的数学操作采用 Destination-Passing Style，算术操作从 arith dialect 转换到支持原地读写的 linalg dialect；在有限 SRAM 下，累加 buffer 可以复用以保存中间结果和最终结果。

### 4.5 Group 4—5：异步 actor 与 CSL 生成

远程区域和局部区域被映射为软件 actor；顶层 timestep 循环被改写为由无参数函数和 local task 组成的控制流 task graph，而不是普通同步 basic block。之后 linalg 操作被降低到 `csl-ir`，并尽量使用 WSE 的 DSD 仿射迭代器和内置 FP32 `@fmacs`。memref allocation/deallocation 和 DSD view/subview 也被降低；最终 `csl-wrapper.module` 被拆成布局模块和程序模块，打印为 CSL 并交给 Cerebras SDK。

### 4.6 运行时通信与优化

论文实现了基于既有 partitionable communication 策略的 CSL 通信库：在四个方向安排异步发送/接收，管理路由更新和完成 task，并触发用户回调。优化 pass 包括 `stencil-inlining` 融合连续 stencil、`varith` 表示变长加法/乘法、重复操作数融合、乘加融合为 FMAC、通信 buffer 的 tensor broadcasting，以及把常数乘法尽量下推到通信过程中。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有神经网络损失函数。论文的关键对象是编译变换和硬件执行代价，而不是可学习目标。

### 5.1 关键表示与变换

论文没有给出需要复述的统一数值目标公式。方法中的核心“规则”可以概括为：

```text
三维 stencil(x, y, z)
→ 二维 PE 网格(x, y) + 每个 PE 的 z 维 tensor
→ 远程依赖 dmp.swap / csl-stencil.prefetch
→ 分块通信 + 远程/局部双区域 apply
→ actor/task/callback 控制流
→ DSD / @fmacs / CSL
```

### 5.2 评价量

实验以吞吐量 GPts/s（也称 GCells/s，每秒处理的十亿网格点数）为主要性能指标，图 4—6 的运行结果取三次运行的平均值，并给出 95% 置信区间误差条。roofline 分析使用 FLOP/s 与 FLOP/Byte；生产力部分用源代码行数（LoC）作近似指标。论文没有把这些指标包装成训练奖励，也没有声称有限运行测试构成形式化正确性证明。

## 6. 实验设置

### 6.1 数据集与基准来源

本文没有传统机器学习数据集；实验使用五个 stencil/HPC benchmark：

| 基准 | 前端/来源 | stencil 与规模信息 |
|---|---|---|
| Jacobian | Flang/Fortran | 3D 六点 stencil，Laplace 扩散，100,000 次迭代，Z=900 |
| Diffusion | Devito | 3D 十三点 stencil，512 次迭代，Z=704 |
| Acoustic | Devito | 3D 十三点 stencil，二阶时间近似，512 次迭代，Z=604 |
| 25-Point Seismic | Cerebras CSL 示例/手写基线 | 3D 二十五点 stencil，100,000 次迭代，Z=450 |
| UVKBE | PSyclone/Fortran | 四个字段、两个字段需跨 PE 通信、两个连续 stencil.apply，1 次迭代，Z=600 |

X/Y 方向测试三种规模：100×100、500×500、750×994；最大规模用于充分占用 WSE2 PE 网格。所有 benchmark 使用 32 位单精度浮点。论文没有把这些 benchmark 称为训练/验证/测试集，也没有报告数据泄漏分析。

### 6.2 模型与工具

论文使用 Cerebras CS-2/WSE2 和 CS-3/WSE3；编译器侧使用 xDSL v0.35、Cerebras SDK 1.3.0 和 Cerebras ML Software 2.4。前端包括 Flang、Devito、PSyclone；IR 和变换使用 MLIR/xDSL dialect、stencil、dmp、arith、tensor、linalg、memref 以及本文的 `csl-stencil`、`csl-wrapper`、`csl-ir`。实现通过公开仓库 `xdslproject/wse-stencil` 发布。

跨平台对比使用：128 张 NVIDIA A100 GPU 的 Tursa 系统，GPU 基线为 MPI + OpenACC，编译器版本为 NVIDIA nvc++ 23.5-0；ARCHER2 Cray-EX CPU 集群使用 128 个 CPU 节点和 MPI + OpenMP，编译器为 Cray Clang 11.0.4。论文说明 GPU/CPU 对比使用了不同的更大网格，并非严格同规模比较。

### 6.3 对比方法

主要对比对象包括：Cerebras 工程师部分手写优化的 25-point seismic CSL kernel；128 张 A100 GPU 的 MPI + OpenACC 实现；128 个 ARCHER2 CPU 节点的 MPI + OpenMP 实现；以及相关工作中已有的 WSE stencil DSL/编译器结果。论文还把自身生成的 WSE2/WSE3 代码互相比较。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| GPts/s 或 GCells/s | 每秒处理的十亿网格点数 | 越大越好 |
| 95% CI | 三次运行平均吞吐量的置信区间误差条 | 越窄越稳定 |
| Speedup / throughput ratio | 相对手写 WSE、GPU 或 CPU 基线的吞吐/时间优势 | 越大越好 |
| FLOP/s、FLOP/Byte | roofline 中的计算性能与算术强度 | 越高通常越好 |
| LoC | 生成/手写 CSL 与 DSL 源代码规模 | 越少表示编程负担较低，但不是完整生产力度量 |

## 7. 实验结果与结论

### 7.1 WSE2/WSE3 与手写代码

论文图 4 比较四个大规模 benchmark 在 WSE2/WSE3 上的生成代码吞吐。作者报告 WSE3 的升级交换逻辑可减少 WSE2 上的自发回传限制，生成代码在不同 benchmark 上获得更高利用率。针对 25-point seismic，论文第 6.1 节报告：相对于 Cerebras 手写 WSE2 版本，编译器生成版本最多高出 7.9%；WSE3 生成版本相对 WSE2 实现最高提升 38.1%。这里的百分比是论文在该特定 benchmark、问题规模和硬件比较下的结果，不是所有 stencil 的普遍保证。

作者将优势归因于更节省内存的通信分块、只发送计算真正需要的列值，以及约减少 50% task 数量的通信库。手写版本在 WSE2 上使用两块通信，而生成版本可以用单块；手写版本还会发送未被计算使用的首尾列值。

### 7.2 与 GPU/CPU 集群比较

在 Devito Acoustic benchmark 上，WSE3 与 128 张 A100 GPU、128 个 ARCHER2 CPU 节点进行时间到解（time-to-solution）比较。论文报告 WSE3 约比 128 张 A100 快 14 倍，约比 128 个 CPU 节点快 20 倍。作者同时明确说明这不是严格同规模比较：CPU/GPU 实验使用了更大的网格，GPU 使用 OpenACC 而不是手写 CUDA，WSE3 在单片晶圆上处理整个域，因此这些数字不能被解释为硬件平台的普适性能倍数。

### 7.3 Roofline 与硬件感知解释

WSE3 roofline 使用 18.22 PB/s 本地内存带宽、3.30 PB/s fabric 带宽和 1.52 PFLOP/s 峰值性能；A100 对照使用 2.04 TB/s DRAM 带宽和 17.59 TFLOP/s 峰值。对 WSE benchmark 分别按本地内存访问和 fabric 访问估计，论文观察到多数点处于计算受限区域；同一 Acoustic benchmark 在单张 A100 上主要受内存带宽限制。该分析支持作者的解释：编译器把未修改的 stencil 程序映射到更适合其访问/通信模式的 WSE 后，瓶颈从传统平台的内存侧转移。

### 7.4 开发者生产力

表 1 比较手写 CSL kernel、完整 CSL（含布局、通信和 host 支持）与 DSL+本文编译器的 LoC：

| Benchmark | CSL kernel | CSL entire | DSL + 本文方法 |
|---|---:|---:|---:|
| 25-point Seismic | 196 | 980 | 81 |
| Acoustic | 211 | 995 | 81 |
| Diffusion | 192 | 976 | 40 |
| Jacobian | 180 | 964 | 28 |
| UVKBE | 203 | 987 | 44 |

作者据此说明 DSL 源码规模和可读性优于直接编写 WSE 特定 CSL，但 LoC 只是生产力的近似指标。

### 7.5 消融与案例分析

论文没有给出一个独立的“去掉每个 pass 后的完整消融表”。不过，正文通过变换设计和案例说明模块作用：UVKBE 的连续 stencil.apply 可经 stencil-inlining 合并；Acoustic 中重复操作数融合可将三个 DSD 加法替换为一个乘法；linalg-fuse-multiply-add 可生成 `@fmacs`；通信库的分块与系数下推支持通信/计算交错。因缺少统一控制变量消融，无法从正文确认每个 pass 对总体加速的独立贡献。

## 8. 主要创新点

### 8.1 创新点一：面向 WSE 的多层 MLIR lowering pipeline

论文不是直接写一个 WSE 专用后端，而是从已有 stencil dialect 出发，新增 `csl-stencil`、`csl-wrapper` 和 `csl-ir` 三层表示，逐步表达远程依赖、布局、内存、异步任务和 CSL 构造。其价值在于把硬件特定知识嵌入可组合的 MLIR/xDSL 编译链，并保留高层 stencil 语义到较晚阶段。

### 8.2 创新点二：把同步 stencil 映射为 WSE actor/task 控制流

论文识别出 WSE 的异步消息执行不能简单用普通顶层循环表达，并设计变换把远程块处理、邻居交换完成和后续局部计算转换为软件 actor 与 callback/task graph。该机制直接处理了 WSE 的 continuation complexity，而不是要求用户手工重写算法。

### 8.3 创新点三：将 WSE 通信与算术最佳实践固化为可复用编译/运行时组件

分块通信库、DSD lowering、stencil fusion、重复操作数融合、FMAC、广播归约和通信中系数应用把已有 WSE 专家经验变成编译器与运行时组件。实验表明这些组件可以同时带来性能和编程负担收益；但“使用 MLIR”或“使用 WSE”本身不是创新，真正贡献是它们在多层 lowering 与异步硬件语义之间的组合方式。

## 9. 局限性

### 9.1 论文明确或正文可确认的局限

1. 通信库当前面向最多三维、星形 stencil 的 partitionable communication；若需要 box-shaped 等其他通信形状，论文说明需要更新库。
2. 论文提到软件路由模式是互补工作，硬件路由配置和 switching update 的代码生成仍是未来工作。
3. 实验中的 GPU/CPU 对比不是严格同问题规模或同优化水平对比；GPU 基线使用 OpenACC，作者明确提醒不能把 14×/20× 当作普适硬件结论。
4. 论文只评估五个 stencil benchmark，覆盖 Flang、Devito、PSyclone 三类前端，但未证明所有 HPC 控制流、非 stencil 算法或复杂通信模式都能直接受益。
5. 论文没有提供完整的模块级消融，因此各个 lowering/优化 pass 的独立收益当前 PDF 内容不足以确认。

### 9.2 阅读后发现的潜在局限

1. WSE 特定 dialect、CSL SDK 和通信库仍形成较强平台耦合，迁移到 RISC-V、GPU 或其他 CGRA 需要重新定义执行模型、内存和通信语义。
2. 通过 LoC 衡量生产力不能覆盖编译时间、调试复杂度、生成代码可解释性和错误定位成本。
3. 大幅性能结果部分依赖 WSE 的硬件规模、单片晶圆通信和特定 stencil 访问模式；若工作负载是非规则稀疏图、分支密集程序或需要全局同步，收益可能不同。
4. roofline 对 fabric 访问使用了上下界式假设；它适合解释瓶颈，但不能替代所有动态通信拥塞、路由冲突和端到端时间测量。

## 10. 阅读后的研究方向反思

这篇论文最值得借鉴的是“保留高层领域语义，延迟到目标硬件信息足够明确时再下降”的编译器组织方式。对于 LLVM IR、RISC-V 或多架构编译，不能简单照搬 WSE 的 actor/task 语义；可迁移的是多层 IR、硬件约束显式化、通信/计算协同和公共 lowering 基础设施。

论文更适合作为传统编译器基础设施和硬件感知 lowering 的方法参考，而不是 LLM 优化的直接 baseline。若把目标平台替换为 RISC-V，本身不足以构成创新；需要新增 RISC-V 向量/片上网络/多核内存模型约束，并证明统一 IR 如何在不同微架构代价模型下选择布局、分块和指令级实现。

对于现有研究方向，本文与“多硬件编译、代价模型、MLIR dialect、RISC-V/RVV 后端”关系较强，与 LLM 作为 Selector/Translator/Generator 的关系有限。一个合理的组合方式是把本文的多层 IR、硬件约束和实测反馈作为传统编译器环境，再让模型选择合法 lowering 配置或优化 pass；不能把论文没有使用的 LLM、强化学习或形式验证补写成本文贡献。

## 11. 可进一步尝试的研究方向

### 11.1 面向 WSE 与 RISC-V 向量后端的统一 stencil IR

#### 研究问题

能否在保留 stencil 邻域、通信依赖和内存布局信息的情况下，用统一 MLIR dialect 同时生成 WSE CSL 与 RISC-V/RVV 多核代码。

#### 与原论文的区别

不是把 WSE 代码简单翻译成 RISC-V，而是定义可区分“邻居通信、向量宽度、局部内存和同步语义”的共享中间层，并允许后端保留不同执行模型。

#### 可能的创新点

统一的通信/计算依赖表示、跨后端合法性约束、面向 RVV 的分块与向量长度无关代码生成，以及 WSE/RVV 的可比代价模型。

#### 实验框架

```text
Fortran/Devito/PSyclone
        ↓
共享 stencil + communication dialect
        ↓
WSE actor lowering 或 RVV 多核 lowering
        ↓
CSL / RVV LLVM IR
        ↓
真实硬件与仿真器上的吞吐、通信和能耗比较
```

#### 可行性

可复用 xDSL/MLIR、LLVM RISC-V 后端、现有 WSE stencil 仓库和 stencil benchmark；需要 RISC-V RVV 硬件或可信模拟器。

#### 主要风险

两种硬件的同步和内存语义差异过大，统一层可能退化为最低公分母；没有真实硬件时，代价模型结论会受模拟器偏差影响。

### 11.2 编译器驱动的通信/计算协同搜索

#### 研究问题

在固定 stencil 语义下，如何自动选择 PE 分解、通信块大小、buffer 复用、系数下推和 FMAC 组合，以降低真实 WSE 或 RISC-V 集群的端到端时间。

#### 与原论文的区别

原论文以规则和人工设计的 lowering 为主；该方向将可验证的变换空间与硬件实测反馈结合，研究配置选择而不是让模型直接生成任意低层代码。

#### 可能的创新点

将通信拥塞、局部 SRAM 峰值、任务激活开销和算术吞吐纳入统一多目标代价模型，并区分静态可行性和动态性能。

#### 实验框架

```text
stencil IR
        ↓
生成合法 lowering 配置
        ↓
编译到 WSE/RVV
        ↓
运行时采集吞吐、通信和内存峰值
        ↓
代价模型/搜索器更新配置
```

#### 可行性

可复用本文的五组 lowering 和 benchmark；若缺乏 WSE 访问，可先用小规模 RISC-V 多核原型验证。

#### 主要风险

硬件运行反馈成本高，性能噪声和配置空间会造成搜索不稳定；需要避免把一次特定规模的最佳配置误认为通用规律。

### 11.3 异步 actor 语义的形式化验证与错误诊断

#### 研究问题

如何验证同步 stencil 到异步 task/callback 图的依赖保持性，并在通信遗漏、死锁或数据竞争时生成可定位的反例。

#### 与原论文的区别

原论文通过运行基准验证功能和性能，但没有给出该 lowering 的形式化证明；该方向把异步控制流的语义保持和错误诊断作为核心问题。

#### 可能的创新点

为 `csl-stencil.apply`、分块通信和 actor graph 定义可检查的依赖不变量，结合静态检查、有限状态探索或 translation validation。

#### 实验框架

```text
同步 stencil IR + 依赖关系
        ↓
异步 actor lowering
        ↓
依赖/消息/内存不变量检查
        ↓
生成 CSL 与运行时测试
        ↓
错误反例或验证通过率
```

#### 可行性

可从小型二维 stencil 和有限 PE 拓扑开始，不需要一开始覆盖完整 WSE。

#### 主要风险

真实 WSE 的路由、task 和 SDK 语义可能不完全公开；有限状态检查不能被误写成对任意规模程序的完整证明。

## 12. 与其他已读文献的关系

本批次只完成本文一篇论文的全文阅读，因此没有其他“当前批次已确认”的论文可进行逐篇横向比较。

就研究角色而言，本文是传统 compiler infrastructure / hardware-aware code generation 工作：它不属于 LLM Selector、Translator 或 Generator。它可作为后续研究中的目标后端、MLIR lowering 参考和硬件-软件协同优化基线；若与 LLM/RL 工作组合，需把模型角色单独记录为 selector 或配置搜索器，不能改变本文的 Supporting 身份。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 将未修改的 HPC stencil 程序编译到 Cerebras WSE |
| 核心问题 | 高层同步 stencil 与 WSE 异步 actor、PE 本地内存和邻居通信之间存在语义鸿沟 |
| 输入 | Flang、Devito、PSyclone 产生的 MLIR/xDSL stencil 表示 |
| 输出 | 面向 WSE2/WSE3 的 CSL 程序、布局与通信代码 |
| 核心方法 | `csl-stencil`、`csl-wrapper`、`csl-ir` 多层 dialect；五组 lowering；WSE 通信库和算术优化 |
| 使用的模型 | 不使用机器学习模型或 LLM |
| 使用的编译器工具 | MLIR、xDSL v0.35、Flang、Devito、PSyclone、Cerebras SDK 1.3.0、Cerebras ML Software 2.4 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 论文未报告形式化验证；使用运行基准和性能分析 |
| 数据集规模 | 无传统数据集；五个 stencil benchmark，三种 X/Y 规模 |
| 主要指标 | GPts/s、95% CI、FLOP/s、FLOP/Byte、LoC |
| 最重要实验结果 | 相对手写 WSE2 seismic 最高提升 7.9%；WSE3 相对 WSE2 最高提升 38.1%；特定非同规模比较中约快于 128 A100 14×、128 CPU 节点 20× |
| 核心创新 | 将高层 stencil 语义与 WSE 异步 actor/通信/布局逐层连接，并复用 MLIR 生态生成跨 WSE 代际的 CSL |
| 主要局限 | 通信库覆盖的 stencil 形状有限；跨平台对比不严格；缺少独立 pass 消融；强依赖 WSE/CSL |
| 与 RISC-V 研究的相关性 | 中：可借鉴 MLIR、硬件感知 lowering 和通信/计算协同，但论文未研究 RISC-V |
| 最适合作为 | 多硬件编译基础设施、MLIR dialect/lowering 方法参考、硬件感知编译基线 |

> 这篇论文最值得学习的是：把领域语义保留到后期，并将异步通信、PE 布局、内存和算术优化作为同一条 lowering 管线的一部分；最主要的局限是 WSE 特定通信和执行模型限制了直接迁移范围，且跨平台结果不是严格同规模对比；如果用于后续研究，最合理的使用方式是把它作为硬件感知编译基础设施和后端参考，而不是简单替换为 RISC-V 或把未使用的 LLM/RL 机制补进论文贡献。
