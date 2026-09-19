# iDSE 文献阅读总结

论文题目：**iDSE: Navigating Design Space Exploration in High-Level Synthesis Using LLMs**  
作者：Runkai Li，Jia Xiong，Xi Wang  
发表时间：2025；本文使用的 PDF 为 arXiv:2505.22086v2（2025-05-31），正文标注为 preprint / under review。  
发表平台：arXiv / CoRR 预印本  
论文链接或编号：DOI 10.48550/arXiv.2505.22086；arXiv:2505.22086  
关键词：高层次综合（High-Level Synthesis, HLS）、设计空间探索（Design Space Exploration, DSE）、大语言模型（LLM）、优化指令、Pareto 前沿、QoR

> 本笔记依据 staging 中的 iDSE PDF 正文生成。论文事实、阅读分析和后续建议分开记录；本笔记不代表正式入库。

## 1. 研究背景

高层次综合把 C/C++/SystemC 等行为级描述转换为面向 FPGA 的硬件实现，并允许设计者通过 `PIPELINE`、`UNROLL`、`ARRAY_PARTITION` 等优化指令定制微架构。指令的组合与参数会形成巨大的离散设计空间，设计者需要反复调用 HLS 工具、查看 latency 和资源利用率，再手工调整配置。

论文指出，传统启发式 DSE 可以减少穷举，但仍受初始采样质量、搜索预算和 HLS 评估成本限制；基于 ML/DL 的 QoR 预测方法则存在训练成本、跨工作负载泛化和跨 HLS 环境迁移问题。作者希望用 LLM 的结构分析和语言推理能力，改善多目标 HLS DSE 的初始化与局部搜索。

## 2. 论文要解决的问题

### 2.1 组合指令空间过大

即使是小型循环和数组，也可能产生数百万个有效配置。论文给出的向量 Hadamard 示例约有 1.58M 个有效设计；若每个配置的 HLS 评估约需一分钟，穷举会达到约三年。

### 2.2 初始采样和搜索方向不佳

传统进化算法可能从低质量或重复的配置开始，导致有限预算下无法覆盖有代表性的 Pareto 前沿。模型预测方法能够降低部分评估成本，但不一定能稳定理解指令、循环结构和资源约束之间的关系。

### 2.3 需要同时平衡 latency 与资源利用率

只追求 latency 可能造成 DSP、LUT、FF 或 BRAM 超用；只追求资源又可能牺牲性能。论文主要研究：如何让 LLM 在剪枝后的 HLS directive 配置空间中生成高质量初始样本，并利用 QoR 反馈进行多路径 refinement，以较少 HLS 评估逼近多目标 Pareto 前沿。

## 3. 核心方法概述

iDSE 是一个由 LLM 导航、由 HLS 工具实际评估的多目标 DSE 框架。LLM 不直接生成最终 RTL 或硬件实现，而是输出结构化的 directive configuration；脚本把配置插入原始 HLS 设计，Vitis HLS 完成综合/仿真并返回 QoR。

```text
原始 HLS C/C++ 设计
        ↓
结构特征提取与 directive configuration 构造
        ↓
Feature-Driven Pruning：删除非法或明显不合理配置
        ↓
LLM Seed Directive Generation：生成多目标初始配置
        ↓
配置注入原始 HLS 设计
        ↓
Vitis HLS 综合/仿真，得到 latency 与资源利用率
        ↓
Pareto 排序、轨迹反思与瓶颈分析
        ↓
LLM 进行 convergent tuning 与 divergent search
        ↓
重复评估并输出非支配 Pareto 设计集合
```

三个阶段是：

1. **Preprocessing**：从设计结构提取循环、数组和指令参数；剪去过度并行、冗余或可能导致综合失败的配置。
2. **Warm-Start**：LLM 根据设计结构、可行配置和目标偏好生成结构化 seed configurations，并生成 Tcl 配置脚本。
3. **Adaptive Optimization**：根据 Pareto rank、crowding distance、QoR 和 bottleneck 反馈，执行面向已有方案的 convergent search，以及用于跳出局部区域的 divergent search。

