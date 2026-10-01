# AutoPPA 文献阅读总结

论文题目：**AutoPPA: Automated Circuit PPA Optimization via Contrastive Code-based Rule Library Learning**<br>
作者：Chongxiao Li、Pengwei Jin、Di Huang、Guangrun Sun、Husheng Han、Jianan Mu、Xinyao Zheng、Jiaguo Zhu、Shuyi Xing、Hanjun Wei、Tianyun Ma、Shuyao Cheng、Rui Zhang、Ying Wang、Zidong Du、Qi Guo、Xing Hu<br>
发表时间：2026（arXiv v2，2026-04-28；首版 2026-04-20）<br>
发表平台：arXiv，cs.LG、cs.AR<br>
论文链接：<https://arxiv.org/abs/2604.18445><br>
本地正文：`../03_LLM_as_Generator/02_Optimization_Rule_Transform_Generation/C228-AutoPPA-arXiv2026/paper.pdf`，16 页，`%PDF-` 签名，正文提取 44,346 字符。

## 1. 研究背景

RTL 设计的面积、时延和动态功耗（PPA）高度依赖代码结构。传统人工优化需要硬件经验，直接让 LLM 对每个 RTL 输入反复改写又存在正确性、成本和复用性问题。已有方法或依赖综合后的 PPA 反馈，或依赖工程师手工总结的优化规则，难以自动形成规模化知识。

AutoPPA 研究一个更具体的问题：能否从原始 RTL 和 LLM 生成的等价改写中，自动归纳可复用的 PPA 优化规则，并让这些规则指导之后的 RTL 搜索？论文的关键产物不是某个样例电路的一次性改写，而是可检索的 `(snippet, condition, action)` 规则库。

## 2. 论文要解决的问题

论文围绕三个难点展开：

1. 如何判断候选 RTL 改写仍保持功能等价，并且确实改善面积、时延或功耗。
2. 如何从大量等价代码对中抽取有用、去噪且具有泛化能力的规则。
3. 如何把抽象规则匹配到新 RTL，并在多步搜索中避免规则过专用、检索噪声和规则组合失控。

论文同时比较自动规则库与人工规则库，检验自动归纳的规则是否能在新的电路上复用。

## 3. 核心方法概述

AutoPPA 由两个阶段组成：Contrastive Code-based Rule Library Learning 和 Adaptive Rule-based PPA Optimization。

```text
原始 RTL
  ↓ LLM 多次采样
多个候选 RTL 改写
  ↓ Yosys/测试台/综合
功能等价代码对 + PPA 标签
  ↓ LLM 规则归纳与重应用筛选
可复用规则库：(snippet, condition, action)
  ↓ 条件检索、规则适配、beam search
新 RTL 优化候选
  ↓ 等价验证 + SiliconCompiler 综合
保留高 PPA 收益结果
```

阶段一的 Explore 使用 Qwen2.5-Coder-7B-Instruct 对每个 RTL 设计采样多次改写。Evaluate 通过测试台、Yosys 和综合流程保留功能等价且 PPA 有差异的代码对。Induce 再让 LLM 为代码对生成候选规则，并通过多次重新应用规则的结果评分，只把平均得分超过 0.7 的规则加入规则库。

阶段二的 ARAO 先让 LLM 为目标代码概括结构条件和动作，再检索最相关的三条规则，由 LLM 适配到当前代码，最后生成优化代码。多步 Rule-based Enhanced Search 使用带多样性项和 PPA 项的 beam search 扩展候选。

## 4. 实验框架与训练流程

论文使用 RTLRewriter benchmark 的 54 个设计，并把 3 个大型实践设计拆成 6 个模块，共 60 个设计。每个设计配备测试台，论文报告非冗余代码达到 100% 行覆盖和分支覆盖。

规则学习阶段不是端到端训练一个新模型，而是使用 LLM 进行代码采样、规则归纳和规则适配。主要模型包括 Qwen2.5-7B-Instruct、Qwen2.5-Coder-7B-Instruct 和 DeepSeek-V3-0324；还比较 CodeV、HaVen 和 DeepSeek-R1-Distill-Qwen-7B。生成温度统一为 0.6。

每个原始设计可按 15、50、104 或 210 个 RTL 样本的搜索规模运行。规则库学习阶段针对每个保留代码对默认生成 2 条候选规则。规则经多次重应用和等价/综合评估后进入库中，而不是未经筛选地直接用于新输入。

## 5. 奖励函数、损失函数或关键公式

