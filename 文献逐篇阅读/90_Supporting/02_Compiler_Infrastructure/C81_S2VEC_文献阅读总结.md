# S2VEC 文献阅读总结

论文题目：**S2VEC: Compiler-Driven Stream Specialization for Linearized Vectorization**

作者：Luís Crespo、Ana Fernandes、Gabriel Falcao、Pedro Tomás、Nuno Roma、Nuno Neves

发表时间：2026

发表平台：40th ACM International Conference on Supercomputing（ICS 2026），pp. 119–131，正式 proceedings 论文
元数据核验来源：[ACM ICS 2026 DOI](https://doi.org/10.1145/3797905.3807866)；[作者 PDF](https://web.tecnico.ulisboa.pt/~ist14359/wordpress/nfvr_pubs/ics26.pdf)

论文链接或编号：DOI [10.1145/3797905.3807866](https://doi.org/10.1145/3797905.3807866)；作者公开 PDF：[ics26.pdf](https://web.tecnico.ulisboa.pt/~ist14359/wordpress/nfvr_pubs/ics26.pdf)
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

关键词：Compiler、LLVM、Intermediate Representations、Stream Specialization、SIMD、Vectorization

> 本文是对正式 ICS 2026 论文正文的总结。论文事实以 PDF 正文为依据；研究建议与事实分开标注。本文不是大语言模型论文，正文未将 LLM 作为系统组成部分。

## 1. 研究背景

论文第 1 节指出，数据移动和通信常成为现代处理器的主要瓶颈；SIMD（Single-Instruction Multiple-Data，单指令多数据）虽能提高数据级并行吞吐，却可能增加内存带宽和延迟压力。传统自动向量化器对复杂、间接或带依赖的内存访问往往难以暴露并行性，导致漏向量化和较差性能（第 1 节）。

数据流（data streaming）机制把地址生成、内存访问与核心计算解耦，使硬件可以专门处理内存流，并减少循环中的索引、地址计算和显式访存。论文关注的是如何让 LLVM 编译器自动识别这些内存流，并将普通程序编译为支持流专门化（stream specialization）的向量代码。

## 2. 论文要解决的问题

### 2.1 编译器表示与流式执行的语义差距

流式硬件使用隐式内存访问和控制流，而现代 LLVM 编译基础设施主要采用显式的 SSA（Static Single Assignment，静态单赋值）表示。仅扩展传统向量化器，无法自然地消除显式 load/store 与归纳变量驱动的循环控制（第 1 节）。

### 2.2 复杂访问模式的自动识别

现有方法对多维、间接、跨循环或依赖其他归纳变量的内存访问支持有限。论文希望自动抽取这些访问模式，并用可优化的中间表示表达其依赖关系（第 1、3 节）。

### 2.3 流专门化与向量化的联合优化

论文不只检测数据流，还要在同一编译流中完成计算图合并、流循环折叠和流感知向量化，并为 UVE（Unlimited Vector Extension）RISC-V 流向量 ISA 生成目标代码（第 1、3 节）。

> 本文主要研究：如何在 LLVM 中自动把普通循环的复杂内存访问转换为流-数据流表示，并生成适用于 RISC-V 流向量扩展的向量代码。

## 3. 核心方法概述

S2VEC 是一个 LLVM-based compilation toolchain。它从源程序经 Clang 和 LLVM IR 开始，在标准 LLVM 优化之前/过程中识别循环、归纳变量、数据流和计算图，构造 Stream-Dataflow IR，再进行流感知优化，最后生成 UVE RISC-V 代码（图 3、第 3 节）。

```text
源程序
  ↓ Clang + LLVM IR
LLVM 循环与访存分析
  ↓
Stream-Dataflow IR
  ├─ 归纳变量、数据流、控制流
  └─ 计算图与依赖关系
  ↓
流感知向量化、计算图合并、流循环折叠、DCE/IC
  ↓
UVE RISC-V 流向量代码生成
  ↓
Spike 功能验证 / gem5 周期级性能评测
```

Stream-Dataflow IR 的顶层对象是 StreamLoop；其内部包含控制流、计算图、DataStream 和 Induction Variable。数据流用基地址及每个循环维度的 offset、size、step 描述；依赖记录一个归纳变量或数据流如何影响另一个模式字段（第 3.2 节）。

论文中的系统角色是传统编译器基础设施和后端代码生成器，不是选择 pass 的 LLM，也不是直接生成源代码的 LLM。因此按 Taxonomy v2，建议归为 `SUPPORTING / B2_Compiler_Infrastructure`，并加 RISC-V/RVV、LLVM、向量化、数据流等横向标签。

## 4. 实验框架与训练流程

### 4.1 编译执行流程

论文不涉及模型训练、SFT、强化学习、提示词推理或工具调用。它采用静态编译分析和目标代码生成流程：先识别 LLVM 循环层次；再分析 PHI 节点得到归纳变量的 offset、size、step；随后沿 GetElementPtr 和操作数递归追踪 load/store，抽取数据流；再以 DFS 抽取计算图和嵌套 StreamLoop（第 3.1、3.3 节）。

### 4.2 中间表示优化

系统在 Stream-Dataflow IR 上执行三类专门优化：

1. 流式向量化：对计算图向量化，把流数据填入向量寄存器；条件控制转为 mask/predication；可向量化的归约被改写为向量到标量的归约。
2. 计算图合并：在适合的存储/加载关系下合并计算图，处理归约退出点并避免可复用值的重复加载。
3. 流循环折叠：在外层没有计算图或其他嵌套流循环时折叠完美嵌套循环，减少循环控制以及数组索引所需的模除和除法开销（第 3.4 节）。

此外，系统复用 LLVM 的 DCE（Dead Code Elimination，死代码消除）和 IC（Instruction Combining，指令合并）等功能。

### 4.3 代码生成与验证

代码生成阶段把每个数据流编码为 UVE 的 start/append/end 配置指令，把依赖表示为静态、动态或 scatter-gather modifier，并生成由流状态或硬件循环驱动的循环。生成代码先在修改后的 Spike ISS 上做功能验证，再进入性能模拟（第 3.5、4.1、5.1 节）。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习，因此不存在强化学习奖励函数，也没有神经网络损失函数。

论文的关键形式化定义是 Stream-Dataflow IR 的结构，而不是训练目标：

```text
StreamLoop = (I, S, B)
ComputeGraph = (V, E)
DataStream = (T, A, D)
Pattern = (offset, size, step)
```

其中 `I` 是归纳变量集合，`S` 是控制流，`B` 是有序循环体；`V/E` 分别表示计算图节点和操作数边；`T` 是来源内存指令，`A` 是基地址，`D` 是数据流维度；每个维度由归纳变量、访问模式、是否向量化和依赖集合描述（第 3.2 节）。这些结构的目标是精确表达循环迭代和地址序列，而不是最大化奖励。

## 6. 实验设置

### 6.1 数据集来源

论文使用 31 个向量化 benchmark，来源包括 PolyBench、MachSuite 和作者自定义 benchmark（表 1、第 4.1 节）。表 1 记录了每个 benchmark 的 kernel 数、最大嵌套循环深度、数据流数量、最复杂访问模式，以及 XSSR、RVV、ARM SVE、当前 UVE 编译器和所提工具链能否向量化。

代表性程序包括 GEMM、GEMM-ncubed、GEMM-blocked、GEMVER、SYMM、SYRK、SYR2K、ATAX、2MM、3MM、BICG、DoitGen、MVT、Cholesky、Gram-Schmidt、LU、TriSolv、FDTD-2D、Jacobi、Convolution、Covariance、KNN、SpMV 和 SpMV-ind。论文没有将这些 benchmark 称为训练集/验证集/测试集，也没有报告机器学习意义上的数据泄漏分析。

### 6.2 模型与工具

| 项目 | 正文信息 |
|---|---|
| 编译器 | LLVM-based toolchain；文中说明使用 LLVM 21 生成 ARM SVE 与 RISC-V RVV 对照代码（第 4.1 节） |
| 目标 ISA | UVE RISC-V stream-vector ISA extension；对照为 RISC-V RVV、ARM SVE 与标量 RISC-V |
| 功能验证 | 修改后的 Spike Instruction Set Simulator，输出动态指令数 |
| 性能模拟 | 基于 NDPmulator 的 UVE 向量协处理器和 gem5/周期级模拟流程 |
| 流向量协处理器 | 单发射、顺序执行、五级流水；4×512-bit 物理向量寄存器，峰值向量吞吐 512 bit/cycle |
| 顺序基线 | 4-wide in-order RISC-V CPU，512-bit 向量数据通路，匹配内存层次与算术单元 |
| OoO 基线 | 8-wide superscalar RISC-V core，按公开 AMD Zen 5 信息建模，峰值向量吞吐 1024 bit/cycle |

### 6.3 对比方法

主要对比包括：手写 UVE 流专门化代码、LLVM 生成的 RVV 代码、标量 RISC-V 代码、ARM SVE 自动向量化结果，以及相同程序在 4-wide 顺序核心和 8-wide OoO 核心上的运行结果（第 4.3、5 节）。表 1 还与原 UVE 编译器和 XSSR/stream semantic register 方案比较覆盖能力。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| Vectorization coverage | benchmark/kernel 是否被工具成功向量化 | 越高越好 |
| Retired instruction reduction | 相对标量代码的已退休指令减少比例，见图 6/7 | 越高越好 |
| Speedup | 流向量代码相对参考 CPU 的性能加速比，见图 9 | 越高越好 |
| Functional validation | Spike 上生成 UVE 代码的功能验证 | 通过才可作为有效代码 |

## 7. 实验结果与结论

### 7.1 覆盖与代码质量

表 1 显示，所提工具链能够向量化表中的全部 31 个 benchmark；相比 RVV、ARM SVE 和较早 UVE 编译器，它对复杂依赖链、间接访问和多维模式覆盖更广。对照编译器在 GESUMMV、BICG、SpMV 等部分 kernel 上不能向量化，而所提工具链仍可生成流专门化向量代码（表 1、第 4.2 节）。

与手写 UVE 代码相比，自动生成代码在大多数 benchmark 上具有相近的动态指令减少；3MM 和 Covariance 等案例中手写代码利用了编译器尚未完全支持的 FMA，因此存在一定差距（第 4.3 节）。

### 7.2 与 RVV 和标量代码的比较

图 6 显示，结合 SIMD 与数据流专门化的代码在各 benchmark 场景中通常比 RVV 代码执行更少指令；论文指出 RVV 在 2MM、TRMM 等案例中甚至可能不如标量代码。图 7 的 512-bit 向量实验表明，随着数据集规模、循环操作数或输入数据流数量变化，所提工具链保持较高指令减少，而 RVV 在内存受限方向呈下降趋势（图 6、图 7）。

### 7.3 与顺序核心的性能比较

在图 9A 的 4-wide in-order 参考核心比较中，所提协处理器在仅考虑 kernel 执行时的全配置几何平均加速比为 5.93×；包括协处理器设置开销时为 5.56×。只看双方均能向量化的配置时，分别为 5.58×（kernel-only）和 5.07×（含 setup）。2MM、3MM、SYRK、SYR2K 的最大加速比分别达到 12.4×、11.5×、11.5×、10.8×；参考 RVV 编译器不能向量化的 GESUMMV、BICG、SpMV 案例最高达到 27.4×（第 5.2 节、图 9A）。

### 7.4 与 OoO 核心的性能比较

相对于 8-wide、按 AMD Zen 5 建模的 OoO RISC-V 核心，所有 benchmark 的平均加速比为 1.54×（仅 kernel 执行）和 1.43×（含 setup）。限制到双方都能向量化的 kernel 后，加速比分别为 1.42×和 1.29×；2MM、3MM、SYRK、SYR2K 的增益最高达到 6.3×，BICG 最高达到 9.2×。论文也报告少数 slowdown，原因包括向量循环之外的标量区域效率和 OoO 核心本身更高的计算吞吐（第 5.3 节、图 9B）。

## 8. 主要创新点

### 8.1 创新点一：Stream-Dataflow IR

论文定义了在 LLVM IR 与机器代码之间的专用 IR，用参数化数据流表达访存，用数据流图表达计算，并显式记录跨归纳变量/数据流的依赖。其价值在于把流式硬件需要的隐式访存语义纳入常规 LLVM 编译流程（第 3.2 节）。

### 8.2 创新点二：面向复杂模式的自动抽取

系统从 PHI、GetElementPtr、load/store 和操作数 DFS 中自动恢复多维、间接和依赖访问，而不是只处理简单仿射模式。表 1 的 31 个 benchmark 覆盖和 SpMV-ind 等复杂案例支持这一贡献的有效性（第 3.3、4.2 节）。

### 8.3 创新点三：把流专门化与向量化/循环变换结合

流感知向量化、计算图合并和流循环折叠共同作用，使编译器不仅识别流，还能减少循环控制、重复访存和索引计算。论文将这些优化纳入完整 LLVM middle-end/backend 流程（第 3.4、3.5 节）。

### 8.4 创新点四：面向最新 UVE 规范的完整代码生成与评测

论文指出既有 UVE 编译器使用过时 ISA 或不可复现工具链，因此实现了可处理当前 UVE 规范的协处理器模型，并将功能验证、指令数分析和周期级性能评测连成一体（第 4.3、5.1 节）。

## 9. 局限性

### 9.1 论文明确体现的局限

1. FMA 与寄存器累加在涉及 store stream 时尚未被编译器完全区分和支持，因此 3MM、Covariance 等案例与手写代码存在差距（第 4.3 节）。
2. 少数 scalar region、向量协处理器设置开销以及 OoO 核心更高的计算吞吐会造成 slowdown（第 5.3 节）。
3. 论文目标后端是 UVE；正文称 UVE 是当时唯一支持流专门化 SIMD 执行的 RISC-V 扩展，因此不能把结果直接等同于标准 RVV 硬件结果（第 3.5 节）。
4. 既有 UVE 工具链因旧 ISA 和不可复现问题无法进行直接比较，部分比较只能通过 RVV、SVE、手写代码和作者模拟器间接完成（第 4.2 节）。

### 9.2 阅读后的潜在局限

1. 性能结果主要来自 Spike、gem5/NDPmulator 和模型化 CPU，并非真实流向量芯片上的硬件测量；因此应表述为模拟性能。
2. 论文没有给出编译时间、内存开销、生成代码可移植性或错误恢复成本的系统实验。
3. 该方法依赖特定流向量 ISA 的指令与运行时语义；移植到标准 RVV、ARM SVE 或 GPU 需要新的后端映射，不是简单更换目标三元组。
4. 31 个 benchmark 的覆盖较广，但论文没有报告真实应用规模、跨函数优化或并行线程场景，结论不应外推到所有 HPC 工作负载。

## 10. 阅读后的研究方向反思

### 10.1 值得借鉴的思想

最值得借鉴的是把“复杂访存模式”从普通 SSA 细节提升为显式的、可优化的中间语义，再让多个优化 pass 围绕该语义协同工作。对 RISC-V/RVV 研究而言，这比仅增加一个后端 pattern 更接近编译器基础设施问题。

### 10.2 不能简单照搬的部分

S2VEC 的核心贡献是 Stream-Dataflow IR 和 UVE 后端，不是把 UVE 名称替换为 RVV。若只把目标 ISA 换成 RVV，通常属于工程移植；需要新增 RVV 的可变向量长度、掩码、segment/strided 访存和成本模型协同，才可能形成独立研究问题。

### 10.3 与当前方向的关系

本文适合作为 `SUPPORTING/B2` 编译器基础设施和 RISC-V 流/向量代码生成的参考。它没有 LLM、强化学习或 pass 选择模型，因此不能直接作为 LLM selector/translator 的方法论文；可以作为这些方法的后端执行器或可验证性能评测平台。

## 11. 可进一步尝试的研究方向

### 11.1 RVV-aware Stream-Dataflow Lowering

#### 研究问题

如何把 Stream-Dataflow IR 的数据流与依赖，映射到标准 RVV 的 `vl`、掩码、strided/segmented 访存和循环控制，同时保留对复杂间接访问的覆盖。

#### 与原论文的区别

不是更换目标名称，而是解决 UVE 隐式流指令与标准 RVV 显式向量内存语义之间的编译映射和成本权衡。

#### 可能的创新点

RVV-specific stream lowering、VL-aware 依赖切分、流与普通 RVV 指令的混合成本模型。

#### 实验框架

```text
LLVM IR → Stream-Dataflow IR → RVV/UVE 双后端 → Spike/gem5/真实 RVV 板卡 → 指令数与运行时间
```

#### 可行性与主要风险

可复用本文 IR 和 31 个 benchmark；风险是标准 RVV 缺少 UVE 的隐式访存机制，可能无法保持全部加速。

### 11.2 Learned stream-pattern cost model

#### 研究问题

在不同 RVV 向量长度、cache 层次和内存带宽下，如何预测某个数据流是否值得专门化，以及应使用何种向量化/循环折叠策略。

#### 与原论文的区别

原论文使用固定编译规则和模拟评测；该方向引入性能反馈模型来选择变换，但不改变 IR 的语义基础。

#### 可能的创新点

跨硬件 transfer learning、流依赖特征编码、编译开销与运行性能的多目标选择。

#### 实验框架

```text
Stream-Dataflow 特征 → 真实/模拟 RVV 运行样本 → 代价模型 → 变换选择 → 性能验证
```

#### 可行性与主要风险

可由本文 benchmark 生成样本；风险是模拟器与真实硬件的性能排序可能不一致。

### 11.3 Verified stream transformation

#### 研究问题

如何对数据流抽取、计算图合并和循环折叠后的 LLVM IR/汇编建立自动等价性或可检查不变量。

#### 与原论文的区别

原文进行 Spike 功能验证，但没有把每个编译变换表述为形式化证明问题；该方向关注转换正确性证据。

#### 可能的创新点

面向间接数据流的语义摘要、转换前后访问序列约束、与 LLVM translation validation 的结合。

#### 实验框架

```text
LLVM IR → 流摘要与依赖约束 → 变换/代码生成 → 验证器或差分执行 → 失败归因
```

#### 可行性与主要风险

可从第 3 节的 offset/size/step/dependency 定义出发；风险是动态 scatter-gather 和浮点归约的等价性较难完全证明。

## 12. 与其他已读文献的关系

本槽位本轮只完整处理 S2VEC 一篇论文，因此没有当前批次内可进行事实级横向对照的第二篇论文。就本仓库给定的研究范围而言，S2VEC 应被视为编译器基础设施/真实后端支撑材料：它提供 LLVM 中间表示、RISC-V 流向量代码生成和模拟评测平台，而不是 LLM pass 选择、源代码翻译或优化规则生成方法。与已有 LLM/RISC-V 条目的关系需以各自正文为依据，不能仅因都使用 LLVM 或 RISC-V 就认定重复。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLVM 驱动的流专门化与线性化向量化编译 |
| 核心问题 | 复杂/间接内存访问难以被传统自动向量化器利用 |
| 输入 | 可降低到 LLVM IR 的普通程序循环 |
| 输出 | 面向 UVE RISC-V 流向量 ISA 的专门化代码 |
| 核心方法 | Stream-Dataflow IR + 数据流抽取 + 向量化/图合并/循环折叠 |
| 使用的模型 | 无机器学习模型；使用 LLVM、Spike、gem5/NDPmulator 与核模型 |
| 使用的编译器工具 | Clang、LLVM IR、LLVM 21 对照编译、定制 LLVM passes |
| 是否使用强化学习 | 否；本文没有使用强化学习，因此不存在奖励函数 |
| 是否使用形式化验证 | 否；使用修改后的 Spike 做功能验证，不等同于形式化证明 |
| 数据集规模 | 31 个 PolyBench、MachSuite 和自定义 benchmark |
| 主要指标 | 向量化覆盖、已退休指令减少、相对 in-order/OoO 加速比 |
| 最重要实验结果 | 相对等配置 in-order 核心平均最高 5.93× kernel-only；相对 OoO 核心平均 1.54× kernel-only；正文摘要概括为 5.9× 与 1.5× |
| 核心创新 | 适配流式执行的 IR、复杂流自动抽取、流感知向量化与 UVE 代码生成 |
| 主要局限 | 依赖 UVE 流式 ISA 和模拟器；FMA、真实硬件与编译开销仍有限制 |
| 与 RISC-V 研究的相关性 | 高：正文直接面向 UVE RISC-V，并与 RVV 代码及 RVV 核心比较；但不是标准 RVV 后端论文 |
| 最适合作为 | SUPPORTING/B2 编译器基础设施、RISC-V 流/向量后端和评测参考 |

这篇论文最值得学习的是：用显式中间表示把复杂内存访问、数据流依赖和向量化统一起来；最主要的局限是结果依赖 UVE 语义和模拟硬件。如果用于后续研究，最合理的方式是把它作为 RISC-V 后端/评测基础，再研究标准 RVV 映射、跨硬件成本模型或变换验证，而不是简单地把 UVE 名称替换成 RVV。
