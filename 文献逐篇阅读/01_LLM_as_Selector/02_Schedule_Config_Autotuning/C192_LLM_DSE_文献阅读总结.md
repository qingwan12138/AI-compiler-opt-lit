# LLM-DSE 文献阅读总结

论文题目：**LLM-DSE: Searching Accelerator Parameters with LLM Agents**

作者：Hanyu Wang、Xinrui Wu、Zijian Ding、Su Zheng、Chengyue Wang、Neha Prakriya、Tony Nowatzki、Yizhou Sun、Jason Cong

发表时间：2025（arXiv v1 首次公开；本次阅读 PDF 为 v3，2025-11-21）

发表平台：arXiv preprint，cs.AR / cs.AI；正文首页标注 “Preprint. Under review.”

论文链接或编号：arXiv:2505.12188；arXiv-issued DOI: 10.48550/arXiv.2505.12188

关键词：高层次综合（High-Level Synthesis, HLS）、领域专用加速器、设计空间探索（Design Space Exploration, DSE）、LLM agents、directive/parameter tuning、Merlin Compiler

> 本笔记只依据 staging 中的 v3 PDF；未执行正式入库同步。

## 1. 研究背景

论文研究领域是用 HLS 从 C/C++ 等高层描述生成领域专用加速器。HLS 将硬件设计从 RTL 提升到较高抽象层，但最终性能仍取决于 pipeline、parallelism、tiling 等 directive 参数组合。论文第 1 节指出，硬件设计空间可能约为 10^13 级别，而每个设计点都可能需要完整综合流程，反馈延迟可达数小时。

传统方法包括依赖领域知识的启发式搜索（如 AutoDSE）和使用代理模型的搜索（如 HARP）。前者容易受固定启发式偏差影响，后者需要昂贵的样本数据。论文将挑战归纳为：综合反馈慢、搜索空间巨大且 reward 不平滑、低资源设置下 LLM 直接生成配置容易失败。

## 2. 论文要解决的问题

### 2.1 昂贵且不平滑的 directive 搜索

给定 HLS 程序和每个 directive 的可选值，系统需要在满足资源约束的前提下降低 latency。相邻参数变化可能使设计从 timeout 变为高性能，不能简单使用局部梯度或固定邻域。

### 2.2 多目标配置决策

系统既要降低周期数，又要把 LUT、BRAM、FF、DSP、URAM 等资源利用率控制在论文设定的 80% 以下。论文希望由不同角色处理性能和资源目标，再由一个整体仲裁角色选择值得实际综合的更新。

> 本文主要研究：如何让 LLM agent 在有限综合预算内搜索 HLS 加速器的参数配置，而不是直接重写程序代码。

## 3. 核心方法概述

LLM-DSE 把参数搜索建模为闭环、多 agent 的树搜索。节点表示一组完整 directive 配置，边表示一次参数更新。系统从无并行的保守配置开始，逐步生成候选更新、选择少量候选进行综合、解析反馈并剪枝。

```text
HLS C/C++ 程序 + 参数空间 + 默认配置
        ↓
Router 分析历史和瓶颈，分配给专家
        ↓
性能专家 / 资源专家分别提出参数更新
        ↓
Arbitrator 合并并选择少量候选
        ↓
Merlin/其他 HLS 工具综合，得到 cycle、资源和 warning
        ↓
Critic 比较父子配置、剪枝分支、生成自然语言反馈
        ↓
History curator 压缩历史，进入下一轮
```

LLM 的最终输出是 directive 参数更新和搜索决策；HLS 工具执行综合和硬件变换。因此本论文建议分类为 **SELECTOR / S2_Schedule_Config_Autotuning**，不是 Translator。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用提示驱动的推理和工具反馈。

### 4.1 初始设计

系统从通常不启用并行的保守设计开始，以确保至少有一个可综合且满足资源约束的点。每轮从已探索配置中选择父节点。

### 4.2 专家路由与 proposal

