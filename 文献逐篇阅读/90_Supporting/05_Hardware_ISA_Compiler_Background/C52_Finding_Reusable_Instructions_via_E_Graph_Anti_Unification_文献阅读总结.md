# C52 Finding Reusable Instructions via E-Graph Anti-Unification 文献阅读总结

论文题目：**Finding Reusable Instructions via E-Graph Anti-Unification**

作者：Youwei Xiao、Chenyun Yin、Yitian Sun、Yuyang Zou、Yun Liang

发表时间：2026；ASPLOS ’26，Volume 2，pp. 749–763

发表平台：Proceedings of the 31st ACM International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS 2026), Volume 2, pp. 749–763

论文链接或编号：DOI 10.1145/3779212.3790162
元数据核验来源：[ACM DOI](https://doi.org/10.1145/3779212.3790162)；[作者代码仓库](https://github.com/isamore-group/ISAMORE)
代码/数据/工件：作者代码：[ISAMORE](https://github.com/isamore-group/ISAMORE)

代码仓库：[公开仓库](https://github.com/isamore-group/ISAMORE)
来源：[ISAMORE 仓库 README（明确对应 ASPLOS 2026 论文）](https://github.com/isamore-group/ISAMORE)；[作者项目论文列表](https://aps.ericlyun.me/publications/)。
关键词：ISAMORE、自定义指令、RISC-V、e-graph、等式饱和、反统一、硬件/软件协同

> 本文档用于文献阅读、组会汇报和后续研究分析。论文事实与阅读后的研究思考分开描述。正文证据来自 15 页正式论文 PDF；章节、表图编号以论文原文为准。

---

## 1. 研究背景

专用加速器和可扩展 ISA（尤其 RISC-V）可为特定应用提供性能，但自定义指令（custom instructions, CIs）的设计需要同时理解软件热点、跨程序复用模式、硬件实现和成本。既有自动化方案多在热点代码上按语法合并指令序列，生成的自定义指令可能过度专用，较少考虑不同表达式在语义上等价或共享子结构。论文以 CImg 为例指出，语法合并得到的定制指令只用于 8 个热点位置，而语义复用方法可平均加速 93 个位置，并报告 1.17× 更多加速和 90.5% 面积节省（第 1 节）。

另一难点是可扩展性。e-graph 和等式饱和能紧凑表达大量等价程序，但朴素 e-graph 反统一需要探索大量 e-class 对及其组合模式，候选数可能指数增长。作者将通用程序转换为结构化 DSL，再设计分阶段规则应用、智能配对/采样、向量化和硬件感知选择，把 e-graph 用于自定义指令发现（第 2–5 节）。

## 2. 论文要解决的问题

### 2.1 发现语义可复用的指令

如何从多个应用程序中的等价或相似表达式抽取可重复利用的指令模式，减少传统仅按语法合并导致的重复电路和低利用率？

### 2.2 控制 e-graph 搜索成本并兼顾硬件效益

如何把带有控制流的程序表示为 e-graph，并使反统一在真实代码规模下可运行？如何同时考虑软件延迟节省、硬件延迟、面积和向量并行机会，而不只优化程序大小？

> 本文主要研究：如何基于语义等价、跨代码位置的复用和硬件成本模型，从代表性程序中自动识别适合实现为自定义指令的模式。

## 3. 核心方法概述

ISAMORE 的核心是可复用指令识别（Reusable Instruction Identification, RII）。LLVM 前端把优化后的 LLVM IR 转成含 If、Loop、向量操作等结构的 DSL，并编码为 e-graph。RII 分阶段应用重写规则，通过 e-graph 反统一发现共同模式；接着识别可向量化模式，用性能和面积模型选择 Pareto 解，最后抽取程序并通过 HLS/CIRCT 生成硬件实现（图 4、图 7、第 3–6 节）。

```text
代表性 C/C++ 程序 → LLVM 18 IR → 结构化 DSL / e-graph
                                      ↓
GEM5 采集基本块周期与执行次数 → 硬件感知成本模型
                                      ↓
分阶段等式饱和 → 类型/结构相似度筛选 → e-graph 反统一
                                      ↓
种子打包、向量展开与无环剪枝 → 候选模式
                                      ↓
Pareto 选择与成本精化 → 自定义指令模式
                                      ↓
XLS HLS / CIRCT → Verilog / RoCC 加速器
```

论文方法输出的是自定义指令及其硬件实现线索，不是语言模型生成的源码；全文未使用 LLM 来执行候选选择或代码变换。LLM 仅出现在 BitNet 推理的应用场景中。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用程序分析、候选搜索、成本估计和硬件仿真/综合。

### 4.1 程序表示与规则准备

前端 LLVM pass 将 LLVM IR 转成结构化 DSL，并插桩基本块边界。另一个自定义 GEM5 tracer 读取标记，收集各操作的 cycles per operation（CPO）和基本块执行次数。离线规则生成经 SMT 后端枚举等价重写规则；论文报告在 20 小时内生成 1164 条等式重写规则（第 6 节、第 7 节）。

### 4.2 RII 分阶段候选识别

算法先从 DSL 初始化 e-graph 和空 Pareto 解集。前两阶段分别完整饱和整数与浮点的 saturating 规则；后续阶段从非饱和规则中选取规则并限制应用次数。每阶段可把此前发现的模式作为重写继续搜索。智能反统一先按结果类型排除不一致的 e-class 对，再用 64 位结构哈希和 Jaccard 相似度阈值筛选结构相似对；模式端用 boundary 极值采样或 kd-tree 空间采样控制候选数量（第 5.1–5.2 节）。

### 4.3 向量化、选择与硬件输出

向量化先在标量 e-graph 中找可打包种子，把同一基本块中的种子合并为向量项；再用提升重写展开向量构造器，并用贪心提取与 compress 操作剪去冗余打包和 Get→Vec→Get 环，得到无环的混合标量/向量 e-graph。随后进行多目标 Pareto 选择、按提取程序重算成本，再用 HLS/CIRCT 生成硬件描述（第 5.3–5.4 节）。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数。关键目标是自定义指令带来的延迟节省与面积权衡。

对模式 p，论文按使用位置 u 累加软件基本块内操作的估计时间，并减去 HLS 报告的硬件延迟：

```text
ΔL(p) = Σ_{u∈use(p)} (Σ_{o∈p} CPO(bb(o,u)) − L_HLS(p))
S(P) = L_cpu / (L_cpu − Σ_{p∈P} ΔL(p))
A(P) = Σ_{p∈P} A_HLS(p)
```

其中 ΔL(p) 是模式各次复用预计节省的总延迟；CPO 由插桩与 GEM5 profiling 获得；L_HLS(p) 来自轻量 HLS 对模式串行行为进行 ASAP 调度的延迟估计，若有循环则启用流水化；S(P) 是选定模式集合相对一般处理器的估计 speedup，A(P) 是 HLS 估计面积之和（第 5.4.1 节，公式 1–3）。Pareto 选择优先最大化速度，再在速度相同或相近时考虑面积。最终 OpenROAD 面积综合用于报告硬件面积结果（第 7 节）。

## 6. 实验设置

### 6.1 数据集来源

微基准包括 9 个核：2DConv、MatMul、MatChain、FFT、Stencil、Quaternion product、QRDecomp、Deriche、SHA，以及把各核组成的 All 组合基准（表 2）。源代码来自 Diospyros、PolyBench、MachSuite、Coremark-PRO。各 LLVM 优化后核函数 IR 规模为 88–496 LOC，LLVM `-O3` 可通过循环展开暴露机会（第 7.1 节）。真实库案例包括 liquid-dsp、CImg 和 PCL；量化语言模型案例为 BitNet b1.58，密码学案例为 CRYSTALS-Kyber（第 7.2 节）。

### 6.2 模型与工具

实现前端基于 LLVM 18 和 JLM；RII 核心以 Rust 编写，建立在 egg、babble 上，扩展约 20,157 LOC。重写枚举使用 Enumo 特征和 SMT 等价检查；HLS 估计使用 XLS 的回归型操作延迟/面积模型，硬件输出使用 CIRCT。主要实验运行于双 AMD EPYC 7763、Ubuntu 22.04 服务器，单次运行默认限时 2.5 小时、内存上限 30 GB。性能/面积估计中目标频率为 1 GHz，OpenROAD 2.0-16235 面向 ASAP7 PDK 综合（第 6–7 节）。

### 6.3 对比方法

微基准比较 ENUM（受细粒度凸子图枚举启发）、NOVIA（较粗粒度语法合并）、NoEqSat（关闭语义等价探索）和 ISAMORE Default。库案例也将 ISAMORE 与 ENUM、NOVIA 对照。NOVIA 使用 ISAMORE 的 profiling 成本模型作公平比较，并采用较宽松 I/O 约束和 RoCC 风格内存接口（第 7.1.2 节）。

### 6.4 评价指标

报告整体 speedup、OpenROAD 最终综合面积、候选/原始/峰值 e-graph 大小、候选模式数、执行时间与内存；还报告每条自定义指令包含的操作数、指令数量和平均复用次数（表 2–3、图 10–12）。RTL 仿真用于 BitNet 与 Kyber 加速器性能案例；这些是仿真值，不是流片芯片实测结果。

## 7. 实验结果与结论

### 7.1 主要结果

与 vanilla LLMT e-graph AU 相比，RII 将峰值 e-graph 规模降低约 6–39 倍；LLMT 候选模式可增长至 633K，并在所测案例中超过 30 GB 内存限制。启用 RII 后，全部基准在 145 秒内完成，内存最高 799 MB（表 2，第 7.1.1 节）。

在微基准上，ISAMORE 的最大速度方案相对 NOVIA 平均为 1.52×，各基准范围 1.12×–1.94×。相对 NoEqSat，ISAMORE 的平均最大 speedup 高 1.12×，平均面积为 NoEqSat 的 84.9%（第 7.1.2 节）。在启用向量化的情况下，比较 NOVIA 与 ENUM 时平均优势分别为 1.76× 和 1.46×，最高分别为 2.69× 和 1.95×（第 7.1.3 节）。

### 7.2 与传统方法的比较

在 liquid-dsp 案例，ISAMORE 相对 NOVIA 平均 speedup 为 1.39×，面积节省 84.0%，并比 ENUM 高 1.21×。CImg 案例中，ISAMORE 得到 8 条自定义指令、平均复用 93 次、面积 975 μm²、speedup 1.18×；NOVIA 合并出一条 167 操作的大指令，复用 8 次、面积 10314 μm²、speedup 1.01×，ISAMORE 面积节省 90.5%。PCL 案例中，相对 NOVIA 平均 speedup 1.64×、最高 2.73×，面积节省 93.2%（第 7.2.1 节，图 12）。

### 7.3 与其他 LLM 方法的比较

本文没有与 LLM 编译优化方法比较。BitNet 是应用案例之一；RII 本身由编译器、e-graph、成本模型和 HLS 工具执行，并未调用语言模型。

### 7.4 消融实验

AstSize 模式用硬件无关的项大小代替硬件成本目标，所有基准的性能增益最低，支持硬件感知选择的重要性。KDSample 在 QProd、Deriche、All 上优于 Default，但探索时间和内存明显更高。Vector 模式在 10 个微基准中的 8 个优于 Default；MatMul、MatChain、QRDecomp 等受益明显，2DConv 因 LLVM 未对边界检查中的操作执行 if-conversion 而没有充分暴露 DLP（第 7.1.3 节、表 3、图 11）。

### 7.5 案例分析

BitNet b1.58 的低比特点积案例中，ISAMORE 识别向量化 packed low-bit dot product 自定义指令，Rocket tile 上的 RoCC 加速器 RTL 仿真报告 BitLinear speedup 2.15×；OpenROAD 报告面积开销 4.81%，频率未下降（第 7.2.2 节）。CRYSTALS-Kyber 案例中，识别可复用于正向和逆向 NTT 的 butterfly 运算，自定义加速器 RTL 仿真 speedup 5.15×；面积开销 17.67%，频率下降 2.58%（第 7.2.3 节）。这些数值依赖仿真、HLS 与 ASAP7 综合流程。

## 8. 主要创新点

### 8.1 适配一般程序与控制流的结构化 DSL

DSL 将 If、Loop、列表、向量及模式应用纳入 e-graph 项，支持从优化后 LLVM IR 表示带控制流的程序（第 4 节）。

### 8.2 可扩展的分阶段 e-graph 反统一

先完整处理不增加新 e-class 的饱和规则，再受限地应用非饱和规则；以类型检查、结构哈希相似度和 boundary/kd-tree 采样削减 e-class 对和候选模式（第 5.1–5.2 节）。

### 8.3 从标量模式发现向量自定义指令

种子打包、向量构造器展开和无环剪枝让标量代码中的重复操作进入混合向量/标量 e-graph，扩展传统自定义指令识别的粒度（第 5.3 节）。

### 8.4 面积/性能 Pareto 选择与端到端硬件生成

以真实 profiling 与 HLS 性能/面积数据驱动模式选择，并通过 CIRCT 输出硬件实现；论文把语义复用、并行化和硬件成本置于同一流程中（第 5.4、6 节）。

## 9. 局限性

作者指出，贪心无环剪枝可能因贪心选择错过部分向量化或模式识别机会（第 5.3 节）。boundary 与 kd-tree 都是启发式采样，存在候选覆盖与搜索预算的权衡。效果依赖程序代表性、GEM5 profiling、XLS 回归估计、HLS 和 PDK；库与基准覆盖虽包括多领域，但不能据此推断任意处理器或完整真实负载都能获得相同收益。BitNet/Kyber 的速度数值来自 RTL 仿真，面积与频率来自特定综合/实现流程，而非硅片实测。规则预枚举需 20 小时，显示离线规则准备仍有显著成本（第 7 节）。

## 10. 阅读后的研究方向反思

ISAMORE 关注的是“哪些程序结构值得加入 ISA”，与传统编译器从固定指令集中寻找优化变换不同。其成本模型显式使用模式出现次数、软件每操作周期、HLS 延迟和面积，适合分析编译与硬件协同设计；但候选发现的语义基础来自重写规则与 SMT 枚举，需区分搜索启发式和等价性证据。对 RISC-V/RVV 研究而言，重要启发是把 LLVM 真实控制流、复用位置和物理资源成本放在同一评估链中。

## 11. 可进一步尝试的研究方向

### 11.1 将可复用指令发现与 LLVM/RVV 后端联合优化

#### 研究问题

对既有 RISC-V 标量与 RVV 后端，如何在不扩大硬件面积预算的前提下，识别高复用且后端可表达的指令/向量模式？

#### 与原论文的区别

原论文面向自定义硬件/指令与 RoCC 实现；后续方向可以把标准 RISC-V/RVV 后端已有指令成本、寄存器压力与合法化路径纳入同一候选评估。

#### 可能的创新点

把 RTL/HLS 成本与 LLVM 后端真实指令成本、寄存器分配结果及可验证语义联系起来，形成可移植的模式取舍模型。

#### 实验框架

从 LLVM IR 与目标机 profile 生成候选，分别映射到基础 ISA、RVV 或扩展单元；用 LLVM MIR/汇编、模拟器和综合结果比较运行时间、代码尺寸、面积与功耗估计。

#### 可行性

论文已给出 LLVM 前端、GEM5 trace 与 HLS/RTL 路径，可沿用其中部分接口；关键在于建立公开、可复现的目标后端和面积模型。

#### 主要风险

候选模式的语义等价不等同于硬件侧实现正确；仿真/估计偏差和真实工作负载覆盖会影响结论，且需处理编译器后端支持与工具链维护成本。

## 12. 与其他已读文献的关系

与 C20“从 ISA 规格综合指令选择后端”相比，ISAMORE 从应用中找值得专门加速的自定义指令，C20 则自动构造 LLVM GlobalISel 后端规则；两者可以处在 ISA 定制的前后端不同阶段。与 C51“符号尺寸广义矩阵链编译”相比，C51 在既定 BLAS/LAPACK 核集合中生成多版本数值程序；ISAMORE 识别并物化新的自定义硬件指令。ISAMORE 与既有等式饱和向量化工作相关，但其目标是新硬件指令识别，而非仅映射到已有 SIMD 指令。

## 13. 一页式总结

ISAMORE 是一套面向专用指令识别的硬件/软件协同框架。它将优化后的 LLVM IR 转为带控制流结构的 DSL/e-graph，分阶段执行等式饱和，通过结构相似度、类型和采样控制反统一成本，再用 profiling 与 HLS/面积模型选择兼顾速度和面积的可复用模式，并输出硬件实现。微基准中 RII 显著降低 vanilla e-graph AU 的内存和运行成本；真实库与 BitNet、Kyber 案例显示出较高复用度与加速潜力。证据限于论文所用工作负载、仿真和综合流程；方法不是 LLM 编译优化器，分类为 SUPPORTING / B5_Hardware_ISA_Compiler_Background。
