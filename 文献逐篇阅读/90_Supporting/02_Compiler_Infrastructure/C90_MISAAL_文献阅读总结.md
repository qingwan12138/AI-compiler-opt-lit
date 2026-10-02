# MISAAL 文献阅读总结

论文题目：**MISAAL: Synthesis-Based Automatic Generation of Efficient and Retargetable Semantics-Driven Optimizations**

作者：Abdul Rafae Noor、Dhruv Baronia、Akash Kothari、Muchen Xu、Charith Mendis、Vikram S. Adve

发表时间：2025 年

发表平台：PLDI 2025 / PACMPL 9(PLDI), Article 198

论文链接或编号：DOI 10.1145/3729301
元数据核验来源：[PLDI 官方论文页](https://pldi25.sigplan.org/details/pldi-2025-papers/52/MISAAL-Synthesis-Based-Automatic-Generation-of-Efficient-and-Retargetable-Semantics-)；[ACM DOI](https://doi.org/10.1145/3729301)
代码/数据/工件：作者公开代码仓库：[RafaeNoor/MISAAL](https://github.com/RafaeNoor/MISAAL)。

关键词：程序综合、编译器代码生成、形式语义、重写规则、等价饱和、向量指令、可重定向编译器、多硬件

> 本笔记只依据 staging 中核验过的 24 页作者公开 PDF；论文事实与阅读后的分析分开描述。

---

## 1. 研究背景

深度学习、图像处理和张量/随机访问工作负载需要利用 x86、Qualcomm Hexagon HVX、ARM Neon 等 ISA 的专用向量、点积、归约和数据搬移指令。传统 Halide、TVM、MLIR 等编译器通常为每个目标手工编写模式匹配后端，存在工程成本高、规则脆弱、容易漏掉高性能匹配和维护困难的问题（第 1 节）。

近年来，Diospyros、Isaria、Rake、Pitchfork、Hydride 等工作用程序综合替代部分手工后端，并可利用形式语义验证生成代码。但仍有三类瓶颈：大 ISA 导致指数级枚举；跨 lane 计算与隐式数据 swizzle 需要长序列和复杂布局；离线生成的大量规则会让编译期重写和 e-graph 爆炸。Hydride 能覆盖较大的 ISA，但对单个表达式的综合仍可能耗时数分钟到数小时。

因此本文研究的是：如何把目标 ISA 的形式语义用于压缩综合搜索空间、自动发现复杂 swizzle，并把离线综合结果压缩成编译期可快速应用的重写规则。

## 2. 论文要解决的问题

### 2.1 大 ISA 上的程序综合不可扩展

直接在数千条具体指令上枚举表达式会产生极大的搜索空间。论文以 x86 为例：深度 2 的具体指令表达式约有 3×10^12 项，使用 AutoLLVM 等价类后约为 2.9×10^8 项，但更深表达式仍超过 10^32 项（第 3.4 节）。

### 2.2 复杂跨 lane 指令需要数据重排

点积、widening multiplication 等指令的伪代码可能以交错或非连续方式读取向量元素。若只综合算术操作，无法生成正确且高效的输入布局；手工设计 swizzle 又依赖专家知识，难以迁移到新 ISA。

### 2.3 规则数量和编译期代价过大

即使把综合放到离线阶段，具体目标指令、向量宽度和元素位宽的组合仍会产生大量规则。论文要在保留覆盖范围的同时，把这些规则抽象成参数化、目标无关的规则，并用轻量重写替代在线 SMT 综合。

> 本文主要研究：如何从输入 IR 和目标 ISA 的形式语义自动生成可验证、可重定向、能在秒级编译中应用的向量代码生成重写规则。

## 3. 核心方法概述

MISAAL 在 Hydride 的自动 ISA 语义和 AutoLLVM IR 基础上，把流程分为离线编译器构造和在线轻量编译两阶段。AutoLLVM IR 用参数化等价类表示一组语义相似的具体目标指令；MISAAL 在该压缩表示上综合规则，自动从 ISA 语义派生复杂 swizzle，用 RDSL（Relevance Domain-Specific Language）建立相关操作集合，再抽象为目标无关规则。

```text
厂商 ISA 伪代码 + Halide/前端 IR 形式语义
        ↓
Hydride 生成 ISA 语义与 AutoLLVM 等价类
        ↓
AutoLLVM 等价类枚举 + Concretization Grammar
        ↓
从 bit-vector 访问模式自动发现复杂 swizzle
        ↓
用 RDSL 与语义相关集合剪枝搜索空间
        ↓
验证并抽象具体规则为参数化、目标无关规则
        ↓
在线 EggLog 等价饱和重写 Halide IR → AutoLLVM IR
        ↓
下沉到 x86 / HVX / ARM 的 LLVM intrinsic 或后端指令
```

LLM 不在该系统中扮演角色；本文也不使用机器学习模型。编译器的核心输出是自动生成的重写规则和目标指令选择结果。

## 4. 实验框架与训练流程

本文不涉及模型训练、SFT、强化学习或提示词推理，主要采用程序综合和规则重写。

### 4.1 离线编译器构造

1. Hydride 从 x86、HVX、ARM 的 ISA 伪代码生成形式语义，并把相似指令折叠为参数化 AutoLLVM IR。
2. MISAAL 用 AutoLLVM 等价类而不是具体 ISA 指令进行语法引导枚举；Concretization Grammar 用 `choose*` 表示具体化选择，并以符号输入等价性寻找合法重写模板（第 3.1 节）。
3. 对每条指令分析输出 lane 访问的 bit-vector slices，自动生成 interleave、deinterleave 和 packing 等复杂 swizzle 的 AutoLLVM 类（第 3.3 节）。
4. 根据目标指令的语义生成 Compute-Only 相关集合。RDSL 仅包含向量/位向量算术、扩展、截取、饱和、归约等操作，从而排除与目标语义无关的操作（第 3.4 节）。
5. 对 concrete patterns 做参数抽象，生成跨向量宽度、元素位宽和目标架构可复用的 rewrite rules，并自动验证其正确性。

### 4.2 在线编译

EggLog 读取 Halide IR 重写规则和目标相关 AutoLLVM 规则，运行若干轮等价饱和，优先把 Halide IR 转换为 AutoLLVM IR；若存在抽象 swizzle，再用 swizzle lowering 规则把它们合法化为目标指令序列。

## 5. 奖励函数、损失函数或关键公式

本文没有强化学习奖励函数，也没有神经网络损失函数。关键目标是语义等价和编译代价，而不是学习奖励。

### 5.1 重写规则等价

论文把规则表示为 `(LHS, RHS, ≃)`。传统具体规则要求所有符号输入上 `LHS ≡ RHS`。MISAAL 在 AutoLLVM 层定义模板有效性：

```text
∃ l ∈ Concretization(LHS_IR), ∃ r ∈ Concretization(RHS_IR), 使 l ≡ r
```

这意味着模板只需存在一组具体化使左右表达式等价，随后由具体化和抽象过程产生可用规则；符号等价性通过 SyGuS/Rosette 求解。

### 5.2 相关集合

对目标表达式 `R`，若某个 AutoLLVM 指令 `I` 能作为 RDSL 或自身的一部分参与生成与 `R` 语义等价的表达式 `E`，则把 `I` 放入 `RelevanceSet(R)`。该集合用于限制枚举，不是运行时奖励。

### 5.3 代价设定

在线 EggLog 中，Halide IR 操作成本设为 10,000，AutoLLVM IR 操作成本设为 1；先进行 5 轮等价饱和，直到提取结果只含 AutoLLVM IR，再处理 swizzle lowering。该成本是重写提取偏好，不是学习损失。

## 6. 实验设置

### 6.1 数据集来源

论文没有使用传统训练/测试数据集。评估使用 33 个图像处理和深度学习 Halide kernel，包括 dilation、blur、edge detection、矩阵乘、pooling、卷积、softmax，以及融合的 `matmul + bias + activation` 等。Halide schedule 经人工调优；论文说明 ARM/HVX 的部分基准由 Qualcomm 和 Adobe 调优。

### 6.2 模型与工具

| 项目 | 论文设置 |
|---|---|
| 编译器/框架 | MISAAL、Hydride、Rake、Halide 13、EggLog、Rosette |
| 前端 | Halide IR |
| 目标 | x86、Qualcomm Hexagon HVX、ARM |
| x86 硬件 | Intel Xeon Silver 4216，16 核、2.1 GHz，关闭超线程 |
| HVX 评估 | Qualcomm Hexagon SDK 3.5.2 的 cycle-accurate simulator |
| ARM 硬件 | Apple M2，3.49 GHz，16 GB 内存，16 MB L3 |
| 离线生成 | AMD EPYC 7453，28 核，2.7 GHz；枚举流程并行化为 64 个进程 |

论文未报告 LLM、训练框架或模型参数，因为本文不使用模型训练。

### 6.3 对比方法

主要对比 Halide 的手工优化后端、Hydride 和 Rake。Isaria 因不支持这些目标架构而未比较；Pitchfork 因不支持数据 swizzle 生成而未比较。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| 编译时间 | 在线编译或离线构造组件所需秒/分钟/小时 | 越小越好 |
| 峰值内存 | 编译过程峰值物理内存 | 越小越好 |
| 相对性能/Speedup | MISAAL 或 Hydride 相对 Halide 的运行性能 | 越大越好 |
| 规则数量 | concrete 与 abstract rewrite rules 的数量 | 越少通常越易维护/应用 |
| 搜索空间 | 枚举项数量或压缩倍数 | 越小越易扩展 |

## 7. 实验结果与结论

### 7.1 主要结果

在 33 个 benchmark 的几何平均上，相对 Hydride，MISAAL 的在线编译时间在 x86、HVX、ARM 分别降低约 16×、9×、10×；极端情况下，对需要复杂 swizzle 的 kernel，Hydride 超过 5 小时，而 MISAAL 为秒级，最高降低 224×（第 5.2 节）。相对 Rake 的 HVX 小集合，MISAAL 几何平均为 272.2 秒对 Rake 的 1784 秒（约 6.8×），且 Rake 无法编译 33 个 benchmark 中的 27 个（表 3）。

### 7.2 内存与搜索空间

相对 Hydride，MISAAL 在 x86 和 ARM 的几何平均峰值内存分别约降低 18×和 26×，HVX 约降低 3×。Hydride 在某些 x86 benchmark 达到约 15 GB；MISAAL 的最大内存约 2.5 GB，但离线生成组件的持续峰值约 8 GB。x86 深度 2 的枚举从约 3×10^12 项降到 2.9×10^8 项，约 9000×。

### 7.3 运行性能

- x86：相对 Halide 后端几何平均提升约 10%，最大加速 2.08×；MISAAL 能识别点积可复用于 widening multiply-add，并减少部分 shuffle。
- HVX：相对 Halide 几何平均约 0.98×，总体接近；但在部分高斯/卷积 kernel 上较慢，Halide 的人工规则仍能更激进地融合 swizzle 和生成 4-point dot product。
- ARM：相对 Halide 几何平均约 1.02×，最大 1.21×；MISAAL 能更灵活地识别 dot product 和 `umlal` 等组合。

这些是运行性能结果；HVX 使用 cycle-accurate simulator，不应表述为真实 HVX 芯片测量。

### 7.4 规则抽象与构造成本

自动派生的复杂 swizzle 从 x86/HVX/ARM 共 376 个 concrete swizzle 压缩到 27 个 AutoLLVM 类，约 13.9×；具体规则总数从 41,841 个压缩到 2,215 个，约 18.9×（表 2、表 5）。其中 x86、HVX、ARM concrete rewrite 分别为 5,806、4,586、4,085，抽象后为 329、412、190；Halide 规则为 27,364→1,284。

代价是离线枚举仍很重：论文报告深度 4、最多 5 个 terminal 的 AutoLLVM 枚举每个目标超过 1 个月；compute-only relevance 约 10–21 小时，具体规则提取约 3.5–6.3 小时。作者明确指出仍需进一步工程优化。

## 8. 主要创新点

### 8.1 创新点一：基于等价类的可扩展枚举

本文不是直接枚举数千个目标指令，而是在 Hydride 自动生成的 AutoLLVM 参数化等价类上枚举，再用 Concretization Grammar 找具体化。价值在于保留目标指令覆盖范围的同时减少重复结构；表 1 显示 x86 的 2,029 条 ISA 指令被表示为 136 个 AutoLLVM 类。

### 8.2 创新点二：从 ISA 语义自动发现复杂 swizzle

论文从伪代码的 bit-vector 访问模式推导 interleave、deinterleave 和 packing，避免为每个新架构手工制作 swizzle。这个机制使复杂跨 lane 指令进入综合搜索，同时把派生 swizzle 再折叠为 AutoLLVM 类。

### 8.3 创新点三：语义驱动的 relevance pruning

RDSL 和 `BVSlices`、`UniqueLanes`、reduction factor 等语义属性共同构造与目标指令相关的操作集合；它不是依赖人工为每条 ISA 编写启发式操作表，而是用小位宽/小 lane 的综合问题自动判定相关性。

### 8.4 创新点四：目标无关规则抽象和轻量在线编译

具体化后的规则被抽象为带符号向量宽度、元素位宽和操作变体的参数化规则，并在在线阶段使用 EggLog 等价饱和而非 SMT 综合。实验显示规则数量约 18.9×压缩，同时保持跨 x86、HVX、ARM 的竞争性性能。

## 9. 局限性

### 9.1 论文明确暴露的局限

1. 离线枚举成本仍然很高：论文报告深度 4 的每目标枚举超过 1 个月；作者建议进一步消除交换律、结合律和分配律变体。
2. MISAAL 的在线编译对简单 kernel 可能有固定规则载入开销，因此个别 benchmark 比 Hydride 更慢，例如 HVX 的 max pool。
3. HVX 上若需要更大的向量寄存器，MISAAL 可能在较小寄存器上执行加法并缺少拼接，导致部分 kernel 慢于 Halide。
4. 评估只覆盖 x86、HVX 和 ARM，以及 Halide/图像处理/深度学习 kernel；论文没有给出 RISC-V、GPU 或通用 C/C++ 编译的实验。

### 9.2 阅读后发现的潜在局限

1. 自动生成的语义依赖厂商伪代码质量和 Hydride 的语义抽象；论文没有证明所有 ISA 语义都能无损解析。
2. 规则正确性主要依赖符号语义等价，但运行性能仍需要具体硬件、后端 lowering 和调度共同决定；形式等价不等于性能最优。
3. 目标无关抽象可能隐藏架构特有代价。论文已经展示 HVX 的复杂布局和手工 Halide 后端仍有优势，因此迁移到 RISC-V 向量扩展时不能只替换 ISA 名称。
4. 论文没有使用 LLM、强化学习或在线反馈学习，不能直接作为 LLM 编译优化系统的训练方法。

## 10. 阅读后的研究方向反思

MISAAL 对“大语言模型与编译器优化、LLVM IR、RISC-V、多架构优化”的直接启发在于：把 LLM 生成候选与语义约束/符号验证分离，让模型提出高层候选，编译器 IR、ISA 语义和验证器负责筛选正确性。本文真正的核心贡献是语义驱动综合、swizzle 派生和规则压缩；不能把它简单改写成“把 x86 换成 RISC-V”或“把 Rosette 换成 LLM”。

在 Taxonomy v2 中，建议类别为 **SUPPORTING / B2_Compiler_Infrastructure**：它是传统程序综合编译器基础设施，最终输出是重写规则和代码生成能力，不是 LLM 的 Selector、Translator 或 Generator 角色。与 RISC-V 的相关性为中等：它提供了面向新 ISA 的语义抽象、规则自动生成和 retargeting 方法论，但正文未评估 RISC-V。

## 11. 可进一步尝试的研究方向

### 11.1 方向一：面向 RVV 的语义驱动规则生成

#### 研究问题

RISC-V Vector Extension 的可变向量长度、mask、tail policy 和 LMUL 参数如何进入 AutoLLVM 类、swizzle 派生与 relevance pruning？

#### 与原论文的区别

不是只替换目标 ISA，而是研究 RVV 的 VLEN/LMUL/mask/tail 语义如何改变等价类和规则抽象，并在真实 RVV 硬件或模拟器上验证。

#### 可能的创新点

设计能表达可变向量长度与 mask 状态的参数化 IR；定义跨 VLEN 的规则有效性；把 RVV 特有的 tail/mask 行为纳入语义剪枝。

#### 实验框架

```text
RVV 规范/伪代码 → 形式语义 → 参数化等价类 → swizzle/relevance 综合
→ LLVM/RVV lowering → QEMU 或真实 RVV 板卡 → 正确性与性能评估
```

#### 可行性

需要 LLVM RVV 后端、RVV 模拟器或开发板、SMT/SyGuS 与一组向量 kernel。

#### 主要风险

RVV 的动态 VL 和 mask 状态可能使规则参数化爆炸；模拟器性能也可能不能代表真实硬件。

### 11.2 方向二：LLM 提议、MISAAL 验证的混合规则发现

#### 研究问题

LLM 能否从 IR/ISA 语义提出更有希望的 rewrite skeleton，而 MISAAL 的符号等价检查、relevance set 和 EggLog 负责正确性与最终选择？

#### 与原论文的区别

LLM 只作为候选排序/搜索提议器，不替代论文中的形式语义和等价性判定。

#### 可能的创新点

用语义摘要和历史规则作为上下文；用验证失败反例进行局部修复；比较 LLM 引导、随机枚举和传统启发式在相同验证预算下的覆盖与成本。

#### 实验框架

```text
IR/ISA 语义 → LLM 生成候选 skeleton → SyGuS/SMT 等价验证
→ 反例反馈修复 → 规则抽象 → EggLog 编译 → 真实性能测试
```

#### 可行性

可复用 MISAAL 的 Rosette、EggLog 和 Halide 接口，但需要明确的规则语法和验证预算。

#### 主要风险

LLM 可能生成语法正确但语义无效的规则；验证成本和提示词泄漏会影响公平比较，且不能把有限验证写成全面证明。

### 11.3 方向三：语义与硬件代价联合的规则选择

#### 研究问题

如何在保持形式等价的前提下，将寄存器压力、swizzle 成本、吞吐量和代码尺寸纳入规则提取？

#### 与原论文的区别

重点从“能生成等价规则”扩展到“在目标微架构上选择更合适的规则”，而不是单纯增加 ISA 覆盖。

#### 可能的创新点

构造跨架构的 cost model；对 RVV、HVX 等复杂向量布局加入 swizzle 代价；在 EggLog extraction 中实现多目标 Pareto 选择。

#### 实验框架

```text
等价规则集合 → 静态资源/硬件计数器特征 → 多目标 extraction
→ 真实硬件测量 → 校准 cost model → 重新选择规则
```

#### 可行性

需要 LLVM/目标汇编分析、性能计数器或微基准，并可从本文 33 个 kernel 开始。

#### 主要风险

静态估计可能与真实运行时间不一致；不同输入尺寸和调度会改变最优规则。

## 12. 与其他已读文献的关系

本 slot 只完成一篇论文，因此没有把其他论文的未核验内容写入横向比较。就研究定位而言，MISAAL 可作为传统编译器基础设施和语义驱动代码生成 baseline，与 LLM pass 选择、源代码翻译、LLM 测试生成系统形成互补：MISAAL 负责从语义生成和应用可验证规则，而 LLM 系统可以在其上层负责候选提议或规则优先级选择。它不是 RISC-V 实验的直接 baseline，除非先实现 RVV 语义与后端接口。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 从 ISA 形式语义自动生成可重定向的向量代码生成优化规则 |
| 核心问题 | 大 ISA 枚举、复杂 swizzle、在线综合和规则数量不可扩展 |
| 输入 | Halide IR、厂商 ISA 伪代码、Hydride/AutoLLVM 形式语义 |
| 输出 | 参数化 rewrite rules、AutoLLVM IR、目标 ISA 代码 |
| 核心方法 | 等价类枚举、swizzle 自动派生、RDSL relevance pruning、规则抽象、EggLog 重写 |
| 使用的模型 | 无神经网络模型；使用 Rosette/SyGuS 与程序综合 |
| 使用的编译器工具 | Hydride、Halide 13、EggLog、Rosette、LLVM intrinsic/backend |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 是；使用符号语义等价/综合约束验证 rewrite rules |
| 数据集规模 | 33 个图像处理和深度学习 kernel；非训练数据集 |
| 主要指标 | 编译时间、峰值内存、运行性能、规则数量和搜索空间压缩 |
| 最重要实验结果 | 相对 Hydride 在线编译几何平均约快 16×/9×/10×；规则 41,841→2,215，约 18.9× |
| 核心创新 | 从 ISA 语义自动生成复杂 swizzle 与 relevance set，并将具体规则抽象为目标无关规则 |
| 主要局限 | 离线深度枚举仍可能超过 1 个月；HVX 部分性能落后手工 Halide；未评估 RISC-V |
| 与 RISC-V 研究的相关性 | 中：方法适合新 ISA retargeting，但正文没有 RVV 实验 |
| 最适合作为 | 传统编译器基础设施 baseline、语义验证模块、规则生成/压缩方法参考 |

> 这篇论文最值得学习的是把目标 ISA 的形式语义直接用于搜索空间压缩、复杂数据布局发现和规则抽象；最主要的局限是离线构造仍很昂贵且依赖特定 ISA 语义基础设施；如果用于后续研究，最合理的方式是作为 RVV 或 LLM-验证混合系统的规则生成与正确性基线，而不是简单替换为 RISC-V 或把有限符号检查宣传为完整编译器证明。