Router 根据当前配置、性能/资源反馈和历史，决定应优化哪个参数以及交给哪类 specialist。性能专家关注 unroll、pipeline 等对周期的影响；资源专家关注 tiling、array 等对片上资源的影响。

### 4.3 两阶段过滤与工具反馈

第一阶段每个 specialist 对其负责的参数提出一个更新，允许跳到较远的合法值。第二阶段 Arbitrator 综合各 proposal，考虑参数敏感度和剩余预算，只保留少量候选交给 HLS 工具。Critic 解析综合报告，比较只改变一个 directive 的父子节点并剪枝。

### 4.4 上下文控制

History curator 按性能和参数多样性选取代表性设计，避免 router 上下文线性膨胀；报告解析器只保留 cycle、资源和常见错误/timeout 等必要字段。

## 5. 奖励函数、损失函数或关键公式

论文没有使用强化学习奖励函数，也没有训练损失函数作为本文系统的核心目标。

论文将设计点表示为 `d=(d1,d2,...,dn)`，目标是：

```text
minimize latency in clock cycles
subject to resource utilization < 80%
and d belongs to the program-specific feasible design space
```

论文中的“reward”主要指性能反馈和搜索中的优劣判断，不是一个单独定义的 RL reward。综合失败、timeout 或资源超限会影响节点可行性和分支剪枝。

## 6. 实验设置

### 6.1 数据集来源

- 主实验使用 HLSyn 中 10 个代表性 workload；这些 kernel 来自 ML4HLS contest 第二阶段。
- 扩展实验使用 Rosetta benchmark 的 `conv2d`、`spam-filter`、`knn`、`3d-rendering` 四个较大程序。
- 论文报告 HLSyn 最大程序约 77 LoC，Rosetta 程序约 118--304 LoC，且 Rosetta 最多有 7 层嵌套循环。
- 论文没有把这些数据描述为 LLM 训练集；它们用于 DSE 和综合评估。

### 6.2 模型与工具

- LLM 通过 OpenAI API 使用；正文没有在主实验段统一给出单一模型名，token 分析段以 GPT-4o 估算成本。
- 主后端是 Merlin Compiler，8 小时搜索 timeout；Stratus 作为 ASIC flow，Vitis 作为片上 FPGA flow 做迁移实验。
- 机器为 60 核 AMD EPYC 7V13、240 GB RAM。
- 框架使用 Python；报告解析器和 history curator 是外部工具模块。

### 6.3 对比方法

- AutoDSE-8 / AutoDSE-24：启发式 HLS directive 优化，分别使用 8 小时和 24 小时预算。
- HARP-8 / HARP-24：用 GNN 代理模型做 breadth-first search，代理数据由 AutoDSE 生成。
- RALAD：作为已有 LLM 直接配置/代码优化方法的比较对象。
- Appendix 中还比较了直接 zero-shot/one-shot 参数生成。

### 6.4 评价指标

- cycle count：综合后完成程序所需周期数，越低越好。
- speedup：baseline cycle count / LLM-DSE cycle count，越高越好。
- resource utilization：LUT、BRAM、FF、DSP、URAM 等，需满足资源约束。
- win/tie：相对 baseline 的配置胜负统计。
- token consumption：8 小时搜索期间的输入/输出 token 数和估算成本。

## 7. 实验结果与结论

### 7.1 主结果

在 Merlin、每个程序 8 小时、每个实验重复两次的设置下，10 个 HLSyn workload 上，LLM-DSE 相对 AutoDSE-8 的平均 speedup 为 2.55×，相对 AutoDSE-24 为 1.60×，相对 HARP-24 为 1.16×；几何平均分别为 1.87×、1.30×、1.04×。speedup 的定义是 baseline 周期数除以 LLM-DSE 周期数。

### 7.2 较大程序

