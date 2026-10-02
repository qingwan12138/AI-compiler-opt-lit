# ASPEN 文献阅读总结

论文题目：**ASPEN: LLM-Guided E-Graph Rewriting for RTL Datapath Optimization**

作者：Niansong Zhang，Chenhui Deng，Johannes Maximilian Kuehn，Chia-Tung Ho，Cunxi Yu，Zhiru Zhang，Haoxing Ren

发表时间：2025

发表平台：MLCAD 2025, pp.1–9

论文链接或编号：DOI 10.1109/MLCAD65511.2025.11189222
元数据核验来源：[IEEE DOI](https://doi.org/10.1109/MLCAD65511.2025.11189222)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：RTL 数据通路、LLM、e-graph、等价饱和、rewrite rule、PPA 优化

> 本笔记依据作者公开的完整 PDF 正文（包括正文、表格、案例、消融和附录）整理。论文事实与阅读后的分析分开描述。

## 1. 研究背景

本文研究寄存器传输级（Register-Transfer Level, RTL）数据通路优化。传统方法依靠设计者手工改写，或依靠 ASIC 综合工具中的启发式变换；它们需要在功耗、性能、面积（Power, Performance, Area, PPA）和功能正确性之间反复权衡。RTL 变换顺序会改变最终 PPA，形成 phase ordering problem。论文以 N-bit prefix adder 和 2048-bit multiplier 的巨大表达式空间说明搜索困难（第 1 节）。

等价饱和（equality saturation）通过 e-graph 同时保留大量等价表达式，缓解变换顺序问题，但仍依赖人工维护 rewrite rule 集合和代理成本模型。规则过多会造成 e-graph 膨胀，固定的代理成本又不一定反映综合后的真实面积/延迟。LLM 能提出设计相关的规则，却缺少结构化等价空间和正确性保证。ASPEN 将 LLM 规则发现、e-graph 和真实 EDA PPA 反馈结合起来。

## 2. 论文要解决的问题

### 2.1 自动发现可复用、设计相关的规则

人工规则难以覆盖不同 RTL 数据通路。论文研究 LLM 能否根据输入 Verilog、初始规则和示例证明，提出新的 rewrite rule，并仅保留等价检查通过的规则（第 III-B 节）。

### 2.2 选择适量的规则并避免 e-graph 膨胀

规则太少可能错过高质量表达式，规则太多又会拖慢饱和。论文研究如何根据设计特征和探索历史选择 critical rule set（第 III-C 节）。

### 2.3 让抽取成本反映真实综合结果

固定或全局的 e-node 成本可能与真实 PPA 不一致。论文研究如何把综合网表和报告中的面积/延迟信息映射回 e-node，并动态调整局部成本（第 III-D 节）。

> 本文主要研究：如何让 LLM 生成并选择可复用的 RTL rewrite rules，再借助 e-graph 和真实综合反馈探索保持功能等价的 PPA 设计点。

## 3. 核心方法概述

ASPEN 的最终编译器角色是 GENERATOR：LLM 生成可复用、可加入规则池的 rewrite rules；规则选择和 e-graph 执行共同形成 RTL 优化框架。系统包含规则提议 agent、规则选择 agent 和 PPA 反馈 agent。

```text
输入 Verilog 数据通路 + 初始规则 + 示例证明
        ↓
规则提议 agent 生成候选 lhs=rhs 与 Z3 proof program
        ↓
证明验证通过的候选加入 rewrite rule pool
        ↓
规则选择 agent 根据设计与历史选择 critical rules
        ↓
egglog 对输入表达式做 equality saturation
        ↓
抽取表达式 → 翻译为优化 Verilog → 商业综合
        ↓
获得 netlist、面积/延迟 PPA 反馈
        ↓
LLM 识别相关 e-node 并增减局部 e-node cost
        ↺ 继续规则选择和 e-graph 抽取，形成 Pareto 前沿
```

规则提议只在流程开始执行一次；外层循环探索规则子集，内层循环依据 PPA 反馈更新 e-node 成本并重新抽取。系统还记录最近的成本更新和结果，允许回溯导致退化的更新（第 III-D 节）。

## 4. 实验框架与训练流程

### 4.1 规则提议阶段

输入是 Verilog 设计、表 1 的初始代数/位级规则以及现有规则的示例证明。LLM 输出候选规则和对应的 Z3 证明程序；只有证明成功的规则才加入规则池。论文没有报告对这些 LLM 做 SFT、预训练或强化学习。

### 4.2 规则选择与饱和阶段

规则选择 agent 接收 Verilog、扩展后的规则池和探索历史，输出一个 critical rule set。egglog 版本为 0.4.0，使用 extraction-gym 的 fast greedy 与 fast bottom-up extractor（第 IV-A 节）。

### 4.3 PPA 反馈与回溯阶段

e-graph 抽取结果被翻译成 Verilog，并由商业 RTL 综合工具生成网表和报告。PPA feedback agent 将网表按 K-way graph partition 切分并序列化为文本，利用实例命名模式把网表子图与 e-node 组关联，之后增减局部成本。若历史表明更新使 PPA 变差，系统可回溯更新。

### 4.4 推理模型

论文使用 `gpt-4.1-202504142` 负责规则选择和 PPA 反馈，使用 `o3-20250416` 负责规则提议（第 IV-A 节）。这是提示驱动的 agent 流程，不是训练新模型的流程。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数，也没有报告 SFT、PPO 或 GRPO 损失。关键机制是 e-node 成本的启发式增减：Algorithm 1 中，PPA feedback agent 返回需要增加或减少成本的 critical nodes，系统分别执行 `cost[c-node] += Δinc` 和 `cost[c-node] -= Δdec`（第 III-D 节、Algorithm 1）。论文未给出一个统一的可学习损失或明确的数值奖励公式。

抽取通常最小化 e-node 成本总和；但 ASPEN 不把固定的算子类型成本当作真实 PPA，而是用综合反馈不断调整局部成本。图 4 展示 `a × 11`：统一高 MUL 成本会导致完全展开且面积更大，PPA 反馈引导的非均匀成本能得到更小面积的混合表达式。

## 6. 实验设置

### 6.1 数据集来源

实验使用 7 个开源 RTL 数据通路设计，而非论文所称的训练数据集：FIR、8-point DCT、4-degree Polynomial、Watermark、MCM(3,7,21)、MCM(7,19,31)、MCM(5,93)（表 III）。目标时钟频率分别为 FIR 1 GHz、DCT 1 GHz、Polynomial 400 MHz、Watermark 2 GHz、三个 MCM 设计均 2 GHz。论文中未报告训练/验证/测试划分，也未报告数据泄漏分析。

### 6.2 模型与工具

使用 egglog 0.4.0、extraction-gym extractor、Z3 proof program、商业 RTL 设计综合工具和商业 RTL equivalence checker。综合工具的 area、power、delay effort 均设置为 high，并使用 ASAP7 PDK；作者未在正文明确给出商业工具名称。输入设计先在 250 MHz 至 5 GHz 扫描频率，选择面积开始明显上升的频率作为目标。PPA 反馈实验实际报告面积和延迟；论文称方法可扩展到功耗。

### 6.3 对比方法

- Commercial Synthesis Baseline：直接用商业综合工具得到的基础面积/延迟。
- ROVER：作者使用表 1 固定 rewrite rules 和固定 e-node costs 的复现版本。
- ASPEN：LLM 规则提议、LLM 规则选择和 PPA-aware e-node cost 的完整系统。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| Area (μm²) | 综合后面积 | 越小越好 |
| Delay (ps) | 综合后延迟 | 越小越好 |
| Improvement (%) | 相对指定 baseline 的面积或延迟下降比例 | 越大越好 |
| Pareto frontier | 面积-延迟折衷下不被其他点同时支配的设计点集合 | 越广且越优越好 |

## 7. 实验结果与结论

### 7.1 主要结果

表 II 的 ASPEN 结果从 Pareto frontier 中分别独立选择最小面积和最小延迟点，不是同一个综合点的两个指标。相对商业 baseline，ASPEN 平均面积下降 24.23%、平均延迟下降 12.43%；相对 ROVER，平均面积下降 16.51%、平均延迟下降 6.65%。相对商业 baseline 的最大单项下降出现在 MCM(7,19,31)：面积 43.76%、延迟 31.62%。

相对 ROVER，MCM(7,19,31) 面积下降 43.76%、延迟下降 22.01%；MCM(5,93) 面积下降 26.01%、延迟下降 6.63%；FIR 延迟下降 13.86%。Watermark 和 Polynomial 的 ASPEN 最优点与 ROVER 相同或接近，表明收益并非对所有 benchmark 都增加。

### 7.2 规则案例

附录 A 中通过等价检查的规则包括 FIR 的 `12*x → (x<<3)+(x<<2)`、DCT 的嵌套乘法因式分解、Watermark 的 and-or 到 MUXAR、以及 MCM 中针对 5、7、19、21、31、41、93 的常数乘法规则。它们包含常数乘法分解、公共移位/乘数提取、重结合和条件表达式重构。

### 7.3 PPA 反馈案例

图 4 的 `a×11` 和图 5 的 FIR 迭代说明，统一成本会选择不理想的完全展开或保持乘法；综合反馈可以先提高 MUL 成本，随后又提高靠近叶节点的 ADD 成本，从而得到局部混合表达式。该反馈来自综合后的 PPA，而非真实运行时测量。

### 7.4 消融实验

图 6 比较移除规则提议（ASPEN-w/o-P）、移除 PPA feedback（ASPEN-w/o-PF）和同时移除提议/学习选择（ASPEN-w/o-PS）的变体。MCM(7,19,31) 上完整 ASPEN 面积/延迟改善为 43.8%/22.0%，去掉规则提议后为 31.5%/5.2%。w/o-PS 在 FIR 上甚至比商业 baseline 差 24.4% 面积。作者据此认为规则提议、规则选择和 PPA feedback 三者均重要。

### 7.5 Pareto 设计空间

图 7 中 ASPEN 评估许多候选点并得到面积-延迟 Pareto frontier；ROVER 通常只有一个固定规则产生的设计点。该结果说明 ASPEN 的贡献不只是找一个点，也包括扩展可探索的等价设计空间。

## 8. 主要创新点

### 8.1 创新点一：LLM 生成并验证设计相关 rewrite rules

论文把 LLM 的输出约束为可加入规则池的 `lhs=rhs` 变换，并要求对应 Z3 proof program 通过后才采纳。价值在于把通用代数规则与面向特定数据通路的常数乘法/因式分解规则统一到可复用规则层。实验附录列出了大量通过验证的规则，但并未证明所有规则对任意 RTL 都普遍最优。

### 8.2 创新点二：规则选择与 e-graph 执行分离

规则选择 agent 依据设计和历史挑选 critical set，egglog 执行等价饱和。这样可以限制 e-graph 膨胀，并让同一规则池适应不同设计；图 6 的消融支持这一模块有效。

### 8.3 创新点三：由真实综合反馈驱动局部抽取成本

论文没有使用固定 proxy cost，而是通过 netlist 分区、LLM 映射和局部成本更新，将综合面积/延迟反馈用于下一轮 e-graph extraction。该机制把生成式规则发现与真实 PPA 评估连接起来；表 II 和图 4/5 支持其有效性。

## 9. 局限性

### 9.1 论文明确或正文可见的限制

1. 评估只有 7 个开源 RTL 数据通路，且使用商业综合器和商业等价检查器，复现这些结果需要相应工具和 ASAP7 配置。
2. 网表可能有数十万至数百万实例，作者必须做 K-way 分区和文本序列化；这反映了 LLM 上下文窗口和图表示不匹配问题。
3. 论文只展示面积和延迟反馈；功耗被描述为可扩展方向，正文没有给出功耗结果。
4. ASPEN 的面积和延迟是从 Pareto frontier 独立取最小值，不能理解为一个设计点同时取得两个平均改善。
5. 作者没有报告训练集、验证集、测试集或数据泄漏实验，因为系统主要是推理期 agent 流程。

### 9.2 阅读后的潜在限制

1. LLM 提议规则和 PPA 反馈都可能产生较高 API/综合成本，论文没有给出端到端 token、调用次数或总时间成本。
2. 规则提议依赖命名模式将综合网表映射回 e-node；不同综合器、网表重命名或更强结构重写可能降低映射可靠性。
3. 等价检查保证的是输入与优化 RTL 的功能等价，不等于在所有时序、功耗、电气约束下都满足实现要求。
4. 结果面向 RTL/EDA 数据通路，不能直接外推到 LLVM IR、通用 C 程序或 RISC-V 后端。

## 10. 阅读后的研究方向反思

值得借鉴的是“LLM 生成候选、形式/等价工具过滤、编译器结构化表示执行、真实硬件反馈评价”的分工。ASPEN 的核心并不是单独使用 LLM，而是把 LLM 输出限制在 rewrite-rule 接口内，并将正确性和 PPA 评估交给工具链。

不能把 ASPEN 简单改成 RISC-V 就视为同等创新：需要重新处理 LLVM IR/SelectionDAG 与 RISC-V 指令语义、寄存器约束、向量状态和成本映射。对当前语料库而言，ASPEN 更适合作为 G2 的方法参考或 RTL rewrite 生成 baseline，不是 LLVM/RISC-V 现成方案。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V Vector 的验证驱动规则生成

#### 研究问题

LLM 能否从 RVV 指令语义和 LLVM IR pattern 生成可复用的 many-to-many rewrite rules？

#### 与原论文的区别

把 RTL 网表 PPA 映射替换为 LLVM IR/SelectionDAG、RVV intrinsic 和指令选择成本，同时处理向量长度和尾部策略，而不是只替换目标 ISA。

#### 可能的创新点

将 RVV 状态约束编码到规则接口，并用 Alive2 或等价的语义检查器筛选候选。

#### 实验框架

```text
LLVM IR/RVV 语义 + seed rules → LLM 提议规则 → 形式等价检查
→ LLVM 指令选择/编译 → 真实 RISC-V 硬件或周期模型反馈 → 规则排序
```

#### 可行性

需要 LLVM/RVV 后端、语义验证器、QEMU 或真实 RISC-V 平台，以及受控的规则语法。

#### 主要风险

向量状态和硬件实现成本可能使“语义等价”与“性能更优”脱钩。

### 11.2 规则池的跨设计泛化评估

#### 研究问题

ASPEN 生成的设计专用规则能否迁移到未见过的 RTL 数据通路？

#### 与原论文的区别

原文主要报告按 benchmark 生成和选择规则；新实验应严格按设计划分，测试规则的跨设计可复用性和失败模式。

#### 可能的创新点

建立规则的前置条件、适用形状和 PPA 变化档案，研究“通用规则 + 专用规则”的组合策略。

#### 实验框架

```text
训练设计生成规则 → 按设计隔离规则池 → 未见设计匹配/等价检查
→ 综合 PPA → 统计迁移成功率与成本
```

#### 可行性

可复用本文 7 类数据通路并加入更多公开 RTL benchmark。

#### 主要风险

规则数量、综合预算和设计分布偏差会影响结论。

## 12. 与其他已读文献的关系

本轮只完成 ASPEN 一篇新增论文，因此没有把未在本轮通读的论文当作横向实验对象。就仓库 taxonomy 的角色而言，ASPEN 与 G2 `Optimization Rule/Transform Generation` 直接对应：LLM 生成可复用规则；这不同于 SELECTOR 的 pass/配置选择，也不同于 TRANSLATOR 直接输出一个优化后的程序。它与已有 RTL e-graph 工作的关系是将 LLM 规则发现和真实 PPA 反馈接入等价饱和流程；与纯规则综合或固定 e-graph 成本模型相比，新增了生成和反馈环节。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLM 引导的 RTL 数据通路 e-graph rewrite 优化 |
| 核心问题 | 人工规则不可扩展、固定成本不反映真实 PPA |
| 输入 | Verilog 数据通路、初始 rewrite rules、示例证明 |
| 输出 | 通过等价检查的规则池、优化 RTL、Pareto 设计点 |
| 核心方法 | 规则提议 + critical rule 选择 + e-node 成本反馈 |
| 使用的模型 | gpt-4.1-202504142、o3-20250416 |
| 使用的编译器工具 | egglog 0.4.0、extraction-gym、商业综合器、商业等价检查器、Z3 proof program |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 使用等价检查/Z3 证明筛选规则；工具名称未完全说明 |
| 数据集规模 | 7 个开源 RTL 数据通路；无训练/验证/测试划分报告 |
| 主要指标 | 面积 μm²、延迟 ps、相对改善、Pareto frontier |
| 最重要实验结果 | 相对 ROVER 平均面积下降 16.51%、延迟下降 6.65% |
| 核心创新 | LLM 生成可复用规则，并用真实综合 PPA 反馈更新局部抽取成本 |
| 主要局限 | benchmark 小、依赖商业 EDA、LLM/综合开销未充分报告 |
| 与 RISC-V 研究的相关性 | 中：规则生成与验证思路可迁移，但本文实验对象是 RTL 数据通路，不是 RISC-V/LLVM |
| 最适合作为 | G2 方法参考、RTL rewrite 生成 baseline、PPA 反馈工具链参考 |

> 这篇论文最值得学习的是把 LLM 的规则发现能力限制在可验证的 rewrite-rule 接口内，再用 e-graph 管理等价设计空间、用真实综合结果反馈抽取成本；最主要的局限是实验规模和工具依赖有限，且没有证明规则对其他编译器或架构的直接迁移性。用于后续研究时，合理方式是借鉴“生成-验证-反馈”架构并重新定义目标 IR/ISA 与成本模型，而不是只把 RTL 平台替换成 RISC-V。
