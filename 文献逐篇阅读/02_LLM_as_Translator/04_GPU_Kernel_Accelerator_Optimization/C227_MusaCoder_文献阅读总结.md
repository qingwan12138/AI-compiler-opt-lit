# MusaCoder 文献阅读总结

论文题目：**MusaCoder: Native GPU Kernel Generation with Full-Stack Training on Moore Threads GPU**

作者：Kun Cheng、Songshuo Lu、Sicong Liao、Tankun Li、Yafei Zhang、Dong Yang、Qiheng Lv、Hua Wang、Zhi Chen、Yaohua Tang

发表时间：2026 年

发表平台：arXiv 预印本（2026）

论文链接或编号：arXiv:2606.04847
元数据核验来源：[arXiv（正式标题）](https://arxiv.org/abs/2606.04847)；[MooreThreads 官方模型工件](https://huggingface.co/MooreThreads/MusaCoder-27B)
代码/数据/工件：模型权重工件：[MusaCoder-27B](https://huggingface.co/MooreThreads/MusaCoder-27B)；源代码仓库待核验

关键词：GPU kernel generation、CUDA、MUSA、LLM、SFT、RFT、GRPO、execution feedback、MooreEval、KernelBench

代码元数据：`Code_Status=NOT_FOUND`；`Code_URL` 为空；`Code_Checked_At=2026-10-01`。本次检索未定位作者或机构官方代码仓库，未以第三方汇总页作为代码证据。

> 本文档基于下载的 arXiv 官方 PDF（55 页，文件头 `%PDF-`，正文可提取；抽查第 24 页表 1 渲染正常）。论文事实与阅读后的研究思考分开描述。

## 1. 研究背景

现代深度学习系统依赖高性能 GPU kernel 执行矩阵运算、归约、注意力和融合算子。cuBLAS、cuDNN、CUTLASS 等库适合常见算子，但面对快速变化的模型结构、长尾算子和新融合模式时，人工编写原生 kernel 的成本很高。论文第 1 节将问题限定为：从高层 PyTorch 张量语义直接生成可编译、数值正确并且具有实际运行收益的低层 CUDA/MUSA kernel。

通用代码模型在 GPU kernel 生成上遇到三类困难：线程和多维索引语义较复杂；输出需要满足编译、运行时安全、数值正确和硬件约束；从零生成的失败比例高，使执行反馈强化学习出现稀疏奖励。论文还指出模型可能通过调用被禁止的 `torch.matmul` 或 `aten::*` 高层实现获得表面正确性，形成奖励投机。

MUSA 是 Moore Threads 的 GPU 软件栈。论文选择 CUDA 与 MUSA 两个后端，试图验证一套包含数据合成、监督微调、执行验证和强化学习的完整流程能否服务于新兴加速器，而不是只在现成 CUDA 代理系统上做推理期搜索。

## 2. 论文要解决的问题

### 2.1 从高层语义生成原生 kernel

模型需要把 PyTorch reference 转为自定义 CUDA 或 MUSA kernel，避免把计算转回高层库。输出必须可以编译、通过随机输入正确性检查，并符合禁止 fallback 的约束。

### 2.2 让执行反馈强化学习获得稳定信号

论文关注全失败 rollout 组、奖励投机和异步多轮训练中的 off-policy drift。它要解决的问题是如何利用编译、运行时、数值和性能反馈训练模型，同时保持首轮生成质量。

### 2.3 覆盖新兴后端

除了 CUDA，论文还构建 MUSA 版 KernelBench 评测流程，研究同一训练管线能否生成具有实际运行收益的 MUSA kernel。

> 本文主要研究：如何通过渐进式数据构造、正确性优先的执行验证和分阶段 SFT/RFT/RL，使语言模型直接生成 CUDA/MUSA 原生 GPU kernel。

## 3. 核心方法概述

MusaCoder 是一个端到端 kernel 生成训练框架。模型的最终输出是 `ModelNew` 中的自定义 CUDA/MUSA 扩展和 kernel source；MooreEval 负责编译、随机正确性验证、fallback 检查、同步计时和反馈生成。

```text
PyTorch reference + 任务约束
        ↓
三阶段 kernel 数据合成：任务扩展、形状/索引推理、反馈轨迹
        ↓
多任务 SFT → 多样性保持 RFT
        ↓
单轮 GRPO 执行强化学习
        ↓
多轮反馈 RL：模型生成 kernel → MooreEval 编译/运行/检查
        ↓                         ↑
结构化错误、正确性、合法性、性能反馈 ──┘
        ↓
CUDA 或 MUSA 原生 kernel
```

LLM 直接产生和修订 kernel source，属于 `TRANSLATOR/T4_GPU_Kernel_Accelerator_Optimization`。MooreEval、GRPO、BDR 和 MirrorPop 是训练与验收机制，不是最终输出角色。

## 4. 实验框架与训练流程

### 4.1 三阶段数据合成

第 3 节和图 3 将 SFT 数据构造分为三阶段。

1. 任务扩展与基础正确性：收集开源 PyTorch 模块、GitHub 中的真实模块、NNSmith 生成图、基本算子变体、GPU kernel 知识问答和自动单元测试；清洗外部状态、不稳定控制流和难以验证的样本。
2. 结构化推理与空间逻辑：在 prompt 中加入 tensor shape、stride、dtype 等元数据，并要求模型按照语义分解、标量数学、形状计算、索引映射、边界处理和错误预分析六步推理。
3. 多轮 RL 准备：加入编译错误、运行时失败、正确性不匹配、profiling、优化改写和多轮修复轨迹，让模型学习解释执行反馈。

### 4.2 多任务 SFT 与多样性保持 RFT

多任务 SFT 学习基础 kernel 生成、知识问答、错误诊断和反馈解释。随后进行 diversity-preserving rejection sampling fine-tuning（RFT，拒绝采样微调）：执行 sandbox 过滤不能编译或不能通过验证的样本，但不只保留单个最快实现，而是保留多个异质且已验证正确的实现，以避免过早熵塌缩并维持后续 RL 的探索多样性。

### 4.3 MooreEval 执行环境

MooreEval 将 CPU 编译和 GPU 执行拆成两个阶段。它记录编译状态、stderr、运行时异常、数值正确性、禁止 fallback、基线时间、候选时间和 speedup。只有通过编译、随机正确性和合法性检查的候选才进入性能测试。

错误类别包括 `compile error`、`runtime error`、`correctness error`、`cheating` 和 `infra`。静态语法检查与运行时 hook 用于捕获禁止的 `aten::*`/高层 fallback，以及异常 speedup。

### 4.4 单轮 RL 与多轮反馈 RL

单轮 RL 使用 MooreEval 对首轮候选打分，目标是提高无反馈时直接生成合法、正确和高性能 kernel 的能力。论文使用 Group Relative Policy Optimization（GRPO，组相对策略优化），每个任务采样一组候选，以组内归一化奖励计算 advantage。

多轮 RL 在失败时把错误摘要追加到上下文，模型据此修复编译、形状、dtype、数值或性能问题，最多 3 个模型响应轮次；通过后提前结束。论文默认 `PrimeEcho` 的 `alpha=0.75`，策略梯度只对第一轮响应计算，后续轮次主要提供轨迹评价和奖励信号。

### 4.5 稳定化机制

- **PrimeEcho**：把第一轮分数与整条轨迹最高分结合，并给前两轮成功额外奖励，防止模型过度依赖后续反馈。
- **Buffered Dynamic Retry（BDR）**：将全失败 prompt、失败代码和 MooreEval 错误放入有限 FIFO buffer，以较小概率作为反馈修复任务重新训练。
- **MirrorPop**：用绝对 log importance ratio 聚合整条响应，检测正负 token 偏差抵消造成的高风险 off-policy 样本，并对其做序列级屏蔽。

## 5. 奖励函数、损失函数或关键公式

### 5.1 MooreEval 分段奖励

第 4.3.2 节给出的候选代码奖励为：

```text
s(c) = -1                                      # 提取、编译或运行失败
s(c) = -1                                      # 检测到禁止 fallback
s(c) = -1                                      # correctness q = 0
s(c) = -0.5 + 0.5q                             # 0 < q < 1
s(c) = 1 + λ * min(max(ν - 1, 0), νmax)        # q = 1 且实现合法
```

其中 `q` 是随机测试中的 correctness rate，`ν` 是相对 PyTorch baseline 的运行 speedup，`lambda` 控制性能奖励权重，`νmax` 截断异常大的 speedup。该设计让部分正确实现获得有限 shaping signal，但只有完整正确且合法的 native kernel 才能得到正的基础奖励和性能 bonus。

### 5.2 GRPO

组内 advantage 为：

```text
A_i = (r_i - mean(r_group)) / (std(r_group) + epsilon)
```

策略目标使用 clipped importance ratio 和 KL 正则。论文第 4.4.1 节给出 GRPO 目标及上下裁剪边界 `1-epsilon_low`、`1+epsilon_high`。

### 5.3 PrimeEcho

```text
R_tau = alpha * s_first + (1-alpha) * max_k(s_k) + b_early
```

`s_first` 是第一轮分数，`max_k(s_k)` 是整条轨迹最高分，`b_early` 奖励第一轮或第二轮成功，且第一轮奖励更高。论文用这一目标平衡首轮质量和多轮修复探索。

### 5.4 MirrorPop

```text
m_i,t = max(rho_i,t, 1 / rho_i,t) = exp(abs(log rho_i,t))
M_i = 1[ mean_t(abs(log rho_i,t)) <= delta ]
```

与 signed log-ratio 不同，MirrorPop 不会让大于 1 和小于 1 的偏差相互抵消，因此可以屏蔽整条响应中 policy drift 过大的样本。

## 6. 实验设置

### 6.1 数据集来源

论文使用 KernelBench 以及作者构建的 MUSA 迁移版。MUSA 版保留原 benchmark 的任务结构和难度层级，将 `.cuda()`、CUDA kernel 接口和编译命令迁移为 MUSA 版本，并用 Musify 转换 CUDA API 和 launch 接口；转换后的示例还要经过编译与正确性验证。

训练数据来自开源任务、GitHub 模块、NNSmith 计算图、算子变体、GPU kernel 知识问答、shape/stride 增强、profiling 分析、优化改写和 MooreEval 验证样本。论文正文没有给出完整 SFT 语料的统一总样本数、明确 train/validation/test 数量或去污染报告，因此这些信息不能补写。

### 6.2 模型与工具

| 项目 | 论文设置 |
|---|---|
| 基础模型 | MusaCoder-9B 初始化自 Qwen3.5-9B；MusaCoder-27B 初始化自 Qwen3.6-27B |
| SFT | AdamW，学习率 `1e-5`，warmup 3%，weight decay 0.01，bf16，最大序列长度 40K，全局 batch 256，1 epoch |
| RL | GRPO；学习率 `1e-6`，warmup 0.1，weight decay 0.1，gradient clipping 0.5；rollout group 8，batch 64 |
| 推理/训练框架 | DeepSpeed（SFT）；Megatron + SGLang（RL 与异步 rollout） |
| 编译与执行 | MooreEval；CUDA/MUSA 编译、随机正确性测试、同步 CUDA event 计时 |
| 硬件 | 64 台 Moore Threads MTT S5000，每台 8 张 80GB accelerator card |
| 目标输出 | 原生 CUDA/MUSA kernel 与 `ModelNew` 扩展 |

论文未报告 LLVM、GCC、Alive2 或形式化等价验证器的使用。

### 6.3 对比方法

KernelBench 主实验比较 Claude Opus 4.7、GLM-5.1、Kimi K2.6、DeepSeek-V4-Pro、DeepSeek-V4-ProMax，以及基础 Qwen3.5-9B、Qwen3.6-27B。所有模型使用相同 prompt、采样配置、验证脚本和硬件。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| Pass@8 | 8 个样本中至少一个通过 MooreEval | 越大越好 |
| Avg.@8 | 8 个样本中通过验证的平均比例 | 越大越好 |
| Faster Rate | 先正确且合法，再相对 baseline 达到大于 1.1x speedup 的候选比例 | 越大越好 |
| First-turn accuracy | 无反馈首轮通过比例 | 越大越好 |
| Best-turn accuracy | 最多 T 轮内至少一次通过比例 | 越大越好 |

MUSA 实验只报告相对 PyTorch eager 的 Faster Rate，因为论文说明当前 MUSA 平台尚未完成 `torch.compile` benchmark。

## 7. 实验结果与结论

### 7.1 CUDA KernelBench 主要结果

根据表 1，Overall 结果如下：

- Qwen3.5-9B 基础模型 Pass@8 为 23.6%、Avg.@8 为 7.05%；Qwen3.6-27B 为 67.2% 和 35.60%。
- MusaCoder-9B-RL 达到 Pass@8 83.6%、Avg.@8 77.20%；MusaCoder-27B-RL 达到 93.2% 和 88.60%。
- Claude Opus 4.7 为 87.2% 和 77.30%，因此 MusaCoder-27B-RL 相比该 baseline 高 6.0 和 11.30 个百分点。
- Level 3 上，MusaCoder-27B-RL 的 Pass@8/Avg.@8 为 72%/65.75%，Claude Opus 4.7 为 54%/39.25%。

Faster Rate 也要求正确、合法且相对 baseline 超过 1.1x：MusaCoder-27B-RL 相对 PyTorch eager/compile 为 15.0%/9.2%，Claude Opus 4.7 为 11.8%/7.5%。这些是候选比例，不是所有 kernel 的平均 speedup。

### 7.2 MUSA KernelBench 结果

表 4 显示 MusaCoder-27B-RL Overall Pass@8/Avg.@8/Faster 为 92.4%/81.7%/12.5%；MusaCoder-9B-RL 为 84.4%/72.3%/6.4%。DeepSeek-V4-Pro 的对应值为 92.0%/56.9%/5.7%，说明 27B RL 模型的至少一次成功率相近，但 8 次采样平均正确率和通过 1.1x 加速门槛的比例更高。Level 3 上，MusaCoder-27B-RL 的 Avg.@8/Faster 为 54.1%/3.3%，两个外部 baseline 的 Faster 为 0.0%。

### 7.3 组件消融

表 2 的 Overall 结果显示：

- 去掉 RFT：Pass@8 从 84.8% 降到 82.6%，Avg.@8 从 79.40% 降到 75.10%。
- 去掉单轮 RL warmup：Pass@8 从 93.2% 降到 90.8%，Avg.@8 从 88.60% 降到 84.25%。
- 去掉 PrimeEcho：Pass@8/Avg.@8 降到 88.4%/83.50%。
- 去掉 BDR：降到 88.6%/83.20%。
- 去掉 MirrorPop：降到 86.0%/80.75%，是表 2 中影响最大的稳定化组件；相对 eager 的 Faster Rate 也从 15.0% 降至 13.1%。

表 3 的短程 BDR 训练中，Qwen3-8B Pass@8 从 59.6% 升至 62.4%，Qwen3.5-9B 从 73.2% 升至 74.4%；论文将其解释为恢复原本全失败样本的学习信号。

### 7.4 训练流程结论

论文第 5.4 节观察到 Level 1 更容易首轮通过，Level 2/3 更依赖多轮反馈。多轮训练过程中平均交互轮数下降、奖励上升，说明模型逐渐减少对延迟修复的依赖。该结论基于论文自己的 KernelBench 验证，不等于在其他硬件或开放任务上已得到保证。

## 8. 主要创新点

### 8.1 创新点一：面向 kernel 能力的渐进式数据管线

论文把 PyTorch-to-CUDA/MUSA 对、shape/stride 信息、六步推理、错误诊断、profiling 和修复轨迹放在同一数据构造流程中，为后续执行训练提供初始化。价值在于把“会写代码”拆成语义映射、空间索引、边界处理和反馈理解等能力；表 1 与 RFT 消融支持该管线对正确性稳定性的贡献。

### 8.2 创新点二：正确性优先的 MooreEval 奖励闭环

MooreEval 先判断提取/编译/运行/数值正确性和 fallback 合规，再计算性能奖励，并用结构化 telemetry 生成多轮错误反馈。该层级设计把速度奖励限制在合法且正确的 native kernel 上，针对 GPU kernel 生成中的奖励投机问题给出具体机制。

### 8.3 创新点三：首轮锚定与 off-policy 稳定化

PrimeEcho 将首轮质量放在轨迹奖励中，BDR 处理全失败组，MirrorPop 用绝对 log-ratio 检测序列级偏移。消融结果表明这些机制对最终 Pass@8、Avg.@8 和 Faster Rate 均有影响，尤其是 MirrorPop。

### 8.4 创新点四：CUDA/MUSA 统一的原生 kernel 生成训练

论文将 MUSA 迁移版 benchmark 接入同一数据、验证和 RL 管线，展示了从 CUDA 到 MUSA 的后端扩展路径。该贡献的范围是论文评估的 MUSA 任务，不能泛化为所有非 CUDA 加速器均可直接迁移。

## 9. 局限性

### 9.1 论文明确或正文可见的限制

- 论文以 KernelBench 及其 MUSA 迁移版为主要评测，任务分布与真实生产 workload 的差异需要进一步验证。
- MUSA 实验没有 `torch.compile` 对照，Faster Rate 只相对 eager 报告。
- 论文没有给出完整训练语料的统一规模、标准 train/validation/test 数量或数据去污染审计。
- 多轮反馈、编译和真实硬件执行的成本较高；论文强调首轮训练以降低部署时的反馈依赖，但没有给出完整推理成本表。
- 论文采用执行测试和 anti-hacking 检查，而不是形式化语义等价证明；有限随机测试不能证明所有输入和所有形状正确。

### 9.2 阅读后的潜在限制

- MUSA 和 CUDA API、编译器诊断及计时语义不同，统一 prompt 或 reward schema 不能自动保证跨 ISA 泛化。
- Faster Rate 是超过 1.1x 的候选比例，不是一个候选在全部输入上的稳定加速保证。
- 训练依赖 64 台 MTT S5000 集群，复现门槛高；论文没有报告缩小硬件规模后的性能退化。
- `MirrorPop`、`PrimeEcho` 等 RL 稳定化效果可能依赖 rollout 长度、模型规模和 off-policy 程度，需要跨任务验证。

## 10. 阅读后的研究方向反思

MusaCoder 最适合作为 Translator/T4 的训练型 baseline。它的清晰接口是“PyTorch reference → 原生 kernel source”，而不是“LLM 选择某个已有编译器 pass”。对 RISC-V 的直接相关性为中等：数据合成、正确性优先奖励和反馈分类可以借鉴，但 CUDA/MUSA 的线程层级、内存模型和工具链不能直接替换为 RVV 或自研加速器后宣称完成迁移。

值得借鉴的是把编译、运行、正确性、合法性和性能指标分层记录，并把具体诊断反馈回传给模型。不能直接照搬的是 MooreEval 的禁止 operator 列表、CUDA/MUSA 编译命令和 1.1x 速度阈值；在 RVV 场景需重新定义 intrinsic 合法性、向量长度、尾部处理、对齐约束与真实硬件计时。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RVV 的原生 kernel 生成与验证闭环

#### 研究问题

语言模型能否从高层 tensor/PyTorch 或 MLIR 语义生成满足 RVV intrinsic、VL/VTYPE 和内存对齐约束的 kernel。

#### 与原论文的区别

目标不是把 MUSA API 替换成 RVV API，而是把向量长度变化、尾处理和 LLVM/RVV 后端诊断纳入输出契约。

#### 可能的创新点

设计 RVV 特有的 legality checker、向量长度分层 reward 和跨实现的语义测试集。

#### 实验框架

```text
MLIR/tensor reference → LLM 生成 RVV C/intrinsic 或 LLVM IR
        ↓
LLVM/RVV 编译 + 模拟器/真实板卡执行
        ↓
语义、VL、对齐、性能反馈 → 继续生成
```

#### 可行性

需要 LLVM/RISCV 后端、Spike 或 QEMU、RVV 硬件和可复用 tensor kernel 任务。

#### 主要风险

模拟器与真实板卡性能排序可能不一致；随机测试仍不能替代形式化或大范围输入验证。

### 11.2 编译器诊断驱动的跨后端 kernel 修复

#### 研究问题

如何将 CUDA、MUSA 和 RVV 后端不同格式的诊断归一化为可供模型修复的结构化反馈。

#### 与原论文的区别

MusaCoder 在两个 GPU 后端上共享训练管线；该方向关注诊断 schema 的跨后端迁移和错误类别对齐。

#### 可能的创新点

建立错误类型、源代码位置、硬件约束和修复动作之间的可验证映射，并评估反馈是否降低跨后端迁移成本。

#### 实验框架

```text
同一高层任务 → 多后端生成候选 → 后端编译/运行
        ↓
统一诊断 schema → LLM 修复 → 跨后端比较
```

#### 可行性

可以复用 MooreEval 的分层状态思想，但需要各后端 compiler log、运行时检查和统一任务接口。

#### 主要风险

错误日志中的硬件细节可能无法统一；过强的统一 schema 可能丢失后端特有信息。

### 11.3 将性能奖励拆成可解释的硬件事件目标

#### 研究问题

在正确性通过后，如何让模型学习内存带宽、向量利用率、cache 行为和分支代价，而不只追逐单一 wall-clock speedup。

#### 与原论文的区别

MusaCoder 的性能项主要是相对 baseline 的 speedup bonus；该方向把硬件计数器和代码变换类型纳入可解释目标。

#### 可能的创新点

研究多目标 reward 与跨输入稳定性的关系，并检测模型是否通过计时噪声获得虚假奖励。

#### 实验框架

```text
合法 kernel → 真实硬件计时与 counters → 多目标 reward
        ↓
LLM 改写 → 正确性/稳定性/性能联合验证
```

#### 可行性

需要稳定的 profiler、固定输入协议和足够多的重复测量；RVV 上需先确认可用性能计数器。

#### 主要风险

计数器跨硬件不可比，训练成本和噪声都可能高于单一 speedup reward。

## 12. 与其他已读文献的关系

- 与仓库中的 C187 TritorX、C130 AccelOpt 和 C142 Xe-Forge 一样，MusaCoder 的最终产物是可执行 accelerator kernel，因此应归入 Translator/T4；区别是 MusaCoder 把能力主要内化到模型权重，通过 SFT/RFT/RL 训练，而不是主要依赖冻结模型的推理期 agent 搜索。
- 与 C199 CUDA Agent、C200 DRTriton 相近，均使用执行反馈生成 GPU kernel；MusaCoder 的独立点是 CUDA/MUSA 双后端、RFT、PrimeEcho、BDR 和 MirrorPop 的完整训练链路。
- 与 C212 AscendKernelGen 相似之处是面向非 CUDA 加速器直接生成 kernel；MusaCoder 的目标后端是 MUSA，并且正文把 CUDA/MUSA 统一纳入 post-training 与执行验证。
- KernelBench 是本文的评测基础设施，不应作为本论文的同工作版本；本文主线是模型训练与原生 kernel 输出。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLM 直接生成 CUDA/MUSA 原生 GPU kernel |
| 核心问题 | 低初始成功率、执行奖励稀疏、奖励投机和多轮 off-policy 不稳定 |
| 输入 | PyTorch reference、shape/stride 元数据、任务约束和执行反馈 |
| 输出 | `ModelNew` 中的 CUDA/MUSA kernel source |
| 核心方法 | 三阶段数据合成、SFT、diversity-preserving RFT、单轮/多轮 GRPO、MooreEval |
| 使用的模型 | MusaCoder-9B/27B；初始化自 Qwen3.5-9B/Qwen3.6-27B |
| 使用的编译器工具 | MooreEval、CUDA/MUSA 编译与执行、同步 CUDA event 计时、SGLang/Megatron/DeepSpeed |
| 是否使用强化学习 | 是；单轮与多轮 GRPO |
| 是否使用形式化验证 | 否；使用编译、随机正确性、合法性和性能检查 |
| 数据集规模 | KernelBench 与 MUSA 迁移版；训练语料总规模论文未统一报告 |
| 主要指标 | Pass@8、Avg.@8、Faster Rate、首轮/最佳轮准确率 |
| 最重要实验结果 | MusaCoder-27B-RL 在 CUDA Overall 达到 93.2% Pass@8、88.60% Avg.@8、15.0%/9.2% Faster Rate；MUSA 达到 92.4%/81.7%/12.5% |
| 核心创新 | 正确性优先 MooreEval、首轮锚定 PrimeEcho、BDR、MirrorPop 与 CUDA/MUSA 统一训练 |
| 主要局限 | 主要依赖 KernelBench；缺少形式化证明、完整数据规模和 MUSA `torch.compile` 对照 |
| 与 RISC-V 研究的相关性 | 中；反馈与验证思想可迁移，CUDA/MUSA 具体接口和阈值不可直接照搬 |
| 最适合作为 | Translator/T4 训练型 baseline、执行反馈奖励设计参考 |

这篇论文最值得学习的是把高层语义到原生 kernel 的生成、编译、正确性、合法性和性能反馈放入一个可训练闭环；最主要的局限是评测和工具链仍集中在 KernelBench、CUDA/MUSA 生态，且没有形式化等价保证；如果用于后续研究，最合理的使用方式是作为执行反馈 kernel 生成 baseline 和 reward 设计参考，而不是简单地把 CUDA/MUSA 后端名称替换成 RISC-V 就视为完成跨架构研究。
