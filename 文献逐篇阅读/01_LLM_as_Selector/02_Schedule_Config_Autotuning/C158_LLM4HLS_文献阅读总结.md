# LLM4HLS 文献阅读总结

论文题目：**High-level Synthesis Directives Design Optimization via Large Language Model**

作者：Xufeng Yao、Wenqian Zhao、Qi Sun、Cheng Zhuo、Bei Yu

发表时间：2025

发表平台：ACM Transactions on Design Automation of Electronic Systems (TODAES), 30(5), Article 78（2025）

论文链接或编号：DOI 10.1145/3747291
元数据核验来源：[ACM DOI](https://doi.org/10.1145/3747291)；[CUHK 作者机构页](https://research.cuhk.edu.hk/en/publications/high-level-synthesis-directives-design-optimization-via-large-lan/)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：高层次综合（High-Level Synthesis, HLS）、FPGA、综合 directives、设计空间探索（Design Space Exploration, DSE）、贝叶斯优化、Gaussian Process、Pareto 优化、PPA、LLM

> 本文档依据论文正文和表图阅读整理。论文事实、阅读分析和后续建议分开描述；本文将 HLS directive 译为“综合指令/配置”，指示 HLS 工具如何并行化、分块、展开和组织存储。

---

## 1. 研究背景

HLS 将 C/C++ 等高层描述转换为 Verilog/VHDL 等硬件描述，使硬件设计者可以通过综合工具快速获得 RTL。directive 是 HLS 设计中控制并行化、存储分配和资源使用的重要配置；不同 directive 组合会导致不同的功耗、延迟和资源利用率（引言、图 1–2）。

HLS directive 设计通常被表述为黑盒设计空间探索（DSE）：每个候选配置必须经过综合和实现流程才能得到可信的 PPA（Power、Performance、Area）结果。穷举搜索成本高，而传统代理模型常把原始 directive 手工编码为数值，可能丢失 directive 的上下文、层级关系和语义信息（第 1 节）。

论文的动机是利用 LLM 的预训练知识和上下文建模能力：一方面从原始代码和 directive 中抽取特征，另一方面比较两个候选配置并优先选择可能更好的配置，再以综合报告中的实际 PPA 反馈微调模型。

## 2. 论文要解决的问题

### 2.1 冷启动和搜索成本

DSE 初始采样如果集中在低质量区域，后续 BO 可能陷入局部最优；逐点运行 FPGA/HLS 综合又非常耗时。论文希望用 LLM 的先验比较能力改善初始采样。

### 2.2 原始 directive 的表示问题

将 unroll factor、数组分区等 directive 直接归一化为数值，难以表达代码上下文、循环层级和不同 directive 之间的关系。论文用 LLM 倒数第二层隐状态作为 directive 特征，再交给 Gaussian Process 建模。

### 2.3 多目标 Pareto 探索

功耗、LUT 使用量和延迟彼此冲突，目标不是单一最优点，而是尽量逼近真实 Pareto 前沿。论文将层次化区域选择、GP/UCB 和 LLM 两两比较组合为 coarse-to-fine 搜索。

> 本文主要研究：如何利用 LLM 的 directive 理解、候选比较和综合反馈能力，减少 HLS directive 多目标设计空间探索的代价并提高 Pareto 解质量。

## 3. 核心方法概述

论文提出一个建立在 Bayesian Optimization（BO，贝叶斯优化）和 Gaussian Process（GP，高斯过程）之上的 LLM 辅助 HLS DSE 框架。LLM 不直接生成硬件 RTL，而是分析代码和 directive、输出候选比较结果及隐藏特征；现有 FPGA/HLS 工具负责综合并产生 PPA 报告。

```text
HLS C/C++ 代码 + directive 空间 + FPGA/约束信息
        ↓
LLM 初始分析、两两比较、分层 tournament 采样
        ↓
FPGA/HLS 综合工具生成 Verilog 与 PPA 报告
        ↓
解析功耗、LUT、延迟等结果，形成 directive-PPA 数据对
        ↓
微调 LLM；抽取隐状态特征并更新 GP
        ↓
UCB1 选区域 → UCB2 选区域内 top-k → LLM 比较筛选
        ↓
重复试验并输出 Pareto set
```

LLM 的最终编译系统角色是 SELECTOR：它输出更可能优越的 directive 索引/配置，或为配置提供搜索策略；编译器和 HLS 后端实施实际变换。

## 4. 实验框架与训练流程

### 4.1 初始采样阶段

论文使用“先分析、后比较”的 prompt。给定 HLS 代码和两组 raw directives，LLM 先生成分析 thought，再在追加 comparison prompt 后预测更好的 directive 编号。为使输出结构化，只读取首个生成 token 的概率，在候选编号中取概率最大的项（第 3.2 节、图 6）。

候选空间按较粗粒度的 RAM pragma/区域、较细粒度的循环展开因子和顺序等分层；每层使用 tournament selection，候选两两比赛，败者淘汰，胜者进入下一轮（图 7）。初始比较不要求对 LLM 做微调，论文指出闭源 ChatGPT 也可以用于该阶段。

### 4.2 综合反馈微调阶段

每个被综合的配置产生 Verilog 和 synthesis report。脚本解析报告中的 PPA，构造包含 raw code、directives、PPA 和分析 prompt 的训练样本。LLM 生成多个 thought，通过实际综合结果确定正确 directive，并选择其正确编号概率最高的 thought。

论文受 RLHF 启发，但这里采用监督式 next-token 训练，不是 PPO/GRPO 等在线强化学习。训练损失由 contrastive-like logits loss 和 directive-index cross-entropy 组成。

### 4.3 BO 与二阶段采样

微调后的 LLM 对每个候选抽取倒数第二层隐状态特征，GP 在这些特征上拟合综合结果。第一阶段用 UCB1 在预定义区域之间选取区域，第二阶段用 UCB2 在该区域选出 top-k directive，再交给 LLM 做两两比较。选出的配置送入综合，结果加入数据集并进入下一轮。

### 4.4 运行循环

算法 1 的输入包括 LLM、试验次数 N、分区后的 HLS 空间、prompt、GP、UCB1/UCB2；每轮依次更新 LLM、抽取特征、选择区域、选区域内候选、用 LLM 筛选并累积候选，最终从累积点中取 Pareto set。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数；论文将 synthesis feedback 用于微调和 BO 的目标建模。关键公式包括：

1. GP 目标：`f(x) ~ GP(m(x), k(x, x'))`，其中 `m` 和 `k` 分别是均值函数和协方差核函数，观测值含高斯噪声。
2. LLM 比较：`choice = argmax({pr_d1, pr_d2})`。`pr` 是 comparison prompt 后首 token logits 经 softmax 得到的编号概率。
3. Thought 采样：`x_(n+1) ~ Categorical(softmax_T(out))`。温度 `T` 越高，生成分布越均匀，thought 多样性越大。
4. 微调损失：论文公式（10）为 contrastive-like 项与交叉熵项之和。`d*` 是由综合结果得到的较优 directive 编号，`d` 是另一个编号；前一项提高 `d*` 的 logits、压低其他编号，后一项提高正确编号概率。
5. 区域 UCB：`UCB1(x) = μ1(x) + β σ1(x)`，其中 `μ1` 是区域期望值，`β` 平衡探索和利用，`σ1(x) = sqrt(2 log(T) / n(x))`。
6. 区域内 UCB：`UCB2(x) = μ2(x) + λ σ2(x)`，用于在选定区域内取得 top-k 候选；`λ` 控制探索与利用。
7. PPA 目标：`PPA = α1 * Power + α2 * LUT + Delay`。论文将 LUT 作为主要面积指标，Delay 取 latency 与 clock period 的乘积；α 的具体数值在正文中未明确给出。

论文还使用 ADRS 衡量学习 Pareto 集与真实 Pareto 集之间的平均距离。ADRS 越低，表示学习到的 Pareto 集越接近参考前沿。

## 6. 实验设置

### 6.1 数据集来源

实验使用开源 PolyBench 中的 7 个 FPGA benchmark：atax、bicg、gemm、gesummv、k3mm、syr2k、syrk。每个 benchmark 约生成 2,000 个 directive 设计点，并解析对应综合目标。论文没有给出独立 train/validation/test 的明确划分，也未报告去重和数据泄漏检测细节。

### 6.2 模型与工具

LLM 为开源 Llama-13B；最大 token 长度为 2048；使用 4 张 NVIDIA A100 80G 做微调和推理，并使用 DeepSpeed、FlashAttention。目标 FPGA 为 Xilinx xc7z030sbg485-1。综合报告来自 Vivado HLS/FPGA 设计工具；单个综合点通常需数分钟，而一次 LLM directive 比较只需数秒。

### 6.3 对比方法

表 1、表 2 对比 FPL18、ANN、BT、DAC19、TODAES22 和本文方法。它们分别代表基于 BO/GP 的线性多保真方法、人工神经网络、Boosting Tree、回归模型引导的 DSE，以及基于 Bayesian/GP 的非线性多保真方法。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| normalized ADRS | 学习 Pareto 集到真实参考 Pareto 集的平均距离，并按 ANN 归一化 | 越低越好 |
| normalized ADRS std | 多次结果的归一化标准差，反映稳定性 | 越低越好 |
| Power | 综合报告中的功耗 | 越低越好 |
| LUT | FPGA 查找表使用量，作为主要面积指标 | 越低越好 |
| Delay | latency × clock period | 越低越好 |

论文明确说明未把 BRAM、DSP、FF 等其他资源纳入当前主要目标，计划留待未来工作。

## 7. 实验结果与结论

### 7.1 主要结果

表 1 在 7 个 benchmark 上给出 normalized ADRS。本文方法的平均值为 0.51，FPL18、ANN、BT、DAC19、TODAES22 分别为 0.86、1.00、0.98、1.18、0.73。相对于第二好的 TODAES22（0.73），论文报告本文方法平均降低 ADRS 约 15%。在 7 个 benchmark 上本文方法均为最低值：atax 0.24、bicg 0.49、gemm 0.53、gesummv 0.54、k3mm 0.73、syr2k 0.68、syrk 0.39。

### 7.2 稳定性

表 2 的平均 normalized ADRS 标准差为 0.05，FPL18、ANN、BT、DAC19、TODAES22 分别为 0.41、1.00、0.22、0.91、0.20。以 gesummv 为例，本文方法标准差为 0.04，而 ANN 为 1.00；但论文未在表中给出未归一化标准差或每个方法的重复次数。

### 7.3 Pareto 点与 PPA

图 10 显示本文方法得到的 Pareto 点总体更接近真实前沿。论文在 gesummv 上报告相对于竞争方法最高约 12% 的功耗下降和约 8% 的延迟下降，在 syr2k 上约 10% 的功耗下降和 6% 的延迟下降；这些是图示比较中的任务级结果，不是所有 benchmark 的平均值。

### 7.4 LLM 比较准确率

图 11 比较 Llama-13B、ChatGPT 和利用 synthesis feedback 微调后的模型。论文称微调模型在所有 benchmark 上超过 ChatGPT，提升幅度随 benchmark 约为 4%–10%；图中给出的是 directive 比较准确率，但正文没有逐点列出全部原始百分比。

### 7.5 特征和复杂度分析

图 12 的 t-SNE 显示 LLM 抽取的特征比 deep-kernel 特征更分散、可区分性更强。图 13 的推理时间随设计代码行数呈线性趋势。论文还指出，一次 LLM 采样的候选生成成本相对不随 directive 维度显著增加，但综合仍是主要瓶颈。

## 8. 主要创新点

### 8.1 用 LLM 比较 directive 并改善冷启动

论文把两个 directive 的性能比较改写为“预测更好编号”的选择问题，利用 LLM 先验和分析 thought 进行 tournament sampling。价值在于减少无信息的初始综合点；图 11 和整体 ADRS 结果支持该设计有效，但并未证明 LLM 判断可以替代综合验证。

### 8.2 用 LLM 隐状态替代手工 directive 特征

论文直接使用 LLM 倒数第二层隐状态表征 raw code/directive，再让 GP 学习 PPA 关系，避免完全依赖把 directive 数值化的人工特征工程。t-SNE 可视化提供了特征分布证据，但没有给出独立的特征消融来量化每个表示组件的贡献。

### 8.3 synthesis feedback 驱动的 LLM 微调

论文把实际综合得到的 directive-PPA 对转成比较训练样本，并以 contrastive-like loss 加 cross-entropy 训练，使模型逐步提高 directive 选择概率。这是面向 HLS DSE 的反馈适配机制，不等同于完整 RLHF 或在线 RL。

### 8.4 层次化 BO + LLM 的 coarse-to-fine 搜索

UCB1 先选 promising region，UCB2 再选 region 内 top-k，LLM 最后做候选比较，兼顾区域探索、局部利用和语义筛选。该组合是本文方法框架层面的主要贡献。

## 9. 局限性

### 9.1 论文明确承认的局限

论文承认当前框架不专门面向极复杂 HLS 设计，受到输入代码长度、可用 transformation/primitives 多样性以及 LLM 语法错误的影响；目前主要面向规则的 dense linear algebra 和 affine loop nest。当前实验只用 LUT、功耗和 delay，BRAM 等资源未纳入主要目标；论文把 transfer learning、更多 prompt 结构、复杂设计处理和额外资源指标留作未来方向。

### 9.2 阅读后的潜在局限

第一，实验只覆盖一个 FPGA 板卡和 7 个 PolyBench kernel，跨板卡、ASIC、稀疏/不规则控制流的泛化仍未被实验证明。第二，论文把较优 directive 标签从综合结果导出，但没有充分说明每轮样本如何划分及是否存在同一 kernel 的近邻泄漏。第三，LLM 输出比较结果最终仍必须经过综合，若综合耗时为主要瓶颈，LLM 推理加速不一定转化为端到端 DSE 加速。第四，`α1`、`α2` 和一些采样超参数的具体设置不完整，复现实验需要额外信息。第五，LLM 隐状态特征的可解释性和对不同 HLS 工具/指令语法的可迁移性仍有限。

## 10. 阅读后的研究方向反思

本文适合作为 SELECTOR / S2 的方法参考：LLM 选择并比较 directive，BO 负责数值化搜索，HLS 工具提供可信执行反馈。它不是 RISC-V 论文；若只把 FPGA 换成 RISC-V 后端，创新性不足。可借鉴的是“语义感知候选选择 + 编译器/综合反馈 + 代理模型”的接口，而不是直接照搬 Llama-13B、Vivado 或 PolyBench 设置。

与本仓库已有的 LLVM pass 选择、模型服务调度和 GPU kernel 搜索相比，本文的差异是 HLS directive/PPA/Pareto 场景；它可以作为跨层硬件配置选择的对照基线，但不应与直接输出源代码/IR/RTL 的 TRANSLATOR 混淆。

## 11. 可进一步尝试的研究方向

### 11.1 RISC-V 向量 HLS/编译配置选择

#### 研究问题

能否让 LLM 选择 RVV-aware 的循环展开、向量宽度、内存分区和编译 flag，使 HLS 或 LLVM 后端在功耗、代码大小和延迟之间取得 Pareto 解？

#### 与原论文的区别

目标从 FPGA directive 扩展为 RISC-V/RVV 编译配置，并引入真实 RVV 硬件或周期级模拟器反馈，而不是简单替换目标字符串。

#### 可能的创新点

联合建模 RVV 向量化合法性、寄存器压力、访存行为和能耗；使用硬件计数器校准 GP 的不确定性。

#### 实验框架

```text
LLVM IR/C 源码 + RVV 配置候选 → LLM 选配置 → LLVM/汇编/模拟器
        ↓ 性能、代码大小、计数器反馈
        → GP/UCB 更新 → Pareto 配置集
```

#### 可行性

需要 LLVM/RVV、QEMU 或 RTL/周期模拟器、可用 RISC-V 板卡和小规模 kernel 集合。

#### 主要风险

真实硬件噪声、RVV 实现差异和编译器版本变化可能使 PPA 标签不稳定。

### 11.2 编译器证据约束的 directive 比较

#### 研究问题

在让 LLM 比较候选前，加入依赖分析、资源预算和 legality 检查，能否降低非法/灾难性候选？

#### 与原论文的区别

原论文主要依赖 LLM 先验和综合后的标签；新方向在选择前加入可验证证据，并把拒绝原因反馈给模型。

#### 可能的创新点

将候选比较从纯概率排序改成 evidence-calibrated ranking，并报告非法率、综合失败率和单位搜索成本。

#### 实验框架

```text
候选 directive → 依赖/资源预检 → LLM 证据排序 → HLS 综合 → 结果校准
```

#### 可行性

可从现有 HLS pragma parser、LLVM 分析和 Vivado/Bambu report parser 开始。

#### 主要风险

预检器可能过度过滤有潜力的候选；证据模块的误报会损害探索性。

### 11.3 跨架构 directive 迁移

#### 研究问题

一个在 FPGA 或 x86 上学习的 directive 语义表示，能否迁移到 RISC-V、GPU 或另一块 FPGA？

#### 与原论文的区别

研究重点从单平台 DSE 改为跨平台表示迁移和少样本校准，需显式测量平台差异。

#### 可能的创新点

把 directive 的代码上下文、硬件约束和历史 PPA 分成可迁移与平台特定因素，并采用 adapter/低秩微调。

#### 实验框架

```text
源平台 directive-PPA 数据 → LLM/GP 表示 → 少量目标平台综合点
        ↓ 平台校准 → 目标平台 Pareto 搜索
```

#### 可行性

需要至少两个可重复的编译/综合目标和统一 benchmark；可先以 FPGA 板卡对比，再扩展到 RVV。

#### 主要风险

不同平台 Pareto 前沿可能不具备可比尺度，且少量目标数据可能不足以校准模型。

## 12. 与其他已读文献的关系

本批次只有本文一篇完成全文阅读，因此不能虚构与其他新笔记的实验级横向结论。根据正式 corpus 的已读记录，本文与 SELECTOR/S2 的已有工作共享“输出配置/调度并由编译器执行”的角色，但研究对象是 HLS directive 和 FPGA PPA Pareto DSE；与 SELECTOR/S3 的 LLVM pass 序列/RL 工作不同，本文没有使用 PPO/GRPO，而是 BO 加综合反馈微调；与 SELECTOR/S4 的 agent/tool-action 工作相比，本文的主要贡献是候选采样和比较，而非开放式多工具编排。它可以作为 HLS directive 选择基线或 synthesis-feedback 模块参考，不能作为直接 IR/源码翻译基线。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLM 辅助 HLS directive 多目标设计空间探索 |
| 核心问题 | 综合昂贵、directive 特征工程丢失语义、Pareto 搜索困难 |
| 输入 | HLS C/C++ 代码、raw directives、FPGA/约束信息 |
| 输出 | 被选择的 directive 配置与 Pareto set |
| 核心方法 | LLM 比较 + 隐状态特征 + GP/UCB + synthesis-feedback 微调 |
| 使用的模型 | Llama-13B；初始比较也可用 ChatGPT |
| 使用的编译器工具 | Vivado HLS/FPGA 综合工具、PPA 报告解析脚本 |
| 是否使用强化学习 | 否；受 RLHF 启发的监督微调，不是 PPO/GRPO |
| 是否使用形式化验证 | 未报告形式化验证；依赖 HLS/综合流程获得结果 |
| 数据集规模 | 7 个 PolyBench benchmark，每个约 2,000 个设计点 |
| 主要指标 | normalized ADRS、其标准差、Power、LUT、Delay |
| 最重要实验结果 | 平均 normalized ADRS 0.51，较 TODAES22 的 0.73 约低 15% |
| 核心创新 | 语义感知 directive 比较、LLM 特征、两阶段 coarse-to-fine 采样 |
| 主要局限 | 单板卡、规则 dense kernel、资源指标不全、跨平台泛化未证实 |
| 与 RISC-V 研究的相关性 | 中；方法接口可迁移到 RVV 配置选择，但原实验不是 RISC-V |
| 最适合作为 | SELECTOR/S2 方法参考、HLS DSE baseline、synthesis-feedback 模块 |

> 这篇论文最值得学习的是把 LLM 放在 directive/config 选择环节，并让综合结果反过来校准模型；最主要的局限是实验范围和复现细节有限。用于后续研究时，合理方式是把它作为 HLS/RVV 配置选择基线并补充 legality、硬件计数器和跨平台实验，而不是把 LLM 改成直接生成 RTL 或仅替换硬件平台。
