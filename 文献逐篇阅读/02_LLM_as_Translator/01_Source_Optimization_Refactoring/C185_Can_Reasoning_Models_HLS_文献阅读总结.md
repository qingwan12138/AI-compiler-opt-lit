# Can Reasoning Models Reason about Hardware 文献阅读总结

论文题目：**Can Reasoning Models Reason about Hardware? An Agentic HLS Perspective**

作者：Luca Collini、Andrew Hennessee、Ramesh Karri、Siddharth Garg

发表时间：2025

发表平台：ICLAD 2025, pp.188–194

论文链接或编号：DOI 10.1109/ICLAD65226.2025.00028；arXiv:2503.12721
元数据核验来源：[ICLAD 正式接收论文清单](https://iclad.ai/lad25-accepted-papers)；[IEEE DOI](https://doi.org/10.1109/ICLAD65226.2025.00028)；[arXiv](https://arxiv.org/abs/2503.12721)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：大语言模型、推理模型、高层次综合、HLS、pragma、设计空间探索、整数线性规划、硬件优化、agent

> 本文档只依据本地 PDF 正文记录论文事实；“阅读后的分析”与论文事实分开。

## 1. 研究背景

本文研究高层次综合（High-Level Synthesis，HLS）：工具把 C/C++ 规格和 pragma/directive 转换为 RTL。HLS 的代码到硬件转换已经较自动化，但如何选择 pragma、子模块实现和跨模块资源分配，仍依赖工程师反复试验。论文第 1 节指出，设计空间探索工具通常需要远多于专家的样本量。

论文关注 DeepSeek-R1、OpenAI o3-mini 等带显式推理机制的 LLM，问题是它们在数学和代码任务中的推理优势能否迁移到硬件设计。HLS 中 latency、area、并行依赖和模块组合关系相互影响，单纯把 LLM 当作代码生成器并不能自动解决这些约束。

## 2. 论文要解决的问题

### 2.1 推理模型是否能理解 HLS 设计约束

论文比较推理模型与非推理模型在 HLS 优化 agent 中的表现，关注它们是否能根据代码、面积、延迟和工具反馈作出有效行动。

### 2.2 如何把局部 pragma 优化扩展到全系统优化

论文希望自动重写代码、插入 pragma，并在总面积约束下为多个 kernel 选择组合，使系统延迟尽可能低。

> 本文主要研究：如何让 LLM agent 在 HLS 工具和 ILP 求解器反馈下，完成 kernel 级和全系统级的面积—延迟受约束优化，并比较不同 LLM 的推理能力。

## 3. 核心方法概述

论文提出一个两任务、工具调用式的 HLS 优化 agent。任务 1 对每个 kernel 生成多个 pragma/代码变体；任务 2 根据各 kernel 的候选结果建立 ILP，选择满足面积约束且延迟较低的系统组合。

```text
C/C++ HLS 设计
        ↓
LLM 查看 kernel 并生成代码/pragma 变体
        ↓
Catapult HLS 综合，获得 area/latency
        ↓
LLM 生成 ILP、调用求解器选择组合
        ↓
按 area 约束筛选并继续综合/反馈
        ↓
输出最终 HLS 设计
```

LLM 可执行的工具动作包括 Inspect Kernel、Make ILP Problem、Synthesize Solution 和 Select Solution；论文第 3 节说明，任务在 agent 选择 ILP 返回的方案后结束。最终输出不是单独的 pass 或 flag，而是包含代码重构和 pragma 的优化后 HLS 设计；因此从 taxonomy 的最终 artifact 规则看更接近 `TRANSLATOR / T1_Source_Optimization_Refactoring`，但其 ILP 选择子流程具有 `SELECTOR / S2` 性质。

## 4. 实验框架与训练流程

### 4.1 运行时任务 1：kernel 级优化

对每个 kernel，LLM 观察代码并给出不同 area-performance trade-off 的 pragma 或代码变换。每个 benchmark 的优化 kernel option 数设置为 5。每次候选通过 Catapult HLS 综合后，反馈面积和延迟。

### 4.2 运行时任务 2：全系统组合

LLM 使用任务 1 的候选，尝试为多个函数建立 one-hot 变量、面积模型、延迟模型和目标/约束，然后调用 ILP 求解器。并行模块的延迟不能简单相加，论文专门分析了 LLM 对数据流图和并行关系的建模错误。

### 4.3 模型与推理配置

实验比较 DeepSeek-V3（非推理基线）、DeepSeek-R1 和 o3-mini（推理模型）。两个任务由两个独立对话完成，以减少 token 使用和上下文长度压力。论文没有进行 SFT、PPO、GRPO 或其他任务专用训练；它主要使用提示、工具调用和运行时反馈。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数，也没有训练损失函数。系统优化目标是让总面积接近或不超过目标面积，同时降低延迟。

论文第 4 节描述推理模型有时采用如下多目标形式：

```text
min(α · latency + |area - target|)
```

这里 `latency` 是估计或综合得到的延迟，`area` 是组合面积，`target` 是面积目标，`α` 是权衡系数。该形式来自模型生成的 ILP/目标设计，并非一个经过训练的 reward。论文指出，若把不可达的面积目标硬编码为约束，求解可能失败；推理模型会尝试放宽限制并重试。

## 6. 实验设置

### 6.1 数据集来源

论文使用 12 个 benchmark：6 个人为构造的合成 benchmark SYN1–SYN6，覆盖顺序、并行、嵌套和循环中的数据流组合；6 个真实 benchmark，包括 AES、Present、SHA256、KMP、FIR+IIR 和 Needleman-Wunsch。合成 benchmark 用于测试模型是否能理解面积、延迟、调用和并行关系，并降低训练数据污染疑虑。每个 benchmark 运行 10 次，初始设计面积下调 10% 作为目标。

### 6.2 模型与工具

使用 DeepSeek-V3、DeepSeek-R1 和 OpenAI o3-mini。论文用 Python 实现约 1500 行的自动化流程；综合工具是 Catapult HLS，目标库是 `nandgate45`。DeepSeek API 实验期间出现服务中断，部分 DeepSeek-R1 运行通过第三方 provider 完成，论文将两类运行合并分析。

### 6.3 对比方法

主要比较三个 LLM：DeepSeek-V3 作为无显式推理的基线，DeepSeek-R1 和 o3-mini 作为推理模型。论文还在重叠 benchmark 上报告手工实现和 C2HLSC 的结果，但主实验的系统性比较集中在三个 LLM。

### 6.4 评价指标

| 指标 | 含义 |
|---|---|
| Success rate | agent 是否完成流程并给出结果，不要求结果达到面积目标 |
| Area | HLS 综合得到的面积，论文图表以平方微米表示 |
| Latency | HLS 报告中的周期数 |
| Cost | 输入/输出 token 数乘 API 单价 |
| Runtime | 完成实验流程所需时间 |
| Target-area count | 运行结果是否满足目标面积并达到相应最低延迟 |

## 7. 实验结果与结论

### 7.1 主要结果

论文图 2 比较 12 个 benchmark、每个 10 次运行的成功率、成本和运行时间。DeepSeek-R1 成功率最高，只出现一次因 task 1 引入 bug 而失败；o3-mini 通常更快但成本更高；DeepSeek-V3 成本最低，且在一些 benchmark 上成功率可达 100%。论文强调“成功”只表示流程完成，不表示满足面积目标。

表 II 的汇总计数显示，在满足面积目标且达到该目标下最低延迟的情形中，DeepSeek-V3、DeepSeek-R1、o3-mini 的总计分别为 28、23、18；当不满足面积目标而取最低面积时，三者总计为 0、3、6。论文据此指出，没有一个模型在所有 benchmark 上都优于其他模型。

### 7.2 与传统方法的比较

论文第 4 节的主比较不是与完整传统 DSE 算法的统一对照，而是与手工实现和已有 C2HLSC 结果的重叠 benchmark 比较。手工实现仍有明显优势；表 IV 中不同模型在 AES、SHA256、Present 上的面积和延迟差异很大，作者没有宣称推理模型全面超过人工设计。

### 7.3 与其他 LLM 方法的比较

本文发现推理模型没有像在数学和代码 benchmark 中那样稳定超过非推理模型。DeepSeek-V3 在某些 kernel 上设计更好；DeepSeek-R1 在 AES 隔离的全系统任务中得到面积 3596 平方微米、延迟 736 cycles，DeepSeek-V3 对应最好结果为 3707 平方微米、763 cycles，o3-mini 的最好结果面积为 4339 平方微米，未达到该面积目标。

### 7.4 消融与行为分析

论文分析了 ILP 生成和 action 数量。三个模型都能正确生成 one-hot 变量，但普遍难以从数据流图推导并行延迟；只有无并行的 AES 或偶然情况下的 Needleman-Wunsch 得到正确 ILP 延迟模型。DeepSeek-V3 平均执行更多 Inspect/Synthesis 动作，推理模型更直接，但 DeepSeek-R1 甚至很少检查代码，导致无法识别依赖和并行。

### 7.5 案例分析

论文展示了推理 token 中的错误推理：DeepSeek-R1 承认延迟存在并行关系，却仍用延迟求和代替最大值；DeepSeek-V3 会偏好“更接近目标面积但延迟更差”的方案。结果表明，LLM 具备局部硬件优化直觉，但并不稳定地理解全系统数据流。

## 8. 主要创新点

### 8.1 创新点一：推理模型在 agentic HLS 中的直接比较

本文把 DeepSeek-R1、o3-mini 和 DeepSeek-V3 放入同一 HLS 工具调用流程中，研究“推理能力是否迁移到硬件优化”。实验结果反而显示迁移并不自动成立。

### 8.2 创新点二：kernel 级变换与全系统 ILP 选择的组合

系统不仅生成单个 pragma，还把多个 kernel 的候选变体组合成 ILP 选择问题，目标是在总面积约束下优化系统延迟。该组合是本文主要的系统设计贡献；ILP 本身不是新算法。

### 8.3 创新点三：记录推理模型的硬件决策过程

论文使用 DeepSeek-R1 的 reasoning tokens 分析模型如何选择 Inspect、Synthesis 和 ILP 动作，并具体暴露其并行建模错误。这是对硬件场景推理能力的诊断性分析，而非形式化证明。

## 9. 局限性

### 9.1 论文明确承认的局限

- 三个模型在 area-latency 质量上没有稳定的全局优胜者。
- 推理模型仍会修改功能、无法修复 bug 或耗尽上下文。
- LLM 不能可靠地从函数依赖推导并行延迟，ILP 公式存在错误。
- DeepSeek API 中断导致部分运行使用第三方 provider。
- 论文只展示了有限 benchmark 和特定 HLS 工具环境。

### 9.2 阅读后的潜在局限

- 成功率不等于功能语义等价；论文使用 HLS 编译/流程结果，但没有把它描述为形式化语义证明。
- latency 主要是综合报告周期，不是完整 FPGA/ASIC 实测运行时间。
- 面积目标设置为初始面积的 90%，会影响不同 benchmark 的难度，跨论文比较需谨慎。
- 论文的主 artifact 同时包含代码重构和配置选择，若用于 taxonomy 分类，需优先区分“最终交付设计”与“内部选择动作”。
- 没有 RISC-V、LLVM IR 或跨架构实验，不能直接推出对 RISC-V 后端的有效性。

## 10. 阅读后的研究方向反思

值得借鉴的是“LLM 提议—HLS 工具综合—面积/延迟反馈—再决策”的边界设计，以及把全系统数据流显式交给求解器，而不是让 LLM 单独承担所有组合推理。论文已经把推理模型比较和 HLS agent 流程作为核心贡献，不能只把 Catapult HLS 换成另一工具就宣称新颖。

对于 RISC-V，单纯把 HLS 目标换成 RISC-V FPGA 不足以构成等价创新。更有价值的问题是：如何把 HLS 中的面积约束、代码/pragma 变体和后端真实指令或向量资源约束连接起来，并验证 LLM 的选择是否跨后端稳定。本文更适合作为 HLS agent baseline、工具反馈接口和分类中的相邻 Translator/Selector 案例，而不是现成的 LLVM/RISC-V 方法。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V/RVV 的跨层 HLS—后端选择

#### 研究问题

HLS 设计中的 pragma/代码变换如何影响最终 RISC-V/RVV 指令、向量长度和存储行为？

#### 与原论文的区别

不只优化 HLS 报告的 area/latency，而是把后端代码生成和目标硬件计数器纳入反馈。

#### 可能的创新点

建立从 HLS 候选到 RISC-V 后端代价的跨层 surrogate 或可验证选择接口。

#### 实验框架

```text
HLS 代码/pragma 候选 → HLS 综合 → RISC-V/RVV lowering → 仿真/硬件计数器 → 反馈与选择
```

#### 可行性

需要开源 HLS 工具、LLVM/RVV 工具链、FPGA 或可用的 RISC-V 硬件。

#### 主要风险

HLS 周期、RISC-V 实际运行时间和综合面积不是同一指标，反馈成本可能过高。

### 11.2 面向数据流图的约束生成器

#### 研究问题

如何让 LLM 只负责提出候选约束，而由静态分析或求解器检查并修正并行延迟模型？

#### 与原论文的区别

将论文暴露出的“LLM 会把并行延迟错误相加”变成显式验证模块。

#### 可能的创新点

把调用图、依赖图和 pipeline 关系自动转换为 ILP 约束，并用反例反馈修正候选。

#### 实验框架

```text
程序依赖图 → LLM 提议约束 → 图分析检查 → ILP 求解 → HLS 综合反馈
```

#### 可行性

可先在论文的 SYN1–SYN6 上复现，再扩展到真实 kernel。

#### 主要风险

完整 HLS 语义和跨函数资源共享可能无法由简单图模型表达。

## 12. 与其他已读文献的关系

- 与阶段 1 报告中的 A1-02 相比，本文先研究推理模型能力和两任务 agent；A1-02 将多 agent scaling 作为一等搜索变量，并扩展到跨函数代码级变换。
- 与正式库中的 C158 LLM4HLS、C171 MPM-LLM4DSE 和 C143 MailoHLS 相邻，均涉及 HLS directive/DSE，但本文更强调推理模型比较、工具 action 和 ILP 全系统选择。
- 与 C99 ComPilot 的角色边界相似之处是 LLM 通过编译器反馈选择优化动作；区别是本文主要是 HLS 代码/pragma 与系统组合，C99 已在正式库中归为 `SELECTOR/S4`，本条目因最终输出含源代码重构而暂拟 `TRANSLATOR/T1`。
- 适合作为 baseline：HLS agent 的工具反馈流程和推理模型对照。适合作为工具模块：ILP 组合和合成反馈接口。不能把其 HLS latency 结果直接当作 LLVM/RISC-V 性能证据。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 比较推理/非推理 LLM 在 agentic HLS 优化中的能力 |
| 核心问题 | 在面积约束下自动生成 kernel 变体并选择低延迟系统设计 |
| 输入 | C/C++ HLS 设计、kernel 代码、面积/延迟反馈 |
| 输出 | 含代码重构和 pragma 的 HLS 设计及组合选择 |
| 核心方法 | kernel 级变体生成 + 全系统 ILP 选择 + HLS 反馈 |
| 使用的模型 | DeepSeek-V3、DeepSeek-R1、OpenAI o3-mini |
| 使用的编译器工具 | Catapult HLS、`nandgate45` library、ILP solver |
| 是否使用强化学习 | 否；论文使用推理和工具反馈，不训练 RL policy |
| 是否使用形式化验证 | 未报告形式化语义验证；使用 HLS 流程和结果检查 |
| 数据集规模 | 12 个 benchmark，6 个合成、6 个真实，每个运行 10 次 |
| 主要指标 | 成功率、area、latency、cost、runtime、目标面积计数 |
| 最重要实验结果 | 推理模型成功率较高但成本更高；没有模型在所有 benchmark 上全面优胜 |
| 核心创新 | agentic HLS 两任务流程、全系统 ILP 组合、硬件推理行为分析 |
| 主要局限 | ILP 延迟建模错误、有限工具/benchmark、无跨架构验证 |
| 与 RISC-V 研究的相关性 | 中低；可借鉴约束反馈接口，但论文未评估 RISC-V |
| 最适合作为 | HLS agent baseline、工具反馈模块、推理模型对照 |

> 这篇论文最值得学习的是把 LLM 放在 HLS 工具和 ILP 求解器之间的决策环路中；最主要的局限是模型不能可靠地理解跨模块并行延迟，也没有跨架构实验；如果用于后续研究，最合理的使用方式是作为 HLS/约束反馈 baseline，而不是简单把目标硬件替换成 RISC-V。
