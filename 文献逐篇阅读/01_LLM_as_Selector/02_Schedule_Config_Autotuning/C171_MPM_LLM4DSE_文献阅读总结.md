# MPM-LLM4DSE 文献阅读总结

论文题目：**MPM-LLM4DSE: Reaching the Pareto Frontier in HLS with Multimodal Learning and LLM-Driven Exploration**  
作者：Lei Xu、Shanshan Wang、Chenglong Xiao  
发表时间：2026；arXiv 首次公开于 2026-01-08，论文首页注明 accepted for DATE 2026  
发表平台：DATE 2026
论文链接或编号：DOI 10.23919/DATE69613.2026.11539388；arXiv:2601.04801
元数据核验来源：[arXiv（明确列出代码链接）](https://arxiv.org/abs/2601.04801)；[作者代码仓库](https://github.com/wslcccc/MPM-LLM4DSE)；[DOI](https://doi.org/10.23919/DATE69613.2026.11539388)
代码/数据/工件：作者代码与模型：[MPM-LLM4DSE](https://github.com/wslcccc/MPM-LLM4DSE)
关键词：高层次综合（High-Level Synthesis, HLS）、设计空间探索（Design Space Exploration, DSE）、多模态图学习、LLM、pragma 配置、QoR 预测

## 1. 研究背景

本文研究 HLS 设计空间探索。HLS 把 C/C++ 等高层行为描述转换为 RTL，再面向 FPGA 或 ASIC 综合。设计者可以通过 pragma/directive 控制 tiling、unrolling、pipeline 等结构与调度决策，但配置组合空间呈组合爆炸，逐个运行 HLS 工具取得延迟、LUT、DSP、FF、BRAM 等质量结果（Quality of Results, QoR）代价很高（引言、图 1）。

既有方法一方面使用 CDFG（Control and Data Flow Graph，控制与数据流图）和 GNN 作为 HLS 工具的 QoR 代理模型，另一方面使用进化算法、强化学习或多目标优化算法探索 pragma 组合。论文认为，仅有 CDFG 难以表达 pragma 在源代码中的语义范围和程序员意图；传统搜索也没有系统利用 pragma 对 QoR 的影响知识。本文引入 LM/LLM：CodeBERT-c 用于 QoR 预测特征，LLM4DSE 用 LLM 生成和更新设计配置。

## 2. 论文要解决的问题

### 2.1 QoR 预测缺少源代码语义

论文第 I 节和第 II-A/B 节指出，CDFG 主要表达结构、控制流和数据流，难以完整表示源代码语义、pragma 作用范围和隐含意图。例如 `UNROLL factor=4` 可能被加入图节点，但图模型未必能理解它影响哪些语句块以及为什么影响延迟或资源。

### 2.2 图与文本特征难以有效融合

ProgSG 等工作尝试对齐图节点和代码 token，但论文认为单一 Transformer 架构仍可能不足以表达复杂的源代码和图结构关系。因此本文研究如何以 GNN 得到结构表示、以 LM 得到文本表示，再用多头注意力和门控机制融合。

### 2.3 大型配置空间中的 DSE 效率

论文第 I、III-C 节关注如何在无需遍历所有 pragma 组合的情况下，利用 LLM 的语义理解与领域提示知识生成高质量配置，并通过 QoR 评估迭代更新。本文主要研究：如何结合多模态 QoR 代理模型与带 pragma 影响知识的 LLM 配置生成，改善 HLS 的 Pareto 设计空间探索。

## 3. 核心方法概述

MPM-LLM4DSE 包含三个模块：数据生成器构建 Graph-Text 样本；MPM（Multimodal Predictive Model，多模态预测模型）融合 CDFG 与 pragma-增强源代码并预测 QoR；LLM4DSE 使用 PEODSE（Prompt Engineering for Optimization in DSE）提示策略生成配置、评估配置并循环更新。

```text
C/C++ 源代码 + pragma 配置 + 目标硬件
        ↓
LLVM / ProGraML 生成 IR 与 CDFG；LM 提取代码语义
        ↓
ECoGNN 提取图表示 + CodeBERT-c 提取文本表示
        ↓
多头注意力与门控融合 → MPM 预测 Latency/LUT/DSP/FF/BRAM
        ↓
PEODSE 将任务知识、优质配置和 CoT 示例提供给 LLM
        ↓
LLM 生成新的 pragma 配置
        ↓
预测/必要时由 HLS 工具评估 → 更新 Pareto 集与提示
```

LLM 的最终角色是配置/候选选择器，而不是直接改写源代码或输出 RTL。论文第 III-C 节给出的探索环节是初始化/生成方案、评价方案、更新最优集合和重构提示，直到达到最大配置数量等终止条件。

## 4. 实验框架与训练流程

本文不训练用于直接生成源码的编译模型；它训练一个 QoR 预测模型，并在 DSE 阶段调用 LLM 生成配置。

### 4.1 数据生成与 QoR 标签

第 III-A 节描述：源代码经 LLVM 得到 IR，再由 ProGraML 转成 CDFG；pragma inserter 向图中加入与 pragma 相关的信息；同时将配置与行为描述合并后送入 CodeBERT-c，得到文本特征。HLS 报告提供 Latency、LUT、DSP、FF、BRAM 标签。

### 4.2 MPM 训练

第 III-B 节中，ECoGNN 处理有方向的图信息，global node attention 得到图级嵌入；文本嵌入作为 Key/Value，图嵌入作为 Query，经多头注意力融合，再由 gated network 控制图特征和融合特征的信息流，最终由 MLP 预测 QoR。模型使用 Adam，隐藏维度 128、batch size 64、学习率 0.001，训练超过 500 次迭代（第 IV-A 节）。

### 4.3 LLM4DSE 推理循环

第 III-C 节中，PEODSE 先指导 LLM 初始化方案；数据生成器生成图文表示；MPM 评估；最优方案集合更新后重构提示，继续生成下一批配置。DSE 对比使用 GPT-4o 与 Qwen3-235B-A22B-Thinking-2507；论文还说明当前实现依赖 API 调用。

### 4.4 训练范式边界

论文没有报告 SFT、PPO、GRPO 或其他 LLM 强化学习训练。LLM 侧主要是提示工程、few-shot/CoT 式示例和迭代配置生成；强化学习出现在相关工作中的既有 HLS DSE baseline，不是本文 LLM4DSE 的训练算法。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数。关键目标和损失如下。

### 5.1 QoR 预测目标

论文第 II-A 节给出：

```text
Target_i = MLP_i(Fuse(hG, hS))
```

`hG` 是 CDFG 的图级结构表示，`hS` 是源代码与配置的文本语义表示，`Fuse` 是参数化融合机制，`Target_i` 是第 i 个 QoR 目标。

### 5.2 文本表示

第 II-B 节给出 `hS` 为各层 CLS 隐状态的平均聚合。输入由设计配置 `dc` 与行为描述 `bd` 合并并经 tokenizer 处理。该表示的目标是同时保留代码语义和配置影响。

### 5.3 MPM 的损失

第 III-B 节说明使用 RMSE loss 更新预测模型。论文没有给出一个额外的 LLM 奖励函数。选择 RMSE 的理由是 latency 数值可达数十万时钟周期，1% 误差对应大量周期；但论文没有给出训练损失的完整数值权重展开。

### 5.4 DSE 评价目标

第 IV-C 节使用 ADRS（Average Distance from the Referenced Set）衡量近似 Pareto 集与参考 Pareto 集的距离：

```text
ADRS(Γ, Ω) = (1 / |Γ|) × Σλ∈Γ minμ∈Ω f(λ, μ)
```

`Γ` 是参考 Pareto 集，`Ω` 是近似 Pareto 集，`f` 是距离函数；ADRS 越低越好。该指标不是奖励函数，而是 DSE 结果评价指标。

## 6. 实验设置

### 6.1 数据集来源

第 IV-A 节基于 GNN-DSE 数据构建多模态数据。训练集包含 MachSuite 的 5 个 kernel 和 Polyhedral benchmarks 的 10 个 kernel；表 II 总计 15 个 benchmark、4,353 个 graph-text 样本，按 70%/15%/15% 划分训练、测试和验证。未见 kernel 用于推理和 DSE，表 III 包含 `heat-3d`、`jacobi-1d`、`jacobi-2d`、`nw`、`seidel-2d`、`stencil`，其设计配置空间从 2,871 到 7,609,187 个配置不等。论文没有专门讨论数据泄漏审计，但训练与未见 kernel 的划分用于检验泛化。

### 6.2 模型与工具

CodeBERT-c 用于 C/C++ 文本特征；ECoGNN 用于 CDFG；LLM DSE 对比使用 GPT-4o 和 Qwen3-235B-A22B-Thinking-2507。工具链包括 LLVM、ProGraML、Vitis-HLS 2022.1、Vivado 2022.1；硬件平台是 AMD Ultrascale+ MPSoC ZCU104。论文表 II 还列出 CDFG 节点特征：节点/指令/函数/基本块类型以及 Latency、LUT、DSP、FF。

### 6.3 对比方法

QoR 预测比较 GNN-DSE、HGBO、IronMan-Pro、ProgSG、ECoGNN-only、LM-only 与 MPM。DSE 比较 NSGA-II、模拟退火（SA）、蚁群优化（ACO）、LLMMH、LLM4DSE（GPT-4o）和 LLM4DSE（Qwen3）。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| Latency | HLS 推出的时钟周期延迟 | 越小越好 |
| LUT/DSP/FF/BRAM | FPGA 资源使用量 | 通常越小越好 |
| RMSE | QoR 预测误差 | 越小越好 |
| ADRS | 近似 Pareto 集相对参考集的平均距离 | 越小越好 |
| Runtime | DSE 搜索耗时 | 越小越好，但需结合 ADRS |

## 7. 实验结果与结论

### 7.1 主要结果

表 IV 报告未见应用上的 QoR 预测 RMSE：MPM 的 Latency/LUT/DSP/FF/BRAM 分别为 `0.3870/0.0004/0.0004/0.0015/0.0005`，总列为 `0.3898`。与 ProgSG 的总列 `0.4226` 相比，论文摘要称 MPM 最多达到约 10.25× 的改进；这是预测误差比较，不是硬件运行速度提升。

表 V 的 6 个未见 benchmark 上，LLM4DSE(Qwen3) 平均 ADRS 为 `0.0305`，低于 NSGA-II `0.0447`、SA `0.0572`、ACO `0.0758` 和 LLMMH `0.0388`；相对 LLMMH 的平均改进为 21.39%。同表中 LLM4DSE(Qwen3) 总运行时间为 9,847 s，LLMMH 为 10,941 s，不能将较低 ADRS 与所有方法的运行时间简单等同。

### 7.2 不同 DSE 方法比较

在表 V 的相同设计数量比较中，LLM4DSE(Qwen3) 的平均 ADRS 最低。表 VI 在相同时间约束下，与 GNN-DSE 比较：`heat-3d` ADRS 0.0486 对 0.0689，`jacobi-2d` 0.0035 对 0.0074，`seidel-2d` 0.0666 对 0.0779；论文报告相应 ADRS 改进为 30.78%、52.70%、14.51%。

### 7.3 提示策略比较

第 IV-C 节和图 7 比较 PEODSE 与其他提示方式。论文结论是 PEODSE 收敛曲线更好，原因是它把任务描述、优质解示例、任务指令和解生成示例结合起来，并显式加入 pragma 对 QoR 的影响知识。图 7 的具体曲线数值在正文中未以表格完整列出。

### 7.4 消融实验

表 IV 包含 ECoGNN-only、LM-only 与 MPM。LM-only 总 RMSE `0.3965`，ECoGNN-only `0.4084`，MPM `0.3898`，说明文本语义单独有价值，而图文融合仍进一步降低预测误差。论文没有把所有 PEODSE 组件拆成独立的数值消融表。

### 7.5 结果解释与失败边界

论文将 MPM 的优势归因于多模态融合和更强的文本语义特征；将 LLM4DSE 的优势归因于 pragma 影响知识、高质量配置示例和 CoT 推理。论文没有报告所有配置在真实芯片上的逐项验证，也没有把 ADRS 直接解释为硬件实测加速比。

## 8. 主要创新点

### 8.1 创新点一：Graph-Text 多模态 QoR 数据

论文把 pragma-增强源代码的 LM 表示与 CDFG 结构放到同一训练样本中，补足仅用图表示时难表达源代码语义的问题。它的价值由 MPM 相对 ECoGNN-only/LM-only 的结果支持，但数据基础来自 GNN-DSE 的扩展而非全新独立大规模语料。

### 8.2 创新点二：ECoGNN 与多头注意力融合

论文扩展 CoGNN 以处理不同方向的信息流，并使用图作为 Query、文本作为 Key/Value 的多头注意力，再用门控网络动态控制融合信息。这是论文的模型设计贡献；不能简单概括为“同时用了 GNN 和 LM”。

### 8.3 创新点三：PEODSE 的领域提示策略

PEODSE 将任务背景、pragma 对 QoR 的影响、动态更新的优质解和 CoT 生成示例组织到提示中，使 LLM 生成配置而非盲目随机搜索。其直接价值由表 V 中 LLM4DSE 的 ADRS 结果体现。

### 8.4 工程组合而非独立创新

LLVM、ProGraML、Vitis-HLS、CodeBERT-c、GPT-4o 和 Qwen3 本身不是本文独立创新。本文的核心是把多模态 QoR 代理与 LLM 配置探索组合成 HLS DSE 流程。

## 9. 局限性

### 9.1 论文明确承认或提出的限制

结论部分提出未来要使用更小、经过任务微调、可本地运行的模型，以降低计算和通信开销，并探索跨平台综合。第 IV-C 节说明当前 LLM4DSE 依赖 API 调用，通信延迟可能显著影响整体运行时间。

### 9.2 阅读后的潜在局限

- 任务集中在 HLS pragma DSE、FPGA/ASIC QoR 和指定 benchmark，不能直接推及 LLVM CPU pass ordering 或 RISC-V 后端。
- MPM 训练集只有 15 个 benchmark、4,353 个样本；未见 kernel 泛化结果有价值，但跨工具、跨 FPGA 家族和跨 HLS 版本的泛化仍未充分证明。
- 表 V 的 LLM DSE 使用大型闭源或超大规模模型与 API，复现成本较高；论文没有报告 token 成本、提示长度或 API 失败率。
- QoR 预测指标使用归一化/误差数值，但正文没有完整展开所有归一化细节；阅读时不能把 RMSE 直接当作周期误差。
- 论文以预测模型评估大量配置，真实 HLS 综合只用于生成训练标签和基线数据；代理误差可能造成搜索方向偏差。
- 论文宣称 Transform/pragma 配置保持语义，但 HLS pragma 对资源、时序和综合可行性的约束仍需工具实际验证；预测成功不等于形式化正确性证明。

## 10. 阅读后的研究方向反思

本文对“大语言模型与编译器优化”的启发主要是：让 LLM 输出受控的配置对象，把语义推理和编译器执行分开，并用结构化 QoR 预测器降低真实综合调用次数。对于 RISC-V，不能只把 Vitis-HLS 换成 LLVM/RISC-V 就声称产生新颖性；那只是平台迁移。更有价值的是把 RISC-V 向量长度、寄存器压力、缓存层级和真实硬件计数器纳入配置语义与跨架构代价模型。

这篇论文更适合作为 SELECTOR/S2 的方法参考和 HLS baseline，而不是直接作为源码翻译器或编译器 pass 生成器。其 PEODSE 与多模态 QoR 代理是可借鉴模块；MPM 的特定 ECoGNN 结构和 pragma 数据格式则需要根据目标编译器重新验证。

## 11. 可进一步尝试的研究方向

### 11.1 RISC-V 向量 pragma/调度的真实硬件闭环

#### 研究问题

LLM 能否根据 LLVM IR、循环/内存特征和 RISC-V Vector（RVV）硬件计数器选择可验证的向量化、展开和调度配置？

#### 与原论文的区别

从 HLS pragma QoR 预测扩展到 RVV 真实执行反馈，目标不是 FPGA 资源预测，而是跨微架构的实际性能与能耗。

#### 可能的创新点

把 `VL/VLEN`、寄存器压力、cache miss、分支行为和可行性约束编码为配置语义，并用不确定性门控拒绝高风险配置。

#### 实验框架

```text
LLVM IR + RVV 特征 → LLM 生成配置 → LLVM 编译 → QEMU/真实板卡计数器
→ 正确性测试与性能反馈 → 更新候选配置
```

#### 可行性

需要 LLVM/RVV、至少一种真实 RISC-V 平台、基准程序、性能计数器和配置搜索器。

#### 主要风险

仿真器与真实硬件排序可能不一致，且跨微架构数据量有限。

### 11.2 代理误差感知的多目标配置搜索

#### 研究问题

如何把 QoR 预测置信区间纳入 LLM 的 Pareto 配置选择，减少模型误判导致的无效综合？

#### 与原论文的区别

不只比较点预测 ADRS，而是把预测不确定性、真实评估成本和探索收益联合建模。

#### 可能的创新点

设计风险约束的 acquisition score，让 LLM 选择“可能更优且值得真实综合”的配置。

#### 实验框架

```text
配置 → 多模态预测均值/方差 → LLM 生成候选 → 选择高价值候选真实综合
→ 校准误差与 Pareto 集 → 继续搜索
```

#### 可行性

可复用本文数据生成器和多目标 benchmark，增加深度集成或保序校准。

#### 主要风险

不确定性估计可能不可靠，且多目标尺度会影响结论。

### 11.3 跨 HLS 工具的配置语义迁移

#### 研究问题

在 Vitis-HLS、Bambu 等工具之间，LLM 能否迁移 pragma 影响知识而不把工具特定规则误当成通用规律？

#### 与原论文的区别

原文主要使用 Vitis-HLS 2022.1/Vivado 2022.1；该方向把工具差异作为显式变量。

#### 可能的创新点

构建工具条件化的 pragma ontology 和版本感知提示，比较零样本、少样本和校准迁移。

#### 实验框架

```text
工具 A 历史配置/QoR → 工具条件化知识 → LLM 生成工具 B 候选
→ B 实际综合 → 迁移误差与 Pareto 质量反馈
```

#### 可行性

需要至少两套可运行 HLS 工具和相同 kernel 的统一输入转换。

#### 主要风险

pragma 语义和 QoR 口径可能并不等价，公平比较成本很高。

## 12. 与其他已读文献的关系

本批次只完成这一篇论文，因此没有可依据本批次正文建立的横向比较。按照正式库去重核对，已有的 DeCOS、Compiler-R1、Reasoning Compiler、AUTOSPARSE、MailoHLS、FlowCompile、PF-LLM 和 Agentic Auto-Scheduling 等条目与本论文处于相邻的 SELECTOR/S2/S3/S4 研究范围，但本笔记不复述它们的实验事实。MPM-LLM4DSE 的区别在于：它将多模态 HLS QoR 预测器与 LLM pragma 配置搜索绑定；其直接 baseline 主要是 HLS DSE 的传统/学习型搜索器和 LLMMH。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | HLS pragma 配置的多目标设计空间探索 |
| 核心问题 | CDFG 语义不足、QoR 预测昂贵、pragma 搜索空间大 |
| 输入 | C/C++ 行为描述、pragma 配置、CDFG、硬件目标与提示知识 |
| 输出 | HLS pragma/design configuration；由 HLS 流程执行/评估 |
| 核心方法 | MPM 多模态 QoR 预测 + PEODSE 提示驱动 LLM4DSE |
| 使用的模型 | CodeBERT-c、ECoGNN、GPT-4o、Qwen3-235B-A22B-Thinking-2507 |
| 使用的编译器工具 | LLVM、ProGraML、Vitis-HLS 2022.1、Vivado 2022.1 |
| 是否使用强化学习 | 本文 LLM4DSE 不使用 RL；RL 仅作为相关工作/baseline 背景 |
| 是否使用形式化验证 | 否；HLS/预测流程不等于形式化证明 |
| 数据集规模 | 15 个训练 benchmark，4,353 个 graph-text 样本；另有 6 个未见 DSE kernel |
| 主要指标 | QoR RMSE、ADRS、Runtime、Latency、LUT/DSP/FF/BRAM |
| 最重要实验结果 | MPM 总 RMSE 0.3898；LLM4DSE(Qwen3) 平均 ADRS 0.0305，相对 LLMMH 改进 21.39% |
| 核心创新 | 图文多模态 QoR 预测与带 pragma 影响知识的 LLM 配置搜索 |
| 主要局限 | HLS/工具范围窄，API 成本与延迟高，跨平台泛化未充分证明 |
| 与 RISC-V 研究的相关性 | 中：配置选择范式可迁移，但论文没有 RISC-V 实验，直接换平台不足以形成创新 |
| 最适合作为 | SELECTOR/S2 方法参考、HLS DSE baseline、QoR 代理与提示工程模块 |

这篇论文最值得学习的是把 LLM 的语义推理限制在结构化配置输出，并让编译器/HLS 工具负责实际执行与评价；最主要的局限是实验依赖特定 HLS 工具、benchmark 和 API，且预测代理不等于真实硬件闭环。如果用于后续研究，合理方式是把它作为配置搜索与代理建模 baseline，再加入 RISC-V 真实计数器、跨工具迁移或不确定性门控，而不是简单地把 HLS 平台替换成 RISC-V。