依据 Taxonomy v2，本文拟归类为 `SELECTOR / S2_Schedule_Config_Autotuning`：模型输出的是供现有 HLS 工具执行的配置与调度动作。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用提示词推理、结构化配置生成和 HLS 工具反馈。

### 4.1 Preprocessing

输入是 HLS 设计代码。系统提取循环层次、循环 trip count、数组维度等特征，并把 `PIPELINE`、`UNROLL`、`ARRAY_PARTITION` 编码为离散/整数特征。LLM 根据结构化 JSON 和规则删除非法配置；另外使用脚本做二次检查，防止 LLM 单独剪枝遗漏指令之间的依赖。

### 4.2 Warm-Start

LLM 被要求针对三类目标生成唯一配置组合：偏性能、偏资源利用率、性能-资源折中。输出被转换为 Tcl 指令并注入 HLS 工程。论文还用 12 个初始样本作为统一实验设置，以平衡覆盖和上下文长度。

### 4.3 Adaptive Optimization

每轮先用 HLS 评估候选并记录 QoR，再以非支配排序和 crowding distance 选择父代。LLM 反思优化轨迹并进行瓶颈分析：检查循环依赖、内存访问/分区匹配和资源饱和。随后生成 oriented/convergent configurations 与 non-oriented/divergent configurations，继续交给 HLS 评估。

### 4.4 模型与提示词

论文实验比较了 DeepSeek-R1、Claude 3.7 Sonnet、Gemini 2.5 Pro、GPT-4.1、Grok-3 和 o1-preview 等 LLM。论文没有把这些模型作为训练后统一模型，而是将其作为提示词驱动的配置生成器；不同模型在不同 benchmark 上没有稳定的单一赢家。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有报告 SFT、PPO 或 GRPO 训练损失。核心是多目标搜索和 QoR 评价。

### 5.1 多目标优化目标

```text
λ(φ*) = arg min over φ∈Φ [ Lat(H, λ(φ)), Util(H, λ(φ)) ]
```

其中 `φ` 是 directive configuration，`λ(φ)` 是注入配置后的 HLS 设计，`H` 是 vendor HLS tool，`Lat` 是综合时序分析得到的 latency，`Util` 是资源利用率。

### 5.2 资源利用率

论文将资源利用率定义为加权和：

```text
Util = 0.30·LUT + 0.25·FF + 0.30·DSP + 0.05·BRAM
```

各资源项是归一化使用比例。权重提高了 LUT/DSP 的影响，并降低了 BRAM 的权重；这是一种论文实验中的目标定义，不应直接解释为所有 FPGA 的通用成本模型。

### 5.3 ADRS

ADRS 衡量探索 Pareto 前沿 `PE` 与参考前沿 `PR` 的平均距离：

```text
ADRS(PE, PR) = (1 / |PR|) · Σ over γ∈PR min over ω∈PE d(γ, ω)
```

ADRS 越小，表示探索前沿越接近参考前沿。论文说明 latency 来自综合 timing analysis，而不是布局布线后的真实端到端硬件运行时间。

## 6. 实验设置

### 6.1 数据集来源

实验使用 12 个 HLS benchmark，来自 PolyBench、CHStone 和 MachSuite。论文列出 `atax`、`bicg`、`gemm`、`gesummv`、`mvt`、`md-knn`、`spmv`、`stencil2d`、`stencil3d`、`viterbi`、`sha`、`autocorr` 等设计，并统计每个设计的循环数、数组数和 directive space 大小。

这些不是 LLM 预训练数据集，也没有单独的训练集/验证集/测试集。参考 Pareto 前沿由随机采样和广度优先搜索等方式构造；具体参考前沿的完整覆盖并不等于真实全局 Pareto 前沿。

### 6.2 模型与工具