在 Rosetta 四个程序上，相对 AutoDSE-8 的几何平均 speedup 为 1.22×。单项结果为 `conv2d` 1.13×、`spam-filter` 2.12×、`knn` 1.15×、`3d-rendering` 0.81×，说明扩展到更大程序并非所有 workload 都获益。

### 7.3 与 RALAD

论文表 5 在可比较的配置上报告：LLM-DSE 相对 RALAD 的平均 speedup 为 132.91，RALAD 为基准 1.00；几何平均分别为 71.06 和 58.32。表中存在 timeout/缺失项，不能把它解释为所有程序上的完整平均硬件加速。

### 7.4 消融实验

- 去除 Arbitrator 后，几何平均相对简化架构的 speedup 为 1.58×；只保留性能专家时为 1.10×。
- 逐步将 agent 替换为启发式时，A+S+R+C 的几何平均相对 AutoDSE-8 为 1.87×，win ratio 为 8/10；A、A+S、A+S+R 的几何平均分别为 0.69×、0.95×、1.24×。
- 论文认为 router-specialist 和 specialist-arbitrator 交互有助于有限综合预算下的探索；resource specialist 可帮助从接近最优但资源非法的分支恢复。

### 7.5 成本与局限结果

8 小时 Merlin 运行在 10 个程序上约消耗 4×10^5--2×10^6 输入 token、4×10^3--3×10^4 输出 token；以 GPT-4o 估算约 1--7 美元。论文明确承认仍需数小时才能取得较好结果。

## 8. 主要创新点

### 8.1 创新点一：把 HLS 参数搜索组织成多 agent 树搜索

不同于让 LLM 一次性生成完整配置，论文把父节点选择、参数更新、候选筛选和分支剪枝拆成协作角色，并让外部 HLS 工具提供真实反馈。实验中的配置和工具链结果支持该设计的有效性。

### 8.2 创新点二：性能/资源 specialist 与 arbitrator 的双阶段 proposal 过滤

specialist 负责局部参数提案，arbitrator 负责跨参数比较和预算感知选择；这减少了昂贵综合次数，也让大跳跃式更新可以被后续迭代恢复。

### 8.3 创新点三：面向上下文和报告的工具化控制

history curator、报告解析和父子单变量比较使 agent 不必读取全部历史或完整日志。它们是针对 DSE 长上下文和噪声反馈的工程机制，不应简单等同于新的编译算法。

## 9. 局限性

### 论文明确承认的局限

- 当前主要优化 DSA 参数，不处理更大的代码变换搜索空间。
- 获得良好性能仍需数小时。
- 论文没有把方法证明为跨所有 HLS 工具或所有硬件平台都有效。

### 阅读后的潜在局限

- 主结果依赖 Merlin/HLS 综合反馈，部署成本和可重复性受商业/复杂工具链影响。
- LLM API、prompt 和模型版本对结果可能有影响；正文没有给出完整可独立复现实验所需的 prompt/token 日志。
- 资源约束采用 80% 门槛，未证明该门槛在不同 FPGA/ASIC 目标上的普适性。
- RALAD 对比中存在 timeout 和缺失项，跨方法的公平性需要按原始配置和预算再次复核。

## 10. 阅读后的研究方向反思

值得借鉴的是“LM 选择配置、编译器负责执行、反馈用于下一轮”的职责分离，以及父子单变量比较带来的可审计增量证据。其核心贡献已经是 HLS 多 agent DSE，不能只把 Merlin 替换成 LLVM 或 RISC-V 就宣称同等创新。对 RISC-V 的直接相关性有限：可以借鉴配置搜索接口，但需要重新定义 RISC-V 向量长度、寄存器分配、tile、unroll 和后端资源反馈，不能假设 HLS directive 空间可直接迁移。该论文更适合作为 Selector baseline 和工具反馈架构参考。

## 11. 可进一步尝试的研究方向

### 11.1 跨硬件后端的配置策略迁移

#### 研究问题

学习从一个 HLS/FPGA 配置空间迁移到另一种目标后端时，哪些配置规律仍然有效。

