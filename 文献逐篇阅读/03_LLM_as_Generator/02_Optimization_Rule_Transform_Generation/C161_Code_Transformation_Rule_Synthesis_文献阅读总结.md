# Code Transformation Rule Synthesis 文献阅读总结

论文题目：**Code Transformation Rule Synthesis using LLMs: Potential and Limits**

作者：Axel Allain、Aymeric Blot、Djamel Eddine Khelladi、Mathieu Acher

发表时间：2026

发表平台：ASE 2026 Research Papers（已列入正式论文页；会议尚未举行）

论文链接或编号：DOI 10.1145/3832783.3837504；arXiv:2609.03592
元数据核验来源：[ASE 官方作者/论文页](https://conf.researchr.org/profile/ase-2026/mathieuacher)；[arXiv](https://arxiv.org/abs/2609.03592)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：Large Language Models、Transformation Rules、Comby、Ast-Grep、GritQL、Program Repair、API Migration

> 本文档用于文献阅读、组会汇报和后续研究分析。论文事实与阅读后的研究思考分开描述。

## 1. 研究背景

软件维护中经常出现重复性的 API 修复、程序修复、重构和版本迁移。直接让大语言模型（Large Language Model，LLM）逐个文件改代码，虽然能够处理复杂变化，但输出缺少确定性、可解释性和规模化复用能力。传统文本替换或正则表达式又难以表达语法结构，容易误匹配。

论文关注代码转换领域特定语言（Domain-Specific Language，DSL）中的规则，例如 Comby、Ast-Grep 和 GritQL。这类规则通常由匹配模式、重写模式及元变量组成，可以把一次代码变化抽象成可重复执行的转换。论文的动机是：让 LLM 输出“可复用转换规则”，而不是直接输出某个文件的一次性修改结果，从而兼顾 LLM 的模式归纳能力与规则系统的确定性。

## 2. 论文要解决的问题

### 2.1 能否生成可执行规则

给定真实代码变化的 before/after diff，LLM 能否生成满足目标 DSL 语法、并能被对应规则引擎执行的转换规则？

### 2.2 规则是否具有完整性与复用性

生成的规则是否覆盖代码变化的主要结构，是否使用元变量形成抽象，是否能在同一数据集的其他实例上再次匹配，而不是只记住单个样例？

### 2.3 规则应用结果是否接近人工变化

规则应用后的代码与人工编写的目标代码在文本、AST（抽象语法树）和测试行为层面有多接近？

### 2.4 不同模型和 DSL 的差异是什么

论文比较 GPT-5.4、GPT-oss-120B 和 Llama3.1-8B，并比较 Comby、Ast-Grep、GritQL 三种 DSL；同时将 LLM 与 anti-unification（反统一）基线比较。

> 本文主要研究：如何从真实代码 diff 中合成可执行、可解释且可复用的 DSL 代码转换规则，以及这种能力在不同模型、任务和 DSL 下的边界。

## 3. 核心方法概述

论文提出的是一项系统实证研究，而非新的编译器优化 pass 实现。其核心流程是将代码变化示例交给 LLM，要求 LLM 生成目标 DSL 的匹配/重写规则，再运行规则并评估结果。

```text
真实代码变化（V1 before、V2 after）
        ↓
生成 unified diff 与 DSL 文档/示例
        ↓
LLM 生成 Comby、Ast-Grep 或 GritQL 规则
        ↓
DSL 引擎检查并应用规则到 V1
        ↓
比较规则输出与人工 V2
        ↓
统计可执行性、复用性、AST/文本一致性与测试修复率
```

LLM 的角色是规则生成器：输出可保存、可审查、可重复应用的 match/rewrite 规则。对于 Comby，论文还提供了可选 RAG（Retrieval-Augmented Generation，检索增强生成），从 MELT 收集的 1,700 多条 Comby 规则中检索 8 条相关示例。Ast-Grep 与 GritQL 使用固定的官方文档示例，不使用同等规模的 RAG。

## 4. 实验框架与训练流程

### 4.1 模型与提示

论文评估 GPT-5.4、GPT-oss-120B 和 Llama3.1-8B。三者温度均设为 0，以降低输出波动。论文没有为本研究训练一个新的规则生成模型，主要采用提示推理，并向提示加入目标 DSL 的 8 个规则示例。

### 4.2 规则生成与执行

对每个 before/after 代码对，模型生成一个或多个目标 DSL 规则。规则先通过 DSL 解析/执行检查，再应用到 before 代码。应用结果与 ground truth after 代码比较。

### 4.3 RAG 条件

Comby 有 RAG 与 no-RAG 两种条件；RAG 从 MELT 规则库检索 8 条示例。Ast-Grep 与 GritQL 只使用固定的 8 条官方文档示例。因此，RAG 结果只在 Comby 内部比较，不能直接解释为三种 DSL 的统一 RAG 对比。

### 4.4 统计检验

论文对 Rule Applicability、Exact Match 和 AST Match 使用配对 McNemar 检验，对 Tree Distance 使用 Wilcoxon signed-rank 检验，并对多重比较使用 Benjamini-Hochberg 校正；规则数、规则大小等聚合指标使用 sign test。

本文不涉及 SFT、PPO、GRPO 或强化学习奖励函数；没有模型训练阶段，主要是固定温度的提示推理、规则执行和离线评测。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习，因此不存在强化学习奖励函数或训练损失函数。论文的关键是评测指标：

| 指标 | 含义 |
|---|---|
| `F` | 规则不可应用、DSL 无法执行、输出有语法错误或指标无法计算的失败数 |
| `RA%` | Rule Applicability，语法有效且可被 DSL 引擎执行的规则比例 |
| `NT` | LLM 生成规则时消耗的 token 数 |
| `#R` | 每个代码变化生成的平均规则数 |
| `SR` | 规则大小，论文按字符序列长度统计 |
| `MTR` | Metavariable Token Ratio，匹配模式中元变量 token 所占比例 |
| `EM` | 应用规则后的代码与人工目标代码完全文本匹配的比例 |
| `AM` | 应用结果与人工目标代码 AST 完全匹配的比例 |
| `TD` | 两棵代码树之间的 Tree Edit Distance；论文表格将其报告为相似度分数 |
| `Reuse score` | 将正确规则应用于同一数据集全部实例时额外匹配的次数/统计量 |

## 6. 实验设置

### 6.1 数据集来源

论文使用真实、人工编写的 before/after 代码对，覆盖六个数据集：

| 数据集 | 语言 | 规模 | 典型变化 |
|---|---:|---:|---|
| Galappaththi et al. API Misuse | Python | 43 | 语句级 API misuse 修复 |
| ManySStuBs4J | Java | 528 | 单语句 bug 修复子集 |
| Defects4J | Java | 427 | 函数级 bug 修复子集 |
| BugsInPy | Python | 559（测试表相关结果出现 493 的有效子集） | 函数级 bug 修复 |
| PyMigBench | Python | 605（表 5 结果出现 661 的评估口径） | 多语句 API 迁移 |
| JMigBench | Java | 45 | Java 8 到 Java 11 的方法级迁移 |

ManySStuBs4J 与 Defects4J 为受资源限制抽取的随机子集；PyMigBench 来自 3,096 个迁移变化，实验取 605 对。Defects4J 和 BugsInPy 具有可执行测试套件，其他数据集主要使用代码/AST 对齐指标。论文没有报告 LLVM IR、GCC 或 RISC-V 数据。

### 6.2 模型与工具

模型为 GPT-5.4、GPT-oss-120B、Llama3.1-8B。转换 DSL 为 Comby、Ast-Grep、GritQL；RAG 规则库来自 MELT。Ast-Grep 与 GritQL 依赖 Tree-sitter AST 能力。语义测试使用 Defects4J 和 BugsInPy 的测试套件。论文未把 LLVM、Alive2 或 RISC-V 作为实验工具。

### 6.3 对比方法

主要对比包括三个 LLM、Comby 的 RAG/no-RAG 条件，以及 anti-unification。Anti-unification 从代码样例对中抽取共同结构并以占位变量泛化，但属于确定性结构算法，不具备 LLM 的上下文理解能力。

### 6.4 评价指标

论文从四个层面评估：规则是否能执行（F、RA、NT）、规则结构是否完整和抽象（#R、SR、MTR）、应用结果是否匹配目标变化（EM、AM、TD），以及在 Defects4J/BugsInPy 上是否通过测试修复 bug（attempted repairs、successful repairs、success rate）。

## 7. 实验结果与结论

### 7.1 主要结果

正文表 2 显示，GPT-5.4 的 RA 范围为 68.2%–100%，在 Ast-Grep、GritQL 和 Comby+RAG 的多数配置中失败较少；GritQL 在三个模型和多个数据集上通常最稳定，常接近或达到 100%。GPT-oss-120B 在 Comby no-RAG 上有竞争力，并且生成 token 通常比 GPT-5.4 少约 20%–40%。

### 7.2 规则结构与复用

表 3 显示 GPT-5.4 通常产生更大的、更显式的规则，规则大小在 JMigBench no-RAG Comby 条件下达到 285.1（论文定义的规则长度单位），但 MTR 较低；GPT-oss-120B 规则更紧凑、元变量比例更高，MTR 在多个程序修复条件达到约 19%–20%。Anti-unification 的 MTR 最高，可达 62，但论文认为这经常导致过度泛化和难读规则。

表 4 的复用实验表明，许多 LLM 规则能够在同一数据集其他实例上再次匹配；但高复用往往由少数重复模式驱动，且过于通用的规则可能换来较低的 AM/EM。论文因此将“抽象程度—覆盖范围—准确性”视为需要平衡的因素。

### 7.3 与基线及不同模型比较

表 5 中 GPT-5.4 在多数数据集/DSL 配置的 EM、AM、TD 上最好。例如 Defects4J 的 Ast-Grep AM 为 49.4%，BugsInPy 的 Ast-Grep AM 为 54.1%；JMigBench 的 Comby+RAG EM/AM 为 60.0%。GPT-oss-120B 在简单任务和 Comby 条件竞争力较强；Llama3.1-8B 在复杂变化上常低于 30%，Defects4J 某些条件低至 5.2%。Anti-unification 最佳 EM/AM 不超过 36.4%，在 Defects4J 等复杂数据集可低至约 1.2%–2.4%。

### 7.4 测试验证与消融

表 6 显示，当规则输出没有 AST Match 时，仍有少量修复能通过 Defects4J/BugsInPy 测试，但成功率很低。例如 Defects4J 上 GPT-oss-120B + GritQL 有 8 次成功修复、成功率 3.0%；BugsInPy 上 GPT-5.4 + Ast-Grep 有 38/212、成功率 17.9%。论文没有把 AST Match 等同于完整语义证明。

RAG 不是普遍有效：作者观察到 Comby RAG 可能因规则库偏向 Python 或相关性不均而降低性能。论文还报告 GPT-5.4 相对 GPT-oss-120B 在 24 个配置中的 12 个配置上 AST Match 显著更好，而 GPT-oss-120B 只在 2 个配置上显著胜出。

### 7.5 失败案例

失败主要来自 DSL 配置格式、额外自然语言注释、缺少 Ast-Grep 规则分隔符、试图匹配多个不相关 AST 节点、遗漏编辑、重复规则和规则过度具体/过度泛化。复杂的函数级修复、多语句迁移和需要插入新语句的变化尤其困难。

## 8. 主要创新点

### 8.1 创新点一：把 LLM 输出从一次性代码修改提升为可复用规则

论文研究的对象不是单个 transformed file，而是可保存、可审查、可在代码库中重复应用的 Comby/Ast-Grep/GritQL 规则。这与 Generator 角色相符：模型生成的是 reusable transformation artifact。

### 8.2 创新点二：跨 DSL、跨任务、跨模型的系统评估

论文在三个规则 DSL、三个规模不同的 LLM、四类软件演化任务和六个数据集上统一报告可执行性、抽象性、复用性、结构匹配与测试结果，较清楚地揭示了 DSL 表达能力和任务复杂度的影响。

### 8.3 创新点三：实证刻画抽象与准确性的张力

论文发现 GPT-5.4 倾向更大、更显式的规则，GPT-oss-120B 倾向更紧凑、更多元变量的规则，而 anti-unification 容易过度泛化。该结果为后续自动 pass/rule 生成系统设计“受控泛化”提供了直接证据；统计结果也支持模型差异并非完全来自偶然波动。

## 9. 局限性

### 9.1 论文明确承认的局限

论文指出，提示词、温度、示例选择和 DSL 表达力都会影响结果；不同 DSL 的直接比较并不完全公平。规则应用率和 AST/文本匹配不能充分代表语义正确性，只有 Defects4J 和 BugsInPy 提供了测试验证。研究只覆盖 Java 与 Python，无法代表所有语言。数据泄漏风险也无法完全排除，因为专有模型的训练数据不可见。复杂代码库、深层嵌套结构和全局上下文变化可能降低可扩展性。

### 9.2 阅读后发现的潜在局限

该论文的“转换规则”主要面向软件维护/演化 DSL，不是 LLVM pass、MLIR rewrite pattern 或 RISC-V backend rule；因此它不能直接证明生成的规则能保持编译器语义或改善机器码性能。实验主要以重现人工变化为目标，EM/AM 可能奖励表面一致而不是性能收益。论文中的成功修复依赖既有测试，不能等同形式化等价证明。另一个需要注意的内部口径问题是：表 1 与表 5 对 BugsInPy、PyMigBench 的有效样本数存在差异，使用结果时应以具体表格条件为准。

## 10. 阅读后的研究方向反思

值得借鉴的是“LLM 生成可审查规则，执行器负责批量应用”的分工，以及对 rule applicability、元变量比例、复用率和正确性的分层评测。对于 AI 编译器，该思想可以迁移到 MLIR pattern、LLVM InstCombine 规则或后端 peephole pattern 的候选生成。

不能简单照搬的是把 Java/Python DSL 规则直接当作编译器优化 pass。编译器规则还必须满足 IR 合法性、支配关系、别名/内存语义、目标指令约束及性能收益。仅替换语言或平台为 RISC-V 也不足以形成新贡献；真正的新问题应是如何让规则感知 RVV 的向量长度、LMUL、尾部策略、别名分析和目标成本模型，并通过等价验证与真实硬件性能筛选。

因此，本文更适合作为 G2 的规则生成方法参考和评测设计参考，而不是 RISC-V 优化 baseline。它对 Generator 角色的直接价值在于：输出一个可复用的 transformation rule，而不是只输出一次性修改后的程序。

## 11. 可进一步尝试的研究方向

### 11.1 LLVM/MLIR 规则 DSL 的受控生成

#### 研究问题

LLM 能否从 IR before/after 对生成可编译、可验证、可复用的 LLVM/MLIR rewrite pattern？

#### 与原论文的区别

输入从 Java/Python 代码 diff 改为带类型、控制流和分析约束的 IR；输出要求能注册为 pass pattern，并通过 Alive2 或 MLIR verifier。

#### 可能的创新点

引入类型/支配/副作用约束，显式区分匹配条件、重写动作和收益条件。

#### 实验框架

```text
IR 变化对 → LLM 生成 rewrite rule → verifier/编译器检查 → benchmark 性能筛选 → 统计复用与收益
```

#### 可行性

需要 LLVM/MLIR、Alive2 或等价验证器、公开 pass 数据和可重复的 microbenchmark。

#### 主要风险

规则通过局部等价验证并不保证全局收益；复杂 side effect 和分析失效可能导致规则不可应用。

### 11.2 面向 RISC-V/RVV 的 peephole rule synthesis

#### 研究问题

LLM 能否生成带 RVV 指令约束的 peephole 规则，并在不同 VLEN/LMUL 配置下保持正确和可迁移？

#### 与原论文的区别

目标不是一般代码迁移，而是输出可复用的后端 pattern/selector rule，并把 ISA 语义与硬件成本纳入筛选。

#### 可能的创新点

构建跨 VLEN 的规则语料，联合指令语义验证、寄存器约束检查和真实 RVV 测量。

#### 实验框架

```text
LLVM/RVV IR 对 → 生成后端规则 → ISA/寄存器合法性检查 → 仿真器/真实 RVV 测量 → 规则保留或修订
```

#### 可行性

需要 LLVM RISC-V backend、QEMU 或实际 RVV 板卡、指令语义测试集和性能计数工具。

#### 主要风险

仿真器结果与真实硬件可能不一致；VL/VTYPE 状态和尾部策略会使局部规则失效。

## 12. 与其他已读文献的关系

本次只完成这一篇论文的正文阅读，因此不把正式 corpus 中已有论文的未重新阅读内容当作本节证据。就任务定位而言，本文与 RuleFlow、SemOpt、ASPEN 等 G2 候选属于同一大方向的“规则/变换生成”，但本论文研究的是通用软件演化 DSL，未进入 LLVM/RTL 专用优化器。它可以作为规则生成质量指标和失败模式的补充参考；若与 LLVM/MLIR 或 RTL 规则生成论文组合，需分别保留语言层级、语义验证和性能评价的差异。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 从真实代码 diff 合成可复用 DSL 转换规则 |
| 核心问题 | LLM 能否生成可执行、正确、可泛化的规则 |
| 输入 | Java/Python before/after 代码对及 diff |
| 输出 | Comby、Ast-Grep、GritQL match/rewrite 规则 |
| 核心方法 | LLM 提示生成 + DSL 执行 + 多层规则评测 |
| 使用的模型 | GPT-5.4、GPT-oss-120B、Llama3.1-8B |
| 使用的编译器工具 | Comby、Ast-Grep、GritQL、MELT；非 LLVM |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；使用 DSL 执行、AST 比较与测试验证 |
| 数据集规模 | 六个数据集；表 1 合计约 2,207 对，具体评测表有有效子集 |
| 主要指标 | RA、NT、#R、SR、MTR、EM、AM、TD、Reuse、测试修复率 |
| 最重要实验结果 | GPT-5.4 RA 为 68.2%–100%，复杂任务 AST/文本匹配总体最佳；GritQL 最稳定 |
| 核心创新 | 评估 LLM 生成可复用代码转换规则的能力与边界 |
| 主要局限 | 非编译器专用规则；仅 Java/Python；语义证明与性能收益有限 |
| 与 RISC-V 研究的相关性 | 中-低：规则生成范式可借鉴，但未涉及 RVV、LLVM 或机器码性能 |
| 最适合作为 | G2 方法参考、规则生成评测与失败模式 baseline |

> 这篇论文最值得学习的是把 LLM 的一次性代码改写转化为可审查、可批量复用的规则生成问题；最主要的局限是其规则 DSL 与编译器 IR/pass 之间仍有语义和性能鸿沟；如果用于后续研究，最合理的使用方式是借鉴其受控泛化和分层评测，再面向 LLVM/MLIR/RISC-V 规则重新建模，而不是直接把 Java/Python 规则迁移到后端。