| 项目 | 论文信息 |
|---|---|
| LLM | DeepSeek-R1、Claude 3.7 Sonnet、Gemini 2.5 Pro、GPT-4.1、Grok-3、o1-preview 等 |
| HLS 工具 | Vitis HLS 2022.1 |
| 目标平台 | Xilinx ZCU106 MPSoC |
| 主机 | Intel Xeon Platinum 8378A，Ubuntu 20.04.6 |
| 指令 | PIPELINE、UNROLL、ARRAY_PARTITION |
| 评估输出 | synthesis latency、LUT、FF、DSP、BRAM 及加权 utilization |

### 6.3 对比方法

主要 baseline 包括 NSGA-II、ACO、MOEA/D、Lattice 和 HGBO-DSE；同时比较 Random Sampling、U-shaped Beta Sampling、Latin Hypercube Sampling 与 iDSE 的 Warm-Start。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| ADRS | 探索 Pareto 前沿到参考前沿的平均距离 | 越小越好 |
| Latency | Vitis HLS 综合 timing analysis 得到的设计 latency | 越小越好 |
| Utilization | 加权 LUT/FF/DSP/BRAM 资源使用比例 | 越小越好 |
| Search budget | 达到目标 ADRS 所需的探索设计数 | 越小越好 |
| Speedup | 相对 baseline 达到相同目标 ADRS 的预算比例 | 越大越好 |

## 7. 实验结果与结论

### 7.1 主要结果

在 12 个 benchmark 上，论文报告 iDSE 相对启发式 DSE 方法的 Pareto 前沿逼近质量提升范围为 5.1×–16.6×；摘要还报告 iDSE 在仅使用 NSGA-II 探索设计数 4.6% 的预算时达到相近表现。表 1 的几何平均改进相对 NSGA-II 为 16.5955×，相对 ACO、MOEA/D、Lattice 和 HGBO-DSE 分别为 12.7×、9.9×、15.3× 和 5.1×左右。

这些数字针对 ADRS/搜索效率，不是某个固定 FPGA 程序的端到端运行加速；HLS latency 由综合时序分析得到。

### 7.2 与传统 DSE 比较

表 2 给出的目标 ADRS 搜索预算为：NSGA-II 在 PolyBench、MachSuite、CHStone 上分别需要 108、108、108 个设计；iDSE 分别需要 4、5、7 个设计。论文据此报告 iDSE 相对传统元启发式方法的几何平均搜索效率提升约 11.0×，相对现有 DSE 方法约 4.4×。

### 7.3 Warm-Start

Warm-Start 与随机、U-shaped Beta、Latin Hypercube 初始采样比较时，论文报告其作为后续搜索初始化能够改善 ADRS。表 2 的 iDSE 预算为 4、5、7，整体几何平均 speedup 为 25.07×（相对表中的 NSGA-II 参考）。Warm-Start 结果来自 5 轮 DeepSeek-R1 调用的平均值。

### 7.4 消融实验

消融分为 Baseline、S1、S2、S3：

- Baseline：只让 LLM 根据 HLS 设计和 QoR 生成不同配置。
- S1：加入剪枝后的设计空间。
- S2：加入优化轨迹反思。
- S3：再加入瓶颈分析和设计重构操作。

论文报告：未加剪枝时低质量/非法设计约占 12.4%；S1 使非法设计减少 89.9%；S2 带来 20.5% 的有效性提升；S3 在此基础上额外提升 45.0%。这些比例是论文消融设置下的相对 ADRS 改进，不应外推为通用提升。

### 7.5 失败与模型差异

论文明确承认不同通用 LLM 在不同 benchmark 上表现不同，没有一个模型始终最好；有限上下文和过长输出也会导致质量下降。LLM 生成的初始配置仍有少量低质量项，会浪费 vendor HLS 工具的评估时间。

## 8. 主要创新点

### 8.1 创新点一：Feature-Driven Pruning

论文把 HLS 设计结构映射为结构化 directive configuration，并利用循环嵌套、trip count、数组维度和并行度规则压缩可行空间。价值在于让 LLM 面向较小且具有语义约束的配置空间工作，而不是直接面对所有 pragma 组合。