论文没有训练损失函数，也没有使用 PPO、GRPO 等强化学习。规则质量使用等价门控的 PPA 评分：

```text
s_i = 1_eq(i) · clip( α + β · (PPA_n - PPA_i)/(PPA_n - PPA_o), 0, 1 )
```

其中 `PPA_n` 是未优化代码的指标，`PPA_o` 是代码对中的优化版本指标，`PPA_i` 是规则第 i 次重应用所得结果；`1_eq(i)` 只允许功能等价结果获得非零分。论文默认 `α=0.25`、`β=0.5`，平均分超过 0.7 的规则进入规则库。

最终优化结果使用面积、周期时延和动态功耗，并以 `Impr = 1 - PPA_opt/PPA_original` 计算收益。对于多样本搜索，论文提出 `Impr@k`，估计从 n 个样本中选 k 个时的期望最优收益。beam search 的候选分数由多样性分数与 PPA 分数组合而成。

## 6. 实验设置

主要综合流程使用 SiliconCompiler、Yosys、OpenSTA 和 FreePDK 45nm；跨工艺对比还使用商业 EDA 工具和 12nm、65nm 工艺。基线包括 RTLRewriter、人工 Verilog 工程师、普通 LLM 采样，以及 RTL 专用模型和通用模型。

规则库用 `gte_Qwen2-7B-instruct` 为规则和检索查询生成嵌入。ARAO 根据结构条件和动作检索三条规则。搜索配置分别为 `2-3-3`、`3-5-4`、`3-8-5` 和 `5-10-5`，对应 15、50、104 和 210 个搜索样本。

论文报告 60 个设计上的面积导向和时延导向优化，也报告与人工规则库的消融比较。代码对先经过功能等价检查，再经过综合评估，避免把不等价或 PPA 变差的结果算作有效提升。

## 7. 实验结果与结论

在面积导向设置中，AutoPPA-DeepSeek-V3 在最大搜索规模下达到 15.31% 的面积改善、0.43% 的时延改善和 15.97% 的功耗改善；在时延导向设置中，AutoPPA-Qwen 达到 11.28% 的时延改善、8.00% 的面积改善和 10.87% 的功耗改善。

相对于 vanilla DeepSeek-V3 采样，AutoPPA 在最大搜索规模下的面积收益高 4.11 个百分点，时延收益高 3.58 个百分点。与 RTLRewriter 对比，AutoPPA 在 11 个电路中的 10 个取得最小面积，论文报告平均面积收益提高 7.56%。与 SymRTLO 的自报告结果相比，AutoPPA 在给定的复杂电路和工艺设置上达到更高面积改善，但作者承认两者综合脚本和工艺并不完全一致。

规则库消融显示，自动 E²I 规则在 `Impr@50` 下达到 8.83% 面积、3.25% 时延和 9.97% 功耗改善，而人工规则为 6.56%、0.80% 和 5.46%。这说明规则库作为可复用中间能力，对后续 LLM 搜索有实际贡献。

## 8. 主要创新点

### 8.1 LLM 最终生成可复用优化规则

论文明确把规则定义为 `(snippet, condition, action)` 三元组。LLM 在 Induce 阶段直接输出解释 PPA 改善的规则候选；这些规则经过重应用评分后存入库，并在后续不同电路上检索和复用。根据 taxonomy v2 的角色定义，这一输出是可复用的 optimization rule/transform artifact，符合 `GENERATOR/G2`，而不仅是一次性 RTL 变换。

### 8.2 对规则进行等价和 PPA 双重筛选

规则必须在功能等价约束下取得足够 PPA 得分才能进入库。该机制把 LLM 的开放式代码归纳与确定性的测试、等价检查和综合反馈连接起来，减少了低质量规则污染。

### 8.3 规则检索与多步搜索结合

论文没有在每个新输入上盲目生成代码，而是先根据结构条件检索规则，再由 LLM 适配，最后进行 beam search。规则库因此成为可以脱离原始代码对重复使用的编译优化知识层。

## 9. 局限性

论文的优化对象是 RTL 电路，不是 LLVM IR、MLIR 或 RISC-V 指令。规则主要由 LLM 以自然语言三元组表示，实际代码仍由后续 LLM 根据规则生成，因此规则的确定性和可执行性弱于直接编译器 DSL 中的 rewrite pattern。

功能等价依赖测试台和工具链；论文报告高覆盖率，但没有把所有规则都用形式化等价证明验证。商业 EDA 对比与 SymRTLO 的工艺和脚本不完全相同，跨论文数字不能视为严格公平比较。

