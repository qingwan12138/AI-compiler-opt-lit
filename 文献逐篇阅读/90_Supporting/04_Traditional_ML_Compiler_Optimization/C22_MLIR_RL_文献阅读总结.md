# MLIR RL 文献阅读总结
代码/数据/工件：作者代码：[MLIR-RL](https://github.com/Modern-Compilers-Lab/MLIR-RL)；[Zenodo 作者工件](https://doi.org/10.5281/zenodo.17987133)
元数据核验来源：[论文/来源](https://doi.org/10.1109/CGO68049.2026.11394838)；[论文/来源](https://arxiv.org/abs/2409.11068)；[论文/来源](https://github.com/Modern-Compilers-Lab/MLIR-RL)

论文题目：**A Reinforcement Learning Environment for Automatic Code Optimization in the MLIR Compiler**

作者：Mohammed Tirichine、Nassim Ameur、Nazim Bendib、Iheb Nassim Aouadj、Djad Bouchama、Rafik Bouloudene、Riyadh Baghdadi

发表时间：首次公开于 2024 年；正式发表于 CGO 2026，pp. 696–710。

发表平台：CGO 2026；[DOI 10.1109/CGO68049.2026.11394838](https://doi.org/10.1109/CGO68049.2026.11394838)；[arXiv:2409.11068](https://arxiv.org/abs/2409.11068)。作者公开代码：[Modern-Compilers-Lab/MLIR-RL](https://github.com/Modern-Compilers-Lab/MLIR-RL)；Artifact：Zenodo DOI 10.5281/zenodo.17987133。

关键词：MLIR、强化学习、自动代码优化、循环变换、多离散动作空间、PPO、编译器反馈

> 本文档基于 16 页完整 PDF（含工件附录）整理。论文事实来自正文；第 10 节之后的扩展设想均标明为阅读后的分析。

## 1. 研究背景

高性能循环代码通常依赖专家手工选择 tiling、融合、循环交换、并行化和向量化参数。整数线性规划、代价模型加树搜索等自动方法可以减少人工工作，但搜索候选数大，且常需要预设搜索顺序。已有强化学习（Reinforcement Learning，RL）编译工作多集中在 LLVM pass ordering、高层 DNN 图变换或研究型编译器，难以直接表达 MLIR 中“变换类型 + 参数 + 作用位置”的结构化决策（第 1–2 节）。

MLIR（Multi-Level Intermediate Representation，多层中间表示）拥有多 dialect、参数化循环变换和广泛前后端，适合作为通用研究环境；但 CompilerGym 的固定 LLVM pass 动作不能直接覆盖 MLIR 的 tile size、融合目标和具体循环层等组合，且完整训练需要反复编译和执行，代价很高。论文据此提出面向 MLIR Linalg dialect 的原生 RL 环境。

## 2. 论文要解决的问题

### 2.1 如何表达 MLIR 的大规模参数化动作空间

一个动作不仅要选变换，还要选每层 tile size、循环排列或融合对象。平铺成单一离散集合会产生组合爆炸；循环交换本身有 `N!` 个排列。

### 2.2 如何构造可学习的程序状态与可执行反馈

状态既要描述 Linalg 操作、循环范围、迭代类型和访问映射，也要保留动作历史；奖励必须反映真实执行时间，同时避免每一步都执行代码造成训练过慢。

### 2.3 环境是否能跨两个领域产生有效优化

论文不仅测试深度学习算子和模型，还测试 Lattice Quantum Chromodynamics（LQCD，格点量子色动力学）长循环代码，以检验环境是否局限于单一 DNN 图模式。

> 本文主要研究：如何在 MLIR Linalg 上构建一个能选择参数化循环变换、以真实执行时间反馈训练、并可复现实验的强化学习优化环境。

## 3. 核心方法概述

```text
PyTorch / LQCD DSL 生成 MLIR Linalg
        ↓
按 consumer→producer 逆序选择当前操作对
        ↓
提取操作类型、循环范围、迭代类型、访问矩阵、算术计数、动作历史
        ↓
LSTM 编码 producer-consumer + 三层 512 单元 MLP backbone
        ↓
策略头先选 6 类变换，再选该变换参数
        ↓
MLIR 执行 tiling / tiled parallelization / tiled fusion /
interchange / vectorization / no transformation
        ↓
动作掩码排除非法或明显失控的选择
        ↓
整条变换序列结束后编译、运行并计算 log(speedup)
        ↓
PPO 更新 actor-critic；训练后用策略生成优化序列
```

六类动作包括 Tiling、Tiled Parallelization、Tiled Fusion、Interchange、Vectorization 和 No Transformation。多离散动作空间先选变换，再按循环层分别选择参数。对循环交换，`Level Pointers` 逐位置选择尚未使用的循环层，用动作掩码完成一个排列，从而覆盖全部排列而不显式枚举 `N!` 个动作（第 4 节与附录 B）。

## 4. 实验框架与训练流程

### 4.1 环境与动作执行

MLIR RL 一次优化一个 Linalg 操作，并从消费者向生产者逆序遍历，以保留更多 tiled fusion 机会。动作掩码根据当前状态过滤非法动作；例如内层循环迭代次数超过 512 时屏蔽会导致完全展开和巨大代码的 MLIR vectorization。

### 4.2 状态编码

特征包含：Linalg 操作类型 one-hot；循环上界和 parallel/reduction 类型；向量化前置条件；由 affine indexing map 构造的访问矩阵；加、减、乘、除、exp 数量；以及 tiling/interchange 的逐步动作历史。策略只编码当前 producer 与 consumer，由 512 单元 LSTM 得到关系表示，再进入三层 512 单元 ReLU backbone（第 4–5 节）。

### 4.3 PPO 训练

作者使用 Proximal Policy Optimization（PPO，近端策略优化）训练 actor-critic。训练 10,000 步；每步收集 64 个代码样本的轨迹，mini-batch 32，执行 4 个 PPO epoch。学习率 0.001、clip range 0.2、折扣因子 `γ=1.0`、GAE `λ=0.95`、value loss 系数 0.5、entropy 系数 0.01。最大循环层数 12、候选 tile size 8、数组数 14、访问 rank 12、变换序列长度 5。训练在 16 个 CPU 节点上约耗时 5 天 7 小时（第 7.1 节）。

### 4.4 推理与编译

训练后的策略对每个操作生成变换序列；MLIR 19 应用变换并降至 LLVM IR/OpenMP/CPU。策略网络 CPU 推理平均每代码样本 0.028 秒；完整变换应用平均为单算子 0.089 秒、LQCD 应用 0.8 秒（第 7.2 节）。

## 5. 奖励函数、损失函数或关键公式

终止奖励为速度提升的对数：

```text
speedup = T_baseline / T_optimized
R_terminal = log(speedup)
R_t = 0，若 t 不是轨迹最后一步
```

使用对数是因为连续变换的加速比可以在对数域相加，更适合累计回报。作者也测试了每步执行代码并发放即时奖励的方案：最终平均性能与终止奖励相近，但训练更慢，因此采用终止奖励（第 4.3 节和图 7）。

PPO 的标准 clipped policy objective、value loss 与 entropy bonus 被用于训练，但论文没有在正文中重新给出完整 PPO 损失公式。奖励只衡量运行时间，没有代码大小、能耗、编译时间或正确性项；变换合法性依赖 MLIR 变换实现和动作掩码。

## 6. 实验设置

### 6.1 数据集来源

训练集共 3,959 个样本。深度学习单算子来自 TensorFlow Hub 和 Hugging Face 的 121 个模型：MatMul 187、Conv2D 278、MaxPool 250、Add 271、ReLU 149，共 1,135 个；另随机生成长度 5 的算子序列。LQCD 部分从未发表的领域编译器 7 个大型测试中抽取循环嵌套并变换输入尺寸，共 691 个样本。随机算子序列的独立条目数论文中未明确说明。

测试包含未在训练中出现的输入尺寸/形状：五类深度学习算子；ResNet-18、VGG、MobileNetV2 三个完整模型；以及三种 1,000–8,000 行 MLIR Linalg 的 LQCD 应用。论文没有给出严格的模型级去重审计，因此只能确认测试形状未见，不能据此断言不存在算子结构重合。

### 6.2 模型与工具

RL 策略为 LSTM + MLP 的 actor-critic，算法为 PPO。编译器为 LLVM/MLIR 19，模型转换使用 Torch-MLIR；PyTorch 2.8.0 使用 Intel oneDNN/MKL-DNN 3.7.1。硬件为双路 Intel Xeon E5-2680 v4（每路 14 核、2.40 GHz），总内存 64 GB；训练使用 16 个相同节点。

### 6.3 对比方法

- MLIR baseline：禁用 MLIR RL 循环级选择，但保留相同 lowering 和 LLVM `-O3`。
- PyTorch eager 与 `torch.jit.script` 的 PyTorch compiler。
- Halide RL：只用于单算子比较。
- Halide Mullapudi autoscheduler：用于 LQCD 比较。

### 6.4 评价指标

主指标是相对未做循环级优化的 MLIR baseline 的 speedup。MLIR 代码运行 5 次取中位数；PyTorch 先 warm-up 10 次，再测 11 次取中位数。还报告策略推理时间、变换应用时间和训练时间。

## 7. 实验结果与结论

### 7.1 单算子结果

相对同一 MLIR baseline，MLIR RL 在 Add 和 ReLU 上与 PyTorch/PyTorch compiler 竞争，在 MaxPool 上报告平均 3.3 倍更高的 speedup。MatMul 与 Conv2D 则分别平均比 PyTorch/PyTorch compiler 慢 2.16 倍和 6.71 倍，因为动作空间没有暴露 oneDNN 使用的寄存器分块、专用 kernel 和更激进向量化。相对 Halide RL，MLIR RL 在 MatMul 上平均获得 5.32 倍更高 speedup，但 MaxPool 平均落后 1.25 倍（第 7.3.1 节、图 5）。

### 7.2 完整模型结果

相对 MLIR baseline，MLIR RL 在 ResNet-18、MobileNetV2、VGG 上分别为 25.43、6.93、54.64；PyTorch compiler 分别为 411.26、28.23、328.77。因此 PyTorch compiler 分别比 MLIR RL 快 16.17、4.07、6.02 倍。主要瓶颈是 MatMul/Conv2D，且当前系统不能在完整模型中将 Conv2D 转换为 Img2col + GEMM（表 3）。

### 7.3 LQCD 结果

相对 MLIR baseline，MLIR RL / Halide autoscheduler 的 speedup 分别为：hexaquark-hexaquark 13.25 / 1.17；dibaryon-dibaryon 7.57 / 5.15；dibaryon-hexaquark 2.15 / 4.68。前两项 MLIR RL 更优，最后一项 Halide 更优；论文把最大相对优势概括为约 11 倍（表 4）。

### 7.4 消融实验

Level Pointers 的平均 speedup 为 18.7，受限枚举循环交换为 14.5。多离散动作空间前期收敛较慢，但最终探索到更高平均 speedup；论文图 6 未给出可独立抄录的最终精确数字。即时奖励与终止奖励最终表现相近，但即时奖励因每步执行程序显著增加训练时间（第 7.4 节）。

## 8. 主要创新点

### 8.1 面向 MLIR 参数化变换的原生 RL 环境

贡献不只是“在 MLIR 上用 RL”，而是把作用位置、变换类型和参数组织为可执行、可屏蔽的动作接口，并发布包含代码、预训练模型、约 700 MB 数据和复现实验脚本的工件。

### 8.2 多离散动作空间

将“变换 + 所有参数”的大平面动作拆成多个小分类分布，使策略能组合未被固定候选表穷举的参数，并在消融中获得更高最终 speedup。

### 8.3 Level Pointers 循环排列决策

循环交换不再从有限候选中一次选择，而是逐层构造排列；配合动作历史和掩码既覆盖完整排列空间，又避免显式 `N!` 输出头。18.7 对 14.5 的消融为该设计提供直接证据。

### 8.4 跨 DNN 与 LQCD 的同一低层环境

系统在 Linalg 级工作，展示了同一动作与状态接口可处理深度学习和物理计算循环，而非只调 DNN 图节点。

## 9. 局限性

### 9.1 论文明确承认的局限

动作空间设计约耗费四人年；完整训练需 16 个 CPU 节点 5 天 7 小时。当前只支持 Linalg，迁移到其他 dialect 仍需重做动作空间和特征抽取。MatMul/Conv2D 缺少专用 kernel、寄存器分块和 Img2col + GEMM，明显落后 PyTorch；LQCD DSL/compiler 仍未公开发表。

### 9.2 阅读后发现的潜在局限

奖励仅使用终端运行时间，较稀疏且不约束代码大小、编译成本或能耗。正确性主要依赖 MLIR 变换“构造即合法”或依赖分析，不是形式化等价证明。实验只有单一 x86-64 CPU 族，没有 GPU、RISC-V 或多硬件迁移证据。训练数据来自公开模型算子，但正文只保证测试形状未见，未做严格结构泄漏审计。结果对线程数、频率稳定性、库版本和 baseline 定义敏感；工件也预计相似硬件上约有 ±5% 波动。

## 10. 阅读后的研究方向反思

最值得借鉴的是“编译器动作接口的结构化”，不是 PPO 本身。对 LLVM/RISC-V 研究，简单把 CPU target 换成 RISC-V 不足以形成新贡献；真正的问题是把 RVV 的 VLEN/LMUL/SEW、`vsetvli` 状态、后端合法性和真机 PMU 反馈编码进动作、状态与多目标奖励，并研究策略跨微架构迁移。

本文适合作为 RL 编译环境和动作空间 baseline。其多离散动作、Level Pointers 和工件接口可复用；“MLIR 原生 RL 环境”本身已是论文核心贡献，后续不能只复制环境并替换 benchmark。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RVV 状态的分层动作空间

#### 研究问题

如何联合决定 Linalg 变换、LLVM/RVV 向量策略与 `vsetvli` 配置，并在不同 VLEN 的真机上稳定获益。

#### 与原论文的区别

原文只优化 Linalg 循环并在单一 x86 CPU 上评估；新问题把可伸缩向量状态和后端反馈作为显式决策对象。

#### 可能的创新点

IR—后端分层动作、非法原因掩码、跨 VLEN 鲁棒奖励、PMU 校准和少样本策略迁移。

#### 实验框架

```text
MLIR Linalg → 高层变换候选
→ LLVM/RVV 合法化与成本解释
→ QEMU 预筛 + 多块 RVV 真机 PMU
→ 分层策略更新 → 跨 VLEN 评估
```

#### 可行性与主要风险

可先限于 MatMul/归约 20–50 个 kernel；风险是动作空间再次膨胀、真机测量噪声和浮点重排语义。

### 11.2 低成本离线编译反馈学习

#### 研究问题

能否用历史编译/运行轨迹训练离线策略或世界模型，减少 16 节点在线 PPO 成本，同时保持对未见程序的安全改进。

#### 与原论文的区别

原文每条训练轨迹需要真实编译执行；新方案强调离线数据、置信度和在线少量校准。

#### 可能的创新点

反事实 pass 轨迹、保守策略选择、不确定性门控、按编译器错误类型主动补样。

#### 实验框架

```text
历史 IR/动作/运行时间 → 离线数据清洗
→ 训练策略/代价模型 → 不确定性筛选
→ 少量真机执行校准 → 与在线 PPO 比较
```

#### 可行性与主要风险

可复用 MLIR RL 工件；主要风险是历史策略覆盖不足和离线外推导致错误选择。

## 12. 与其他已读文献的关系

与 CompilerGym/AutoPhase 类工作相比，本文不只选固定 pass，而是同时选择变换参数和作用循环。与现有 C05 CITROEN、C10 Protean 等 phase-ordering 条目相比，它更接近循环级 autotuning 环境。与 C21 OML-vect 的规则化 MLIR→RVV 向量化不同，MLIR RL 学习变换序列但没有 RISC-V 实验；二者可组合为“高层策略选择 + RVV 后端真实性能反馈”。与 C23 的多层 RISC-V 后端相比，本文依赖通用 LLVM CPU lowering，而 C23 通过保留目标语义获得专用后端性能，揭示了只优化高层 Linalg 仍可能被后端信息瓶颈限制。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 构建 MLIR 原生的强化学习自动循环优化环境 |
| 核心问题 | 参数化动作空间巨大、训练反馈昂贵、现有 LLVM pass 环境不适配 MLIR |
| 输入 | MLIR Linalg 操作序列及 producer-consumer 特征 |
| 输出 | tiling、并行、融合、交换、向量化等变换序列 |
| 核心方法 | 多离散动作空间、Level Pointers、动作掩码、终端执行时间奖励 |
| 使用的模型 | 512 单元 LSTM + 三层 512 MLP actor-critic，PPO |
| 使用的编译器工具 | LLVM/MLIR 19、Torch-MLIR、OpenMP、Clang |
| 是否使用强化学习 | 是，PPO |
| 是否使用形式化验证 | 否；依赖 MLIR 合法性/变换实现 |
| 数据集规模 | 3,959 个训练样本；其中单算子 1,135、LQCD 691 |
| 主要指标 | 相对 MLIR baseline 的 speedup、推理/变换/训练时间 |
| 最重要实验结果 | Level Pointers 18.7 对枚举 14.5；两项 LQCD 明显胜 Halide，但完整 DNN 显著落后 PyTorch compiler |
| 核心创新 | 将 MLIR 结构化参数动作变成可训练、可复用的 RL 环境 |
| 主要局限 | 训练昂贵、Linalg/x86 单目标、专用 kernel 与跨硬件能力不足 |
| 与 RISC-V 研究的相关性 | 中；方法可迁移，但论文没有 RISC-V/RVV 实验 |
| 最适合作为 | RL 编译环境、动作空间与实验基线 |

> 这篇论文最值得学习的是把 MLIR 参数化变换拆成可组合动作并用消融验证设计；最主要的局限是训练成本和通用 CPU 后端能力不足。后续最合理的使用方式是把它作为环境与动作空间 baseline，再加入 RVV 状态、后端反馈和跨硬件迁移，而不是只把 target triple 改成 RISC-V。
