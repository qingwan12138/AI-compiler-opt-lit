# C38 LiteCoOp 文献阅读总结

论文题目：**LiteCoOp: Lightweight Multi-LLM Shared-Tree Reasoning for Model-Serving Compiler Optimizations**
作者：Annabelle Sujun Tang、Christopher Priebe、Lianhui Qin、Hadi Esmaeilzadeh
发表时间：2026（PDF：arXiv:2602.01935v2，2026-05-21）
发表平台：arXiv 预印本
论文链接或编号：[arXiv:2602.01935](https://arxiv.org/abs/2602.01935)；作者公开代码仓库未确认。
元数据核验来源：[论文/来源](https://arxiv.org/abs/2602.01935)；[论文/来源](https://arxiv.org/abs/2602.01935v2)
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）
关键词：多模型协作、MCTS、TVM、张量编译、成本感知搜索

> 阅读标记：gpt-5.6-luna，2026-09-05；依据 PDF v2（32 页）。本文事实与阅读分析分开。

## 1. 研究背景

模型服务中的调度、融合、布局和张量切分会显著影响目标硬件延迟，但变换序列组合爆炸、硬件相关且存在长程相互作用。规则启发式和随机搜索缺少程序级推理；单一大模型引导搜索又带来高 API 与编译成本。本文研究的是 TVM/MetaSchedule 中面向神经网络算子的调度搜索，不是 LLVM Pass 生成，也不是源码级重写。

## 2. 论文要解决的问题

现有方法通常让一个大 LLM 贯穿整个搜索，小模型单独使用时能力和可靠性不足。核心问题是：异构 LLM 能否在不引入外部多智能体协调开销的情况下，共同搜索语义保持的编译变换，并以低于单一大模型的成本获得更好目标性能？

## 3. 核心方法概述

LiteCoOp 把“程序状态+当前 LLM”作为 MCTS 节点，把“编译变换+下一个 LLM”作为联合动作。当前模型提出变换序列并选择下一个模型；所有模型共享一棵树，回传的代价模型奖励跨模型传播。LA-UCT 在性能奖励之外偏好小模型；连续两次小模型回归时，剪掉最近节点并调用最大模型进行 course alteration（纠偏）。

```text
初始 IRModule → MCTS选择(程序,模型) → LLM提出变换及下一模型
 → TVM应用变换 → 随机短 rollout → XGBoost代价模型估计
 → 奖励回传共享树 → 必要时大模型纠偏 → 输出最佳schedule
```

## 4. 实验框架与训练流程

本文不涉及预训练、SFT、PPO/GRPO 或强化学习训练；是推理阶段的 MCTS 搜索框架。每次迭代进行 selection、expansion、rollout、backpropagation。rollout 使用随机变换，奖励来自 TVM 未修改的硬件无关 XGBoost cost model；最终性能另在真实 GPU/CPU 上测量。模型通过 OpenAI 与 Nscale API 串行调用，避免并发协调成本。

## 5. 奖励函数、损失函数或关键公式

编译目标为 `max f(p_T)`，其中 `p_{t+1}=o_t(p_t)`，`f` 是目标硬件性能指标。联合状态为 `S=P×LLMSet`，联合动作为 `<o_t,llm_{t+1}>`。LA-UCT（PDF 第4页）为：

```text
LA-UCT = (1-λ) W/N + λ φ_small(llm) + c sqrt(ln N(parent)/N(child))
φ_small = [log θ(llm_max)-log θ(llm)] /
           [log θ(llm_max)-log θ(llm_min)+ε]
```

λ=0 时为奖励导向 UCT；λ 越大越偏好参数更少的模型。附录 A 证明在固定父节点下等价于对变换奖励 `(1-λ)R+λφ_small` 使用 UCB1，并渐近集中到该 surrogate 均值最佳的子节点。不存在强化学习奖励函数；奖励是 cost model 的预测，不应误写成真实硬件时间。

## 6. 实验设置

### 6.1 数据集来源

五个生产神经网络计算 kernel：Llama-3-8B attention、DeepSeek-R1 MoE、FLUX attention、FLUX convolution、Llama-4-Scout MLP；另有端到端 Llama-3-8B。论文未报告传统训练集/验证集划分，搜索对象是对应 TVM IRModule。

### 6.2 模型与工具

目标硬件是 NVIDIA 2080 Ti GPU 与 Intel Core i9 CPU；Apache TVM v0.20.0、MetaSchedule、XGBoost cost model。2 模型配置为 GPT-5.2+gpt-5-mini；4 模型增加 DeepSeek-R1-Distill-Qwen-32B、Llama-3.1-8B-Instruct；8 模型再加入 DeepSeek-R1-Distill-Qwen-7B、Qwen3-8B、Qwen3-14B、Devstral-Small-2505。λ=0.5、c=√2、branching factor=2；每项实验重复 10 次并报告均值。

### 6.3 对比方法

单一 GPT-5.2、单一 gpt-5-mini；另以 Llama-3.3-70B-Instruct 替代最大模型做稳健性比较。随机/round-robin 模型选择、无 course alteration 和每一次回归即纠偏是附录消融基线。

### 6.4 评价指标

Speedup=原始未优化 IRModule 延迟/最终 schedule 延迟，越大越好；延迟在目标硬件直接测量。还报告总编译时间、API cost reduction、LLM 调用比例；每个数字均按 GPU/CPU 或五 benchmark 平均口径给出。

## 7. 实验结果与结论

### 7.1 主要结果

8-LLM（最大 GPT-5.2）在五个 benchmark 的最终平均 speedup 为 GPU 30.1×、CPU 10.9×；相对单一 GPT-5.2，GPU/CPU 编译时间减少 1.95×/1.74×，API 成本减少 4.47×/4.32×，最大模型总调用比例仅 23.1%/23.9%（摘要与第1、7页）。聚合十个 benchmark-硬件组合时，编译时间减少 1.84×、API 成本减少 4.39×。

### 7.2 与传统方法的比较

论文没有与传统人工/规则 Pass 单独对比；比较重点是单一 LLM 与异构协作。TVM cost model 是搜索反馈，不等于传统编译器 pass 的真实性能保证。

### 7.3 与其他 LLM 方法的比较

8 模型均优于单一 GPT-5.2 的五个 GPU benchmark。GPU 最终平均 speedup 随 2/4/8 模型为 29.3×/29.6×/30.1×；CPU 8 模型平均 10.9×，相对单大模型几何平均提高 14.4%。换最大模型为 Llama-3.3-70B-Instruct 时，8 模型在每个 benchmark 仍最好，1000 samples 相对该基线提高 4.8%–27.5%。

### 7.4 消融实验

λ=0.5 在五 benchmark、1000 samples 的算术平均 speedup 为 10.86×，优于 λ=0、0.25、0.75、1.0。course alteration 每两次小模型回归是成本-质量折中：相对每一次回归，最大模型纠偏率下降，编译时间平均减少 1.38×、API 成本 1.59×，最终 speedup 平均差距仅 1.3%。共享树的状态相关选择在所有 benchmark 优于随机和轮询；相对两者 sample efficiency 平均分别提高 1.37×、1.42×。

### 7.5 案例分析

附录提示展示了根据当前/父/祖父程序、变换历史、预测分数和模型命中率产生 JSON 变换序列及下一模型；纠偏提示会提供失败的小模型提议和预测回退分数。附录 E 对 log speedup ratio 做单侧 matched-block 检验并用 Dunnett 校正；所有配置报告显著性，但这不代表跨硬件泛化证明。

## 8. 主要创新点

### 8.1 共享搜索树即协调机制

模型选择内生于编译搜索状态，避免外部 router、共享记忆或并发 agent controller；这是核心机制，不能简化为“多模型投票”。

### 8.2 LA-UCT 成本感知搜索

以模型参数规模的对数归一化偏好调节树搜索，在性能足够接近时减少大模型调用，并有附录理论分析支撑。

### 8.3 面向回归的稀疏大模型纠偏

连续小模型回归触发剪枝和最大模型的短提示纠偏，在保持大模型能力的同时减少常规调用。

## 9. 局限性

论文明确范围是 TVM 张量程序和五个 kernel/一个整模，依赖特定 TVM cost model、模型 API 与硬件；候选变换需来自给定列表，且未提供源码项目、LLVM IR 或 RISC-V 后端实验。潜在局限是 cost model 与真实执行存在偏差、API 价格随服务商变化、模型输出和搜索结果受提示与采样影响；“减少编译时间”不能外推为所有编译任务。

## 10. 阅读后的研究方向反思

可借鉴的是把模型路由与优化状态统一建模、记录模型可靠性并让结果跨模型回传。其核心贡献不是“换成 RISC-V”即可复现：RISC-V 需要重新定义后端动作、真实硬件代价和合法性约束。LiteCoOp 更适合作为多模型搜索/成本路由 baseline 或张量编译工具模块，而不是 LLVM Pass 优化完整框架。

## 11. 可进一步尝试的研究方向

### 11.1 后端感知的共享树调优
#### 研究问题
在 RISC-V/RVV 真实硬件上，模型选择偏好是否应同时考虑编译延迟、代码尺寸和运行时延迟？
#### 与原论文的区别
不只替换目标平台，而是把后端合法性、VL、寄存器压力和真实测量引入状态/奖励。
#### 可能的创新点
多目标 LA-UCT 与真实硬件校准 cost model。
#### 实验框架
`RVV IR → 异构 LLM 提议 → LLVM/TVM 后端编译 → 硬件测量 → 多目标回传`。
#### 可行性与风险
需要 RVV 工具链和板卡；风险是测量噪声、模型 API 成本和搜索空间过大。

### 11.2 可靠性与正确性联合路由
#### 研究问题
在变换可能不合法时，如何让模型错误率与性能共同决定下一模型？
#### 与原论文的区别
增加 verifier/编译失败信号，而非只依据 cost model 回归。
#### 可能的创新点
编译通过率、语义等价和延迟的联合 surrogate。
#### 实验框架
`变换提议 → 编译/验证 → 失败分类 → LA-UCT 回传`。
#### 可行性与风险
可用 LLVM verifier/Alive2；风险是形式验证覆盖范围有限。

## 12. 与其他已读文献的关系

与 LLVM-Bench 的对象不同：LiteCoOp 是 TVM 模型服务调度搜索，LLVM-Bench 是 LLVM issue 修复能力评测；前者可作为搜索成本路由方法参考，后者可提供大型编译器维护场景的能力边界。与 POLO 也不同：POLO 重写项目源码并用运行时反馈，而 LiteCoOp 操作张量编译 schedule。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 多 LLM 协作的张量编译调度搜索 |
| 核心问题 | 降低大模型引导优化的编译/API 成本 |
| 输入/输出 | TVM IRModule / 最佳 schedule |
| 核心方法 | 共享 MCTS、LA-UCT、course alteration |
| 使用模型 | 2/4/8 个异构 LLM，最大 GPT-5.2 或 Llama-3.3-70B |
| 编译器工具 | TVM v0.20.0 MetaSchedule、XGBoost cost model |
| 强化学习/形式验证 | 无 RL 训练；无形式化验证 |
| 数据集规模 | 5 kernel + Llama-3-8B 端到端 |
| 主要指标 | 硬件 latency、speedup、编译时间、API cost |
| 最重要结果 | 在最终搜索预算、五个 GPU/CPU benchmark 的口径下，8 模型 GPU/CPU 平均 speedup 为 30.1×/10.9×；相对单一 GPT-5.2，API 成本分别降低 4.47×/4.32× |
| 核心创新 | 将模型选择纳入共享搜索树，并以大模型稀疏纠偏 |
| 主要局限 | TVM/模型服务/有限 kernel，cost model 与后端范围受限 |
| RISC-V 相关性 | 低到中：可借鉴路由，尚无 RISC-V 实验 |
| 最适合作为 | 多模型搜索与成本路由 baseline |

这篇论文最值得学习的是把协作信号直接放进搜索树；主要局限是实验集中于 TVM 张量调度和有限硬件。后续应把它作为成本感知搜索参考，而不是简单声称其已解决 LLVM/RISC-V 后端优化。