规则库可能出现相互矛盾、过度具体或过度抽象的规则。检索依赖嵌入相似度和 LLM 的条件概括，错误检索会导致不等价或低收益改写。论文也没有展示规则在更大规模真实 RTL 工程、不同 HDL 或不同综合器之间的长期稳定性。

## 10. 阅读后的研究方向反思

本文对 AI 编译器研究最直接的启发是把“模型生成代码”和“模型生成可复用变换知识”分开。若将 AutoPPA 的规则三元组映射到 MLIR rewrite pattern、LLVM peephole rule 或 RISC-V backend pattern，规则就能成为编译器基础设施中的持久中间产物。

但 RTL PPA 规则不能直接等同 LLVM 或 RISC-V 规则。编译器规则还需要处理类型、控制流、内存别名、副作用、寄存器约束和目标指令成本。RISC-V/RVV 场景尤其需要将 VLEN、LMUL、VL/VTYPE 状态、尾部策略和掩码语义写入条件，避免自然语言规则在不同硬件配置下失效。

## 11. 可进一步尝试的研究方向

### 11.1 LLVM/MLIR 可执行规则生成

构建 `snippet/condition/action` 到 MLIR pattern 或 LLVM matcher 的结构化映射。LLM 只生成候选规则，MLIR verifier、Alive2 和编译测试负责检查合法性与等价性；规则还需在多程序上测量复用率和实际性能收益。

### 11.2 RISC-V/RVV 目标感知规则库

从 LLVM IR 到 RVV 指令序列的等价改写对中归纳规则，将指令选择、寄存器压力和 VTYPE/VL 状态作为规则条件。不同 VLEN 的仿真和真实板卡测量可用于筛选可迁移规则。

### 11.3 规则库的冲突消解和版本化

为每条规则保存来源代码对、验证条件、目标工艺/ISA、收益分布和适用范围；对相互冲突的规则建立优先级和回滚机制，减少规则库持续扩张后的检索噪声。

## 12. 与其他已读文献的关系

AutoPPA 与本地已有的 RuleFlow、SemOpt、ASPEN、Code Transformation Rule Synthesis 等 G2 工作同属“生成可复用规则或变换能力”方向，但对象分别是 Pandas、通用代码优化、RTL e-graph 和软件演化 DSL。与 SymRTLO 相比，AutoPPA 的差别在于显式构造并筛选规则库；SymRTLO 的最终主要输出仍是单个 RTL 改写，因此已按 TRANSLATOR/T1 归类。

AutoPPA 与 LLM-VeriOpt 等 Translator 工作的边界也很清晰：后者的模型最终输出优化 LLVM IR；AutoPPA 的论文贡献同时包含 LLM 生成的规则三元组及规则库，并将规则重新应用到后续输入。即使最终阶段也生成优化 RTL，规则库这个持久化中间产物使其具备 Generator/G2 角色依据。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 研究任务 | 从等价 RTL 代码对中归纳可复用 PPA 优化规则 |
| 主要输入 | Verilog/RTL 代码、LLM 改写代码对、综合 PPA 标签 |
| LLM 最终可复用输出 | `(snippet, condition, action)` 优化规则及规则库 |
| 主分类 | `GENERATOR` |
| 二级分类 | `G2_Optimization_Rule_Transform_Generation` |
| 角色证据 | 规则经评分后写入库，并检索、适配、重应用到后续电路 |
| 关键流程 | Explore → Evaluate → Induce → Retrieve/Adapt → Beam Search |
| 验证 | 测试台/Yosys 功能等价检查与综合 PPA 评估 |
| 数据规模 | RTLRewriter 54 个设计加 6 个拆分模块，共 60 个设计 |
| 主要模型 | Qwen2.5 系列、DeepSeek-V3、CodeV、HaVen 等 |
| 最佳报告结果 | 面积改善 15.31%，时延改善 11.28%，功耗改善 15.97% |
| 代码状态 | `PUBLIC_REPO`；<https://github.com/j-silv/autoppa>；许可证未确认 |
| 主要局限 | RTL/PPA 专用；规则自然语言化；测试等价不等同形式化证明 |
| 对本语料库价值 | Generator/G2 的新候选，适合研究规则生成与规则库复用 |

结论：AutoPPA 通过 LLM 生成并筛选可复用的 `(snippet, condition, action)` 规则，将规则库作为持久化编译优化能力用于后续 RTL 输入。全文证据支持 `GENERATOR/G2`，但其硬件 RTL 目标与 LLVM/MLIR/RISC-V 编译器仍存在迁移鸿沟。