#### 与原论文的区别

不只是换工具，而是显式学习配置语义与硬件效应之间的可迁移表示。

#### 可能的创新点

加入后端效应标签、失败边界和不确定性门控。

#### 实验框架

```text
源后端配置轨迹 → 语义/硬件效应表示 → 目标后端候选配置
        → 编译/测量 → 迁移收益与失败边界
```

#### 可行性

需要两个可访问的编译器后端、统一 benchmark 和真实性能反馈。

#### 主要风险

不同后端的配置空间可能没有稳定同构关系。

### 11.2 面向 RISC-V/RVV 的证据约束 Selector

#### 研究问题

LM 如何只选择可由 LLVM/RVV 后端解释且具备性能证据的 pass/参数组合。

#### 与原论文的区别

输出从 HLS pragma 配置转为 LLVM/RVV pass 与后端参数，并要求每个动作附带可验证反馈。

#### 可能的创新点

将硬件计数器、代码生成失败和语义检查组成统一的候选证据契约。

#### 实验框架

```text
LLVM IR + RVV 目标 → LM 提出 pass/config → LLVM 执行
        → 语义/性能证据 → 保留、回退或剪枝
```

#### 可行性

需要 LLVM、RISC-V 交叉编译和可重复的仿真或板级测量。

#### 主要风险

仿真性能与真实硬件性能的相关性可能不足。

## 12. 与其他已读文献的关系

- 与本批次 RALAD 相似之处：都使用 LLM 处理 HLS 优化问题并结合工具反馈；差异是 RALAD 直接生成带 pragma 的源代码，LLM-DSE 只生成参数更新，因此前者是 Translator、后者是 Selector。
- 与本批次 AI4DSE 相似之处：都把 LM 用于 HLS DSE 配置搜索；LLM-DSE 用多 agent、真实综合闭环和分支剪枝，AI4DSE 用 GNN QoR predictor 加 LLM-enhanced meta-heuristic，后者更依赖代理评估器。
- 三者可作为角色边界对照：LLM-DSE 是 Selector baseline，AI4DSE 是 Selector 的代理模型/元启发式变体，RALAD 是 Translator baseline。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | HLS 领域专用加速器 directive 参数搜索 |
| 核心问题 | 搜索空间巨大、综合反馈慢、资源和性能目标冲突 |
| 输入 | HLS 程序、参数空间、历史设计和工具报告 |
| 输出 | 下一步 directive 参数更新及最终配置 |
| 核心方法 | Router、specialists、arbitrator、critic 组成闭环树搜索 |
| 使用的模型 | 通过 OpenAI API 调用 LLM；正文未统一固定单一主模型 |
| 使用的编译器工具 | Merlin；补充 Stratus、Vitis |
| 是否使用强化学习 | 否；论文不涉及 RL 训练奖励 |
| 是否使用形式化验证 | 否；依赖 HLS 工具的可行性/正确性流程 |
| 数据集规模 | HLSyn 10 个 workload；Rosetta 4 个较大程序 |
| 主要指标 | cycle、speedup、资源利用率、win/tie、token |
| 最重要实验结果 | 相对 AutoDSE-8 平均 2.55×、几何平均 1.87× speedup |
| 核心创新 | 面向 HLS 配置搜索的多 agent 协作和双阶段过滤 |
| 主要局限 | 仍需小时级搜索，不处理代码变换 |
| 与 RISC-V 研究的相关性 | 中低；可借鉴配置选择和证据闭环，但需重新建模 RVV 后端 |
| 最适合作为 | Selector baseline、工具反馈架构参考 |

> 这篇论文最值得学习的是让 LM 负责配置决策、让编译器负责实际变换并保留父子配置证据；最主要的局限是搜索成本仍高且依赖 HLS 工具；如果用于后续研究，最合理的使用方式是作为 Selector/DSE baseline，而不是简单把 HLS 参数名替换为 RISC-V 参数。
