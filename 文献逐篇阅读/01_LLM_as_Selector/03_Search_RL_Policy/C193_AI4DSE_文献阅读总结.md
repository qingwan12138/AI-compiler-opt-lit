# AI4DSE 文献阅读总结

论文题目：**AI4DSE: Leveraging Dynamic Graph Neural Networks and Large Language Models for Optimizing High-Level Synthesis Design Space Exploration**

作者：Lei Xu、Shanshan Wang、Emmanuel Casseau、Chenglong Xiao

发表时间：2026（ACM TODAES online publication 2026-06-14；本次正文 PDF 为同一工作的 arXiv v3，2025-10-15）

发表平台：ACM Transactions on Design Automation of Electronic Systems，31(6)，Article 1--25；作者同时提供 arXiv 版本，标题在 arXiv v3 中仍显示为 Intelligent4DSE: Optimizing High-Level Synthesis Design Space Exploration with Graph Neural Networks and Large Language Models。

论文链接或编号：DOI [10.1145/3805806](https://doi.org/10.1145/3805806)；arXiv:2504.19649；本次使用 [arXiv v3 PDF](https://arxiv.org/pdf/2504.19649) 作为 ACM 正式版 PDF 的可读同工作版本。

关键词：HLS、DSE、ECoGNN、QoR prediction、LLM-enhanced meta-heuristic、Pareto frontier、pragma configuration

> 本笔记以 arXiv v3 正文为事实依据，并记录 ACM DOI 版本关系；未把 arXiv 标题误当成另一篇论文。

## 1. 研究背景

AI4DSE 研究 HLS 设计空间探索。HLS 允许用 C/C++ 和 pragma/directive 生成大量硬件实现，不同配置在 latency、资源和功耗之间形成权衡。完整综合所有配置代价高，因此常用 QoR 预测模型替代部分综合，再用多目标搜索寻找 Pareto 前沿。

论文认为已有 message-passing GNN 在长距离依赖、过平滑和表达能力方面存在问题；NSGA-II、SA、ACO 等元启发式方法则依赖人工设置 crossover、mutation、temperature、pheromone 等参数，面对百万级设计空间时搜索效率下降。论文提出 ECoGNNs-LLMMHs：用任务自适应 GNN 做 QoR 预测，用 LLM 的 in-context learning 改进元启发式候选生成。

## 2. 论文要解决的问题

### 2.1 QoR 预测

论文希望同时改进 post-HLS 和 post-implementation QoR 的预测，减少传统 GNN 的表达限制。

### 2.2 多目标 DSE

论文希望在有限候选评估和时间预算下，找到更接近参考 Pareto 集的 directive configurations，并降低元启发式参数人工调节的依赖。

> 本文主要研究：如何把动态消息传递的 GNN 评估器与 LLM-enhanced 元启发式搜索结合，用于 HLS pragma 配置的多目标选择。

## 3. 核心方法概述

系统由数据生成、ECoGNN 训练、推理和 LLMMH DSE 四阶段构成。LLM 不直接输出新的 HLS 源码；它根据 prompt 中的任务说明、配置示例和知识生成初始 population、offspring 或邻域解，后续由预测器评估并由元启发式规则选择。因而本论文建议分类为 **SELECTOR / S3_Search_RL_Policy**，可附带 S2 配置搜索标签。

```text
C/C++ + pragma 配置 + HLS/implementation reports
        ↓
LLVM/ProGraML 转为图，训练 ECoGNN QoR predictor
        ↓
LLM 读取任务说明、已有解和候选配置示例
        ↓
LLM-enhanced GA / SA / ACO 生成或更新配置候选
        ↓
ECoGNN 预测 latency/resource，保留 Pareto 候选
        ↓
有限 DSE 迭代，输出近似 Pareto 配置集
```

## 4. 实验框架与训练流程

### 4.1 数据生成

论文从 C/C++ 源码和 pragma 配置生成 LLVM IR/图表示，并结合 HLS report 与 implementation report 形成标签。post-HLS 任务包含 latency、LUT、DSP、FF、BRAM 等；post-implementation 任务还包含 CP、POWER 等指标。

### 4.2 ECoGNN 训练

ECoGNN 使用 action network 和 environment network 动态决定消息传播拓扑。论文用不同 MPNN 组合形成 ECoGNN 变体，并训练预测 QoR。HGBO-DSE 的 10 个应用中，6 个用于训练，剩余应用用于 unseen inference；另一个 GNN-DSE 数据集按 70%/15%/15% 划分训练、测试和验证。

### 4.3 LLMMH 推理

论文不对 LLM 做 SFT、PPO 或 GRPO。LLM 以 in-context learning 方式读取任务描述、solution examples 和已有知识。LLM-enhanced 版本包括 LLMGA、LLMSA、LLMACO，分别对应遗传算法、模拟退火和蚁群优化。LLM 生成/更新的是 pragma configuration candidate，ECoGNN 作为快速 evaluator。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数；LLMMH 是提示驱动的元启发式搜索。

### 5.1 Pareto 条件

文中用资源 `A(d)` 和 latency `L(d)` 定义 Pareto 支配关系：若某配置不劣于另一配置的资源和 latency，且至少一项更优，则后者被支配。

### 5.2 ADRS

论文用 ADRS 衡量近似 Pareto 集 `Ω` 与参考 Pareto 集 `Γ` 的距离：

```text
ADRS(Γ, Ω) = average over λ in Γ of min over μ in Ω of f(λ, μ)
```

其中 `f` 取归一化资源和 latency 相对差异中的最大值。ADRS 越低，近似 Pareto 集越接近参考集。资源函数把 FF、LUT、BRAM、DSP 按目标平台最大资源归一化平均；latency 使用倒数形式。

### 5.3 GNN 预测损失

论文使用 RMSE、MAPE 和 MAE 评价预测误差；具体训练损失/优化器配置若未在当前正文段落明确给出，应以“论文中未明确说明”为准，不将 ADRS 当作 GNN 训练损失。

## 6. 实验设置

### 6.1 数据集来源

- post-HLS 预测使用 GNN-DSE 和 HARP 相关数据。
- post-implementation 预测使用 HGBO-DSE 数据，包含 10 个应用。
- DSE 使用 5 个未参与训练的应用：`atax`（3,354 configurations）、`doitgen`（179）、`gemm-p`（409,905）、`heat-3d`（71,511）、`mvt`（3,059,001）。
- 图数据由 LLVM 和 ProGraML 处理 C/C++ 配置。

### 6.2 模型与工具

- ECoGNN 是任务自适应图神经网络。
- LLM 组合包括 DeepSeek-R1、GPT-4o、GPT-4.1、o3-mini。
- LLMMH 组合为 3 种元启发式 × 4 种 LLM，共 12 个组合。
- 论文使用 HLS/implementation report 作为数据标签，并使用 predictor 替代部分真实综合；正文没有给出一个统一的实际 FPGA 板卡型号作为所有 DSE 运行环境。

### 6.3 对比方法

- SA、NSGA-II、ACO：传统元启发式。
- GNN-DSE：GNN 驱动的 DSE。
- IRONMAN-PRO 的 AC-FT、PG-FT：有限探索预算下的比较方法。
- QoR 预测部分还比较 HGP+SAGE+GF、PNA-R、PowerGear、HARP 等模型。

### 6.4 评价指标

- RMSE：预测值和标签的均方根误差，越低越好。
- MAPE：相对百分比误差，越低越好。
- MAE：平均绝对误差，越低越好。
- ADRS：近似 Pareto 集到参考集的距离，越低越好。
- runtime：DSE 运行时间，越低越好，但必须结合探索质量解释。

## 7. 实验结果与结论

### 7.1 QoR 预测

在 GNN-DSE 相关数据上，ECoGNNs(β,α) 相对 GNN-DSE、HGP+SAGE+GF、IRONMAN-PRO、PNA-R、PowerGear 的 RMSE 降幅分别报告为 47.69%、36.32%、52.51%、26.07%、25.80%。这些是预测误差比较，不是真实硬件加速比。

在 HGBO-DSE unseen 应用上，ECoGNNs(β,α) 的 LUT/FF/CP/POWER 的 MAPE 为 6.82%、5.98%、7.54%、10.01%，DSP/BRAM 的 MAE 为 0.5339 和 0.1035。LUT 上 PNA-R 的 6.40% 更低，论文没有把 ECoGNN 宣称为每个指标都最优。

### 7.2 DSE 结果

5 个 unseen 应用上，LLMACO(DeepSeek-R1) 的平均 ADRS 为 0.0339；SA、NSGA-II、ACO 分别为 0.3182、0.2405、0.2842。相对这三个传统方法，论文报告 LLMACO(DeepSeek-R1) 的平均 ADRS 改善分别为 89.34%、85.90%、88.07%。在 `mvt` 这个约 305.9 万配置的空间上，LLMACO(DeepSeek-R1) ADRS 为 0.1516，仍高于部分小空间任务的结果。

### 7.3 与 GNN-DSE/IRONMAN-PRO

在相同或受限预算比较中，LLMACO(DeepSeek-R1) 平均 ADRS 为 0.0339，GNN-DSE 为 0.1065，IRONMAN-PRO 的 AC-FT 和 PG-FT 为 0.0918 和 0.0847；论文报告相对 GNN-DSE、AC-FT、PG-FT 分别改善 68.17%、63.07%、59.98%。

### 7.4 消融和代价

论文比较了 3 种元启发式和 4 个 LLM。小空间任务上部分 LLMMH 结果不如传统元启发式，论文将其与 LLM hallucination 联系起来。LLMMH 运行时间较高，主要来自每轮 API 调用、LLM 推理和网络延迟；LLMACO(DeepSeek-R1) 在 5 个任务上的总 runtime 为 839 秒。

## 8. 主要创新点

### 8.1 创新点一：任务自适应 ECoGNN

ECoGNN 动态选择消息传播拓扑，试图缓解固定 MPNN 的过平滑和长距离依赖问题，并同时覆盖 post-HLS 与 post-implementation QoR 预测。

### 8.2 创新点二：LLM-enhanced meta-heuristic

论文不是让 LLM 直接输出硬件源代码，而是让 LLM 参与生成/更新元启发式中的配置候选。配置仍由预测器和 Pareto 选择规则评估。这是本论文进入 Selector 队列的主要依据。

### 8.3 创新点三：预测器与搜索器结合

ECoGNN 负责廉价 QoR 估计，LLMMH 负责候选搜索，避免每个候选都运行完整 EDA 流程。该组合对大配置空间尤其重要，但其收益依赖预测器精度和 prompt 质量。

## 9. 局限性

### 论文明确承认的局限

- LLMMH 运行时间高，LLM API 延迟和网络延迟明显。
- LLM 选择质量受 prompt 配置影响。
- 小设计空间上 LLM 可能因 hallucination 产生不如传统算法的候选。
- 论文未来工作提出更系统地选择 LLM、减少 runtime、改进 prompt。

### 阅读后的潜在局限

- DSE 主要由 ECoGNN 预测器评估，预测误差可能造成搜索器偏向静态 QoR，而不是实际板级运行时间。
- ACM 正式版与 arXiv v3 的标题不同，交付时需要保留 DOI/arXiv 版本关系，不能按两篇论文计数。
- LLM 生成候选的有效性、格式约束和非法配置比例需要正文/代码复现进一步核验。
- 论文同时包含 GNN 预测和 Selector 搜索两条贡献线，正式 taxonomy 需要以最终 LM 输出配置这一角色为主，而不是把整个框架归为 Generator 或 Supporting。

## 10. 阅读后的研究方向反思

最值得借鉴的是把“预测器”和“候选选择器”明确拆分，并让 LM 只负责搜索策略/配置候选。它与 LLM-DSE 可形成互补：前者使用代理 QoR 预测，后者强调真实 HLS 闭环。仅把 GNN 换成 RISC-V 性能模型不足以形成创新；需要处理预测误差、跨硬件迁移和真实反馈校准。该论文适合作为 Selector 的代理模型增强 baseline，也可作为后续研究的候选生成模块。

## 11. 可进一步尝试的研究方向

### 11.1 预测器不确定性感知的配置选择

#### 研究问题

当 QoR predictor 对未见 kernel 不可靠时，LM 如何决定是否调用真实编译器。

#### 与原论文的区别

增加预测置信度和真实测量触发条件，而不是只最小化静态 ADRS。

#### 可能的创新点

把不确定性、失败边界和编译成本联合为选择策略。

#### 实验框架

```text
候选配置 → predictor 均值/不确定性 → LM 选择真实测量或代理评估
        → 真实反馈校准 predictor → 更新 Pareto 集
```

#### 可行性

需要可访问的编译器和可重复的 QoR 数据。

#### 主要风险

不确定性估计本身可能失真。

### 11.2 跨 ISA 配置语义迁移

#### 研究问题

如何把 HLS/FPGA 上的配置效应迁移到 GPU 或 RISC-V/RVV 的 tiling、vectorization 和 backend 参数。

#### 与原论文的区别

学习跨 ISA 的配置语义，而不是简单替换硬件平台名称。

#### 可能的创新点

建立配置效应图和跨后端对齐损失。

#### 实验框架

```text
源后端配置轨迹 → 效应抽取 → 目标 ISA 候选配置
        → 编译/仿真/真实测量 → 迁移与失败分析
```

#### 可行性

需要 LLVM/RISC-V 或 GPU 编译链和统一 kernel 集合。

#### 主要风险

不同后端配置并不存在一一对应关系。

## 12. 与其他已读文献的关系

- 与 LLM-DSE：均为 Selector；AI4DSE 使用 ECoGNN 代理评估与 LLM-enhanced GA/SA/ACO，LLM-DSE 使用 Router、Specialists、Arbitrator、Critic 和真实 HLS 工具反馈。
- 与 RALAD：AI4DSE 输出/更新配置候选，RALAD 直接生成带 pragma 的 HLS 源码；前者是 Selector，后者是 Translator。
- 三者可组成一个清晰的对照链：AI4DSE（代理模型 Selector）—LLM-DSE（真实工具闭环 Selector）—RALAD（代码生成 Translator）。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | HLS 多目标 DSE 与 QoR 预测 |
| 核心问题 | MPNN 表达受限、元启发式参数依赖人工调节 |
| 输入 | pragma 配置、LLVM/图表示、QoR 数据 |
| 输出 | Pareto 配置候选/近似 Pareto 集 |
| 核心方法 | ECoGNN predictor + LLM-enhanced GA/SA/ACO |
| 使用的模型 | ECoGNN；DeepSeek-R1、GPT-4o、GPT-4.1、o3-mini |
| 使用的编译器工具 | LLVM/ProGraML；HLS 与 implementation reports |
| 是否使用强化学习 | 否；使用元启发式和 in-context learning |
| 是否使用形式化验证 | 否 |
| 数据集规模 | 5 个 DSE unseen 应用，最大约 3,059,001 配置 |
| 主要指标 | RMSE、MAPE、MAE、ADRS、runtime |
| 最重要实验结果 | LLMACO(DeepSeek-R1) 平均 ADRS 0.0339，相对 NSGA-II 改善 85.90% |
| 核心创新 | 动态 QoR predictor 与 LLM 搜索候选结合 |
| 主要局限 | API/推理成本高，预测器误差可能影响搜索 |
| 与 RISC-V 研究的相关性 | 中；可借鉴代理评估和配置选择，但需重新校准后端效应 |
| 最适合作为 | Selector baseline、代理模型搜索模块 |

> 这篇论文最值得学习的是将 LLM 限定为配置候选/搜索策略角色，并用独立 QoR predictor 扩展搜索；最主要的局限是搜索质量依赖 predictor 和 prompt，且 API 成本高；如果用于后续研究，最合理的使用方式是作为代理模型 Selector baseline，而不是把 GNN 或 LLM 本身当作新的编译器后端。
