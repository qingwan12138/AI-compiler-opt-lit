# MOKA 文献阅读总结

论文题目：**To Cross, or Not to Cross Pages for Prefetching?**

作者：Georgios Vavouliotis、Martí Torrents、Boris Grot、Kleovoulos Kalaitzidis、Leeor Peled、Marc Casas

发表时间：2025

发表平台：HPCA 2025, pp.188–203

论文链接或编号：DOI 10.1109/HPCA61900.2025.00025
元数据核验来源：[IEEE DOI](https://doi.org/10.1109/HPCA61900.2025.00025)
代码/数据/工件：未找到可确认的作者代码仓库或工件

PDF：`MOKA_HPCA2025_To_Cross_or_Not_to_Cross_Pages_for_Prefetching.pdf`

关键词：页面跨界预取、L1D 预取器、VIPT cache、TLB、感知器预测器、运行时自适应阈值、硬件-软件协同优化

> 本笔记依据公开的 16 页同行评审版本全文整理。论文事实、阅读后的分析和后续建议分开描述。

---

## 1. 研究背景

本文研究第一层数据缓存（L1D）旁路的硬件数据预取器。L1D 通常采用 VIPT（Virtually Indexed, Physically Tagged，虚拟索引、物理标记）缓存，因此预取器可在虚拟地址空间中判断访问模式。若下一次预取跨过虚拟页边界，预取器可能需要查询 TLB（Translation Lookaside Buffer，地址转换缓存），甚至触发页表遍历。

页面跨界预取有双重效应：准确时可以提前带来下一页的地址转换和数据，减少 TLB/cache miss 并改善预取及时性；不准确时则可能引入最多 4 次推测性页表遍历访问和 1 次缓存预取访问，造成缓存和 TLB 污染、额外带宽与能耗。论文指出，已有学术 L1D 预取器通常直接丢弃页面跨界请求，而厂商处理方式没有公开说明。

第 2 节的实证动机表明，不同 workload 和执行阶段对页面跨界预取的收益方向不一致：固定“总允许”和固定“总丢弃”都不是普适最优策略。论文因此把问题从“设计更激进的预取器”转为“在已有预取器之后增加一个能够识别页面跨界请求价值的过滤器”。

## 2. 论文要解决的问题

### 2.1 静态页面跨界策略不适应异构 workload

论文在 Berti、BOP、IPCP 三种 L1D 预取器上比较总允许（Permit PGC）与总丢弃（Discard PGC）。有些程序从跨页请求中获益，有些程序因页表遍历和污染而受损；同一程序也可能在不同执行阶段表现不同。

### 2.2 既要保留有效跨页请求，又要避免有害请求

一个理想过滤器应只让可能带来真实 L1D demand hit 的页面跨界预取通过，同时不破坏原有预取器的工作方式。论文特别关注对误丢弃请求（false negative）的反馈，因为被过滤掉的请求如果后来对应 demand miss，说明过滤器可能过于保守。

### 2.3 需要低硬件开销并适应阶段变化

过滤器不能依赖庞大的存储结构，也不能用固定阈值覆盖所有工作负载。论文研究如何组合程序特征、系统状态特征和基于 epoch 的阈值调整。

> 本文主要研究：如何在 VIPT L1D 预取器旁增加低开销、可在线更新的页面跨界过滤机制，使跨页预取在不同 workload 和执行阶段中更加有效。

## 3. 核心方法概述

论文提出 MOKA（用于设计 Page-Cross Filter 的框架），并用它实现 DRIPPER 原型。MOKA 使用针对程序特征的哈希感知器预测器、针对系统状态的饱和计数器，以及运行时自适应激活阈值。最终系统不替换 Berti、BOP 或 IPCP，而是在它们产生跨页请求之后决定 issue 或 discard。

```text
L1D 访问
    ↓
已有 L1D 预取器生成请求
    ↓
判断是否跨越虚拟页边界
    ├─ 否：按原有路径处理
    └─ 是：MOKA/DRIPPER 读取程序特征与系统特征
              ↓
       累加权重并与激活阈值比较
              ├─ 通过：查询 TLB，必要时触发推测性页表遍历并发出预取
              └─ 丢弃：记录到 vUB，等待后续 demand miss 反馈
    ↓
L1D hit/eviction 反馈更新权重
    ↓
epoch 统计更新阈值
```

MOKA 的关键硬件组件如下：

* Weight Tables（WT）：保存程序特征对应的感知器权重，权重由饱和计数器实现。
* System Feature Weights：保存系统状态特征的权重，例如 sTLB MPKI 或 miss rate。
* Virtual Update Buffer（vUB）：保存被过滤的虚拟地址及哈希索引，用于识别误丢弃的跨页请求。
* Physical Update Buffer（pUB）：保存已发出请求的物理地址及哈希索引，用于根据实际 cache 命中和驱逐结果更新权重。
* Adaptive Thresholding Scheme：根据准确率、IPC、LLC miss rate、ROB 压力和 L1I MPKI 等运行时统计调整激活阈值。

论文中的“训练”是在线硬件权重更新，不是神经网络离线训练，也没有 LLM、SFT、PPO 或 GRPO。

## 4. 实验框架与训练流程

本文不涉及模型预训练或机器学习模型离线训练，主要采用 ChampSim 中的微架构模拟和在线预测器权重更新。

### 4.1 离线特征探索

MOKA 框架共设计 55 个程序特征。作者先在 218 个已见 workload 上评估单特征过滤器，再按 geomean IPC speedup 排序，逐步加入能使 geomean IPC 额外提升超过 0.3% 的特征。优化指标选用 IPC speedup，而不是单独最大化 coverage 或 accuracy，因为不同 miss 的性能关键性不同，单纯追求 accuracy 可能牺牲性能。

### 4.2 在线预测

对于每个跨页请求，系统提取程序特征并哈希到相应 WT，读取程序特征权重；如果系统特征超过其条件阈值，则加入相应 system-feature weight。所有有效权重求和得到 `wfinal`，当 `wfinal > Ta` 时发出预取，否则丢弃。

### 4.3 在线权重更新

* 被过滤的请求后来导致 L1D demand miss：vUB 命中，相关程序和系统权重增加，表示过滤器出现 false negative。
* 被发出的请求后来被 L1D demand hit 使用：pUB 命中，相关权重增加。
* 被发出的请求在驱逐前没有产生任何 L1D hit：相关权重减少，表示该请求无效。

作者给每个 L1D block 增加 Page Cross Bit（PCB），以标记该 block 是否由跨页预取填充，从而在 demand hit 和 eviction 时进行对应更新。

### 4.4 epoch 阈值调整

epoch 内统计 useful/useless page-cross prefetch 数、IPC、LLC miss rate、ROB pressure 和 L1I MPKI。高 ROB 压力且有大量未完成 L1D miss，或准确率低于阈值时，系统提高 `Ta`；高 L1I 压力时使用中等阈值；极高 LLC 压力时暂时禁用跨页预取。epoch 结束后根据准确率变化调整 `Ta`，若 IPC 下降也会将阈值至少调到中等水平。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有神经网络损失函数。其核心是一个基于阈值的在线预测目标：

```text
wfinal = Σ(program-feature weights) + Σ(active system-feature weights)
若 wfinal > Ta：发出页面跨界预取
否则：丢弃页面跨界预取
```

在线更新不是梯度下降：

* 有效预取或此前误丢弃的请求：相关权重增加。
* 发出但在驱逐前没有命中需求访问的预取：相关权重减少。

论文把“useful prefetch”定义为：该跨页预取在其生命周期内至少服务过一次 L1D demand access。实验性能优化的最终目标是 IPC，而不是只提高 useful 请求比例。

## 6. 实验设置

### 6.1 数据集来源

论文使用 ChampSim 的 trace-driven 微架构模拟，不是传统意义上的训练数据集。workload 来源包括：SPEC CPU2006、SPEC CPU2017、Geekbench 5、GAP、Ligra、PARSEC，以及 Qualcomm 提供的 CVP-1 工业整数和浮点 workload。主要实验使用 LLC MPKI 至少为 1 的 memory-intensive workload，共 396 个：218 个用于 DRIPPER 设计的 seen workload，178 个未参与设计的 unseen workload。另有覆盖完整 benchmark suite 的非 memory-intensive workload 评估。

单核实验按 workload 使用 SimPoint trace；SPEC、GAP、PARSEC、Ligra 通常先 warm-up 250M instructions，再执行 250M instructions；Qualcomm workload 使用 50M warm-up 和 100M simulation。多核实验使用 300 个随机生成的 8-core mixes。

### 6.2 模型与工具

| 项目 | 设置 |
|---|---|
| 模拟器 | ChampSim |
| L1D 预取器 | Berti、BOP、IPCP |
| CPU | 1–8 cores，4 GHz，352-entry ROB，6-wide issue |
| TLB | dTLB 64-entry、sTLB 1536-entry，含 page-table walker |
| Cache | L1I 32 KB、L1D 48 KB、L2 512 KB、每核 LLC 2 MB |
| 页面 | 主要为 4 KB；另评估 4 KB 与 2 MB 混合页面 |
| 主存 | 单核 4 GB，8 核 16 GB，3200 MT/s |
| 过滤器 | MOKA 框架的 DRIPPER 原型 |
| 存储开销 | 每核 1.44 KB |

该论文不使用 LLVM、GCC、MLIR、LLM 或真实硬件执行；结果是基于 ChampSim 的微架构模拟结果。

### 6.3 对比方法

* **Discard PGC**：始终丢弃页面跨界预取，是主要 baseline。
* **Permit PGC**：始终允许页面跨界预取。
* **Discard PTW**：只有目标转换已在 TLB 中时才允许跨页预取，否则丢弃。
* **ISO Storage**：把 DRIPPER 的存储预算用于扩大底层预取器的相关结构。
* **PPF**：将 Perceptron-Based Prefetch Filtering 改造为页面跨界过滤器。
* **PPF+Dthr**：在 PPF 上加入 MOKA 的动态阈值机制。
* **DRIPPER-SF**：只使用选中的 system features，用于消融程序特征的作用。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| IPC speedup | 相对 Discard PGC 的指令吞吐提升；单核使用 weighted geomean，多核使用 normalized weighted speedup | 越大越好 |
| Prefetch coverage | 预取覆盖的 demand miss 比例 | 越大越好 |
| Prefetch accuracy | 发出的预取中实际有用的比例 | 越大越好 |
| MPKI | 每千条指令的 dTLB、sTLB、L1D 或 LLC miss 数 | 越小越好 |
| Storage overhead | 每核新增过滤器硬件存储 | 越小越好 |

## 7. 实验结果与结论

### 7.1 主要结果

在 Berti 上，DRIPPER 相对 Discard PGC 的单核 geomean IPC speedup 为：218 个 seen workload 上 +1.7%，178 个 unseen workload 上 +1.2%；相对 Permit PGC 则分别高 2.5% 和 2.1%。在 300 个随机 8-core mixes 上，DRIPPER 相对 Discard PGC 的 geomean speedup 为 2.0%，相对 Permit PGC 为 3.3%。

在所有考虑的 Berti、BOP、IPCP 预取器上，DRIPPER 都获得最高的 geomean IPC gains。论文报告其每核存储开销为 1.44 KB。

### 7.2 与静态策略和存储扩展比较

Permit PGC 在少数 workload 上有效，但总体因错误跨页请求造成损失；Discard PGC 更保守。Discard PTW 能避免部分推测性页表遍历，但会错过目标转换不在 TLB 中、同时又有价值的请求。ISO Storage 与 Permit PGC 性能相近，说明单纯扩大已有预取器存储不能解决页面跨界判断问题。

### 7.3 过滤准确性与缓存/TLB 影响

以 Berti 为例，Permit PGC 和 DRIPPER 在所有 workload 上相对 Discard PGC 的平均 miss coverage 提升分别为 4.2% 和 4.1%，说明 DRIPPER 基本保留了有效跨页预取带来的 coverage。平均 accuracy 方面，DRIPPER 相对 Discard PGC 提升 1.2%，而 Permit PGC 下降 2.6%。DRIPPER 相对 Discard PGC 平均降低 dTLB、sTLB、L1D、LLC MPKI 的绝对值分别为 0.6、0.1、2.1、0.2。

### 7.4 消融实验

DRIPPER 的特征选择为：Berti 使用 `Delta`、`sTLB MPKI`、`sTLB Miss Rate`；BOP 和 IPCP 使用 `PC XOR Delta`、`sTLB MPKI`、`sTLB Miss Rate`。单特征对比表明组合特征优于单独使用 Delta、PC XOR Delta 或系统特征。DRIPPER 相对只使用 system features 的 DRIPPER-SF，在 218 个 workload 上额外提高 0.9% geomean IPC。

相对转换后的 PPF，DRIPPER 在 Berti、BOP、IPCP 上分别高 2.4%、1.4%、1.6% geomean IPC。论文将差异归因于 DRIPPER 同时使用系统特征、预取器无关的程序特征和动态激活阈值。

### 7.5 页面大小、L2 预取器和未见 workload

在同时使用 4 KB 和 2 MB 页面时，DRIPPER 相对 Discard PGC 的 geomean IPC uplift 为 1.3%，相对 Permit PGC 为 2.2%。超过 99% 的跨页预取并不跨越 2 MB 边界，因此在 4 KB 边界进行过滤仍有价值。加入不同 L2 预取器（无、SPP、IPCP、BOP）后，DRIPPER 仍保持最高 speedup 趋势。

对 178 个设计阶段未见过的 workload，DRIPPER 相对 Permit PGC 和 Discard PGC 的 geomean 优势分别为 2.1% 和 1.2%。这说明论文报告的收益并不只来自对 218 个 seen workload 的特征选择。

### 7.6 论文结论

作者的结论是：页面跨界预取不应采用对所有 workload 一律允许或一律拒绝的静态策略；结合程序特征、系统状态和在线阈值调整的 DRIPPER 能以较低硬件开销保留有效请求并减少有害请求。

## 8. 主要创新点

### 8.1 创新点一：首次系统分析 VIPT L1D 的页面跨界预取

论文在三种 L1D 预取器和 396 个 workload 上展示了跨页预取收益高度依赖 workload 和执行阶段，并区分了 coverage、accuracy、TLB/cache MPKI 与 IPC 的关系。这是问题定义和设计空间分析，而非单纯增加一个预取器。

### 8.2 创新点二：MOKA 的程序特征与系统状态联合过滤

相较仅用程序特征的 PPF，MOKA 还根据 sTLB MPKI、sTLB miss rate 等系统状态决定是否启用相应权重，使同一程序模式在不同资源压力下可以得到不同判断。论文的实验消融支持这种联合设计有效。

### 8.3 创新点三：使用 vUB/pUB 处理误丢弃与真实命中反馈

vUB 捕获被过滤但后来造成 demand miss 的 false negative；pUB 结合物理地址和 PCB 识别实际有用或无用的已发出请求。该设计把过滤判断和后续缓存行为闭环连接起来。

### 8.4 创新点四：面向 workload/phase heterogeneity 的自适应阈值

MOKA 不是用一个固定 activation threshold，而是根据 epoch 内准确率、IPC、缓存/ROB 压力和前端压力调整阈值。论文结果表明，动态阈值与程序特征、系统特征组合后优于静态 PPF 和单一特征配置。

## 9. 局限性

### 9.1 论文明确承认或实验可见的局限

* 论文评估主要基于 ChampSim 模拟，没有真实处理器芯片上的实测结果。
* DRIPPER 的特征组合是针对 218 个 seen workload 离线探索后得到的，虽有 178 个 unseen workload 评估，但不等于对所有程序分布都无偏。
* 多核实验使用单核阶段得到的设计，作者明确指出未在 8-core context 中调优，进一步探索可能获得更高 speedup。
* 论文关注 VIPT L1D；下层物理寻址 cache 的跨物理页预取还涉及安全侧信道问题，不应直接套用该机制。
* DRIPPER 在少数短 workload 或某些 QMM INT/QMM FP workload 上仍可能低于 Discard PGC，原因是它在部分有害阶段允许了跨页请求，或在达到高置信度前过于保守。

### 9.2 阅读后发现的潜在局限

* 特征选择和阈值策略依赖微架构参数、TLB 层次和 workload trace；迁移到 RISC-V、不同页表结构或不同 cache 组织时需要重新校准。
* 论文的“有效”定义是至少服务一次 L1D demand hit，不能完全表达延迟隐藏程度、关键路径贡献、带宽消耗或能耗收益。
* 1.44 KB 是模拟设计中所选配置的 storage overhead，不等同于完整 RTL、时序、功耗和面积验证。
* 页面跨界过滤位于硬件预取器之后，编译器没有向过滤器提供 loop、array、指针别名或阶段语义信息，因此它和 LLVM/MLIR 的结合仍是开放问题。

## 10. 阅读后的研究方向反思

本文不是 LLVM/MLIR 编译器论文，也不直接生成源码、IR 或 compiler pass；它更适合作为硬件反馈、预取策略和自动调优的参考。对 AI 编译器研究最有价值的思想是“用真实运行时状态校正静态或局部预测”，而不是简单把感知器模块移植到编译器中。

值得借鉴的是三层反馈结构：局部程序特征、全局系统状态、跨阶段阈值调节。对于 RISC-V，直接替换 ISA 不足以形成创新；需要研究 RISC-V 页表、TLB 层级、向量访存或异构内存系统特有的反馈信号，并证明这些信号改变了编译器或运行时策略。

论文最适合作为：

* 硬件预取器/地址转换反馈模块的 baseline；
* 编译器自动调优系统中的运行时反馈来源；
* 研究“静态代码特征 + 动态硬件状态”协同决策的架构参考。

## 11. 可进一步尝试的研究方向

### 11.1 RISC-V 向量程序的编译器-预取器协同过滤

#### 研究问题

LLVM/MLIR 能否把向量循环、stride、页边界和内存别名信息编码成轻量 metadata，帮助 RISC-V 向量 L1D 预取过滤器判断跨页请求？

#### 与原论文的区别

原论文的程序特征主要是 PC、虚拟地址、delta 等硬件可见特征；新方向加入编译器产生的循环/向量语义，并针对 RVV 访存行为评估。

#### 可能的创新点

设计不暴露源码的紧凑 metadata、研究 metadata 误分类成本，并比较纯硬件 DRIPPER 与 compiler-assisted DRIPPER。

#### 实验框架

```text
LLVM/MLIR 向量化与 metadata 注入
    ↓
RISC-V/RVV 模拟器或 RTL 预取器
    ↓
页面跨界过滤与 TLB/cache 反馈
    ↓
真实 workload 的 IPC、带宽、能耗和代码尺寸评估
```

#### 可行性

需要 LLVM/RVV 后端、ChampSim 或 RISC-V 微架构模拟器，以及 SPEC、PolyBench 或 ML kernel workload。

#### 主要风险

metadata 可能增加指令/缓存压力；仿真器中的收益不一定能迁移到真实芯片。

### 11.2 异构 CPU-GPU 内存系统的跨页预取自动调优

#### 研究问题

在统一虚拟地址或共享页表的 CPU-GPU 系统中，如何根据 TLB、页迁移和 GPU kernel 阶段动态调节跨页预取？

#### 与原论文的区别

原论文只研究 L1D 和 CPU 风格 TLB/cache；新方向把页迁移、异构内存压力和 kernel phase 纳入目标。

#### 可能的创新点

联合优化预取收益、页迁移成本和 PCIe/CXL 流量，而不只优化 IPC。

#### 实验框架

```text
编译器提取 kernel/访存阶段
    ↓
运行时收集 CPU-GPU TLB、页迁移与带宽状态
    ↓
在线选择预取阈值与目标设备
    ↓
比较执行时间、迁移流量、能耗和公平性
```

#### 可行性

可从 GPU kernel trace 和异构内存模拟器开始，逐步加入 MLIR GPU dialect 或 RISC-V 加速器后端。

#### 主要风险

系统状态更多且反馈延迟更长，可能导致阈值振荡或预取投机放大迁移流量。

### 11.3 面向硬件反馈的编译器 phase/flag 自动调优

#### 研究问题

能否把 DRIPPER 式的“准确率、TLB pressure、cache pressure、IPC”反馈转化为编译器 pass 顺序、prefetch 插入或 code layout 的调优信号？

#### 与原论文的区别

原论文的最终动作是硬件层面 issue/discard；新方向由编译器选择可持久化的程序变换，并把运行时反馈用于跨配置优化。

#### 可能的创新点

建立跨编译配置与硬件状态的多目标代价模型，区分短期预取收益和长期代码尺寸/编译时间成本。

#### 实验框架

```text
源码/LLVM IR
    ↓
候选 pass、prefetch 或 layout 配置
    ↓
编译到目标 ISA 并运行 profiling
    ↓
读取 IPC、TLB/cache MPKI、预取 accuracy 与带宽
    ↓
更新配置选择器，输出下一轮候选
```

#### 可行性

可复用 LLVM、CompilerGym 风格环境和 ChampSim/硬件计数器；先以黑盒搜索建立 baseline，再研究学习型策略。

#### 主要风险

编译和运行成本较高，硬件反馈对输入规模和系统噪声敏感，且不能把微架构模拟结果直接视为真实硬件时间。

## 12. 与其他已读文献的关系

本批次 slot-3 只完成本文一篇论文，尚无同批次可核验全文用于横向比较。与仓库中既有 compiler autotuning 或 CompilerGym 类工作之间的关系只能作为后续去重线索，本文笔记不把未在本批次重新核验的内容当作事实比较。

从角色上看，本文更接近 SUPPORTING：它提供硬件/地址转换/预取反馈机制，可作为 AI 编译器自动调优的底层环境信号，而不是直接的 LLVM pass 选择器、IR translator 或 compiler capability generator。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 优化 VIPT L1D 的页面跨界数据预取 |
| 核心问题 | 静态允许或静态丢弃无法适应 workload/phase 差异 |
| 输入 | L1D 预取请求、虚拟地址/PC/delta、TLB/cache/ROB 状态 |
| 输出 | 是否发出页面跨界预取 |
| 核心方法 | MOKA 框架与 DRIPPER 过滤器 |
| 使用的模型 | 哈希感知器、饱和计数器；无 LLM |
| 使用的编译器工具 | 无 LLVM/MLIR；使用 ChampSim 微架构模拟器 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否 |
| 数据集规模 | 396 个 memory-intensive workload；218 seen、178 unseen；另有完整 suite 评估 |
| 主要指标 | IPC speedup、coverage、accuracy、dTLB/sTLB/L1D/LLC MPKI、storage |
| 最重要实验结果 | Berti 上相对 Discard PGC：seen +1.7%、unseen +1.2%；300 个 8-core mixes +2.0%；开销 1.44 KB/core |
| 核心创新 | 程序特征 + 系统状态 + 在线阈值的页面跨界过滤闭环 |
| 主要局限 | 依赖 ChampSim 仿真和特定微架构；少数 workload 仍受损；未在真实硬件验证 |
| 与 RISC-V 研究的相关性 | 中：可借鉴反馈式预取和地址转换协同，但论文未评估 RISC-V |
| 最适合作为 | 硬件反馈模块、预取/自动调优 baseline、异构内存研究参考 |

> 这篇论文最值得学习的是把“程序模式、系统状态和实际缓存命中结果”组合成在线闭环，而不是只追求更激进的预取；最主要的局限是结果依赖微架构模拟与特定 workload/特征选择；如果用于后续研究，最合理的使用方式是把它作为预取反馈和硬件-编译器协同的 baseline，而不是简单把 CPU 页面边界过滤器移植到 RISC-V 后就宣称完成了新方法。

---

### 建议归类与去重记录（仅供正式维护代理登记）

* 建议 taxonomy 类别：`SUPPORTING / B5_Hardware_ISA_Compiler_Background`。
* 角色依据：论文最终输出是硬件 Page-Cross Filter 的 issue/discard 动作，不是 LLVM/MLIR pass、源码/IR 变换或可复用编译器组件；因此不建议归入 SELECTOR/TRANSLATOR/GENERATOR。
* 去重依据：已检查正式 `taxonomy_v2.csv`、当前角色索引、逐篇阅读目录和年份筛选清单中的 HPCA 记录；未发现同题名、DOI `10.1109/HPCA61900.2025.00025` 或相同作者/版本关系的记录。`slot-3` staging 初始为空。