### 8.2 创新点二：LLM-Guided Seed Directive Generation

论文将 LLM 用作多目标 DSE 的 warm-start 生成器，分别面向性能、资源和折中目标生成种子配置。它输出的是可由 HLS 执行的 directive configuration，而不是最终硬件代码，因此具有明确的 Selector 特征。

### 8.3 创新点三：QoR-Aware Adaptive Optimization

论文把 Pareto 选择、轨迹反思、瓶颈分析、convergent search 和 divergent search 放进一个闭环，使 LLM 根据真实 HLS QoR 进行配置 refinement。其贡献不是单纯“使用 LLM”，而是将 LLM 的结构化配置生成与传统进化搜索和 vendor HLS 评估组合起来。

## 9. 局限性

### 9.1 论文明确承认的局限

- 有限搜索预算下，LLM 无法识别全部 ground-truth Pareto-optimal designs。
- 搜索规模扩大后，LLM 的长上下文限制可能降低有效性。
- 初始采样仍会包含少量低质量 directive configurations，造成 HLS 评估浪费。
- 不同通用 LLM 在不同 benchmark 上表现差异明显，没有稳定的最佳模型。
- 论文建议用更大初始样本、传统 DSE 和面向 HLS 的专门训练数据改善可扩展性。

### 9.2 阅读后的潜在局限

- 评价依赖单一 vendor flow：Vitis HLS 2022.1 和 Xilinx ZCU106；跨 Intel FPGA、ASIC HLS、MLIR-based HLS 或 RISC-V accelerator 的迁移尚未由本文证明。
- latency 是综合 timing analysis，而不是布局布线或真实板卡执行时间；资源利用率还是带权指标，权重改变可能改变 Pareto 前沿。
- 参考 Pareto 前沿由有限采样/搜索构造，ADRS 并不自动等于相对真实全局最优的距离。
- 论文主要处理三个 directive 类型；更复杂的 dataflow、memory banking、跨函数约束和后端时序约束可能超出当前 prompt 与剪枝规则。

## 10. 阅读后的研究方向反思

论文最值得借鉴的是“结构化 action space + 编译器真实反馈 + 多目标 Pareto 记忆”的组合，而不是直接复制某个 LLM 或 HLS 平台。对 LLVM/RISC-V 方向，可以把 directive configuration 的思想迁移为 pass/flag/schedule configuration，但仅替换为 RISC-V 并不足以形成创新。

更有价值的研究问题是：如何把 RISC-V 微架构、RVV 向量长度、缓存/带宽约束和 LLVM pass 依赖编码进可审计的 action space；如何使用真实硬件计数器而不是综合 proxy 作为反馈；以及如何保证 LLM 不生成违反 pass 前置条件或语义安全约束的配置。本文适合作为 Selector baseline 和 action-space 设计参考，不适合作为 RISC-V 端到端优化方案直接照搬。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V/RVV 的安全 Pass Configuration Selector

#### 研究问题

LLM 能否在 LLVM pass 依赖、RVV 向量化约束和目标微架构反馈共同限定的空间中选择有效 pass/flag 序列？

#### 与原论文的区别

从 HLS pragma 选择转向 LLVM IR/pass/config 选择，并加入 pass legality、IR 验证和真实硬件反馈。

#### 可能的创新点

依赖图约束解码、RVV-aware action schema、编译失败与性能退化的可审计回退机制。

#### 实验框架

```text
LLVM IR + RVV 特征 → 合法 pass/config 空间剪枝 → LLM 选择候选
→ clang/opt 编译 → QEMU/真实 RISC-V 运行 → 性能计数器反馈
→ Pareto 记忆与安全回退
```

#### 可行性

需要 LLVM opt/clang、RISC-V 交叉工具链、QEMU 或 RVV 板卡、SPEC/PolyBench/TSVC 等 benchmark。

#### 主要风险

