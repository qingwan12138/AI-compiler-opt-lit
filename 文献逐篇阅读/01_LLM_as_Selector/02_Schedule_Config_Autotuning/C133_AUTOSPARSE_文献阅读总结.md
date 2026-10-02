# AUTOSPARSE 文献阅读总结

论文题目：**LLM-Guided Autoscheduling for Large-Scale Sparse Machine Learning**

作者：Rubens Lacouture、Genghan Zhang、Konstantin Hossfeld、Tian Zhao、Kunle Olukotun

发表时间：2025

发表平台：ML For Systems Workshop at Neural Information Processing Systems (NeurIPS 2025)，属于 NeurIPS 2025 相关 workshop，不是 NeurIPS 主会论文。

论文链接或编号：[OpenReview 官方条目](https://openreview.net/forum?id=7H9qWe8lLO)；[ML for Systems Workshop 官方 PDF](https://mlforsystems.org/assets/papers/neurips2025/paper40.pdf)。论文正文未给出 DOI 或 arXiv ID。代码仓库：未在论文或作者官方页面确认，待核验；名称相近的 AutoSparse 仓库对应另一篇 ICCD 论文，不作为本篇代码。
元数据核验来源：[论文/来源](https://openreview.net/forum?id=7H9qWe8lLO)；[论文/来源](https://mlforsystems.org/assets/papers/neurips2025/paper40.pdf)
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

关键词：LLM-guided autoscheduling、稀疏机器学习、FuseFlow、fusion、dataflow order、MLIR、operational intensity

> 本文档用于 staging 阶段的文献核验与阅读交付。论文事实依据本地 PDF 正文；阅读分析和后续建议单独标注。

## 1. 研究背景

本文研究稀疏机器学习工作负载的编译器自动调度。稀疏模型具有不规则内存访问和很大的调度空间，性能依赖于算子融合粒度、融合区域内的循环/数据流顺序、分块以及并行化方式。人工为完整模型探索这些组合不可扩展；稠密计算中的 Halide、TVM 等自动调度器也不能直接解决稀疏场景中特有的组合约束。

论文把 LLM 引入调度候选提出环节，而不是让 LLM 直接生成最终代码。FuseFlow 编译器负责枚举合法 dataflow order、验证编译器不变量并提供 FLOPs/byte 代价信号；LLM 负责根据计算图、稀疏性与硬件提示提出 fusion group 和 order。这样做的动机是利用 LLM 对离散、层次化设计空间的结构化推理，同时保留编译器的合法性边界。

根据论文第 1 节，本文关注的是“怎样在稀疏模型的融合与数据流顺序空间中，以较少候选评估找到接近专家调度的方案”，而不是训练一个新的通用代码生成模型。

## 2. 论文要解决的问题

### 2.1 稀疏调度空间过大

需要同时决定哪些 producer/consumer 算子融合，以及每个融合区域采用哪一种合法 dataflow order。稀疏存储格式和交集率会改变内存流量、重计算和数据复用关系，使目标函数出现离散阈值和不平滑变化。

### 2.2 盲目完全融合可能恶化性能

论文指出，全融合并不总是好选择。过度融合可能增加内存访问或重计算，因此需要 workload-adaptive 的融合粒度。

### 2.3 现有自动搜索的适配困难

精确求解器需要脆弱的约束编码，规模变大后难以扩展；贝叶斯优化通常假设目标较平滑且维度较低，而本文的 fusion/order 空间是离散、层次化并受稀疏性影响的。论文因此研究 LLM 提案、编译器合法性检查和轻量代价剪枝的组合。

> 本文主要研究：如何让 LLM 在 FuseFlow 提供的合法稀疏调度空间内提出结构化 fusion+order 方案，并用编译器代价信号筛选出接近专家方案的配置。

## 3. 核心方法概述

AUTOSPARSE 是建立在 FuseFlow 之上的 LLM 引导自动调度器。FuseFlow 是在 MLIR 中实现的稀疏 ML 编译器，能够处理模型图、融合表达式、合法数据流顺序和 SAM 风格数据流图。LLM 的最终输出是候选调度配置，因此本文按 Taxonomy v2 应归入 `SELECTOR / S2_Schedule_Config_Autotuning`。

整体数据流如下：

```text
模型计算图、张量形状、稀疏统计、producer-consumer 关系、硬件提示
        ↓
FuseFlow 枚举可用 fusion set 与每个融合区域的合法 dataflow order
        ↓
LLM 识别瓶颈并提出结构化 fusion groups + dataflow orders
        ↓
编译器验证名称、形状和 order 合法性
        ↓
FLOPs/bytes 启发式估计 operational intensity 与内存访问
        ↓
按 roofline 风格分数排序、剪枝，必要时要求替代候选
        ↓
选择最佳调度；再交给代码生成和 Comal/FPGA 后端
```

LLM 不选择 tiling 和 parallelization，这两项在论文实验中只作为上下文。论文采用半结构化 JSON-like 输出：

```text
Plan := {
  "rank": int,
  "score": number,
  "estimated_OI": number,
  "fusion_groups": [
    {"name": string, "ops": [string], "dataflow_order": [string]}
  ]
}
```

当多个方案接近时，系统向用户展示 top-k 计划、LLM 理由和启发式估计；实践者仍可依据特定数据集的稀疏性覆盖选择。由此，LLM 是 schedule/config 选择器，编译器和后端执行变换。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用 LLM 推理驱动的自动调度执行流程。

### 4.1 状态构造

论文第 3.2 节把算子类型、张量形状、稀疏统计、producer-consumer 关系和粗粒度 roofline 瓶颈分类序列化为 prompt state。同时加入允许的 fusion sets 以及编译器枚举的合法执行顺序。硬件提示也作为 LLM 输入。

### 4.2 候选提案与验证

每轮中，LLM 识别瓶颈、选择融合粒度，并从编译器给出的集合中选择合法 dataflow order。系统验证候选的名称、形状和每个 order 是否满足编译器合法性，再使用 FLOPs/bytes 启发式打分。不合格或低分候选被剪枝；若没有候选通过阈值，则要求 LLM 提供替代方案。

### 4.3 排序、人工覆盖与后端执行

候选按 roofline 风格分数排序，保留高质量方案；近似平局时展示 top-k 供人工选择。选定结构后，可由离线 BO 或小型求解器调节数值型 tile/parallel 因子，但该步骤是论文讨论的后续可能性，不是本文 LLM 已选择的变量。选定的 schedule 最后生成 SAM 风格数据流图，并在 Comal simulator 或 FPGA backend 上执行。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习，因此不存在强化学习奖励函数，也没有报告 SFT、PPO、GRPO 或其他训练损失。

论文的关键代价信号是 operational intensity：

```text
Operational Intensity (OI) = FLOPs / Bytes
```

其中 `FLOPs` 是融合 loop nest 的符号化浮点操作量，`Bytes` 是估计的内存访问量。FuseFlow 根据张量维度、稀疏性和交集率进行估计。相同或相近 FLOPs 下，减少 bytes 通常会提高 OI；系统还根据机器平衡把候选粗略分类为 compute-bound 或 memory-bound。

该信号用于候选排序和剪枝，不应被表述为 LLM 的训练奖励。论文没有给出一个更详细的统一数学目标式，也没有报告该启发式与真实运行时间之间的独立校准误差。

## 6. 实验设置

### 6.1 数据集来源

论文使用两层 GCN 和 GraphSAGE，在 Cora、Cora_ML、DBLP、OGB-Collab 四个真实图数据集上评估。表 1 给出了图结构：顶点数分别为 2708、2995、17716、235868；有向边数分别为 10556、16316、105734、2570930。Feature length 在表中按模型输入/隐藏/输出维度列为 `1433-16-7`、`2879-16-7`、`1639-16-4`、`128-16-2`。

论文没有把这些数据集拆成 LLM 训练集、验证集和测试集；本文是推理时的调度提案实验。数据清洗过程和是否存在与 prompt 相关的数据泄漏风险，论文中未明确说明。

### 6.2 模型与工具

| 项目 | 论文正文确认的信息 |
| --- | --- |
| LLM | GPT-5（OpenAI） |
| 编译器 | FuseFlow，基于 MLIR，实现稀疏 ML 编译流程 |
| 前端 | Torch-MLIR 和 MPACT |
| 后端 | SAM-style dataflow graph；Comal simulator 或 FPGA backend |
| 仿真器 | Comal，cycle-accurate dataflow simulator |
| FPGA 校准 | 与 VU9P FPGA 实现校准，R2=0.991 |
| LLM 选择变量 | fusion groups 与 dataflow order |
| 固定变量 | backend blocking 与 parallelization |
| 代码生成 | FuseFlow 生成数据流图 |

论文未明确给出 GPT-5 的具体模型版本、提示词全文、调用温度、token 预算、FuseFlow/MLIR 的软件版本和硬件 FPGA 实验配置。

### 6.3 对比方法

1. `UNFUSED`：编译器默认方案，不做跨算子融合。
2. `FULLY FUSED`：编译器贪心的全融合/顺序方案。
3. `HAND`：专家或人工整理的调度方案。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
| --- | --- | --- |
| End-to-end speedup | 相对于 UNFUSED 的端到端加速比 | 越大越好 |
| FLOPs | 每次前向执行的浮点操作量 | 需结合 bytes 解读 |
| Bytes | 每次前向执行的估计内存流量 | 越小通常越好 |
| OI | FLOPs/Bytes 的 operational intensity | 通常越大越有利 |
| Tested points | 搜索中评估的候选调度数 | 越少表示搜索成本较低，但不是质量指标 |
| Iterations | beam search 迭代轮数 | 需结合收敛质量解读 |

## 7. 实验结果与结论

### 7.1 主要结果

在固定 backend blocking 和 parallelization 的条件下，AUTOSPARSE 的 LLM-guided schedule 在所有图数据集上与人工专家选择相匹配。相对于无融合 `UNFUSED`，GCN 的几何平均加速比约为 `1.85×`，GraphSAGE 约为 `2.22×`，总体约为 `2×`。这些是 Comal cycle-accurate simulator 上的结果，不应直接写成真实芯片运行时间。

### 7.2 与传统方法的比较

与 `UNFUSED` 相比，LLM 选择的融合方案减少了内存流量并获得约 2× 量级加速。与 `FULLY FUSED` 相比，LLM-guided 方案的几何平均运行性能约快 `13–15×`，说明盲目全融合会显著增加内存流量。与 `HAND` 相比，LLM-guided 方案达到基本相同的性能。

### 7.3 与其他 LLM 方法的比较

本文没有报告与其他 LLM 自动调度器的定量对比，也没有与 LLM Compiler、TVM 中 LLM 方法或其他 LLM baseline 做统一实验。因此不能据此宣称 AUTOSPARSE 在所有 LLM 编译优化任务上优于已有方法。

### 7.4 消融实验

论文没有给出独立的模块消融表。实验通过 `UNFUSED`、`FULLY FUSED`、`HAND` 和 `LLM-guided` 的横向比较，间接说明选择性融合和合法顺序选择的价值，但无法把 LLM、合法性验证器和 FLOPs/bytes 剪枝的贡献完全拆开。

### 7.5 案例分析

在 OGB-Collab 上，GCN 的 unfused 与最佳 fused 方案都约为 `1.7 GFLOPs`，但估计 bytes 从 `445.9 MiB` 降到 `186.8 MiB`；GraphSAGE 从 `625.2 MiB` 降到 `316.4 MiB`。这说明主要收益来自减少数据移动，而不是减少算术操作。搜索统计显示，每个 workload 的 beam search 使用约 9–11 轮迭代，测试约 28–36 个候选调度。

## 8. 主要创新点

### 8.1 创新点一：面向稀疏 ML 的 LLM 调度选择器

现有稀疏编译流程通常依赖人工或规则驱动的 fusion/order 选择。本文把 LLM 放到结构化 schedule proposal 位置，让它选择融合组与合法 dataflow order。实验显示其能在四个图数据集、两种模型上达到专家调度的性能。真正的创新是 LLM 与 FuseFlow 调度接口的结合，而不是单独使用 LLM。

### 8.2 创新点二：半结构化计划格式与编译器合法性门控

LLM 输出 JSON-like 计划，编译器验证名称、形状和 order 合法性。这样把自然语言推理的灵活性限制在可执行配置空间内，避免把 LLM 的字符串输出直接当作可信编译结果。论文提出的是系统接口和验证链，未声称形式化证明整个性能目标。

### 8.3 创新点三：FLOPs/bytes 启发式驱动的低成本剪枝

FuseFlow 通过符号化 FLOPs 和 bytes 估计 operational intensity，并按 roofline 风格排序候选。该机制把候选评估从完整后端执行前移到廉价代价估计阶段，使几十个候选就能找到接近专家的方案。其价值由搜索统计和最终性能结果支持，但论文未与 BO/solver 做实际定量对照实验。

## 9. 局限性

### 9.1 论文明确承认的局限

论文结论指出，当前结果主要聚焦 dataflow；对 cache-heavy GPU，需要增加 cache-aware heuristic 才能迁移。论文还把非 LLM 搜索、模拟退火和 RL 引导搜索列为未来可比较方向。

### 9.2 阅读后发现的潜在局限

1. 评估规模较小：正文只报告两层 GCN、GraphSAGE 和四个图数据集，不能代表更深或更多样的稀疏模型。
2. 后端变量被固定：blocking 和 parallelization 没有交给 LLM 选择，因此实验只覆盖 fusion/order 子空间。
3. 主要性能证据来自 Comal 仿真器；虽然论文报告其与 VU9P FPGA 的 `R2=0.991` 校准，但本文表格并不是完整 FPGA 实测结果。
4. LLM、FuseFlow 合法性验证器和 OI 剪枝没有做严格组件消融，难以确定各自贡献。
5. 没有 DOI 或 arXiv ID，正式可追溯性主要依赖 OpenReview/Workshop PDF；该版本的接收状态和是否存在后续正式扩展需要后续人工跟踪。
6. 论文没有提供完整 prompt、随机种子、模型调用配置和软件版本，复现实验可能受 LLM 服务配置影响。
7. 本文没有正确性证明章节。编译器合法性检查能约束 schedule 结构，但不能自动等价于对所有后端执行语义的形式化证明。

## 10. 阅读后的研究方向反思

AUTOSPARSE 最值得借鉴的是“LLM 提出结构候选，编译器定义合法空间，廉价代价模型负责筛选”的职责分离。这比让 LLM 直接生成后端代码更适合作为 SELECTOR 基线。其核心贡献是稀疏 FuseFlow 的接口和融合/顺序搜索设计，后续工作不能仅把 GPT-5 换成另一模型或把 Comal 换成 RISC-V 就宣称新颖。

仅替换成 RISC-V 平台不够形成创新：RISC-V 相关研究至少还需处理 RVV 的 VLEN/LMUL、向量化合法性、寄存器压力、缓存层次和真实板卡测量，并建立与目标硬件相关的 cost signal。与本文相比，RISC-V 方向可把“合法 schedule + 架构条件 + 实测反馈”结合起来研究跨芯片配置迁移，但这是阅读后的研究问题，不是本文已实现内容。

本文更适合作为 selector/autoscheduling baseline、稀疏 schedule proposal 模块和编译器合法性接口参考；不适合作为通用 LLM 代码生成或形式化验证基线。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RVV 的稀疏数据流选择

#### 研究问题

如何让 LLM 在 fusion group、dataflow order 与 RVV 的 VLEN/LMUL、向量化布局之间进行联合选择。

#### 与原论文的区别

原论文固定 blocking/parallelization 且面向 SAM-style dataflow；新方向把 RISC-V 向量配置和真实后端约束纳入选择空间。

#### 可能的创新点

设计 RVV-aware 合法性枚举、向量寄存器压力代价和跨 VLEN 的配置迁移机制。

#### 实验框架

```text
稀疏图 + RVV 硬件特征 → LLM 提案 fusion/order/vector config
        → LLVM/MLIR 合法性检查 → Spike/真实 RVV 板卡测量
        → 多目标性能、代码尺寸、能耗排序
```

#### 可行性

需要 MLIR/LLVM RVV 后端、Spike 或真实 RISC-V 向量硬件、稀疏图数据集和可重复的测量脚本。

#### 主要风险

真实板卡测量噪声、RVV 后端支持不完整以及仿真代价可能使候选预算过高。

### 11.2 代价模型与运行时反馈联合选择

#### 研究问题

FLOPs/bytes 启发式在不同稀疏性和缓存层次下如何与真实 runtime 反馈结合，避免 OI 高但实测慢的方案。

#### 与原论文的区别

原论文主要在生成代码前使用启发式剪枝；新方向增加低预算的真实运行反馈校准。

#### 可能的创新点

研究不确定性感知的代价模型、候选回滚和 runtime/静态信号加权。

#### 实验框架

```text
LLM 生成合法候选 → OI 预筛 → 少量真实运行测量
        → 校准代价模型 → 选择下一批候选 → 输出最终 schedule
```

#### 可行性

可以复用 FuseFlow 接口，并在 Comal 与 FPGA/CPU/GPU 后端上分层评估。

#### 主要风险

硬件测量噪声和编译/运行成本可能抵消搜索节省；需要严格区分仿真值和实测值。

### 11.3 跨硬件 schedule 迁移

#### 研究问题

如何将某一硬件上积累的 fusion/order 经验迁移到具有不同内存和数据流约束的硬件，而不把旧策略错误复制过去。

#### 与原论文的区别

原论文集中于固定后端设置下的工作负载自适应选择；新方向显式研究多硬件条件化和迁移。

#### 可能的创新点

将硬件特征、稀疏统计和历史候选组织为可检索上下文，并用合法性与性能双门控。

#### 实验框架

```text
源硬件历史 schedule/反馈 + 目标硬件描述
        → LLM 生成迁移候选 → 目标编译器合法性检查
        → 少量目标硬件评估 → 迁移成功率与性能比较
```

#### 可行性

需要至少两种后端、统一的 schedule 表示和同一稀疏工作负载集合。

#### 主要风险

schedule 语义可能跨后端不一致，且有限的目标硬件样本容易造成迁移结论不稳定。

## 12. 与其他已读文献的关系

本 staging 批次只完成 AUTOSPARSE 一篇论文的正文阅读，因此没有在本批次内建立可用于定量横向比较的其他已读论文集合。

从任务边界看，AUTOSPARSE 属于 SELECTOR/S2：LLM 选择 fusion group 和 dataflow order，FuseFlow 执行编译与代码生成。它不是 TRANSLATOR，因为 LLM 不直接输出变换后的源码、IR 或汇编；也不是 GENERATOR，因为论文没有把 LLM 输出的 schedule 固化为可复用编译器 pass 或后端组件。后续批次若纳入同一主题，应把它作为稀疏 ML schedule 选择 baseline，并与 pass/phase 选择、LLM 直接 IR 改写和可复用启发式生成分开比较。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | 稀疏 ML 的 fusion group 与 dataflow order 自动调度 |
| 核心问题 | 稀疏调度空间离散、层次化且受稀疏性约束，人工搜索成本高 |
| 输入 | 模型图、张量形状、稀疏统计、producer-consumer 关系、硬件提示、合法候选空间 |
| 输出 | 结构化 fusion groups + 合法 dataflow orders 的 schedule/config |
| 核心方法 | LLM 提案 + FuseFlow 合法性验证 + FLOPs/bytes 启发式剪枝 |
| 使用的模型 | GPT-5（OpenAI）；具体版本和推理参数未说明 |
| 使用的编译器工具 | FuseFlow、MLIR、Torch-MLIR、MPACT、Comal simulator |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；使用编译器不变量和合法性检查 |
| 数据集规模 | Cora、Cora_ML、DBLP、OGB-Collab；GCN 与 GraphSAGE |
| 主要指标 | speedup、FLOPs、bytes、OI、搜索迭代和 tested points |
| 最重要实验结果 | 相对 UNFUSED，GCN 1.85×、GraphSAGE 2.22× 几何平均加速；接近 HAND |
| 核心创新 | 将 LLM 限定为稀疏 schedule 选择器，并以编译器合法空间和 OI 信号约束搜索 |
| 主要局限 | 后端变量固定、规模较小、主要依赖仿真、缺少组件消融和完整复现配置 |
| 与 RISC-V 研究的相关性 | 中：selector 接口和代价门控可借鉴，但论文未评估 RISC-V/RVV |
| 最适合作为 | 稀疏 schedule 选择 baseline、编译器合法性/代价接口参考 |

这篇论文最值得学习的是把 LLM 的结构化建议与编译器的合法性边界、廉价代价模型结合起来；最主要的局限是只覆盖 fusion/order 子空间且主要在 Comal 仿真环境中评估；如果用于后续研究，最合理的使用方式是作为 selector baseline 和稀疏调度模块，而不是简单地替换模型或硬件后宣称形成新的编译器方法。
