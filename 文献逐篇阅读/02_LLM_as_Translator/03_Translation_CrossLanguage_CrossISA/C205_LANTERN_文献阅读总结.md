# LANTERN 文献阅读总结

论文题目：**Unlocking LLM Repair Capabilities Through Cross-Language Translation and Multi-Agent Refinement**

作者：Wenqiang Luo；Jacky Wai Keung；Boyang Yang；Jacques Klein；Tegawendé F. Bissyandé；Haoye Tian；Bach Le

发表时间：2026（ICSE ’26）；arXiv 首次公开 2025，当前正文为 arXiv:2503.22512v5（2026-03-30）

发表平台：2026 IEEE/ACM 48th International Conference on Software Engineering (ICSE ’26)，13 页

论文链接或编号：[arXiv:2503.22512](https://arxiv.org/abs/2503.22512)；DOI [10.1145/3744916.3773227](https://doi.org/10.1145/3744916.3773227)

代码：`PUBLIC_REPO` — [https://github.com/stringing/LANTERN](https://github.com/stringing/LANTERN)，根据论文第 1 节 Availability 声明核验，日期 2026-09-22。

关键词：Automated Program Repair、Large Language Model、Cross-Language Translation、Multi-Agent Refinement、xCodeEval、historical feedback

> 本文档基于本轮 staging 的官方 arXiv PDF 正文阅读。论文事实、阅读后的分析和后续建议分开记录；未将有限测试结果表述为形式化等价证明。

---

## 1. 研究背景

本文属于大语言模型辅助自动程序修复（Automated Program Repair, APR）和跨语言代码翻译交叉领域。LLM 已能生成复杂代码修复，但论文指出修复能力在不同编程语言之间存在明显差异：主流语言拥有更多训练数据和社区支持，Rust、Kotlin 等语言的修复性能较弱。引言以 xCodeEval 为例，初始 LLM 在 Python、PHP 上的 Pass@10 为 89.02% 和 89.93%，Rust 仅为 65.58%，差距超过 24 个百分点。

传统 APR 通常在原语言中定位、生成和验证补丁；多轮修复也常局限在同一种语言上下文中。论文的动机是：若模型在原语言中难以修复某个缺陷，可以把程序翻译到模型更擅长的另一种语言，在该语言中修复，再翻译回原语言。编译器/解释器和测试执行器提供的结果被用于判断候选是否通过，而不是用于形式化证明。

## 2. 论文要解决的问题

### 2.1 低资源语言的修复能力差距

论文研究如何缓解 LLM 在不同编程语言之间不均衡的程序修复能力，尤其关注 Rust、Kotlin、Go、Ruby 等相对低资源或语言约束不同的语言。

### 2.2 原语言内反复修复的局部最优

论文假设：当直接修复在原语言中失败时，切换语言上下文可能暴露算法、类型、内存或语法问题的新视角。目标语言不应固定，而应根据 bug 特征和历史修复反馈动态决定。

### 2.3 翻译与回译造成的语义一致性风险

跨语言翻译可能改变错误类别、资源行为和程序语义。因此论文还研究 bug 翻译前后测试结果的一致性，以及目标语言修复代码回译到源语言后还能否保持正确。

> 本文主要研究：如何利用 LLM 直接生成跨语言翻译和修复代码，并结合测试反馈与历史记录，在多轮流程中提高自动程序修复成功率。

## 3. 核心方法概述

LANTERN（cross-LANguage Translation and multi-agEnt RefiNement）由 repairer、middleware、analyzer 和 translator 组成。初始修复先在原语言中尝试；失败样本进入跨语言修复流程。analyzer 根据 bug 特征、历史修复记录和已尝试语言选择下一目标语言；translator 由 LLM 执行源语言到目标语言的代码翻译，并在目标语言中继续修复；通过测试的目标代码再由 LLM 回译到源语言。

```text
源语言中的缺陷程序
        ↓
LLM repairer 直接生成源语言修复候选
        ↓
编译/执行/测试验证
        ↓ 失败样本进入 middleware
历史修复记录 + bug 特征检索
        ↓
LLM analyzer 选择尚未尝试的目标语言
        ↓
LLM translator 生成目标语言缺陷程序
        ↓
目标语言中的 LLM repairer 生成修复程序
        ↓
编译/执行/测试验证
        ↓
LLM translator 回译到源语言
        ↓
下一轮验证与 refinement
```

根据第 4.1–4.4 节，LLM 的角色有三类：repairer 生成修复程序，analyzer 生成目标语言选择及理由，translator 直接生成目标语言代码和回译后的源语言代码。最终可交付对象是修复后的程序，而不是一个 pass、配置或自然语言建议，因此拟分类为 `TRANSLATOR/T3_Translation_CrossLanguage_CrossISA`。本文没有跨 ISA 实验；T3 中的实际方向是 cross-language source translation/repair。

## 4. 实验框架与训练流程

### 4.1 系统运行流程

本文不涉及模型预训练、SFT、PPO 或其他参数更新训练，主要采用现成 LLM 的提示推理和外部测试反馈。每个 bug 在单次迭代中每个目标语言最多使用一次；实验最大迭代次数为 11，对应 xCodeEval 的 11 种语言。

### 4.2 Middleware

Middleware 包含四个组件：

1. Translation Coordination：接收评估结果并调度未修复 bug。
2. Historical Data Storage & Retrieval：把初始直接修复和后续翻译修复的结果、bug 特征、成功语言等编码到本地向量数据库。
3. Prompt Construction：按任务构造修复、目标语言决策、代码翻译和回译提示。
4. Process Control：监控迭代上限等终止条件。

### 4.3 Analyzer 与 Translator

Analyzer 使用 bug 查询与向量数据库中的 top-k 相似历史记录，为目标语言决策提供上下文。历史直接修复记录反映哪些语言适合相似 bug；历史翻译修复记录反映哪些目标语言曾成功修复相似 bug。Translator 使用 source language、target language、代码和翻译指令生成翻译或回译结果。

### 4.4 Prompt 设计

修复提示包含问题描述、错误类型、输入输出规格、buggy code 和修复指令。翻译提示指定源/目标语言和待翻译代码。目标语言决策提示采用 zero-shot chain-of-thought，整合 bug 特征、历史反馈、候选语言范围和已尝试语言约束。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有训练阶段的损失函数。系统主要通过外部编译、执行和测试结果判断候选是否成功，并通过历史记录影响后续提示。

论文使用的关键评价定义包括：

```text
Pass@k = top-k 个生成补丁中至少有一个修复 bug 的概率
```

实验使用 `Pass@10`，其中每个 bug 生成 `n=20` 个候选。目标语言选择还使用 Precision@k、MAP@k、NDCG@k、Recall@k 和 F1@k，评价排序靠前的语言选择中有多少对应有效修复迭代。

这些是评测指标，不是用于训练 LLM 的奖励函数。论文没有给出一个将正确性、性能或成本加权的显式优化目标。

## 6. 实验设置

### 6.1 数据集来源

主要数据集为 xCodeEval Compact Set，来源于 Codeforces，包含 11 种语言共 5,068 个 bug，难度等级为 800–3,500。各语言数量为：C 733、C# 739、C++ 641、Go 294、Java 716、JavaScript 183、Kotlin 313、PHP 191、Python 710、Ruby 343、Rust 205，总计 5,068。每个问题平均约 50 个测试，ExecEval 将执行结果分为 compilation error、runtime error、memory limit exceeded、time limit exceeded、wrong answer 和 passed。

RQ4 使用：

- Defects4J v1.2 的 255 个单函数 bug 子集；
- Defects4J v2.0 的 78 个过滤后 bug；
- SWE-Bench Lite 的 300 个真实 GitHub issue。

论文未把这些数据集转换为新的训练集；主要是推理、编译/执行和测试验证。潜在风险包括翻译引入语义变化、不同语言编译器产生不同错误类别，以及历史反馈可能造成选择偏置。

### 6.2 模型与工具

主实验在 analyzer、translator 和 repairer 中统一使用 DeepSeek-V3。泛化实验使用 Claude 3.5 Sonnet 和 Qwen2.5-72B-Instruct。论文正文明确给出了这些模型名称，但没有在本文中给出本地部署量化配置或编译器版本的完整清单。

执行环境使用 xCodeEval 的 ExecEval，为不同语言提供编译器/解释器和运行环境；Defects4J 与 SWE-Bench 使用论文对应的 benchmark 设置。论文没有提出 LLVM、GCC、RISC-V、Alive2 或硬件性能测试。

### 6.3 对比方法

RQ1.1–RQ1.2 比较三种目标语言选择策略：

- Greedy：按初始直接修复阶段的历史语言性能排序；
- Random：每轮随机选择目标语言；
- Reasoning：LLM 根据 bug 特征和历史反馈决定目标语言。

RQ1.3 的 APR baseline 包括 direct repair、ChatRepair、Self-Planning 和 Self-Collaboration。Defects4J 对比 ChatRepair，SWE-Bench Lite 对比 AGENTLESS。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| Pass@10 | 20 个候选中前 10 个内至少一个成功修复的概率 | 越大越好 |
| Precision@k | 前 k 次语言选择中有效迭代的比例 | 越大越好 |
| MAP@k | 目标语言有效排序的平均精度 | 越大越好 |
| NDCG@k | 对更早出现的有效语言赋更高权重的排序质量 | 越大越好 |
| Recall@k | 前 k 次选择覆盖有效修复语言的比例 | 越大越好 |
| F1@k | Precision@k 和 Recall@k 的调和平均 | 越大越好 |
| Resolved instances | 真实 issue 中被成功解决的数量/比例 | 越大越好 |

## 7. 实验结果与结论

### 7.1 主要结果

在 xCodeEval 上，reasoning strategy 相比 greedy 和 random 更早选择有效目标语言。其平均翻译路径长度为 2.52，优于 random 的 2.69 和 greedy 的 2.98；前五轮 MAP@k 收敛到 0.938，前五轮 Recall@k 达到 1.00。这里的路径长度和排序指标衡量语言选择效率，不是程序运行时间。

在不同源语言上，翻译修复带来的最终 Pass@10 增益中，Rust 最大，为 22.09%；Go、PHP、Kotlin、C#、Ruby 分别为 11.31%、9.33%、8.87%、8.82%、8.33%。论文报告 Mann–Whitney–Wilcoxon 检验 p=0.0126，Cliff’s Delta=0.64。

### 7.2 与直接修复和 APR 方法比较

LANTERN 在 xCodeEval 的 11 种语言中有 9 种取得最高 Pass@10。相对于 ChatRepair，LANTERN 在 Go、Kotlin、PHP 上分别提高 5.46%、4.91%、6.76%。Self-Collaboration 在部分语言上有效，但在 C++、Java、PHP 上出现下降，PHP 最大下降 26.48%。

在真实 benchmark 上，LANTERN 在 Defects4J v1.2 修复 170 个 bug，ChatRepair 为 141 个；Defects4J v2.0 为 60 对 54。在 SWE-Bench Lite 上，LANTERN 解决 110/300（36.67%）实例，AGENTLESS 为 95/300（31.67%）。这些结果是在论文指定的 benchmark 子集和模型设置下得到，不能直接外推为所有仓库级修复任务的提升。

### 7.3 翻译一致性与回译

在 32,277 个测试结果中，20,750 个粗粒度错误类别保持一致；其中 PASSED 一致 12,225 个，WRONG ANSWER 一致 3,881/4,318，TIME LIMIT EXCEEDED 的 3,021 个结果全部保持一致。COMPILATION ERROR 和 RUNTIME ERROR 更容易因不同语言编译器/运行时而改变。

回译后，论文报告固定 bug 的总体损失为 7.56%，正确样本保留率为 85.95%；回译前后正确样本的统计检验 p=0.39、Cliff’s Delta=0.22。该结果是测试驱动的语义一致性近似，不是形式化语义证明。

### 7.4 消融实验

去掉跨语言翻译后，Rust 的 Pass@10 相比直接修复只少了 16.55% 的翻译增益；PHP、Go、Ruby 分别为 7.27%、6.96%、5.71%。论文报告翻译修复相对无翻译策略的统计检验 p=1.95×10^-3，Cliff’s Delta=0.42。

去掉 historical feedback 后，最终性能接近，但早期迭代明显变差。例如 Rust 第一轮 Pass@10 为 0.82，而不使用历史反馈为 0.74。说明历史反馈主要帮助更早选择有效目标语言，而不是必然提高所有迭代结束后的上限。

### 7.5 模型泛化

Claude 3.5 Sonnet 上，LANTERN 在每种语言都高于 direct repair；Kotlin、PHP、Rust 的增益分别为 16.94%、17.95%、18.30%。Qwen2.5-72B-Instruct 的 Rust 从 direct repair 的 45.42% 提高到 72.00%，C#、Go、PHP 增益分别为 10.42%、10.13%、11.67%。

## 8. 主要创新点

### 8.1 创新点一：把跨语言翻译作为 APR 的修复通道

现有方法通常在原语言内迭代修复。LANTERN 将失败的 buggy code 翻译到另一个目标语言，在该语言中修复，再回译到原语言。创新不在于单独使用 LLM 翻译，而在于把翻译放入 APR 的失败恢复流程，并以测试结果闭环验证。

### 8.2 创新点二：基于 bug 特征和历史反馈的目标语言决策

系统使用 LLM analyzer，结合相似 bug 的历史直接修复记录、翻译修复记录和已尝试语言，动态生成下一目标语言。论文的消融与排序指标表明，该设计主要改善早期语言选择效率。

### 8.3 创新点三：多智能体式的翻译—修复—回译迭代

repairer、analyzer 和 translator 在流程中承担不同任务，middleware 负责记录、调度和提示构造。论文通过 ablation 说明翻译本身和 historical feedback 都对最终修复有效，但没有提出新的参数训练算法。

## 9. 局限性

### 9.1 论文明确承认的局限/威胁

- Historical feedback 随迭代积累，可能使后续推理偏向过去成功的语言；论文限制每个语言在单个 bug 的一次迭代中只能选择一次，并比较 greedy/random 策略。
- Pass@10 不能覆盖程序修复的全部维度；论文增加其他排序指标，但仍以测试通过作为主要正确性信号。
- 翻译可能造成语义不一致，尤其是编译错误、运行时错误和内存行为。论文通过测试结果一致性和回译检查进行缓解。

### 9.2 阅读后的潜在局限

- 目标语言选择、翻译、修复和回译都由 LLM 完成，错误可能在多个阶段累积；测试集未覆盖的行为仍可能错误。
- xCodeEval 主要是程序题 bug，真实仓库实验使用局部函数或上下文窗口；复杂构建系统、跨文件 API、宏、依赖和配置迁移尚未被充分覆盖。
- 论文没有 LLVM IR、汇编、RISC-V 或真实硬件性能实验，因此不能直接证明该方法适用于低层编译优化或跨 ISA 翻译。
- 历史向量检索和最多 11 轮翻译会增加 token、编译/测试和调用成本；论文没有给出完整成本曲线。
- “语义保持”是由测试结果近似衡量，不等同于形式化等价；不同语言运行时资源差异还可能把性能/内存变化误判为修复效果。

## 10. 阅读后的研究方向反思

LANTERN 最值得借鉴的是“改变程序表示或语言上下文以逃离原语言局部最优”的思想，以及把每次翻译和回译都置于可执行测试闭环中。其核心贡献已经是跨语言 APR 流程，不能只把 Python 换成 C++ 或把目标语言换成 RISC-V 汇编就声称形成新方法。

对当前编译器研究的直接相关性为中等：它证明了 LLM 直接输出变换后源码和修复程序的 Translator 角色，但没有研究 LLVM pass、IR 优化、寄存器分配或后端代码质量。若迁移到 RISC-V，应增加 ISA/IR 层级的语义约束、汇编器/模拟器/差分执行和成本指标，而不是仅替换目标语言名称。

## 11. 可进一步尝试的研究方向

### 11.1 IR 锚定的跨语言修复

#### 研究问题

在源语言和目标语言之间加入 LLVM IR 或受控中间表示，能否降低翻译和回译的语义漂移？

#### 与原论文的区别

原论文直接在多种源语言之间翻译，主要依赖测试结果；该方向将 IR 作为显式语义锚点，并验证源/IR/目标三者的一致性。

#### 可能的创新点

设计 IR-aware prompt、局部 IR 对齐和失败回退机制，区分语法修复、控制流修复与数据依赖修复。

#### 实验框架

```text
缺陷源程序 → LLM 生成目标程序与 IR → 编译/IR 验证 → 目标语言修复
          → 回译源程序 → 差分测试/语义检查 → 记录历史路径
```

#### 可行性

需要 Clang/LLVM、可执行测试集、至少两种语言和能处理代码与 IR 的 LLM。

#### 主要风险

LLVM IR 不一定完整表达语言运行时、异常、内存模型和未定义行为；IR 可验证不等于源语言语义完全一致。

### 11.2 面向 RISC-V 的跨 ISA repair/translation

#### 研究问题

当 x86/ARM 低层代码翻译到 RISC-V 后出现汇编器、ABI 或执行错误时，能否借鉴 LANTERN 的“翻译—修复—回译”循环自动修复？

#### 与原论文的区别

原论文处理高级语言之间的 APR；该方向处理 assembly/LLVM IR、ABI、寄存器和内存模型约束。

#### 可能的创新点

将 RISC-V 汇编器错误、模拟器执行结果、ABI 检查和指令级差分作为反馈，并按错误类型选择目标 ISA 或中间表示。

#### 实验框架

```text
x86/ARM 程序 → LLM 生成 RISC-V ASM → assembler/emulator/differential test
             → 错误分类与历史检索 → LLM repair → 再汇编与执行
```

#### 可行性

需要 RISC-V GCC/LLVM、QEMU 或 Spike、ABI 约束、跨 ISA benchmark 和可执行测试。

#### 主要风险

指令级翻译的寄存器分配和调用约定错误可能难以由有限测试发现；跨 ISA 运行时性能不能仅由功能测试推断。

### 11.3 成本约束的目标语言选择

#### 研究问题

目标语言选择是否应同时考虑修复成功概率、翻译路径长度、编译/测试成本和回译风险？

#### 与原论文的区别

原论文使用历史反馈改善有效语言排序，但没有给出显式成本优化目标。

#### 可能的创新点

建立 cost-aware language selection，报告质量—成本 Pareto 前沿，并比较早停、贪心、随机和 LLM reasoning。

#### 实验框架

```text
bug 特征 + 历史成功率 + token/编译成本
        ↓
LLM/排序器选择目标语言
        ↓
翻译—修复—测试—回译
        ↓
按成功率、成本和风险更新历史记录
```

#### 可行性

可复用 xCodeEval 与论文的三种模型，新增 token、调用次数、编译时间和失败率统计。

#### 主要风险

成本度量受模型服务、缓存和硬件环境影响；不同 benchmark 的成本不可直接比较。

## 12. 与其他已读文献的关系

本轮只对 LANTERN 完成正文阅读，未对其它 staging 候选完成阶段 2 正文核验，因此不把其它候选的摘要信息当作“已读文献”事实。

从 LANTERN 正文引用关系可确认：它将代码翻译作为 APR 中间通道，并与 ChatRepair、AGENTLESS 等修复系统比较；它引用 BatFix 作为语言模型 transpilation repair 相关工作，引用 AlphaTrans 作为 repository-level code translation/validation 相关工作。这些工作在本笔记中只作为论文相关工作背景，不作本轮独立候选的实验结论。

在当前语料角色上，LANTERN 更适合作为 `TRANSLATOR/T3` 的跨语言修复 baseline 或方法参考，而不是 Selector。其 analyzer 虽然选择目标语言，但最终研究对象是直接生成翻译/修复代码；该选择子角色不应覆盖论文的主角色。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 跨语言 LLM 自动程序修复 |
| 核心问题 | 原语言修复失败和不同语言修复能力不均衡 |
| 输入 | 多语言 buggy source code、bug 特征、测试结果和历史反馈 |
| 输出 | 目标语言修复代码及回译后的源语言修复代码 |
| 核心方法 | analyzer 选语言，translator 翻译/回译，repairer 修复，middleware 记录反馈 |
| 使用的模型 | DeepSeek-V3；泛化实验使用 Claude 3.5 Sonnet、Qwen2.5-72B-Instruct |
| 使用的编译器工具 | xCodeEval/ExecEval 的编译、解释和测试环境；具体版本未完整说明 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；使用编译、执行和测试一致性检查 |
| 数据集规模 | xCodeEval Compact Set：5,068 bugs、11 languages；另有 Defects4J 和 SWE-Bench Lite |
| 主要指标 | Pass@10、MAP@k、NDCG@k、Precision/Recall/F1@k、resolved instances |
| 最重要实验结果 | Rust 在 xCodeEval 上最高提升 22.09% Pass@10；SWE-Bench Lite 110/300 vs AGENTLESS 95/300 |
| 核心创新 | 将跨语言翻译嵌入 APR 失败恢复流程，并用历史反馈选择目标语言 |
| 主要局限 | 依赖有限测试、翻译/回译可能漂移、成本高，未覆盖 IR/ASM/RISC-V/真实硬件 |
| 与 RISC-V 研究的相关性 | 中：提供跨表示修复流程启发，但没有 RISC-V 或低层代码实验 |
| 最适合作为 | Translator/T3 baseline、跨语言 repair 方法参考 |

这篇论文最值得学习的是把“换一种语言思考”转化为可执行的翻译—修复—回译闭环；最主要的局限是测试驱动的一致性不能替代形式化语义证明，也没有证明对 IR、汇编或 RISC-V 后端有效。如果用于后续研究，合理方式是引入 IR/ISA/ABI 约束和差分验证，而不是简单替换目标语言或平台。

