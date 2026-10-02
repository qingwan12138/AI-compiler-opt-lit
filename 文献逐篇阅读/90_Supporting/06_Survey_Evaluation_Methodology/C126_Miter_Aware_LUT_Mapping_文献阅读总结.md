# Miter-Aware LUT Mapping 文献阅读总结

论文题目：**Miter-Aware LUT Mapping: Aligning Structure and Solvability for Efficient Logic Equivalence Checking**

作者：Jiaying Zhu，Zhengyuan Shi，Mengxia Tao，Kezhi Li，Min Li，Qiang Xu

发表时间：2026

发表平台：DAC 2026

论文链接或编号：DOI 10.1145/3770743.3803929；arXiv:2607.07164
元数据核验来源：[ACM DOI](https://doi.org/10.1145/3770743.3803929)；[DAC official page](https://dac.com/dac-2026-general-society-awards)；[arXiv](https://arxiv.org/abs/2607.07164)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：逻辑等价检查（Logic Equivalence Checking, LEC）、miter、LUT mapping、SAT、XOR、形式验证、EDA、CNF

> 本文档依据 DAC 2026 正文 PDF 通读整理。论文事实、阅读后的分析和后续建议分开描述。本文不分配正式 Paper_ID，也不修改 taxonomy、正式索引、年份清单或总目录。

建议归类（仅供后续正式维护者参考）：`SUPPORTING / B6_Survey_Evaluation_Methodology`。这是一篇 EDA 形式验证与求解建模论文，不是 LLM、LLVM/MLIR 或 RISC-V 论文；最终分类应以维护阶段核验为准。

---

## 1. 研究背景

逻辑等价检查（LEC）用于验证综合前后的 golden design 与 implementation design 是否实现相同的布尔函数。论文第 1 节说明，常见流程把两套设计的对应输出通过 XOR 连接，再将所有 XOR 结果 OR 起来形成 miter；如果 miter 可满足，则存在反例，若不可满足，则两套设计等价。这个问题通常被转换为 SAT（Boolean Satisfiability，布尔可满足性）问题。

现有流程在复杂设计上遇到两个主要困难：

1. 逻辑综合中的 rewriting、resubstitution 和 decomposition 会扰动逻辑锥，使 golden design 与 implementation design 内部原本相同的节点难以匹配。内部等价点减少后，功能合并和冗余消除的机会下降。
2. 算术单元和密码单元等 XOR-dense 电路包含长 XOR 链。论文指出，依赖搜索回溯的现代 SAT 求解器在此类约束上往往传播较弱，需要大量决策才能确定字面量。

论文的基本判断是：LEC 效率不仅取决于 SAT solver，也取决于送入 solver 之前如何建模 miter。既有方法主要在扁平 miter 上做简化，或者使用面向单电路 SAT 的 LUT abstraction，忽视了 golden 与 implementation 两侧之间的结构对应关系。因此，作者把“等价结构恢复”和“solver-friendly（有利于求解器推理）的表示”放到 miter 建模阶段共同考虑。

## 2. 论文要解决的问题

### 2.1 综合扰动造成的跨设计结构失配

论文要解决的问题之一是：在综合已经改变内部逻辑结构的情况下，如何重新发现 golden design 和 implementation design 之间的可匹配子图，以便保留内部等价关系并减少 LEC 的 SAT 实例规模。

### 2.2 XOR-dense 区域的 SAT 求解困难

论文要解决的问题之二是：如何显式提取 XOR 区域中的仿射关系，避免其在普通 CNF 编码后变成不利于传播的结构。

### 2.3 LUT 选择目标与 SAT 行为脱节

论文还研究：LUT mapping 不应只追求结构紧凑或门数少，能否定义与分支、传播、冲突解释和结构吸收相关的 solver-oriented 成本，并据此选择 LUT。

> 本文主要研究：如何在 LEC 的 miter 建模阶段，通过保持跨设计结构对应、显式建模 XOR 仿射关系以及面向 SAT 行为选择 LUT，构造更容易被多种求解器处理的 SAT 实例。

## 3. 核心方法概述

论文提出一个 miter-aware mapping framework。给定 golden design `G` 与 implementation design `I`，框架不直接把两套门级网表简单压平，而是联合映射成 LUT-based miter，再把该表示交给 SAT/LEC 工具。方法包含三个组件：

1. **Equivalence-preserving LUT mapping**：从 primary output 附近的 XOR/XNOR 结构寻找跨两侧的锚点，递归匹配候选 LUT，保留可识别的内部等价关系。
2. **Gaussian-guided XOR modeling**：在 XOR-dense 区域检测 XOR 关系，将 XOR 门写成有限域上的方程并做 Gaussian elimination（高斯消元），保留有意义的仿射关系，将其显式重建为 XOR LUT。
3. **Solver-oriented LUT selection**：对候选 LUT 计算 branch count、sensitivity、determinacy、monotonicity、conflict interpretability 和 structural compactness 等指标，用加权成本选择更利于 SAT 推理的 LUT。

整体数据流如下：

```text
Golden design G + implementation design I
        ↓
构造 LEC miter 问题
        ↓
等价保持的跨设计 LUT mapping
        ↓
XOR-dense 区域的高斯引导 XOR 建模
        ↓
按 solver-oriented 指标选择 LUT
        ↓
生成结构更紧凑、关系更显式的 SAT/CNF 实例
        ↓
Kissat / CaDiCaL / X-SAT / CSAT / CEC 求解
        ↓
SAT 或 UNSAT 与 PAR2、Solved 数等结果
```

本文的 LM 角色：论文没有使用大语言模型。没有提示词、agent、工具调用式 LLM，也没有“模型输出 pass/IR/代码”的过程。编译器相关性主要体现在综合、逻辑 mapping、形式验证和 SAT 建模，而不是 LLVM/MLIR 编译器优化。

## 4. 实验框架与训练流程

### 4.1 系统执行流程

本文不涉及模型训练，主要采用一个面向 LEC 的静态 mapping 与求解前重构框架。输入是等价或不等价的电路对；输出是经 LUT abstraction 和求解器友好化处理后的 SAT 实例。

第一步，针对电路对建立 miter。等价样本将两个电路组合为 miter；不等价样本通过在综合后的 AIGER 中随机修改少量节点引入小的功能扰动，然后构成不等价电路对。

第二步，在两个设计之间执行等价保持的 LUT mapping。mapper 从输出向输入遍历，在 primary output 附近的 XOR/XNOR 节点处建立比较锚点，再依据拓扑深度、扇入/扇出和局部逻辑锥的 functional hash 递归匹配 LUT 候选。

第三步，对 XOR-dense 区域进行高斯引导建模。XOR/XNOR 门被转换成变量方程，有限域上的高斯消元用于提取仿射关系；参与这些关系的 LUT 被保留为显式 XOR LUT，而非被普通逻辑合并掉。

第四步，在不被 XOR 结构主导的区域按 solver-oriented 成本选择 LUT。各项分数在 mapping 前离线预计算并缓存；论文称由于只使用 2/3/4-input LUT，计算开销可以控制在较低水平。

第五步，将处理后的 LUT miter 插入现有 LEC/SAT flow。实验中使用 ABC 的 CEC 作为高级 LEC 引擎，并将框架作为 plugin-like pre-CNF optimization layer 插入其 CNF 转换之前。

### 4.2 训练、强化学习与反馈

论文没有预训练、SFT、PPO、GRPO 或其他强化学习。权重通过 Bayesian optimization 在小验证集上调节，以最小化求解时间；这不是神经模型训练，也不是强化学习奖励优化。

论文使用的是编译/验证工具反馈：最终反馈来自 SAT/LEC 求解时间、PAR2、Solved 数以及消融对比，而不是模型在运行时生成候选代码并迭代修复。

## 5. 奖励函数、损失函数或关键公式

本文没有强化学习奖励函数。关键优化目标是 LUT 的 solver-oriented cost。

### 5.1 LUT 总成本

论文第 3.4 节给出：

```text
Cost(L) = Σ_i w_i f_i(L)
```

其中：

| 符号 | 含义 |
|---|---|
| `L` | 一个候选 LUT |
| `f_i(L)` | 第 `i` 个 solver-friendliness 指标的归一化分数 |
| `w_i` | Bayesian optimization 得到的权重 |
| `Cost(L)` | LUT 候选的综合成本，供 mapping 选择 |

论文强调，较高或较低的分数含义依指标而异。例如，较小的 branch count 表示用更少的输入空间分支刻画函数；较低的 sensitivity 表示单比特翻转较少改变输出；较高 determinacy 和 conflict interpretability 通常表示更有利于传播或局部冲突解释；较小的 structural compactness 分数表示单个 LUT 吸收了更多内部逻辑、引入较少中间变量。

### 5.2 分支效率

定义 `bc(L)` 为覆盖 LUT 所有满足赋值所需的最少 branch pattern 数，`|I|` 为输入数：

```text
f_branch(L) = bc(L) / 2^|I|
```

该值越小，说明函数可由更少的输入区域描述，SAT 搜索可能需要更少 case split。

### 5.3 敏感性

对每个输入赋值 `a`，依次翻转每个输入位并观察输出是否改变：

```text
f_sensitivity(L) = SensCount(L) / (|I| × 2^|I|)
```

论文将较低敏感性解释为局部稳定，通常对应更窄的有效分支空间。

### 5.4 Determinacy 与 monotonicity

`DetCount(L)` 统计部分输入赋值能够唯一确定剩余变量和输出的情形，并按所有非空输入子集归一化：

```text
f_det(L) = DetCount(L) / Σ_{S⊆V, S≠∅} 2^|S|
```

较高的 `f_det` 意味着部分赋值更容易塌缩自由度，有利于 Boolean Constraint Propagation（BCP）。

对于单调函数 `T_L`，若所有输入满足 `a_i ≤ b_i`，则 `T_L(a) ≤ T_L(b)`。论文将 monotonicity 与 Horn-like CNF、较少负字面量和更短传播链联系起来；XOR 等非单调函数会同时产生正负字面量，可能削弱 unit propagation。

### 5.5 冲突可解释性与结构紧凑性

`f_interp` 通过部分赋值下的局部冲突计数归一化，偏好在小局部上下文中暴露冲突的 LUT。结构紧凑性使用 LUT 所吸收的内部 AIG gate 数与输入数归一化：

```text
f_compact(L) = GateCount(L) / |I|
```

论文说明，这些指标只用于候选选择；并没有声称某个单独指标能在所有电路或所有 SAT solver 上单独决定最优映射。

### 5.6 XOR 方程

论文将 XOR/XNOR 转写为有限域方程，例如：

```text
x1 ⊕ x2 = x3  ⇒  x1 ⊕ x2 ⊕ x3 = 0
x1 ⊙ x2 = x3  ⇒  x1 ⊕ x2 ⊕ x3 = 1
```

其中 `⊕` 是 XOR，`⊙` 表示 XNOR。对 XOR 方程做初等行变换后，保留参与仿射关系的 XOR LUT，并将关系重新实例化为显式 XOR 连接。

## 6. 实验设置

### 6.1 数据集来源

论文第 4.1 节构建了综合 benchmark suite，包含来自以下来源的电路对：

* 工业 LEC 问题；
* ForgeEDA；
* ITC99；
* EPFL；
* OpenCore。

等价电路对直接把原始电路和等价电路组合为 miter。不等价电路对通过在综合后的 AIGER 中随机修改少量节点引入小功能扰动，再构造 miter。论文未给出完整的训练/验证/测试集划分，因为本文不是学习模型数据集论文。

所有电路使用 ABC 综合；LUT 网络由作者基于 Mock-turtle 的定制 cost-driven mapper 生成，并限制为 2-input、3-input 和 4-input LUT。作者解释，较大的 LUT 会产生指数增长的 CNF clauses，从而妨碍后续 SAT 求解。

### 6.2 模型与工具

| 项目 | 论文信息 |
|---|---|
| LEC/综合工具 | ABC；CEC 配置作为高级 LEC engine |
| SAT solver | Kissat、CaDiCaL、X-SAT、CSAT |
| LUT mapping | 基于 Mock-turtle 的定制 cost-driven mapper |
| 硬件 | 单个 Intel Xeon Platinum 8375C 核心 |
| 超时 | 每个实例 3,600 秒 |
| 公式级简化 | Clausal Congruence Closure（CCC）可检测 CNF 中 AND/XOR/ITE 并做同余闭包 |
| LLM/训练 | 未使用；论文中未涉及 |
| RISC-V/LLVM/MLIR | 未使用，论文中未说明 |

### 6.3 对比方法

主要比较是传统 baseline 与本文完整框架 `Ours` 的 LEC 求解表现。消融设置包括：

* `Ours.w/o Equiv w/o CCC`：去掉等价保持 mapping 和 CCC；
* `Ours.w/o Equiv w CCC`：去掉等价保持 mapping，但保留 CCC；
* `Ours.w/o CCC`：保留等价保持 mapping，但去掉 CCC；
* `Ours w CCC`：完整方法。

solver-oriented LUT selection 另与两个方法比较：

* `Comp.1`：既有工作的 branching-based LUT cost function，强调结构复杂度；
* `Comp.2`：基于 CNF clause count 的 area-oriented cost function。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| `#Solved` | 在 3,600 秒限制内求解完成的实例数量 | 越大越好 |
| PAR2 | 实际求解时间；超时实例按 7,200 秒惩罚，且包含求解和 LUT mapping 时间 | 越小越好 |
| `Improv.` | 相对配置的改进百分比 | 依指标而定 |
| `Red.` | 相对 baseline 的 PAR2 或运行时间降低比例 | 越大越好 |
| Speedup | 相对另一配置的平均加速 | 越大越好 |

## 7. 实验结果与结论

### 7.1 多 solver 主要结果

表 2 比较了完整方法与 baseline：

| Solver/引擎 | Baseline #Solved | Ours #Solved | Ours PAR2 | Red. |
|---|---:|---:|---:|---:|
| Kissat | 1,729 | 1,731 | 42.3 | 91.44% |
| CaDiCaL | 1,530 | 1,723 | 67.4 | 92.13% |
| X-SAT | 1,125 | 1,498 | 289.4 | 87.35% |
| CSAT | 373 | 996 | 868.7 | 79.17% |
| CEC | 1,735 | 1,735 | 92.8 | 66.49% |

论文摘要和第 4.2 节将完整框架的最高降低幅度概括为 92.1%；该数值对应表 2 中 CaDiCaL 的 Red.。对 CaDiCaL，PAR2 从 856.6 降至 67.4；该 PAR2 包括求解和 mapping 开销。CEC 的 solved 数没有增加，但 PAR2 仍下降 66.49%。

### 7.2 与不同 solver 类型的比较

作者观察到，收益在 CNF-based solver 上尤其明显：Kissat 的 Red. 为 91.44%，CaDiCaL 为 92.13%。对于电路型 CEC 引擎，仍有 66.49% 的 PAR2 降低，但相对 CNF solver 的提升较小。论文将差异归因于：本文在电路级重构结构，且 CEC 自身还会执行 functional reduction 和 CNF-based final checking，因此本文相当于在既有 LEC 工具前增加一个可组合的预处理层。

### 7.3 高斯引导 XOR 建模消融

在 20 个由 32×32 multiplier 构成的 XOR-dense 实例上，完整方法与不使用 Gaussian modeling 的配置比较：

* 完整方法在 2,000 秒内解决全部 20 个实例；
* 不使用 Gaussian modeling 的配置在 2,000 秒内只解决 7 个，在 3,600 秒内解决 19 个；
* 对整个数据集，Gaussian-guided XOR modeling 带来平均 5.3% 的 PAR2 降低；
* 在与 CCC 联用的 20 个 multiplier 实验中，完整配置相对不使用 Gaussian modeling 的配置平均加速 36.8%。

这说明高斯建模的收益主要集中在 XOR-dense 电路，而不是所有电路均匀获得同等收益。

### 7.4 等价保持 mapping 与 CCC 消融

表 3 给出的 Kissat 结果如下：

| 设置 | #Solved | PAR2 | Red. |
|---|---:|---:|---:|
| Ours/w/o Equiv w/o CCC | 1,713 | 396.5 | — |
| Ours/w/o Equiv w CCC | 1,733 | 314.5 | 20.67% |
| Ours/w/o CCC | 1,726 | 56.2 | 85.83% |
| Ours w CCC | 1,731 | 42.3 | 89.33% |

在作者的解读中，CCC 本身能通过 CNF 冗余消除带来收益，但启用 equivalence-preserving mapping 后再配合 CCC，收益更大，说明电路级结构保持与 CNF 级同余闭包具有互补性。

### 7.5 Solver-oriented LUT selection 消融

在只启用 metric-guided LUT selection 的配置下，`Ours.metric` 在整个运行时间范围内优于 `Comp.1` 和 `Comp.2`。相对于 `Comp.1`，它在 3,600 秒内多解决 5.5% 的实例，整体平均加速 34.1%。这里比较的是 solver-oriented 指标组合，不是完整三组件框架的单独端到端结果。

### 7.6 论文结论

论文结论是：在 LEC 中，改变 miter 的表示方式能够在逻辑综合与 SAT 形式验证之间建立桥梁。联合使用等价保持 LUT mapping、高斯引导 XOR 建模和 solver-oriented LUT selection，可以让 miter 结构更对称、代数关系更显式、SAT 实例更容易求解。

## 8. 主要创新点

### 8.1 创新点一：面向 miter 的跨设计等价保持 mapping

已有 LUT mapping 工作主要处理单电路 SAT 或结构紧凑性问题。本文把 golden 与 implementation 两侧联合映射，并以输出附近 XOR/XNOR 作为自然锚点，递归匹配 LUT 候选。这一设计直接针对综合扰动造成的跨设计失配。表 3 的消融结果支持其有效性：保留等价保持 mapping 后，Kissat PAR2 明显低于去掉该模块的配置。

### 8.2 创新点二：把 XOR 仿射关系显式带入 LEC 表示

本文不是简单地把 XOR 逻辑压入普通 CNF，而是检测 XOR 关系、做高斯消元并保留相关 XOR LUT。这样既减少了 XOR-dense 区域中冗长搜索，也让 circuit-based 与 CNF-based solver 更容易利用结构。20 个 multiplier 实验和 36.8% 的相对平均加速表明该机制对 XOR-dense 实例有效。

### 8.3 创新点三：把 SAT solver 行为纳入 LUT 选择目标

本文将分支、敏感性、确定性、单调性、冲突可解释性和结构紧凑性组合成 LUT cost，而不是只用面积、门数或 clause count。与两种既有 cost function 的比较显示，该指标组合能减少求解时间。创新不在“使用 LUT”本身，而在于把 LEC solver 的推理行为前置到 mapping 决策中。

### 8.4 工程集成贡献

框架可以作为现有 LEC 工具的 plugin-like pre-CNF optimization layer 插入，而不是替换所有 SAT/CEC solver。论文同时给出跨 Kissat、CaDiCaL、X-SAT、CSAT 和 CEC 的实验，说明其目标是兼容既有验证 flow。

## 9. 局限性

### 9.1 论文明确或实验直接体现的范围

* 论文只研究组合逻辑 LEC；对时序等价检查、状态映射、时钟关系或顺序逻辑的适用性，论文中未明确说明。
* LUT 被限制为 2、3、4 输入。更大的 LUT 会产生更复杂的 CNF，但论文没有系统研究不同 LUT 上限的完整 trade-off。
* Gaussian-guided XOR modeling 针对 XOR-dense 电路更有效；论文没有声称它对所有电路类别均有同等收益。
* 每个实例使用 3,600 秒超时，PAR2 对超时按 7,200 秒惩罚；结果依赖这一时间口径。
* 权重在小验证集上通过 Bayesian optimization 调整。论文说明结果在 validation sets 上较稳健，但没有给出跨不同设计家族、不同硬件平台的泛化实验细节。

### 9.2 阅读后的潜在局限

* benchmark 包含工业 LEC 问题和公开来源，但正文没有给出每类实例的完整数量、公开性和可复现实验包信息；因此仅凭论文正文无法确认数据集是否全部可公开复现。
* 不同 SAT solver 的指标偏好可能不一致。一个统一的加权 cost 在 Kissat、CaDiCaL、X-SAT、CSAT 与 CEC 上总体有效，但并不等于对任意新 solver 都最优。
* 论文没有真实芯片运行时间或硬件执行性能实验；它优化的是形式验证求解时间，而非生成代码的运行时。
* 本文的 mapping 不提供语义等价的“证明替代品”。最终等价性仍由 SAT/CEC 求解器判定；mapping 和 Gaussian modeling 是求解前表示优化，不能把有限的 solved 结果解释为对所有电路的形式化保证。
* 论文没有 LLVM IR、MLIR、RISC-V、GPU 后端或异构硬件实验，因此不能直接推断其对软件编译器 pass、代码生成或 RISC-V 指令选择有效。

## 10. 阅读后的研究方向反思

### 10.1 值得借鉴的思想

最值得借鉴的是“先设计验证/反馈表示，再进行搜索或生成”的思路。论文将 SAT solver 的传播、分支与冲突行为反向注入 LUT mapping，说明优化目标不应只看表面规模，还应看下游工具是否能利用结构。这对 LLVM/MLIR 优化中的 verifier、cost model 和 test generation 也有方法论启发。

### 10.2 不能简单照搬的部分

不能把“LUT mapping + SAT”直接替换成 LLVM IR rewriting 或 RISC-V instruction selection，就声称得到同等创新。LEC 的布尔等价、CNF 传播和 XOR 仿射关系是本文特定问题结构。迁移到 LLVM/MLIR 必须重新定义中间表示的等价关系、优化目标和反馈信号。

### 10.3 与 RISC-V 研究的关系

相关性为低到中。相关之处在于它研究编译/综合结果进入下游验证器后的结构表示和求解成本；但论文没有 RISC-V ISA、RVV、LLVM backend 或真实硬件数据。若用于 RISC-V 研究，更合理的定位是形式验证/验证前建模方法参考，而不是 RISC-V 优化 baseline。

### 10.4 最合适的研究定位

本文最适合作为：

* 形式验证工具模块的设计参考；
* SAT/LEC 前端表示优化的 baseline；
* 研究“中间表示结构如何影响下游求解”的方法参考。

它不适合作为 LLM 训练方法、LLM agent 框架、LLVM pass 生成器或 RISC-V 后端优化器。

## 11. 可进一步尝试的研究方向

以下是阅读后的建议，不是论文已经实现的工作。

### 11.1 面向 LLVM IR 等价检查的 solver-aware 表示优化

#### 研究问题

能否把本文的“结构对应 + 下游求解友好性”思想迁移到 LLVM IR 变换后的函数等价验证，减少 verifier 或 translation validation 的求解时间？

#### 与原论文的区别

输入从组合门网表变为 LLVM IR，需处理 SSA、内存语义、未定义行为和循环，而不是把 LUT 映射用于布尔 miter。

#### 可能的创新点

定义与 SSA value correspondence、路径条件传播和 SMT/SAT 可解释性相关的 IR region cost，并以 verifier 的真实反馈校准，而不是直接复用 LUT 指标。

#### 实验框架

```text
原始 LLVM IR + 优化后 LLVM IR
        ↓
抽取 SSA/控制流对应关系
        ↓
构造 solver-aware IR region 表示
        ↓
Alive2 或 SMT 验证
        ↓
按验证时间、通过率和反例定位成本评估
```

#### 可行性

需要 LLVM、Alive2/SMT 工具和可复现的优化 pass benchmark；不需要假设 LLM。

#### 主要风险

LLVM 内存模型、未定义行为和循环处理可能使“结构对应”不再足够；形式验证速度提升也可能以构造开销或漏掉匹配为代价。

### 11.2 RISC-V 后端的验证友好代码生成

#### 研究问题

能否在 RISC-V 指令选择或 peephole 优化时，联合考虑生成代码的性能与后续等价验证难度？

#### 与原论文的区别

原论文优化硬件 miter 的 SAT 表示；该方向优化源 IR 到 RISC-V 汇编的变换选择，并要求语义等价验证反馈参与决策。

#### 可能的创新点

定义“验证友好”的指令序列特征，例如可匹配的局部数据流、较短的反例路径和较低的 SMT 编码成本，再与运行时间、代码大小共同构成 cost model。

#### 实验框架

```text
LLVM IR
  ↓
多个 RISC-V 指令选择/peephole 候选
  ↓
性能与代码大小测量 + 等价验证
  ↓
多目标排序
  ↓
选出性能可接受且验证成本较低的候选
```

#### 可行性

需要 LLVM RISC-V backend、Spike 或真实 RISC-V 平台、Alive2/SMT 或等价汇编验证工具。

#### 主要风险

验证成本与真实硬件性能可能冲突；若只用静态估计，不能把结果写成真实运行时提升。

### 11.3 LLM 生成优化候选的验证前筛选

#### 研究问题

当 LLM 生成多个 IR/源码优化候选时，能否用 solver-aware 结构指标优先筛掉难验证且可能无益的候选？

#### 与原论文的区别

原论文没有 LLM，也不生成程序变换；新方向把 LLM 作为候选生成器，把本文的表示/验证思想作为后端筛选模块。

#### 可能的创新点

建立候选的 correctness proof cost、counterexample locality、IR correspondence 和 runtime gain 联合排序；重点是候选筛选与验证预算分配，而非声称 LLM 本身完成证明。

#### 实验框架

```text
原始程序/IR
  ↓
LLM 生成候选变换
  ↓
语法检查与编译
  ↓
solver-aware 特征预筛选
  ↓
形式等价验证 + 真实性能测量
  ↓
保留正确且有收益的候选
```

#### 可行性

需要一个可固定版本的 LLM、编译器、形式验证器和可重复的性能 benchmark。

#### 主要风险

有限测试不能替代形式证明；候选分布、提示词和模型版本会影响结果；验证前筛选可能误删最终性能最好的候选。

## 12. 与其他已读文献的关系

本 slot-6 本轮只完整确认这一篇 DAC 2026 论文，不能虚构与其他论文的事实级横向比较。

就研究位置而言，它与正式 corpus 中 DAC 2025 的 CGRA 映射、RTL 断言修复和可重构架构探索论文存在 venue 层面的关联，但本笔记没有在本轮重新通读那些论文，因此不对其方法、实验数字或重复性作具体判断。本文自身更接近 `SUPPORTING` 中的 EDA 形式验证/评测方法：它不做 pass 选择、不直接生成 IR 或汇编、不生成后端组件，也不使用 LLM。

若与未来的 LLM 编译器论文组合，本文最自然的组合位置是验证/求解反馈模块，而不是把其 LUT mapping 当作 LLM 的训练算法。一个可能的组合关系是：LLM 或其他搜索器产生候选变换，本文式的结构与 solver cost 分析帮助安排验证预算；但这种组合尚未在本文中实现或实验。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 面向 LEC 的 miter 建模、LUT mapping 与 SAT 求解优化 |
| 核心问题 | 综合破坏跨设计结构对应，XOR-dense 区域削弱 SAT 传播 |
| 输入 | Golden design 与 implementation design 的组合逻辑电路对 |
| 输出 | 更适合 SAT/CEC 求解的 LUT-based miter/SAT 实例 |
| 核心方法 | 等价保持 LUT mapping + 高斯引导 XOR 建模 + solver-oriented LUT selection |
| 使用的模型 | 无 LLM；无神经模型训练 |
| 使用的编译器工具 | ABC、CEC、Mock-turtle mapper；不是 LLVM/MLIR |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 是；LEC/SAT/CEC 是最终验证工具，但 mapping 本身不是证明替代品 |
| 数据集规模 | 论文正文未给出完整总实例数；表 2 展示各 solver 的 solved 数，XOR 消融含 20 个 32×32 multiplier 实例 |
| 主要指标 | #Solved、PAR2、运行时间降低比例、平均加速 |
| 最重要实验结果 | CaDiCaL 的 PAR2 从 856.6 降至 67.4，Red. 92.13%；完整方法在 20 个 multiplier 实例中 2,000 秒内解决全部实例 |
| 核心创新 | 将跨设计结构对应和 SAT solver 行为共同前置到 miter/LUT mapping 阶段 |
| 主要局限 | 主要验证组合逻辑，LUT 输入受限；缺少 LLVM/RISC-V/真实硬件实验；数据细节有限 |
| 与 RISC-V 研究的相关性 | 低到中：可借鉴验证友好表示与反馈建模，但论文没有 RISC-V 实验 |
| 最适合作为 | 形式验证工具模块、SAT/LEC 前处理 baseline、solver-aware IR 表示研究参考 |

这篇论文最值得学习的是：下游验证器的推理行为可以反向指导上游中间表示和 mapping，而不仅是等求解器在扁平表示上自行补救；最主要的局限是方法针对组合逻辑 LEC，不能直接等同于 LLVM/RISC-V 代码优化；如果用于后续研究，最合理的使用方式是把它作为验证/求解反馈模块或 baseline，而不是简单把 LUT、SAT 或 DAC 场景替换成 RISC-V 就宣称形成新的 LLM 编译器方法。
