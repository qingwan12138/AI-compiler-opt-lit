# C36 PassNet 文献阅读总结

论文题目：**PassNet: Scaling Large Language Models for Graph Compiler Pass Generation**
作者：Yiqun Liu、Yingsheng Wu、Ruqi Yang、Enrong Zheng、Honglei Qiu、Sijun He、Tai Liang、Jingjing Wu、Yuhan Zhou、Yiwei Zhang、Dongyan Chen、Weihan Yi、Xinqi Li、Siqi Bao（Baidu）
发表时间：2026-05-28；本次阅读日期：2026-09-05
发表平台：arXiv 预印本，2605.29357v1（PDF 版本 v1）
代码/数据/工件：作者公开仓库：[PassNet](https://github.com/PaddlePaddle/PassNet)
元数据核验来源：[arXiv:2605.29357](https://arxiv.org/abs/2605.29357)
关键词：图编译器、编译器 Pass 生成、Tensor Compiler、LLM、正确性与性能评测

> 本文档为 Luna 阅读（2026-09-05）。事实依据为所给 v1 PDF；推测与研究建议单独标明。

## 1. 研究背景

注：第 1 页的 `r=0.013` 指性能缺口与图复杂度的相关性接近于零，不是“算子覆盖与缺口”的相关系数；正文将瓶颈归因于算子覆盖与启发式限制。

PassNet 研究深度学习张量编译器中的图级优化。TVM、XLA、TorchInductor 等把高层计算图降低为设备相关 kernel，并依靠人工编写的融合、切分和布局规则获得较好性能。论文第 1 节对 9,526 个来自 1,000 多个社区模型的子图做 TorchInductor 默认流水线分析：34% 的 kernel 加速低于 1.2×，43% 的端到端结果变慢，8.3% 严格退化（第 1 页）。问题集中在长尾算子组合，而非单纯图规模；论文将瓶颈归因于算子覆盖不足与启发式限制。`r=0.013` 仅对应性能缺口与图复杂度的相关性，不能解释为算子覆盖与性能缺口的相关系数。

论文认为，仅生成独立 GPU kernel 不能自然接入现有编译流水线，还增加部署和验证负担。因此提出“pass generation”：模型输出带模式匹配器和重写器的结构化图变换，直接作为编译器 Pass 使用。100K 模型去重后仅约 18K 个图、约 1,025 个结构模式，说明真实工作负载有强重复性（第 2 页）。

## 2. 论文要解决的问题

### 2.1 数据问题

缺少来自真实模型、覆盖长尾图结构且能支持多形状和多 dtype 泛化的大规模 Pass 生成语料。

### 2.2 评测问题

现有 kernel 评测容易只看输出或单个任务，可能接受调用高层 API 的“投机”实现；也难同时反映正确性、稳定性和性能。

### 2.3 任务定义

给定共享算子序列但形状、dtype 可变的子图任务 T={G1,…,Gk}，生成 Pass π=(M,R)，其中 M 是模式匹配器，R 是语义等价重写器，使全部 Gi 在误差容忍度下正确且总体运行更快（第 3.1 节，第 3 页）。

> 本文主要研究：如何以大规模真实图数据和抗投机评测，训练、比较和改进可组合的图编译器 Pass 生成模型。

## 3. 核心方法概述

PassNet 是数据集、PassBench 基准、PassAgent 工具脚手架和 Error-aware Speedup Score（ESt）指标组成的生态，而非单一模型。

```text
真实模型/GraphModule
        ↓ torch.fx 符号追踪与五项质量过滤
18,086 去重全图 → Recursive Folding / Prefix Analysis / 单算子抽取
        ↓ 形状与 dtype 泛化
约279K分层子图 → PassNet-Dataset / PassBench任务
        ↓ PassAgent（文件编辑 + 评测器）
LLM生成 pattern matcher + rewriter + manifest
        ↓ AST检查、运行时 dispatch 白名单、反向评测
正确性/稳定性/速度反馈 → ESt、AS Score
```

与自由 kernel 生成不同，Pass 必须针对图结构生成可执行的匹配与重写逻辑；同一任务包含多形状、多精度实例，阻止只为单个形状硬编码。论文展示的 MaskFormer roll+slice+layer_norm 和 BGE masked mean pooling 说明 LLM 能恢复被底层算子分解掩盖的复合意图（第 4.4 节，第 9 页）。

## 4. 实验框架与训练流程

### 4.1 数据构建

从 PyTorch、PaddlePaddle 社区库收集 100K+ 模型，提取图、张量元数据、权重和运行时元数据；每图须可运行、可序列化、可分解、可静态分析，且自定义算子源码可访问（附录 A，第 15 页）。

### 4.2 子图构造与任务分组

Recursive Folding 先把图拓扑线性化，再用卷积式哈希递归折叠高频结构；Prefix Analysis 观察前缀算子数到 kernel 数的平台区间发现可融合片段；另保留单算子子图。实例化时使用 10 种形状配置和 3 种 dtype（第 3.2—3.3 节）。PassBench 按算子序列、对数形状桶和 dtype 分组，训练/评测无重叠。

### 4.3 推理、蒸馏与后训练

PassAgent 使用 file_editor 和 pass_evaluator，最多 50 步迭代，每次根据匹配、正确性、性能诊断修改文件。主实验比较 6 个前沿/开源模型。另用 Claude-Sonnet-4.6 生成 4,476 个实例、保留 AS>0.1 的 3,899 条轨迹，对 Qwen3-30B-A3B 和 Qwen3-4B 做 SFT（学习率 2e-5 余弦衰减至 2e-6，batch 8，5 epoch，256K 上下文；附录 C 表 4）。本文没有已实现的 PPO/GRPO；RL from ESt 只是未来方向（第 5 节）。

## 5. 奖励函数、损失函数或关键公式

本文不是强化学习论文，但把 ESt 作为代理迭代和未来训练信号。对第 i 个子图，速度比为 si，错误类别 ci∈{1,2,3}（精度、编译、运行时错误），容忍阈值 t∈{-10,…,|E|+1}：

```text
ŝt,i = si                         （正确且 si≥1）
      = si^(p+1)                  （正确但 si<1）
      = b^(1[t<ci])               （错误）
ESt = (Π_i ŝt,i)^(1/N)
AS = Π_t ESt^(Wt / Σ_s Ws)
```

实验固定 b=0.1、p=0（第 4.1 节，第 7 页）。AS 是跨容忍度的归一化几何平均；正确但变慢会受惩罚，错误在较宽容阈值下才被放过。它是平滑、细粒度的性能/正确性反馈，不是形式化证明。

## 6. 实验设置

### 6.1 数据集来源

PassNet 有 18,086 个去重全图，来自 100K 模型；领域占比 NLP 63.6%、CV 27.0%、多模态 1.7%、音频 1.2%、其他 6.5%。生成 129K fusible、126K classical、24K single-operator 实例，约 279K；PassBench 主要为 200 个评测任务、2,060 个子图级评测，任务每组 1–396 个子图，平均约 10（第 3.3—3.4 节）。

### 6.2 模型与工具

NVIDIA A30 24GB；CUDA/cuDNN 12.8/9.10.2；PyTorch/Triton 2.9.1+cu128/3.5.1；Ubuntu 24.04.1。每项 20 次 warmup、100 次计时；IQR 超过中位数 20% 重跑。模型包括 GPT-5.4、Claude-Opus-4.6、Claude-Sonnet-4.6、GLM-5.1、MiniMax-M2.7、Qwen3-4B/30B。

### 6.3 对比方法

Eager 是 1.0×参考；TorchInductor 默认模式是传统编译器基线。论文明确说明没有用 max-autotune，因为其 CUDA Graph 布局冻结和开销在此规模下抵消收益（第 4.1 节）。

### 6.4 评价指标

fast_1 是相对 eager 至少 1×的正确子图比例；Samp. CR/ Sub. CR 分别是正确样本/子图比例；G-Mean Speedup 是正确子图的几何平均速度比；AS 综合容忍度下的 ESt。

## 7. 实验结果与结论

### 7.1 主要结果

Inductor 的 G-Mean=0.846、AS=0.706；最佳前沿模型 Claude-Opus-4.6 的 G-Mean=0.922，Claude-Sonnet-4.6 的 AS=0.448，低于 Inductor 37%。Claude-Sonnet 的子图正确率 61.9%，Qwen3-30B-A3B 仅 11.8%，AS 相差 3.22×（表 1，第 8 页）。模型整体仍未达到 G-Mean≥1。

### 7.2 个案与训练

MaskFormer 的 roll+slice 由 6 kernel 融合为 1 kernel，相对 eager/Inductor 为 1.65×/3.02×；BGE-Reranker masked pooling 为 1.50×/2.90×，最大误差 0.07，kernel 数 7→1（表 2，第 9 页）。Qwen3-30B-A3B-SFT 的 AS=0.371，相比 0.139 提升 2.67×；正确子图率 48.8%，样本正确率 44.0%（第 4.3 节）。

### 7.3 失败与消融性质

失败主要是融合边界判断错误、缺少硬件代价模型、把局部重写成不透明 kernel 后破坏 FlashAttention 等后续链。单次评测只捕获最终最佳 AS 的 31%–51%；12%–52% 的最终通过样本出现 pass→fail→pass（图 3，第 8 页），说明迭代预算影响显著。论文没有把这些称为严格消融，而是能力与失败模式分析。

## 8. 主要创新点

### 8.1 Pass generation 抽象

把 LLM 输出约束为可组合的 matcher/rewriter Pass，兼顾部署和可验证性；创新不在“使用 LLM”，而在把图变换作为可泛化、多实例任务。

### 8.2 真实图语料与分层生成

18K 去重真实图、递归折叠和前缀平台分析共同覆盖结构重复与融合机会，形成约 279K 分层实例。

### 8.3 抗投机的多维指标与基准

AST 检查拦截 78% 已知违规，PoisonDispatchTensor 独占识别 18%，反向评测修复缓存污染漏洞（第 3.6 节，第 6—7 页）；ESt/AS 把正确性、稳定性、速度统一为连续信号。

## 9. 局限性

**论文明确承认：**当前主要评测单 GPU 推理和 fusible tasks；多设备、训练循环、复杂 classical subgraph 尚未覆盖。数据中 NLP+CV 达 90.6%，科学计算和生成模型代表性不足；反作弊防御不能保证抵御未来攻击（第 5 节，第 9—10 页）。

**阅读后潜在局限：**PassBench 的真实硬件结论绑定 NVIDIA A30、CUDA 12.8 和 PyTorch/Triton 版本；ESt 的数值依赖容忍度和惩罚设定。论文只做有限测试与运行时比较，不是形式化语义证明；把实现迁移到 RISC-V 需要重新定义图后端和硬件代价，不能仅替换设备名。

## 10. 阅读后的研究方向反思

最值得借鉴的是多图、多形状、多 dtype 的 Pass 约束和“正确性—稳定性—性能”联合反馈。核心贡献已经是 PassNet 数据与 PassBench 防投机机制，不能只把 TorchInductor 换成 RISC-V 后宣称新方法。对 LLVM/RISC-V 的关系是间接的：它提供图级 Pass 数据与评测范式，可作为数据/基准方法参考；并不直接证明 LLVM IR 或 RISC-V 后端收益。更适合作为图编译优化的 benchmark、训练数据与工具模块，而不是完整跨架构框架。

## 11. 可进一步尝试的研究方向

### 11.1 面向多后端的硬件代价条件 Pass 生成

#### 研究问题

把硬件代价摘要作为上下文，生成跨 A30、CPU 和 RISC-V/NPU 后端仍有效的融合 Pass。

#### 与原论文的区别

原文主要是单 A30 推理；新问题要求后端条件化和跨设备泛化。

#### 可能的创新点

统一代价描述、后端特定正确性/性能门槛及跨设备鲁棒训练。

#### 实验框架

```text
图与硬件摘要 → LLM生成Pass → 多后端编译/运行 → ESt向量 → 选择泛化Pass
```

#### 可行性

可复用 PassNet 图格式、TorchInductor/MLIR 后端和真实计时设备。

#### 主要风险

跨设备 timing 噪声、算子覆盖不足和 Pass 语义差异。

### 11.2 ESt 驱动的安全后训练

研究把现有 AS 反馈用于 RL，同时保留 AST/dispatch 防投机；区别是验证闭环成为训练约束，而不是论文未来工作中的一句设想。风险是奖励稀疏、反作弊与训练目标冲突。

## 12. 与其他已读文献的关系

与 Magellan 同样用可执行编译器代码和真实性能反馈，但 Magellan 演化 LLVM/XLA 的 Pass 内启发式，PassNet 生成张量图 Pass；前者是 compiler decision logic，后者是 graph rewrite。与 SuperCoder 相比，PassNet 输出可组合图 Pass、正确性由编译/运行测试与误差阈值判断，SuperCoder直接改 x86-64 汇编并用测试+端到端 runtime。三者可组合为“图级候选生成—后端启发式选择—汇编级局部超优化”的分层基线，但不能把其中任一层的结果外推到其他层。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 真实长尾张量图的 LLM Pass 生成基础设施 |
| 核心问题 | 数据稀缺、评测可投机、正确性与性能难联合 |
| 输入/输出 | 子图任务；pattern matcher + rewriter Pass |
| 核心方法 | PassNet-Dataset、PassBench、ESt/AS、PassAgent |
| 使用的模型 | Claude、GPT、GLM、MiniMax、Qwen；Qwen SFT |
| 工具 | torch.fx、PyTorch/Triton、TorchInductor、AST/dispatch防护 |
| 强化学习/形式验证 | 未使用 RL；不是形式化证明 |
| 数据规模 | 100K模型→18,086图→约279K子图；200评测任务 |
| 最重要结果 | 在 PassBench 200 个评测任务上，Inductor AS=0.706；最佳前沿模型 Claude-Sonnet-4.6 的 AS=0.448；Qwen3-30B-A3B 在约 4K 轨迹上 SFT 后 AS=0.371，相对其基座 0.139 提升 2.67× |
| 核心创新 | 多实例 Pass 生成与抗投机联合基准 |
| 主要局限 | 单 A30、推理/fusible 偏重、非形式证明 |
| RISC-V相关性 | 中低：可借鉴基准与数据范式，未直接评测 RISC-V |
| 最适合作为 | 图编译 benchmark、数据来源、训练/评测工具模块 |

这篇论文最值得学习的是把“能写 kernel”收紧为可组合、可泛化的编译器 Pass；最主要的局限是后端和任务覆盖仍窄。如果用于后续研究，合理方式是把其数据和 ESt 作为基准，补充硬件代价和跨后端证据，而不是简单替换为 RISC-V。
