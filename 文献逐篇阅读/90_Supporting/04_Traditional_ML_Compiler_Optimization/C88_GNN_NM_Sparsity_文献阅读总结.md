# N:M-Sparsity-Oriented Graph Reordering 文献阅读总结

论文题目：**Accelerating GNNs on GPU Sparse Tensor Cores through N:M Sparsity-Oriented Graph Reordering**

作者：Jou-An Chen、Hsin-Hsuan Sung、Ruifeng Zhang、Ang Li、Xipeng Shen

发表时间：2025 年

发表平台：PPoPP 2025, pp.16–28；另有 JPDC 213 (2026), Article 105264 的同主题期刊记录（与会议论文的版本关系未确认）

论文链接或编号：DOI 10.1145/3710848.3710881；JPDC DOI 10.1016/j.jpdc.2026.105264
元数据核验来源：[ACM DOI](https://doi.org/10.1145/3710848.3710881)；[JPDC 出版社记录](https://doi.org/10.1016/j.jpdc.2026.105264)
代码/数据/工件：未找到可确认的作者代码仓库或工件

版本说明：目录所列 PPoPP 2025 会议论文与 JPDC 2026 论文为同主题不同版本；出版社记录显示期刊版题为 *N:M sparsity-oriented graph reordering for accelerating GNNs on GPU sparse tensor cores*，JPDC 213, Article 105264，DOI [10.1016/j.jpdc.2026.105264](https://doi.org/10.1016/j.jpdc.2026.105264)。
来源：[PPoPP 2025 作者机构记录](https://www.pnnl.gov/publications/accelerating-gnns-gpu-sparse-tensor-cores-through-nm-sparsity-oriented-graph)；[JPDC 官方卷期目录](https://www.sciencedirect.com/journal/journal-of-parallel-and-distributed-computing/vol/213/suppl/C)。
关键词：GNN、图重排序、N:M 稀疏、V:N:M 稀疏、GPU、Sparse Tensor Core、SpMM、CUDA

> 本文档依据论文正文 PDF 撰写。论文事实与阅读后的研究思考分开描述。本文是传统 GPU/编译运行时优化系统，不是 LLM 编译器论文。

---

## 1. 研究背景

图神经网络（Graph Neural Network，GNN）通过沿图边聚合邻居特征，再执行神经网络更新，已用于社交网络、推荐、分子化学等任务。GNN 的聚合阶段通常可表示为稀疏邻接矩阵与稠密特征矩阵的稀疏矩阵-矩阵乘法（SpMM）。随着图规模和特征维度增大，SpMM 成为重要性能瓶颈（第 1、2 节）。

现代 NVIDIA GPU（论文指出 Ampere 及以后）配备 Sparse Tensor Core（SPTC），可以通过 `mma.sp` 等稀疏融合乘加指令加速满足结构化 N:M 稀疏模式的矩阵运算。N:M 表示每连续 M 个元素最多有 N 个非零元素；V:N:M 是其推广，在 V×M 的 tile 中要求每行满足 N:M，并限制非零列数。VENOM 等软件抽象进一步支持 V:N:M 模式。

问题在于 GNN 的邻接矩阵通常是非结构化稀疏的。论文在 SuiteSparse 的 1356 个图上观察到，原始图中只有约 5%–9% 能符合相关稀疏模式，因而不能直接利用 SPTC。传统做法如剪枝会删除图边，可能损害 GNN 精度；一般矩阵重排序又可能破坏邻接矩阵对称性。论文因此研究只改变顶点编号、同时交换邻接矩阵对应行和列的无损图重排序。

## 2. 论文要解决的问题

### 2.1 将非结构化图稀疏变成硬件可用模式

给定图的邻接矩阵、顶点顺序和目标 V:N:M（N:M 是 V=1 的特例），寻找新的顶点排列，使尽可能多的 segment vector 和 meta-block 满足 SPTC 的稀疏约束。由于只做顶点重编号，图语义不变，邻接矩阵仍保持对称。

### 2.2 在组合搜索不可承受时获得有效的离线近似

最优顶点排列的搜索空间接近 n!；论文指出相关最优重排序问题是 NP-hard。因此方法需要在大图上以较低成本寻找足够好的排列。论文面向离线预处理：重排序本身可以花费一定时间，但结果应服务于图的多次推理。

### 2.3 在保留精度的同时加速 GNN SpMM

论文比较无损重排序与基于幅值剪枝的有损方案，目标是让已有 PyTorch Geometric（PyG）和 Deep Graph Library（DGL）框架能够替换为 SPTC SpMM 内核，获得 GNN 聚合层和端到端性能收益。

> 本文主要研究：如何通过保持图语义和对称性的双层顶点重排序，把非结构化 GNN 邻接矩阵转换为更适合 GPU SPTC 的 V:N:M 形式。

## 3. 核心方法概述

论文提出双层 N:M 稀疏导向图重排序，并实现为 GPU 库 SOGRE（Sparsity-Oriented Graph Reordering）。V:N:M 约束包括：V×M meta-block 中至多 k 列含非零值的垂直约束，以及每个 M 元素行向量最多 N 个非零值的水平约束。算法交替执行两个阶段，直到没有进一步改进或达到最大迭代次数。

```text
输入：图邻接矩阵 A、当前顶点顺序、目标 V:N:M 模式
        ↓
阶段 1：将 segment vector 编成二进制串和 Hamming 位置码，排序行
        ↓
同步交换邻接矩阵的行和列，降低 meta-block 垂直约束违例
        ↓
阶段 2：在高违例 segment 之间评估顶点交换，贪心选择有利交换
        ↓
重复两阶段并计算 MBScore/PScore
        ↓
得到新的顶点排列和 V:N:M 邻接矩阵
        ↓
SOGRE/CUDA 生成适合 SPTC 的表示，调用 Spatha 风格 SpMM
        ↓
PyG/DGL 中完成 GNN 聚合与端到端推理
```

阶段 1 用 Hamming-distance position encoding 给每个二进制 segment vector 编码；违反水平约束的编码取负值，再对各行的编码向量排序，以便把非零位置相似的行放入同一 meta-block。阶段 2 以 PScore 最大的 segment 为 primary segment，遍历 target segment，枚举顶点对，选择能最大减少两者 PScore 的交换，批量应用交换以提高效率。

SOGRE 用 CUDA 实现，使用 BSR（Block Sparse Row）索引、位移/位运算、warp 内 shuffle 和 voting 等 GPU intrinsic。邻接矩阵的行列同步重排保持图对称；对应的稠密特征矩阵也必须按照相同顶点顺序调整。论文将 SPTC SpMM 内核作为 PyG/DGL 的可替换模块。

## 4. 实验框架与训练流程

本文不涉及模型训练、SFT、强化学习或多阶段神经网络训练，主要采用离线图预处理、GPU 内核执行和 GNN 前向推理。

### 4.1 离线模式选择与重排序

论文先尝试 `1:2:M`，从 M=4 开始逐步加倍，直到图不能继续重排为符合模式；随后固定 M，尝试 `V:2:M`，V 从 1 增至 32。实验把垂直和细粒度重排序循环的最大迭代次数设为 10，多数矩阵在 6 次以内收敛。重排序时间不计入 GNN 运行时间，因为作者将其定位为一次性预处理。

### 4.2 阶段 1：垂直约束重排

对每个 segment vector 做二进制编码和 Hamming 位置编码；对每一行的编码向量排序，取得新的顶点顺序；同步重排邻接矩阵的行列，计算 meta-block 违例数 MBScore，并重复该过程。论文给出的理论复杂度为 O(n log n)，其中最大外层迭代被视为常数。

### 4.3 阶段 2：水平约束重排

先排除 PScore 为零的健康 segment；从最差 segment 开始，在 segment 对之间寻找未使用顶点的有利交换。每个 segment 长度为 M，候选顶点对最多为 M²；记录一轮交换后批量应用，再重新计算 PScore。作者给出复杂度 O(ω)，ω 为含有不健康 segment vector 的 segment 数，因此不高于 O(n)（M 和最大迭代数视为常数）。

### 4.4 推理与内核执行

实验将重排序后的矩阵送入 SPTC-based SpMM，再用于 PyG/DGL 的 GNN 前向节点分类。比较四种设置：原始矩阵上的默认 PyG/DGL、重排矩阵上的默认 PyG/DGL、剪枝矩阵上的 SPTC 版本、重排矩阵上的 SPTC 版本（论文方案）。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习，因此不存在强化学习奖励函数，也没有模型损失函数。

### 5.1 GNN 层计算

论文给出简化的 GNN 层公式：

```text
h(v, l+1) = ReLU((Σ[u→v] (e(u,v) ⊙ h(u,l))) ⊗ W(l))
```

其中 `h(u,l)` 是第 l 层节点隐藏特征，`e(u,v)` 是边值（零表示无边），聚合得到中心节点信息后与权重 `W(l)` 做神经更新。论文关注其中基于邻接矩阵的聚合/SpMM 阶段。

### 5.2 重排序改进率

```text
improvement rate = (final ω - initial ω) / initial ω
```

论文把 ω 定义为违反目标稀疏模式的 segment vector 数。按文字表述，实际结果以违例减少百分比呈现；因此阅读时应将其理解为违例计数的相对变化，而不是运行时间 speedup。`MBScore` 表示垂直约束违例的 meta-block 数，`PScore` 表示水平约束违例的 segment vector 数。

## 6. 实验设置

### 6.1 数据集来源

论文使用两类数据：

1. SuiteSparse Matrix Collection 中的 1356 个真实图邻接矩阵，按规模分为 444 个 small、724 个 medium、188 个 large。small 平均约 426 个顶点/4.97K 条边，medium 平均约 3.6K/93.2K，large 平均约 22.6K/878K。
2. 12 个 GNN 图数据集：Cora、Citeseer、Facebook、Computers、CS、CoraFull、Amazon-ratings、Physics，以及 OGBN 的 ogbn-proteins、ogbn-products、ogbn-arxiv、ogbn-papers100M。论文表 2 给出了各自顶点数、边数、特征数和类别数；最大的是 ogbn-papers100M，约 111.06M 个顶点和 1.616B 条边。

论文没有将这些数据称为训练集/验证集/测试集划分；评测重点是 GNN 前向节点分类、SpMM 性能与模式符合率。论文中未明确说明是否存在训练数据泄漏风险。

### 6.2 模型与工具

| 项目 | 论文设置 |
| --- | --- |
| GNN 模型 | GCN、GraphSAGE、ChebNet、SGC |
| 框架 | PyTorch Geometric（PyG）、Deep Graph Library（DGL） |
| GPU | NVIDIA A100，40 GB |
| CUDA | CUDA 11.7 |
| CPU | 第四代 AMD EPYC |
| 稀疏内核 | 基于 Spatha/VENOM 的 SPTC SpMM；PTX `mma.sp.sync`，默认 `m16n8k32` |
| 自研库 | CUDA 实现的 SOGRE |
| 默认基线 | PyG 的 Torchsparse CSR-SpMM、DGL 的 cuSPARSE CSR SpMM（论文指出 DGL 使用 `CUSPARSE_SPMM_CSR_ALG2`） |

### 6.3 对比方法

- `default-original`：原始矩阵上的默认 PyG/DGL，不使用 SPTC。
- `default-reordered`：重排矩阵上的默认 PyG/DGL，不使用 SPTC，用于隔离“仅换顶点顺序”的收益。
- `revised-pruned`：对每个 V:N:M meta-block 按幅值删除最小元素，形成 SPTC 模式；可以获得速度，但有损。
- `revised-reordered`：论文方案，在无损重排矩阵上使用 SPTC SpMM。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
| --- | --- | --- |
| Layer-wise/LYR speedup | 聚合层相对基线的加速比 | 越大越好 |
| End-to-end/ALL speedup | GNN 前向整体加速比 | 越大越好 |
| Pattern violation count | 违反 N:M 或 V:N:M 的 vector/meta-block 数 | 越小越好 |
| Conforming graph ratio | 满足目标模式的图比例 | 越大越好 |
| Reordering time | 离线重排序耗时 | 越小越好，但不计入推理时间 |
| Node classification accuracy | GNN 预测准确率 | 越大越好 |

## 7. 实验结果与结论

### 7.1 主要 GNN 结果

在 8 个中小型 GNN 数据集和 PyG/DGL 框架上，论文表 3 报告了相对 `default-original` 的结果。PyG 上，论文方法对 GCN 的平均聚合层加速约为 1.4–3.3×；SGC 的聚合层最高达到 8.6×，端到端最高约 6.3×。例如 Facebook 上，PyG 的 SGC 聚合层为 8.64×、端到端为 6.37×；DGL 对应为 6.39× 和 4.74×。不同模型差异来自聚合与线性层的执行顺序及乘数矩阵列数。

### 7.2 仅重排但不使用 SPTC 的比较

表 4 显示 `default-reordered` 相对 `default-original` 基本没有显著收益，大多数结果接近 1.0×，个别结果略有下降。这说明性能增益不是来自重新编号本身，而是来自重排序后能够调用 SPTC/V:N:M 内核。

### 7.3 与剪枝方案的比较

`revised-pruned` 获得了与重排方案相近的速度，因为两者都能形成 SPTC 所需模式；两者相对默认基线的 speedup 差异约在 ±0.01–±0.07 内。但表 5 显示剪枝造成准确率下降，而重排序不造成准确率下降。例如 Facebook 上，剪枝使 GCN、GraphSAGE、ChebNet、SGC 准确率分别下降 6.79%、8.04%、7.41%、4.96%；Computers 上 ChebNet 下降 13.4%。论文将图边删除视为可能破坏关键信息的有损操作。

### 7.4 分布式/大图结果

对于 OGBN 数据，作者使用 PyG NeighborSampler 采样子图，并在 4 张 A100 上并行执行 SGC。表 6 的端到端加速分别为：ogbn-proteins 1.159×、ogbn-arxiv 3.229×、ogbn-products 1.399×、ogbn-papers100M 2.449×；对应聚合层加速为 1.140×、6.494×、1.423×、2.781×。

### 7.5 SuiteSparse SpMM 与重排质量

在 1356 个 SuiteSparse 矩阵上，图重排使向量级违例减少约 98.9%–100%。以 1:2:4 为例，small/medium/large 三类的平均初始违例分别为 510.31、12,656.56、33,202.87，平均最终违例为 0.96、21.43、930.11；平均改进率约 99.29%、99.94%、98.87%。平均重排序时间分别为 0.05 s、4.39 s、30.55 s，中位数分别为 0.01 s、0.63 s、11.12 s。

在符合率方面，论文摘要报告原始约 5%–9% 的图提升到 88.7%–93.5%；正文中针对 1:2:4 的统计为 small 8.74%→93.92%、medium 7.32%→87.85%、large 5.91%→89.25%。

SpMM 对 cuSPARSE 的 speedup 在图规模和特征列数增加时通常更明显；论文图 4 以 H=64、128、256、512 评估。约 3.9% 的矩阵出现 slowdown，论文分析其主要是密度极低（多数低于 0.01%），SPTC 为适配模式仍需处理额外零元素。

### 7.6 消融与案例

论文没有以单独“消融实验”标题报告神经模型组件消融；但 `default-reordered` 对照实质上隔离了顶点重排本身与 SPTC 内核的作用，`revised-pruned` 对照隔离了无损重排与有损剪枝的精度影响。论文还说明 V 越大，meta-block 约束越严格，越不容易完全符合；不同矩阵应选择不同 V:N:M，作者建议尝试常见模式后选择最佳模式。

## 8. 主要创新点

### 8.1 创新点一：面向 V:N:M SPTC 的无损图重排序

已有图重排序多用于局部性、负载平衡或存储格式；论文将目标改为满足 GPU SPTC 的 V:N:M 模式。通过同步交换邻接矩阵的行和列，方法只改变顶点编号，不删除边，因此保持图语义、邻接矩阵对称性和 GNN 准确率。论文实验支持了其对比剪枝的优势。

### 8.2 创新点二：双层、可迭代的约束处理算法

阶段 1 用 Hamming 位置码降低 meta-block 的垂直违例，阶段 2 用基于 PScore 的贪心顶点交换降低 segment vector 的水平违例；两阶段迭代处理彼此影响。这不是简单调用通用矩阵排序，而是针对 V:N:M 两级结构设计的搜索启发式。

### 8.3 创新点三：GPU 位级实现与框架集成

SOGRE 用 CUDA、BSR、位编码和 warp intrinsic 加速离线重排，并将 SPTC SpMM 作为 PyG/DGL 的模块化替换。论文在真实 GNN、SuiteSparse 和大图采样上验证了这一工程路径。

## 9. 局限性

### 9.1 论文明确或实验中体现的局限

- 该方法是启发式重排，不保证找到全局最优；最优排列问题具有 NP-hard 性。
- 重排序时间虽然离线，但 large SuiteSparse 图的平均时间达到 30.55 s，中位数 11.12 s；若图频繁变化或只使用一次，预处理摊销优势会减弱。
- 约 3.9% 的极稀疏矩阵在重排后出现 slowdown；SPTC 并非对所有稀疏度都有效。
- 不同矩阵对 V:N:M 的偏好不同，较大的 V 更难满足；模式选择目前主要采用尝试多个模式的策略，论文提到可训练预测器但未实现。
- 实验主要在 NVIDIA A100、CUDA 11.7 和特定 Spatha/PTX 内核上完成，跨 GPU 厂商、跨 ISA 和其他 SPTC 实现的迁移结果论文中未明确说明。
- 论文评估的是 GNN 前向节点分类和 SpMM；对动态图、频繁增删边、训练反向传播、其他 GNN 运算或更复杂稀疏算子的适用性未全面验证。

### 9.2 阅读后发现的潜在局限

- “无损”指图重编号不删边、不做近似，不等于整个实现对任意数值格式、采样顺序或分布式通信都自动保持等价；论文对这些边界条件没有给出形式化证明。
- 论文使用真实硬件运行时间与 kernel speedup，但没有提供跨多代 GPU 的系统性敏感性分析；把结果直接外推到 RISC-V 向量或其他异构加速器并无正文依据。
- 由于重排同时改变顶点顺序，系统集成必须确保特征矩阵、标签、采样器和结果回写使用同一排列；论文描述了对应矩阵需同步交换，但未将所有框架状态的一致性作为独立验证问题展开。

## 10. 阅读后的研究方向反思

论文最值得借鉴的是“硬件结构约束—中间数据表示—离线搜索—内核反馈”的闭环：它没有通过删边强行匹配硬件，而是寻找保持语义的表示变换。对编译器研究而言，SOGRE 更接近传统编译/运行时优化基础设施，而不是 LLM Selector、Translator 或 Generator。

该方法不能简单改成“把 A100 换成 RISC-V”就形成新贡献。若迁移到 RISC-V，需要重新说明目标向量/矩阵扩展的稀疏指令约束、tile 布局、压缩格式、代价模型和真实硬件验证。较合理的 taxonomy 建议是 **SUPPORTING / B4_Traditional_ML_Compiler_Optimization**，横向标签包括 GPU、Graph_Compiler、Sparse_Matrix、Autotuning、Hardware_Aware_Optimization；它更适合作为传统后端优化基线或硬件感知搜索模块。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V 向量/矩阵扩展的稀疏图布局搜索

#### 研究问题

给定 RISC-V RVV 或向量矩阵扩展的实际稀疏加载、分块和向量长度约束，如何保持图语义并生成适配布局？

#### 与原论文的区别

不是替换 GPU 名称，而是重新定义硬件模式、数据布局和成本函数，并在真实 RISC-V 硬件或可信模拟器上验证。

#### 可能的创新点

将 VLEN、向量尾处理、缓存/访存代价与稀疏模式符合率联合建模；比较顶点重编号、边分块和 CSR/BSR 混合布局。

#### 实验框架

```text
GNN 邻接矩阵 → RVV/RVV+矩阵扩展约束分析 → 候选重排序/布局搜索
→ LLVM/MLIR 后端 lowering → RISC-V 真实硬件或模拟器 → 性能与语义回归
```

#### 可行性

需要 LLVM/MLIR 后端、RVV 工具链、至少一种 RISC-V 平台或周期/性能模拟器，以及 GNN/图基准。

#### 主要风险

硬件稀疏指令支持可能不稳定，模拟器与真实硬件的访存模型可能不一致；必须避免把有限测试误写成形式化等价证明。

### 11.2 学习型 V:N:M 模式选择与代价模型

#### 研究问题

能否根据图规模、密度、度分布、特征维度和目标 GPU/加速器，提前预测最有收益的 V:N:M 模式，减少逐模式试探？

#### 与原论文的区别

原论文建议尝试多种模式但没有实现预测器；该方向把模式选择作为独立的学习型编译决策，并评估预测错误的代价。

#### 可能的创新点

构造图统计特征到模式/内核配置的预测器，并将重排成本、SpMM 收益和失败概率纳入多目标选择。

#### 实验框架

```text
图统计特征 → 模式预测器 → SOGRE/候选重排序 → 硬件测量反馈
→ 更新代价模型 → 跨图、跨硬件泛化测试
```

#### 可行性

可复用论文的 SuiteSparse、GNN 数据集和现有 SOGRE 类实现；需要增加跨硬件测量与训练/测试划分。

#### 主要风险

图分布外泛化和硬件版本变化可能使模型失效；必须报告模式选择时间和预测错误，而不能只报告最佳离线结果。

### 11.3 形式化验证支持的无损图重编号编译管线

#### 研究问题

如何验证图重编号、特征/标签同步置换、SpMM 结果逆置换在语义上保持一致？

#### 与原论文的区别

原论文通过构造说明和准确率实验支持无损性，但没有把端到端置换一致性作为形式化验证对象。

#### 可能的创新点

以图同构/矩阵置换关系为规范，结合 SMT 或差分测试验证优化前后聚合结果、标签映射和分布式回写的一致性。

#### 实验框架

```text
原始图与 GNN 层 → 生成置换证明义务 → 重排/后端 lowering
→ SMT/差分执行检查 → 真实 GPU/RISC-V 性能评测
```

#### 可行性

适合做 LLVM/MLIR 外部验证工具或编译器 pass 的 correctness guard；可从小图和单层聚合开始。

#### 主要风险

浮点重排、采样随机性和原子操作顺序会使“位级相等”过强；需要区分数学等价、数值容差和任务准确率。

## 12. 与其他已读文献的关系

本 slot 本轮只完成这一篇论文，因此没有另一篇已通读论文可进行事实性横向比较。就仓库研究方向而言，它与 LLVM/MLIR、GPU/异构代码生成和传统自动调优主题相关，但论文没有使用 LLM，也没有生成 LLVM pass 或 MLIR rewrite。它更适合作为硬件感知稀疏布局/SpMM 优化的传统 baseline，或作为后续 Selector（选择 V:N:M/布局）与 Generator（生成后端配置）研究的被测执行器模块。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | 通过图顶点重排序让 GNN 邻接矩阵适配 GPU SPTC |
| 核心问题 | 非结构化图稀疏不满足 N:M/V:N:M，无法利用稀疏 Tensor Core |
| 输入 | 图邻接矩阵、顶点顺序、目标 V:N:M 模式 |
| 输出 | 保持图语义和对称性的重排邻接矩阵/顶点排列 |
| 核心方法 | Hamming 编码的垂直重排 + PScore 贪心水平重排，迭代执行 |
| 使用的模型 | GCN、GraphSAGE、ChebNet、SGC；无 LLM |
| 使用的编译器工具 | CUDA/SOGRE、Spatha/VENOM 风格 SPTC SpMM、PyG、DGL |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；主要用构造、对照实验和准确率评测支持无损性 |
| 数据集规模 | SuiteSparse 1356 个矩阵；GNN 12 个图数据集 |
| 主要指标 | SpMM/GNN layer speedup、端到端 speedup、模式违例、符合率、准确率、重排时间 |
| 最重要实验结果 | SpMM 相对 cuSPARSE 平均最高 2.3–7.5×；GNN 关键操作最高 8.6×；无损重排保持准确率 |
| 核心创新 | 面向 V:N:M SPTC 的对称、无损、双层图重排序与 GPU 实现 |
| 主要局限 | 启发式且可能耗时；极稀疏图可能 slowdown；依赖 A100/SPTC 栈；未验证 RISC-V 迁移 |
| 与 RISC-V 研究的相关性 | 中：硬件约束感知和无损布局思想可迁移，但正文只验证 NVIDIA GPU |
| 最适合作为 | 传统 GPU 稀疏编译/运行时优化 baseline、硬件感知布局模块、后续模式选择研究的执行器 |

> 这篇论文最值得学习的是把硬件的结构化稀疏约束转化为保持图语义的离线布局搜索；最主要的局限是启发式重排和 NVIDIA SPTC 依赖。用于后续研究时，合理方式是把它作为硬件感知稀疏布局基线或执行器，而不是简单把 GPU 替换为 RISC-V 就宣称产生了新的编译器贡献。
