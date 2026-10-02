# Towards Efficient Compiler Auto-tuning 文献阅读总结

论文题目：**Towards Efficient Compiler Auto-tuning: Leveraging Synergistic Search Spaces**

作者：Haolin Pan、Yuanyu Wei、Mingjie Xing、Yanjun Wu、Chen Zhao

发表时间：2025 年 3 月（CGO ’25 proceedings）

发表平台：Proceedings of the 23rd ACM/IEEE International Symposium on Code Generation and Optimization（CGO ’25），pp. 614–627
元数据核验来源：[CGO 2025 官方论文页](https://2025.cgo.org/details/cgo-2025-papers/48/Towards-Efficient-Compiler-Auto-tuning-Leveraging-Synergistic-Search-Spaces)；[ACM DOI](https://doi.org/10.1145/3696443.3708961)

论文链接或编号：[DOI 10.1145/3696443.3708961](https://doi.org/10.1145/3696443.3708961)；[CGO 2025 官方论文页](https://2025.cgo.org/details/cgo-2025-papers/48/Towards-Efficient-Compiler-Auto-tuning-Leveraging-Synergistic-Search-Spaces)
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

关键词：编译器自动调优、LLVM pass、协同 pass 对、K-means、监督学习、GEAN、遗传算法、强化学习

> 本文档用于文献阅读、组会汇报和后续研究分析。论文事实与阅读后的研究思考严格分开；事实主要依据 14 页 CGO ’25 出版 PDF。

---

## 1. 研究背景

本文研究的是传统机器学习辅助的编译器自动调优（compiler auto-tuning）。编译器通常提供大量优化 pass，用来减少程序的中间表示（Intermediate Representation，IR）指令数、代码尺寸或运行开销。手工决定 pass 的排列顺序很困难；论文指出，在 124 个 pass 中搜索长度为 45 的序列时，组合规模已经极大，无法依赖人工枚举（第 1 节）。

传统自动调优主要包括两类：一类是逐程序的迭代编译和搜索，例如 OpenTuner、BOCA、CompTuner、TPE 等；另一类是利用机器学习或强化学习，直接根据程序特征预测较好的 pass 序列或搜索策略。逐程序搜索通常需要大量编译调用，而跨程序泛化的学习方法又依赖程序表示、模型结构和训练分布，面对未见程序时可能退化（第 1、3 节）。

论文的切入点不是让语言模型直接改写代码，而是利用 LLVM pass 之间的协同关系缩小搜索空间：先找到能够连续连接的有益 pass 对，再把相似程序的 pass 对集合聚类成多个 coreset，最后用监督学习模型为新程序选择更合适的 coreset，并在其中搜索 pass 序列。这样把“选择搜索空间”和“在空间中搜索”分开，以减少无效编译调用。

## 2. 论文要解决的问题

### 2.1 如何减少 pass 序列搜索中的无效空间

论文关注的第一个问题是：完整的 LLVM 优化 pass 空间过大，随机或逐步搜索会把大量预算花在不能继续改善目标的 pass 上。论文先观察到，只保留能降低 IR 指令数的 pass，就可以在较少搜索次数下接近完整空间的最好结果（第 2.1 节）。

### 2.2 如何利用 pass 之间的顺序协同关系

第二个问题是：单独评估 pass 或只依据当前贪心收益，可能错过“前一个 pass 为后一个 pass 创造机会”的组合。论文因此定义 chained synergy pass pair：若 pass B 单独作用有益，且先执行 A 再执行 B 比只执行 B 获得更低的 IR 指令数，则 `(A, B)` 被保留为协同 pass 对（第 4.1 节）。

### 2.3 如何把协同关系泛化到未见程序

第三个问题是：为每个新程序单独构造协同关系仍然昂贵，而且某个程序的 pass 对不一定适用于另一个程序。论文通过 pass 对影响向量和 K-means 聚类建立多个共享 coreset，再训练图神经网络预测新程序更适合的 coreset（第 4.2–4.3 节）。

> 本文主要研究：如何利用 LLVM pass 的链式协同关系、聚类和监督学习，在保持跨程序泛化能力的同时，快速搜索未见程序的近似最优 pass 序列。

## 3. 核心方法概述

论文提出的方法由四步组成：从训练程序中找 chained synergy pass pairs；把相似程序的 pass 对集合合并为 coreset；用 GEAN（Graph Edge Attention Network，带动态边表示的图注意力网络）预测测试程序偏好的 coreset；在预测的 coreset 内用随机、贪心、遗传算法或强化学习搜索 pass 序列。论文把 IR 指令数减少作为主要演示目标，并另外用代码尺寸减少验证搜索空间的实际效果。

```text
CompilerGym 程序
        ↓
LLVM 10.0.0 / opt 评估单个 pass 与 pass 对
        ↓
为训练程序构造 chained synergy pass pair 集合
        ↓
用 pass 对影响向量做 K-means 聚类并合并为 k 个 coreset
        ↓
改进版 ProGraML 程序图特征 + GEAN 监督学习
        ↓
为未见程序预测最合适的 coreset
        ↓
随机 / 贪心 / GA / RL-PPO 在该 coreset 中搜索
        ↓
输出 pass 序列，并由 LLVM 工具产生优化后的 LLVM IR 或二进制
```

输入是 CompilerGym 中经过过滤的程序及其 LLVM IR；中间模型输入是程序图特征；搜索动作是连接 chained synergy pass pairs；输出是一个 pass sequence，不是模型直接生成的源代码、LLVM IR 文本或机器码。论文中的学习模型是 GEAN 图神经网络，属于传统机器学习编译优化流程，不是大语言模型。编译器负责真正执行 pass，`opt`、`llvm-size` 等工具提供目标指标。

论文的核心结构区别于直接预测完整 pass 序列的方法：GEAN 先预测搜索空间，GA/RL 等算法再在被缩小的空间内寻找具体序列。该结构也允许不使用复杂搜索器时，用 100 条随机序列获得较好的结果。

## 4. 实验框架与训练流程

本文不涉及大语言模型预训练或 SFT。它使用监督学习训练 GEAN，并把强化学习作为一种可选的 pass 序列搜索策略，而不是训练语言模型。

### 4.1 Chained synergy pass pair 发现

对训练程序 P，先计算原始或当前基线的 IR 指令数 `V_P`。对每个 pass B，先单独应用 B；只有当 `V_P^B < V_P` 时，才继续枚举每个 pass A，检查先执行 A 再执行 B 后的 `V_P^AB` 是否小于 `V_P^B`。满足条件的 `(A, B)` 放入该程序的集合 S。

论文使用 LLVM 10.0.0 的 `opt` 工具和 124 个 transformation passes，在训练集 19,603 个程序上运行该过程。最终有 8,257 个程序产生非空的 chained synergy pass pair 集合；跨这些集合枚举出 1,548 个不同的 pass 对（第 4.1–4.2 节）。

### 4.2 Coreset 构造

对每个程序，根据 1,548 个枚举 pass 对构造长度为 E 的影响向量。若 pass 对属于该程序的集合，就记录其 IR 指令数减少百分比；否则相应位置为 0。该向量描述程序对不同协同 pass 对的偏好。

论文使用 K-means 对这些影响向量聚类，并对同一簇程序的 pass 对集合取并集，得到 k 个 chained synergy pass pair coresets。论文正文使用符号 k 表示簇数，但在可读正文中没有明确给出最终 k 的固定数值。

### 4.3 GEAN 监督学习选择 coreset

从每个 coreset 中随机抽取 100 条长度为 4–20 的 pass sequence。对训练程序 i，计算每个 coreset 中 100 条序列所能取得的最大 IR 指令数减少百分比，形成长度为 k 的偏好向量。改进版 ProGraML 生成程序图特征，GEAN 根据这些特征预测程序对各 coreset 的概率分布。训练使用温度参数 T 的 KL-divergence-based cross-entropy；训练完成后，对测试程序选择预测概率最高的 coreset。

### 4.4 Coreset 内搜索

论文比较四种搜索方式：

1. 随机策略生成 100 条长度 4–20 的合法序列，选 IR 指令数最小者。
2. 贪心策略从收益最好的 pass 对开始，逐步接上能继续降低 IR 指令数的后继 pass，最大长度为 20。
3. 遗传算法以随机策略生成的候选作为初始种群，按适应度选择、在公共节点处交叉，并在出度大于 1 的节点上变异。
4. 强化学习策略把 coreset 表示为有向图，利用 action masking 只保留与当前末尾 pass 相接的动作；状态使用 Autophase 的 56 个特征，逐步选择 pass 对，直到达到最大步数或没有合法动作。

### 4.5 代码尺寸搜索实验

为验证实际代码尺寸优化，论文把目标从快速评估的 IR 指令数切换为 x86_64-unknown-linux-gnu 上的 `.text` 代码尺寸。该实验不做 coreset 聚类和 GEAN 训练，而是在来自 5 个训练集的 2,700 个 chained synergy pass pairs 上比较随机和 GA 搜索与其他单程序搜索方法。

## 5. 奖励函数、损失函数或关键公式

### 5.1 Chained synergy 判定

论文的 pass 对判定条件可概括为：

```text
V_P^B < V_P  且  V_P^AB < V_P^B
```

其中 `V_P` 是程序 P 的基准 IR 指令数，`V_P^B` 是只应用 B 后的指令数，`V_P^AB` 是先应用 A、再应用 B 后的指令数。该条件只保留 B 本身有益且 A 能进一步帮助 B 的有序 pass 对。

### 5.2 K-means 聚类目标

论文使用 K-means 的簇内平方和：

```text
WCSS = Σ_j Σ_(v∈C_j) ||v - μ_j||²
```

`C_j` 是第 j 个簇，`μ_j` 是该簇均值，`v` 是程序的 pass 对影响向量。最小化 WCSS 的目标是把 pass 对收益模式相近的程序放在一起，以便合并出更有代表性的 coreset。

### 5.3 监督学习的 coreset 偏好

对程序 i 和第 j 个 coreset，论文定义：

```text
p_ij = max_(n=1..100) [InstC_i(O0) - InstC_i(s_j^n)] / InstC_i(O0)
```

`O0` 是未优化程序，`s_j^n` 是第 j 个 coreset 中的第 n 条随机 pass sequence，`InstC` 是 IR 指令数。`p_ij` 越大，表示程序 i 越偏好该 coreset。

### 5.4 GEAN 损失函数

论文用温度 T 调节后的目标分布监督 GEAN 输出分布：

```text
L(D_i, a_i) = - Σ_j Softmax(D_i / T)_j log(a_ij)
```

`D_i` 是程序 i 的 coreset 偏好向量，`a_i` 是 GEAN 输出的概率分布，`T` 是温度参数。该损失希望模型学习程序图特征与适合搜索空间之间的关系。论文没有把这一监督目标称为强化学习奖励，也没有报告该损失的具体数值或 T 的数值。

### 5.5 强化学习搜索奖励

强化学习策略每一步的奖励是相对于上一步 IR 指令数的 reduction ratio，总奖励是各步 reduction ratio 的累积。action masking 约束合法的后继 pass 对。该 RL 只用于搜索 pass sequence；论文没有提出新的 RL 算法或独立奖励设计，且正文指出 action mask 可能导致状态价值不变、训练不稳定。

### 5.6 评价指标

论文以相对于 `opt Oz` 的平均减少百分比评价优化效果：

```text
MeanOverOz = (1 / |P|) Σ_(p∈P) (I_Oz^p - I_πθ^p) / I_Oz^p × 100%
```

`P` 是数据集，`I_Oz^p` 是程序 p 经 Oz 后的 IR 指令数或代码尺寸，`I_πθ^p` 是搜索策略得到的结果。该指标越大表示相对于 Oz 的减少越多。代码尺寸由 `llvm-size` 测量 `.text` section，不等同于 IR 指令数，也不等同于运行时间。

## 6. 实验设置

### 6.1 数据集来源

数据来自 CompilerGym。作者在划分前保留 IR 指令数小于 10K 的程序，以与 GEAN-NVP 的实验设置保持一致；论文明确说明较大程序可能对 pass sequence 有不同响应。训练、验证和测试划分如下：

| 类型 | 数据集 | Train | Val | Test |
|---|---|---:|---:|---:|
| Uncurated | blas-v0 | 133 | 28 | 29 |
| Uncurated | github-v0 | 7,000 | 1,000 | 1,000 |
| Uncurated | linux-v0 | 4,906 | 1,000 | 1,000 |
| Uncurated | opencv-v0 | 149 | 32 | 32 |
| Uncurated | poj104-v1 | 7,000 | 1,000 | 1,000 |
| Uncurated | tensorflow-v0 | 415 | 89 | 90 |
| Curated | cbench-v1 | 0 | 0 | 11 |
| Curated | mibench-v1 | 0 | 0 | 40 |
| Curated | chstone-v0 | 0 | 0 | 12 |
| Curated | npb-v0 | 0 | 0 | 121 |
| **Total** | — | **19,603** | **3,149** | **3,335** |

手工整理的 cbench、MiBench、CHStone 和 NPB 只用于测试时评估真实程序域的泛化，不参与训练。论文未对数据泄漏风险做独立实验；从正文可确认的是，训练/验证/测试按上述数据集划分，且 curated 数据集只出现在测试侧。

### 6.2 模型与工具

* 编译器工具：LLVM 10.0.0、`opt`、`clang`、`llvm-size`。
* 优化动作：`opt` 中可用的 124 个 transformation passes。
* 程序表示：改进版 ProGraML 程序图特征；强化学习搜索另外使用 Autophase 的 56 个特征。
* 预测模型：GEAN 图边注意力网络。
* 搜索方法：随机、贪心、遗传算法、RL-PPO。
* 代码尺寸目标平台：x86_64-unknown-linux-gnu。
* artifact 环境：Ubuntu 22.04-LTS、AMD EPYC 7713 64-Core Processor；论文附录说明其复现实验主要使用 LLVM 10.0.0 和 Python 包 ray、torch、numpy 等。
* 训练模型的参数量、GEAN 的具体层数以及硬件执行时间，论文中未明确说明。

### 6.3 对比方法

IR 指令数实验的主要对比包括 GEAN-NVP、GEAN-NVP Oracle、本文的 Random、Greedy、GA 和 Autophase-RL-PPO。代码尺寸实验比较 OpenTuner、BOCA、GA、RIO、TPE、compTuner，以及本文的 Random 和 GA。相关工作还讨论了 AutoPhase、SRTuner、ICMC 等方法，但它们不都出现在主结果表中。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| IR instruction count reduction | 相对于 `opt Oz` 的 LLVM IR 指令数减少百分比 | 越大越好 |
| Codesize reduction | 相对于 `opt Oz` 的目标二进制 `.text` section 尺寸减少百分比 | 越大越好 |
| Samples/Program | 每个程序使用的搜索或采样预算 | 越少越高效，但不能单独代表质量 |
| Average search time | 搜索一组结果所需平均时间 | 越小越好 |

## 7. 实验结果与结论

### 7.1 主要结果

在十个数据集测试集上，以 `opt Oz` 为基线，表 3 报告的平均 IR 指令数减少为：GEAN-NVP 3.7%、GEAN-NVP Oracle 4.8%、本文 Random 6.7%、本文 Greedy 4.9%、本文 GA 7.5%、本文 Autophase-RL-PPO 6.5%。因此，论文方法中 GA 在平均 IR 指令数指标上最好，与 Oz 相比平均减少 7.5%。这些是跨测试集平均的 IR 指令数结果，不是实际运行时间加速比。

表 4 显示 GA 在多数数据集上最好：例如 NPB 为 25.9%、POJ104 为 9.6%、CHStone 为 11.9%；GitHub 和 Linux 的平均减少分别为 1.1% 和 0.6%。不同数据集的收益差异较大，说明程序域仍会影响 pass 协同关系。

### 7.2 与传统搜索方法的比较

在五个数据集的代码尺寸实验中，所有结果均相对于 x86_64-unknown-linux-gnu 上的 `opt Oz`：OpenTuner 平均减少 11.8%，BOCA 10.4%，普通 GA 11.1%，RIO 9.9%，TPE 10.2%，compTuner 9.0%，本文 Random 13.2%，本文 GA 13.9%。本文 GA 的平均搜索时间为 6 秒，本文 Random 为 4 秒；对照方法的平均时间分别为 200 秒、3310 秒、593 秒、200 秒、905 秒和 10800 秒。代码尺寸结果采用正文表 5 的 13.9%；CGO 官方网页摘要显示 13.6%，与出版 PDF 摘要、表 5 和结论中的 13.9% 不一致，本笔记以本地正式 PDF 正文为准。

### 7.3 与其他机器学习方法的比较

GEAN-NVP 能直接预测长度为 45 的 pass sequence，并为每个 pass 进行采样；本文 Random 只使用 100 条序列就达到 6.7%，高于 GEAN-NVP 的 3.7%。与 GEAN-NVP Oracle 的 4.8% 相比，论文称本文使用约 1/22 的采样预算而得到约 1.4 倍的 IR 指令数减少。本文生成的序列最大长度为 20，而 GEAN-NVP 和 Oz 的序列长度分别为 45 和 97。

### 7.4 消融实验

表 6 的平均结果如下：

* 随机搜索：原始连接规则 6.7%；取消连接规则 4.8%；取消聚类 5.2%；取消监督学习 4.0%。
* GA：原始方法 7.5%；取消聚类 5.4%。

取消连接规则会破坏后继 pass 对的可连接性，尤其在 NPB 上损失明显。取消聚类后，搜索空间不再集中反映相似程序的共同优化机会；GA 的下降尤其明显。取消监督学习后固定使用平均表现最好的 coreset，结果弱于为每个程序单独预测 coreset，说明不同程序确实需要不同的协同搜索空间。

### 7.5 案例分析

论文给出两类协同关系示例：`--dce` 与 `--mem2reg` 可以独立作用于不同代码区域，属于 complementary pair；`--simplifycfg` 与 `--instcombine` 中，先做指令合并可能为控制流简化创造更多机会，属于 contributing pair。图 4 进一步说明两个相接 pass 对可以通过共享节点组合成更长的搜索路径。

论文没有把某个单独成功程序提升为通用保证，也没有报告形式化语义等价证明。相反，结论和附录承认 pass 序列可能导致程序错误或崩溃，未来需要进一步降低该风险。

## 8. 主要创新点

### 8.1 创新点一：可连接的 chained synergy pass pair 定义

现有逐程序搜索往往直接在巨大 pass 空间中试探，或者仅用当前收益做贪心选择。本文用“B 本身有益且 A→B 比 B 单独更有益”的条件构造有方向的 pass 对，并在后续搜索中强制共享节点连接。该定义把 pass 顺序关系转化为可复用的搜索图，表 6 的 Non-Connecting 结果低于 Original，说明连接规则确实影响效果。

### 8.2 创新点二：用聚类得到跨程序可复用的协同搜索空间

本文不是为每个程序保留孤立的 pass 对集合，而是用 pass 对收益向量做 K-means，再把相似程序的集合合并为 coresets。这样既减少了单程序构造成本，又保留了多种程序类型的优化机会。取消聚类后的 GA 平均结果从 7.5% 降到 5.4%，为该设计提供了实验支持。

### 8.3 创新点三：把监督学习用于“搜索空间选择”而不是直接生成完整序列

GEAN 不直接输出一条固定 pass sequence，而是预测每个程序最适合的 coreset，再交给搜索器生成序列。这个分层设计降低了模型直接预测长序列的压力，并允许复用随机、GA 或 RL 等多种搜索策略。本文的贡献是这种协同搜索空间与选择器的组合，不是单独使用 K-means、GNN 或 GA。

## 9. 局限性

### 9.1 论文明确承认的局限

* 未来需要探索更复杂的搜索策略和其他聚类方法。
* 当前实验重点是 IR 指令数和代码尺寸，未来计划扩展到执行时间、能耗等目标。
* pass 序列组合可能引入程序错误或崩溃，论文未来工作计划设计降低错误风险的策略。
* artifact 附录说明完整复现实验准备约需 3 天、全部搜索约需 6 小时，说明大规模重现实验仍有成本。

### 9.2 阅读后发现的潜在局限

* 评测主要基于 LLVM 10.0.0、124 个 pass 和 IR 指令数/代码尺寸，不能直接推断对现代 LLVM 或其他编译器版本同样有效。
* 代码尺寸实验限定在 x86_64-unknown-linux-gnu；论文没有 RISC-V、RVV、真实硬件性能或跨 ISA 迁移实验。
* 10K IR 指令数过滤和 curated 数据集仅用于测试，使结论主要适用于这一规模与数据划分；复杂大程序、跨函数优化和更长循环变换的表现，论文中未明确说明。
* 论文把“搜索空间选择”和“序列搜索”分层，但 coreset 数量 k、GEAN 的完整超参数与模型规模没有在正文中充分给出，影响独立复现和公平比较。
* IR 指令数减少和 `.text` 代码尺寸减少并不等价于运行时间或能耗改善；论文没有用真实执行时间作为主指标，因此不能把 7.5% 或 13.9% 解释为端到端性能加速。
* 论文用有限的编译器执行结果评价候选，没有引入 Alive2 等形式化验证；结论中承认错误/崩溃风险，语义安全性仍是后续问题。

## 10. 阅读后的研究方向反思

### 10.1 值得借鉴的思想

最值得借鉴的是把复杂的编译优化搜索拆成“程序表示 → 搜索空间选择 → 候选序列搜索 → 编译器反馈”。对于 LLVM IR 或 RISC-V 后端研究，这比让学习模型直接输出长 pass 序列更容易解释，也便于替换搜索器和目标指标。

### 10.2 不能简单照搬的核心贡献

chained synergy pair 的定义、K-means coreset 构造和 GEAN 选择器共同构成本文的主要方法。只把 LLVM 10.0.0 换成 RISC-V 或 RVV，若仍采用相同 pass 对定义和同样的 IR 指令数目标，更多是平台迁移，不能自动形成新的研究贡献。若要形成新问题，需要证明 RISC-V/RVV 的指令选择、向量长度、寄存器压力或真实硬件代价会改变 pass 协同关系和搜索空间结构。

### 10.3 与本研究方向的关系

本文与“机器学习辅助 LLVM 优化”直接相关，与 RISC-V 的关系有限：论文的输入和主要工具是 LLVM，代码尺寸实验目标为 x86_64；RISC-V/RVV 未在正文中作为实验平台出现。因此它适合作为传统 ML pass-ordering baseline、协同搜索空间构造方法和实验框架参考，而不是现成的 RISC-V 方法。

### 10.4 更适合的复用位置

在后续研究中，本文最适合作为 baseline 或搜索空间模块：可以把其 coreset 选择器与新的硬件代价模型、运行时 profiling、RISC-V 性能计数器或形式化验证器组合。但不能把本文的 IR 指令数下降直接当成安全的语义保持优化，也不能把 GA/RL 搜索结果直接等同于真实硬件加速。

## 11. 可进一步尝试的研究方向

以下是基于本文的研究设想，不是本文已经完成的工作，也不包含单篇笔记所要求之外的最小 Demo。

### 11.1 面向 RISC-V/RVV 的硬件感知协同 pass 空间

#### 研究问题

RISC-V/RVV 的向量长度、寄存器压力、指令扩展和后端 lowering 是否会产生不同于 x86_64 的 pass 协同关系？

#### 与原论文的区别

不只替换目标架构，而是把 RVV 后端代价、向量化覆盖率和寄存器溢出代价加入 pass 对的定义与 coreset 聚类。

#### 可能的创新点

构造多目标协同向量，同时描述 IR 指令数、RVV 指令数、静态寄存器压力和真实硬件性能；学习能跨 RVV 向量长度迁移的 coreset 表示。

#### 实验框架

```text
LLVM IR / RVV 候选程序
        ↓
LLVM pass 对与 RVV lowering
        ↓
硬件计数器 + 指令统计 + 编译反馈
        ↓
硬件感知协同向量与聚类
        ↓
GEAN 或代价模型选择 coreset
        ↓
搜索 pass sequence 并在多种 RVV 配置验证
```

#### 可行性

需要 LLVM/RISC-V 工具链、Spike 或真实 RVV 硬件、CompilerGym 风格数据和可重复的性能测量流程。

#### 主要风险

硬件噪声和向量长度差异可能使 pass 对收益不稳定；静态指标与真实执行时间的相关性也需要单独验证。

### 11.2 协同搜索与形式化验证联合调优

#### 研究问题

如何在协同 pass 对搜索过程中尽早发现错误或未定义行为，并避免把错误序列继续用于训练或遗传交叉？

#### 与原论文的区别

本文只依赖编译结果和 IR 指标，并在结论中把错误风险留作未来工作；新方向把 translation validation 或等价性检查作为候选过滤/反馈信号。

#### 可能的创新点

设计“协同收益 + 验证成本 + 语义安全”的多信号搜索目标，并研究验证失败如何更新 pass 对图和 coreset。

#### 实验框架

```text
候选 pass sequence
        ↓
LLVM 编译与 IR/机器码生成
        ↓
Alive2 或差分测试检查
        ↓
安全性与性能联合反馈
        ↓
更新 coreset、屏蔽危险边、继续搜索
```

#### 可行性

需要 LLVM IR 语义验证工具、可生成测试输入的基准程序集以及可处理验证失败的搜索器。

#### 主要风险

验证工具可能对复杂 IR 超时；有限测试不能替代形式化证明，验证成本也可能抵消本文强调的搜索效率。

### 11.3 面向执行时间与能耗的动态 coreset 选择

#### 研究问题

当优化目标从 IR 指令数/代码尺寸转为真实执行时间和能耗时，静态 pass 对协同关系是否仍稳定？

#### 与原论文的区别

不再把指令数作为唯一代理指标，而是把输入规模、运行时 profiling 和硬件计数器纳入程序表示，并允许 coreset 随输入或硬件状态变化。

#### 可能的创新点

研究多目标 coreset 偏好预测、噪声鲁棒的收益向量和跨输入规模的动态选择策略。

#### 实验框架

```text
程序与输入规模
        ↓
编译器 pass 搜索
        ↓
真实硬件运行、性能计数器与能耗测量
        ↓
动态收益向量
        ↓
预测 coreset 并搜索
        ↓
验证时间、代码尺寸和能耗的 Pareto 结果
```

#### 可行性

需要稳定的硬件计时/能耗采集、输入分层的 CompilerGym 数据和多目标搜索实现。

#### 主要风险

运行时测量噪声、硬件频率变化和输入分布偏移可能使训练标签不稳定；不同目标之间也可能产生冲突。

## 12. 与其他已读文献的关系

本次并行子任务只正式处理这一篇 CGO 2025 论文，因此没有同批次内可用的第二篇论文可做事实级横向比较。就仓库主题而言，本文与传统 ML 编译优化和 pass-ordering 条目相近，适合作为“监督学习选择搜索空间 + 编译器执行 pass”的 baseline；它与仓库中的 LLM 直接生成源代码/IR 的 Translator 类论文不同，也没有把模型作为 Compiler Pass Generator。

论文自身将 GEAN-NVP、AutoPhase、SRTuner、ICMC、OpenTuner、BOCA 和 CompTuner作为相关工作或对比背景，但本笔记只依据当前 PDF 对它们在本文实验中的角色进行有限描述，不据此扩展其他论文的实验结论。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 利用 pass 协同关系进行 LLVM 编译器自动调优 |
| 核心问题 | 在巨大 pass 序列空间中快速找到未见程序的近似最优序列 |
| 输入 | CompilerGym 程序、LLVM IR、程序图特征 |
| 输出 | 由 LLVM `opt` 执行的 pass sequence 及优化后的 IR/二进制 |
| 核心方法 | chained synergy pass pairs、K-means coresets、GEAN 选择、随机/贪心/GA/RL 搜索 |
| 使用的模型 | 改进版 ProGraML 特征 + GEAN；无大语言模型 |
| 使用的编译器工具 | LLVM 10.0.0、`opt`、`clang`、`llvm-size` |
| 是否使用强化学习 | 使用 RL-PPO 作为一种搜索策略；不是主要监督模型训练方法 |
| 是否使用形式化验证 | 否；论文未使用 Alive2 等形式化验证，并承认错误/崩溃风险 |
| 数据集规模 | Train 19,603、Val 3,149、Test 3,335；均来自 CompilerGym，过滤为小于 10K IR 指令 |
| 主要指标 | 相对 `opt Oz` 的 IR 指令数减少、`.text` 代码尺寸减少、搜索时间 |
| 最重要实验结果 | IR 指令数：本文 GA 平均减少 7.5%；代码尺寸：本文 GA 平均减少 13.9%，平均搜索约 6 秒 |
| 核心创新 | 把 pass 协同关系组织成可连接、可聚类、可预测选择的搜索空间 |
| 主要局限 | 依赖 LLVM 10.0.0 和固定 pass 集；主要用静态代理指标；无 RISC-V/真实硬件/形式化安全验证 |
| 与 RISC-V 研究的相关性 | 中低：LLVM pass 搜索思想可迁移，但正文没有 RISC-V/RVV 实验，不能直接声称适用 |
| 最适合作为 | 传统 ML 编译优化 baseline、协同搜索空间方法参考、后续硬件感知搜索模块 |

> 这篇论文最值得学习的是把“选择搜索空间”和“搜索具体 pass 序列”分层，并用 pass 协同关系提升搜索效率；最主要的局限是语义安全、真实运行时间和跨硬件泛化尚未解决；如果用于后续研究，最合理的使用方式是把它作为传统 LLVM pass-ordering baseline 或搜索空间模块，而不是简单把代码尺寸/IR 指令数下降当成真实硬件加速或形式化正确性证明。
