# C48 Fast Autoscheduling for Sparse ML Frameworks 文献阅读总结

论文题目：**Fast Autoscheduling for Sparse ML Frameworks**

作者：Bobby Yan、Alexander J. Root、Trevor Gale、David Broman、Fredrik Kjolstad

发表时间：2026

发表平台：CGO 2026 主会，pp. 28–43
元数据核验来源：[Stanford Compilers Lab 论文/项目页](https://compilers.stanford.edu/publications/cgo26scorch/)；[作者 PDF](https://fredrikbk.com/publications/scorch.pdf)；[CGO 官方论文页](https://2026.cgo.org/details/cgo-2026-papers/52/Fast-Autoscheduling-for-Sparse-ML-Frameworks)
代码/数据/工件：作者公开实现：[Scorch](https://github.com/bobbyyyan/scorch)

论文链接或编号：DOI `10.1109/CGO68049.2026.11394842`；[CGO 官方论文页](https://2026.cgo.org/details/cgo-2026-papers/52/Fast-Autoscheduling-for-Sparse-ML-Frameworks)

关键词：稀疏张量、自动调度、Scorch、循环排序、混合稀疏稠密 tiling、格式推断、PyTorch

> 本文档用于文献阅读、组会汇报和后续研究分析。论文事实与阅读后的研究思考需要分开描述。

## 1. 研究背景

稀疏张量可减少大型模型中的无效计算，但现有 ML 框架对稀疏算子支持有限。手写内核需要专家知识，离线穷举搜索又可能耗时数小时到数天，不适合 PyTorch eager-mode 的交互式开发。稀疏计算还要求同时选择循环顺序、中间/输出张量格式及面向缓存的分块策略，这些决策会显著影响算法复杂度与性能。

## 2. 论文要解决的问题

论文提出的问题是：能否在保持 PyTorch 张量 API 易用性的前提下，在毫秒级编译时间内自动选择稀疏张量的循环顺序、稀疏/稠密混合循环的 tiling 方式，以及中间与输出张量的存储格式，并避免复杂度级别的性能悬崖。

> 核心目标：以快速、结构化的启发式代替耗时搜索或手工调度，使通用稀疏张量框架在多类 ML 工作负载上获得可预测且有竞争力的性能。

## 3. 核心方法概述

作者实现 Scorch，一个面向 PyTorch API 的稀疏张量编译原型，包含三项自动化算法：基于稀疏 filter 和结构成本模型的循环排序/工作区插入；识别可复用稠密维度、排除稀疏维度及稀疏维度父循环的专用 tiling；以及按运算语义和输入格式逐维推断临时/输出张量格式。生成代码把可组合的稀疏张量操作融合为内核，减少中间物化。

```text
PyTorch 稀疏/稠密 tensor 表达式
  → 分析索引集合、格式和 sparse filters
  → 贪心 loop ordering + workspace 决策
  → mixed sparse-dense 循环选择性 tiling
  → 逐维 format inference
  → 融合稀疏代码生成与 CPU 执行
```

## 4. 实验框架与训练流程

本文没有深度模型训练或强化学习环节。代价模型的常数通过代表性 SuiteSparse 矩阵、候选合法调度的实测表现和 Bayesian optimization 一次调优；论文说明在 ARM 上调参后将同一组常数用于 x86。

### 4.1 自动循环排序

算法先按照索引变量对应 sparse filter 的稀疏程度形成初始顺序，再贪心尝试移动索引。成本项分别近似计算量、临时工作区插入/排序开销和张量转置开销；若移位的加权净成本为负则接受。随后按输入和输出格式建立循环顺序约束图，必要时移除代价最低的冲突边并更新循环顺序。

### 4.2 混合稀疏—稠密 tiling 与格式推断

tiling 算法从张量访问中发现缺失索引所暗示的复用维度，删除稀疏维度及其父循环，再对剩余候选维度分块，避免在稀疏不规则遍历中引入额外搜索开销。格式推断通过递归模式匹配表达式，依据加法、乘法及一元运算对零值的行为，逐维决定输出是 DENSE 还是 SPARSE。

### 4.3 端到端执行

Scorch 扩展 PyTorch 的稀疏 tensor 构造/API dispatch，在 CPU 上执行；实验分别比较 SuiteSparse 核心算子和图神经网络、稀疏自编码器、稀疏 Transformer 任务。

## 5. 奖励函数、损失函数或关键公式

本文无训练奖励函数。循环排序使用加权成本变化：

```text
ΔCost = α·ΔCcomp + β·ΔCws + γ·ΔCtrans
```

其中 `Ccomp` 估算迭代空间计算量；`Cws` 估算中间工作区的插入与排序开销；`Ctrans` 估算满足循环顺序所需的张量转置成本。成本模型追求识别渐近复杂度差异，不直接拟合每种输入的精确运行时间。工作区插入由输出散布需求决定；格式推断则按运算的零保持性质选择稀疏或稠密维度格式。

## 6. 实验设置

### 6.1 数据与工作负载

核心稀疏算子是 SpMV、SpMM、SDDMM、SpMSpM，矩阵取自 SuiteSparse Matrix Collection。端到端任务包括 Cora、CiteSeer、PubMed、OGBN-ArXiv 上的两层 GCN；MNIST、CIFAR-10、CIFAR-100、CelebA 上的稀疏自编码器；以及稀疏 Transformer 数据集/配置。

### 6.2 硬件与软件

CPU 测试平台为 Intel Core i9-14900K（24 核、196 GB 内存）和 Apple M1 Ultra（20 核、64 GB 内存）；GPU 对比使用 RTX 4090。论文将 Scorch 作为 CPU 稀疏张量编译器，与 PyTorch Sparse、Intel MKL 及领域库/专用方法比较。

### 6.3 对比方法与指标

比较对象包括 PyTorch Sparse、PyG、DGL、Nano 和穷举式 autoscheduler Pigeon；消融比较 tiling 与 untiled、仅 k tiling 与 i-k tiling、以及 fusion。报告核心算子时间、端到端 speedup、调度编译时间和预处理成本；区分仅 kernel 时间与包含预处理的总时间。

## 7. 实验结果与结论

### 7.1 核心稀疏算子与自动调度

SpMV 和 SpMSpM 等调度空间受限的算子上，Scorch 大体与 PyTorch 持平；SpMM 的稠密维度 tiling 在较大问题上最高报告 6.36× speedup，较大中等稀疏矩阵最高 5.96×。SDDMM 上，融合避免了 PyTorch 分解操作时产生的大型稠密中间结果，在较大问题上报告超过一个数量级的优势。自动调度在毫秒级完成，较 Pigeon 穷举快多个数量级；作者指出其计划落在 Pigeon 的渐近最优 Pareto 前沿内，同时还自动做 tiling 和格式推断。

### 7.2 Tiling 与端到端应用

只 tiling 稠密 k 维通常胜过同时 tiling i 和 k；在稀疏访问父循环上分块会引入控制/边界成本，并可能反复加载稠密 tile。Highly sparse 数据上的收益较温和（约 1.05–1.07×），符合不规则访问限制缓存收益的分析。端到端结果中，Scorch 相对 PyTorch 在 GCN ARM 测试平均约 2.1×；稀疏自编码器报告 1.39×–5.80×；不同任务与设备的收益有差异，部分 GPU 对比还受启动开销和显存容量影响。

### 7.3 结论

论文的核心结论是，稀疏编译优化可通过轻量结构模型在很短的编译时间内自动化，而无需把所有优化空间交给长时间离线搜索或人工手调。效果来自调度、格式推断、tiling 和算子融合的共同作用，并非单一算法在所有任务上的统一加速比。

## 8. 主要创新点

1. 按 sparse filter 结构排序循环，以计算、工作区与转置成本平衡渐近复杂度和数据布局代价。
2. 提出针对稀疏—稠密混合循环的选择性 tiling 规则，避免机械套用稠密分块策略。
3. 根据运算对零的语义逐维推断临时和输出张量格式，减少用户手工声明。
4. 将稀疏调度接入 PyTorch 风格 API，以毫秒级编译服务交互式 ML 开发。

## 9. 局限性

代价模型依据维度、格式与假设稀疏率估算非零量，可能在高度偏斜、格式与真实密度不匹配时选到次优方案；算法目标不是预测精确运行时间。论文指出这种结构估计对少见边缘情况可能不准确。Scorch 是原型，当前主要面向 CPU，不支持 GPU 代码生成；GPU 结果是 PyTorch/cuSPARSE 对比。离线专用方案在矩阵重复使用时可能因高效 kernel 超过 Scorch。跨数据分布、模型动态变化和更广张量语义的泛化仍需额外验证。

## 10. 阅读后的研究方向反思

Scorch 的关键思想是把稀疏运算的结构信息而非微架构细节作为快速调度依据，同时明确承认近似成本模型的适用边界。对 AI 编译器研究而言，它可以作为稀疏调度的传统非 LLM 基线。可与学习型代价模型结合，但应保留渐近复杂度保护，避免仅凭有限 benchmark 搜索带来难以解释的性能悬崖。

## 11. 可进一步尝试的研究方向

### 11.1 结构模型与硬件反馈的分层混合调度

研究问题是如何先用 sparse filter 规则排除渐近低效 loop nest，再让轻量学习器或运行时抽样选择 tile 参数。评估应同时报告编译延迟、不同稀疏分布下的鲁棒性、首次执行总成本以及重复执行的摊销收益。此方向是阅读后设想，论文尚未验证。

### 11.2 面向 RVV 的稀疏格式与向量化联合选择

将 CSR/COO 逐维格式推断与 RVV 向量长度、访存规律、尾部掩码和真实 RISC-V 核心成本关联，检验 CPU 稀疏工作负载是否能从向量化中获益。需与仅使用结构成本模型及手写 RVV kernel 对比，避免把单平台参数拟合误当可移植性。

## 12. 与其他已读文献的关系

与 C13（TCL 跨硬件张量程序优化）、C19（STENSO 张量程序超级优化）、C21（Transformer 自动向量化）和 C29（Neptune GPU 算子融合）相比，本文聚焦传统稀疏张量编译算法及快速调度，而非 LM 输出或 e-graph 搜索。它也延续 TACO、SparseTIR、Pigeon 等稀疏张量编译工作，并将自动 tiling 与格式推断纳入 PyTorch 风格系统。论文是 CGO 2026 主会论文；分类为 SUPPORTING/B4，不代表其只具背景价值，而是记录其非 LM 核心编译优化定位。

## 13. 一页式总结

**问题：** 稀疏 ML 框架缺少快速、自动、可预测的循环排序、分块与格式决策。

**方法：** Scorch 用结构成本模型贪心优化稀疏循环顺序，选择稠密维度 tiling，并逐维推断输出/临时格式；操作融合减少大型中间结果。

**证据：** SuiteSparse 核心核函数与 GCN、自编码器、Transformer 评估显示其调度毫秒级完成，并在部分任务上明显快于 PyTorch Sparse；加速幅度取决于工作负载、平台与是否计入预处理。

**分类：** SUPPORTING / B4_Traditional_ML_Compiler_Optimization；属于非 LLM 的稀疏张量编译优化。

**研究价值：** 可作为稀疏编译器自动调度的算法基线，并为结构成本模型与 RVV/硬件反馈结合提供对照。