真实硬件预算、性能噪声、pass 依赖遗漏和 LLM 产生不可执行序列。

### 11.2 结构化配置与硬件反馈的双层 Selector

#### 研究问题

能否将静态 IR/pass 选择与运行时 cache、branch、vector utilization 反馈分成两个相互约束的选择层？

#### 与原论文的区别

原论文主要使用 HLS synthesis QoR；该方向将静态编译配置与真实运行时反馈解耦并闭环。

#### 可能的创新点

跨层 credit assignment、硬件反馈置信度、不可重复测量下的鲁棒 Pareto 搜索。

#### 实验框架

```text
静态 IR 分析 → 初始 pass/config 候选 → 编译
→ 真实硬件指标 → 第二层局部参数选择 → 语义/可执行性检查
```

#### 可行性

可先用 QEMU 或 perf 计数器，再迁移到 RISC-V FPGA/开发板。

#### 主要风险

测量噪声和硬件差异可能掩盖优化收益，且搜索成本高。

## 12. 与其他已读文献的关系

本批次已确认的相关材料是 RALAD。RALAD 让 LLM 直接生成带 HLS pragma 的优化后 C 代码，属于 Translator；iDSE 则输出 directive configuration，由 HLS 工具执行，属于 Selector。两者共享 HLS 和 pragma 语境，但最终 LM 输出角色不同，不能合并为同一条候选。

iDSE 还与仓库已有的 HLS Selector 类工作存在近邻关系：共同点是输出供 HLS 执行的 directive/config；区别在于 iDSE 强调 feature-driven pruning、warm-start 和 Pareto-aware adaptive search。当前 staging 只记录本次阅读结果，不做正式索引同步。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 用 LLM 导航 HLS 多目标设计空间探索 |
| 核心问题 | directive 组合爆炸、HLS 评估昂贵、初始采样质量不足 |
| 输入 | HLS C/C++ 设计、结构特征、可行 directive 配置和 QoR |
| 输出 | HLS directive configurations 与 Pareto-optimal design 集合 |
| 核心方法 | Feature-Driven Pruning、LLM Seed Directive Generation、QoR-Aware Adaptive Optimization |
| 使用的模型 | 多个通用 LLM，包括 DeepSeek-R1、Claude 3.7 Sonnet、Gemini 2.5 Pro、GPT-4.1 等 |
| 使用的编译器工具 | Vitis HLS 2022.1 |
| 是否使用强化学习 | 否；使用提示词推理、进化式搜索和 HLS 反馈 |
| 是否使用形式化验证 | 论文未报告形式化等价验证；主要依赖 HLS 综合/仿真和 QoR 检查 |
| 数据集规模 | 12 个 PolyBench、CHStone、MachSuite HLS benchmark；无独立训练集 |
| 主要指标 | ADRS、latency、LUT/FF/DSP/BRAM utilization、search budget |
| 最重要实验结果 | 相对启发式 DSE 的 Pareto 逼近提升约 5.1×–16.6×；iDSE 在较低搜索预算下达到强 Pareto 前沿 |
| 核心创新 | 把 LLM 约束为结构化 directive/config Selector，并接入 Pareto-aware HLS 反馈闭环 |
| 主要局限 | 有限预算无法覆盖全部 Pareto 最优；长上下文、模型差异和单一 Vitis/Xilinx 流限制泛化 |
| 与 RISC-V 研究的相关性 | 中；action-space、Pareto 反馈和安全回退可迁移，但本文没有 RISC-V 实验 |
| 最适合作为 | Selector baseline、HLS action-space 设计参考、RISC-V 编译调优方法参考 |

> 这篇论文最值得学习的是把 LLM 的输出限制为可执行、可评估的结构化配置，并用 Pareto/QoR 反馈驱动搜索；最主要的局限是硬件平台和 directive 类型较窄、评价主要依赖综合 proxy；如果用于后续研究，最合理的使用方式是作为 Selector 与搜索协议 baseline，而不是简单地把 HLS 平台替换成 RISC-V。
