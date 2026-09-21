# Understanding Agent-Based Patching 文献阅读总结

论文题目：**Understanding Agent-Based Patching of Compiler Missed Optimizations**

作者：Batu Guan；Zirui Wang；Shaohua Li

发表时间：2026（arXiv v1：2026-07-02；v2：2026-07-03）

发表平台：arXiv，cs.SE/cs.AI；论文中未明确说明正式会议或期刊版本

论文链接或编号：arXiv:2607.02370；DOI：10.48550/arXiv.2607.02370

关键词：LLVM；compiler missed optimization；coding agent；patch generation；optimization scope；RAG；knowledge distillation；Alive2；llvm-mca

> 本文档用于阶段2正文核验。论文事实依据本轮下载的官方 arXiv PDF；阅读分析和后续建议单独标注。本文不构成正式 taxonomy 入库。

正文文件：[官方 PDF](../../../03_LLM_as_Generator/01_Compiler_Pass_Generation/C207-Compiler-Missed-Optimization-Patching-arXiv2026/paper.pdf)

---

## 1. 研究背景

现代编译器通过优化 pass 识别代码模式，并将其替换为语义等价但更高效的形式。由于优化规则覆盖的模式数量很大，编译器仍会漏掉一些可以安全优化的情况，即 missed optimization。漏优化通常不会直接造成错误执行，但会让生成代码保留不必要的指令或错过更好的规范化形式。

论文指出，传统的超优化器、程序综合和近期 LLM 方法已经能够发现大量漏优化实例，因此瓶颈逐渐从“找到一个机会”转向“把机会实现成质量足够高、能覆盖开发者意图的编译器 patch”。单纯修复 issue 中给出的一个测试用例，可能只得到过拟合的局部规则，不能覆盖相似的操作数、常量、类型或控制条件。

本文引入 coding agent 的动机，是利用其理解 issue、LLVM 源码和历史修改的能力生成候选 patch；但论文并不假定 agent 的首个 patch 就等价于开发者实现，而是把 patch 的优化作用域作为独立评测对象。

## 2. 论文要解决的问题

### 2.1 Agent 能否把 issue 级漏优化修复为可泛化的 LLVM patch

论文不只问 agent 是否能让初始 reproducer 变小，而是比较 agent patch `A` 与开发者 golden patch `G` 的优化作用域关系。目标是判断 agent 是否覆盖开发者意图、只覆盖其子集、与其部分交叠，或在覆盖 golden scope 之外继续泛化。

### 2.2 哪些上下文能帮助 agent 推断开发者意图

论文评估两种补充信息：

1. 在任务指令中明确要求“处理比给定测试更一般的模式”；
2. 从历史 LLVM optimization pull request 中检索相似经验，或把历史经验蒸馏为 pass 级知识文档后提供给 agent。

### 2.3 如何在不泄露 golden patch 的情况下评估泛化范围

测试时不给 agent golden patch 或 golden tests。评估阶段再用 withheld golden tests 和 fuzz-generated LLVM IR 比较 agent-patched LLVM 与 golden-patched LLVM 的行为及优化成本。论文明确承认这种测试代理不能穷举所有真实作用域，也不能形式化证明开发者意图。

> 本文主要研究：如何让 coding agent 为真实 LLVM missed-optimization issue 生成可编译、可测试且更接近开发者优化作用域的源代码 patch，并如何用历史知识改善这种泛化。

## 3. 核心方法概述

本文构建了一个由 LLVM issue、受控 agent harness、编译验证和作用域评测组成的实验系统。LLM 最终输出的是修改 LLVM 优化实现的 patch，而不是单个输入程序的优化后 IR；因此按最终输出角色，候选更接近 `GENERATOR`，但“研究一个 patch 生成能力”与“已经部署一个通用 pass generator”之间仍需谨慎区分。

```text
真实 LLVM missed-optimization issue
        ↓
准备修复前 LLVM workspace、issue 上下文、源码和 LangRef
        ↓
LLM coding agent 查看源码并编辑 LLVM 优化实现
        ↓
构建 LLVM、运行 lit/初始测试，得到 agent patch
        ↓
golden tests 检查是否覆盖开发者修复范围
        ↓
传统变异器 + LLM 差异测试生成 LLVM IR
        ↓
Alive2 检查语义 refinement；llvm-mca 计算优化成本
        ↓
分类 A⊂G、A⋈G、G⊂A 或 A∼G
        ↓
历史 PR 的 RAG 或 pass 级蒸馏知识增强，再重复 patch 评估
```

LLM 的角色包括：issue 级 patch 生成、历史 PR 摘要/知识蒸馏，以及生成用于区分 agent patch 与 golden patch 的 LLVM IR 测试。编译器、lit、Alive2 和 llvm-mca 承担构建、测试、语义检查和成本评估。

## 4. 实验框架与训练流程

本文不报告一个通过 SFT、PPO 或 GRPO 训练的新基础模型；主要采用现成模型、工具调用和离线历史知识增强。论文中未明确说明 agent 是否对基础模型进行参数更新。

### 4.1 Benchmark 构建

作者从官方 LLVM GitHub issue 中筛选同时带有 `fixed` 和 `missed-optimization` 标签的已关闭 issue，排除 duplicate、invalid、wontfix，以及 backend-specific code generation、Clang、MLIR、LLDB 等超出研究范围的组件。作者根据 issue 讨论、关联 PR 和 commit 找到开发者修复，并把开发者改动作为 golden patch。

每个实例需要满足：修复前版本确实在初始测试上漏掉优化；应用开发者修复后重新构建 LLVM，并确认 regression test 产生预期优化输出。最终得到 43 个经过验证的 LLVM missed-optimization 实例。

### 4.2 Agent patch 生成

每个任务使用开发者修复的 parent commit 准备 LLVM workspace。agent 可以使用 `issue_info`、`view_source`、`get_langref` 查看上下文、源码和 LLVM Language Reference Manual；使用 `apply_code` 编辑源码；使用 `build_and_check` 与 `lit_check` 构建和验证。agent 不可访问 golden patch 或 golden test。

### 4.3 作用域评估

golden tests 在生成阶段保留，作为开发者意图的代表性测试。另一路从初始 IR 出发做随机变异；LLM-based generator 则同时看到 agent patch 和 golden patch，尝试生成能暴露二者行为差异的 LLVM IR。每个候选 IR 都由两种 patched compiler 优化，先用 Alive2 检查输出是否为源 IR 的语义有效 refinement，再用 llvm-mca 比较成本。

### 4.4 泛化指令实验

在默认指令之外加入通用的泛化要求，保持任务描述、工具接口、交互预算和评估流程不变。这个实验检验“只在 prompt 中要求更泛化”是否足以改善与 golden scope 的一致性。

### 4.5 历史知识增强

作者从历史 LLVM pull request 中构造两类增强：

- RAG：按 IR-pair 相似度检索历史摘要，使用 Qwen3-Embedding-4B，在单张 NVIDIA RTX A6000 上取 top-3 摘要；
- Distillation：先用 DeepSeek-V4-Pro 对历史 PR 生成摘要，再按优化 pass 聚合并蒸馏为 pass 级知识文档。论文报告使用了 869 个历史摘要；蒸馏为离线操作，不是对被测 agent 的在线训练。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习，因此不存在强化学习奖励函数、PPO/GRPO 目标或在线策略更新。

论文的核心不是一个单一损失函数，而是 patch 作用域的集合关系。设 `A` 为 agent patch 覆盖的优化范围，`G` 为开发者 golden patch 覆盖的范围：

```text
A ⊂ G   agent 只覆盖 golden scope 的一部分
A ⋈ G   两者部分交叠，但各自还有未覆盖区域
G ⊂ A   agent 覆盖 golden scope，并在其外有更广泛优化
A ∼ G   在测试代理下未观察到行为差异
```

这四类不是形式化证明，而是由 golden tests、传统 fuzz 和 LLM fuzz 的观测结果推断的经验类别。若 agent patch 未覆盖 golden tests，通常说明欠泛化；若它在通过 golden tests 后还能让 agent-patched compiler 在 agent-only fuzz case 上获得更低 llvm-mca 成本，则记录为可能的超出 golden scope 的泛化。

## 6. 实验设置

### 6.1 数据集来源

- 主 benchmark：43 个经过验证的真实 LLVM missed-optimization 实例。
- Pass 分布：InstCombine 32 个、SimplifyCFG 4 个、ValueTracking 3 个、ConstraintElimination 2 个、InstructionSimplify 2 个。
- 每个实例包含初始 issue 测试、开发者修复提取出的 golden patch、golden tests 和预期 IR。
- 历史知识增强使用 LLVM 历史 optimization pull requests；论文报告历史摘要数量为 869。
- 真实 IR 评估包含 opencv、uv、linux、llama.cpp 等项目。论文中未给出这些项目的完整版本清单。

### 6.2 模型与工具

基础 agent 模型：GPT-5.5、DeepSeek-V4-Pro、Qwen3.5-Plus、Kimi K2.5。论文没有把这些模型训练成一个新的统一模型。

主要工具：LLVM/LLVM IR、LLVM Language Reference Manual、LLVM build/check、lit、Alive2、llvm-mca、Qwen3-Embedding-4B。历史知识检索使用单张 NVIDIA RTX A6000；其余硬件、LLVM 精确版本和推理框架配置，论文中未明确说明。

### 6.3 对比方法

主要比较包括：

- 现成 coding agent 在默认指令下的 patch；
- 加入泛化指令后的 agent patch；
- RAG 历史知识增强；
- pass 级历史知识蒸馏；
- 开发者 golden patch，作为作用域参照而不是一个 LLM baseline。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| Initial-test applied | patch 能应用、通过检查并优化 issue 初始测试的实例数 | 越大越好 |
| A⊂G / A⋈G / G⊂A / A∼G | agent 与开发者优化作用域的经验关系 | `A∼G` 或 `G⊂A` 通常更接近充分泛化 |
| Alive2 refinement | 优化输出是否为源 IR 的语义有效 refinement | 越多越好；不是作用域证明 |
| llvm-mca cost | LLVM IR 优化结果的静态机器成本估计 | 通常越小越好 |
| Project-level optimization hits | 大型真实项目 IR 中触发优化的累计次数 | 越多越好 |

## 7. 实验结果与结论

### 7.1 主要结果

默认指令下，GPT-5.5 在 43 个 issue 中有 32 个生成的 patch 成功应用并优化初始测试，11 个失败，即 32/43，约 74%。这只说明初始案例被修复，不等价于 patch 已覆盖开发者完整意图。

默认设置的作用域计数如下：

| 模型 | A⊂G | A⋈G | G⊂A | A∼G | Applied | Failed |
|---|---:|---:|---:|---:|---:|---:|
| GPT-5.5 | 4 | 4 | 6 | 18 | 32 | 11 |
| DeepSeek-V4-Pro | 5 | 4 | 4 | 14 | 27 | 16 |
| Qwen3.5-Plus | 9 | 4 | 3 | 11 | 27 | 16 |
| Kimi K2.5 | 11 | 3 | 2 | 10 | 26 | 17 |

对 GPT-5.5 而言，18/32 个已应用 patch 在观测测试下属于 A∼G，6 个属于 G⊂A；其余表现为欠泛化或部分交叠。论文的主结论是：agent 经常能修复给定测试，但生成 patch 的作用域与开发者 patch 不一致仍然普遍存在。

### 7.2 泛化指令结果

单纯增加“请覆盖更一般模式”的指令没有稳定改善结果。GPT-5.5 的 `A∼G + G⊂A` 从默认的 24 个变为 24 个；DeepSeek-V4-Pro 从 18 个降为 15 个；Qwen3.5-Plus 从 14 个降为 10 个；Kimi K2.5 从 12 个降为 10 个。部分模型的失败数增加，说明 prompt 中的泛化要求可能把 agent 从初始案例修复推向更难、但不一定正确的搜索空间。

### 7.3 历史知识增强与真实 IR

在真实项目 IR 上，蒸馏增强把累计优化命中数从 20.2M 提高到 24.3M；RAG 提高到 20.3M。蒸馏在论文评估的每个项目上都优于对应 baseline，RAG 的收益更依赖项目和 IR 模式。

这些结果支持“历史 compiler engineering knowledge 能帮助模型识别优化抽象”的判断，但不表示增强后的 patch 已经被 LLVM 社区接受，也不表示所有命中都等价于真实硬件性能提升。

### 7.4 消融实验

论文主要比较默认指令、泛化指令、RAG 和蒸馏增强。明确的泛化指令没有可靠收益；历史知识增强，尤其是 pass 级蒸馏，在真实 IR 命中数上更稳定。论文中未报告一个将 RAG、蒸馏、工具集合和模型能力全部逐项拆开的完整 factorial ablation。

### 7.5 案例分析

论文以 LLVM issue #158326 为例。初始模式将范围判断与 masked equality 结合；`(a & -2) == 2` 实际上限制 `a` 在 `{2,3}`，因此与 `a < 5` 的合取可以化简。直接 patch 能通过初始测试，但在 5 个 golden tests 中失败 3 个；RAG 提供了历史 masked-comparison 与 range reasoning 的线索，使 agent 把 masked equality 转为 `ConstantRange`，复用已有的 `foldAndOrOfICmpsUsingRanges` 逻辑。该增强 patch 通过 golden tests，且没有被接受的反例。

## 8. 主要创新点

### 8.1 创新点一：把 missed-optimization patch 的作用域作为核心评测对象

既有工作常用“初始测试是否变好”判断 patch 成功。本文把 agent patch 与 developer golden patch 的作用域关系显式化，并用 withheld golden tests 与 fuzz tests 双重观察。这使“修复一个例子”和“实现可泛化优化”能够被区分。

### 8.2 创新点二：构建 43 个严格验证的真实 LLVM benchmark

benchmark 不是任意生成的 IR 集合，而是从真实 issue、开发者 commit 和 regression tests 恢复，并验证修复前漏优化、修复后输出正确。其规模不大，但为作用域研究提供了可追溯的 golden patch 和 golden tests。

### 8.3 创新点三：用历史 compiler engineering knowledge 增强 patch 泛化

论文把历史 PR 作为可检索经验，并进一步按 pass 蒸馏成知识文档，内容包括常见指令形式、操作数变体、谓词、类型约束和 value-tracking 交互。案例表明，相关历史线索能把 agent 从语法近似推进到 ConstantRange 级语义抽象。

## 9. 局限性

### 9.1 论文明确或正文直接承认的局限

- 作用域 `A` 和 `G` 无法被完全枚举，测试代理不能形式化证明 agent patch 与开发者意图相同。
- golden tests 可能遗漏开发者实际想覆盖的情况；agent-only fuzz 发现的额外优化也可能是合理的更广泛泛化，而不一定是错误。
- benchmark 仅包含 43 个实例，且 InstCombine 占 32 个；不能直接代表全部 LLVM pass。
- 研究排除了 backend-specific code generation、Clang、MLIR 和 LLDB，结论集中在 LLVM 中端优化。
- `llvm-mca` 是静态成本估计；论文中的成本改善不等于真实硬件运行时间改善。

### 9.2 阅读后发现的潜在局限

- 论文把 developer patch 当作 golden scope 参照，这适合研究开发者意图，但可能低估“不同且同样正确”的优化实现。
- LLM-based fuzz generator 同时接收 agent patch 和 golden patch，适合离线评估，但不能用于生成阶段；其测试能力可能影响 A⋈G 或 G⊂A 的发现率。
- 论文中未明确说明基础模型训练数据、系统 prompt、完整交互预算和 LLVM 精确版本，复现实验仍需要补齐工程配置。
- 只有局部 patch 被实现并通过测试，并不代表 patch 已被 LLVM upstream 接受或具备长期维护质量。

## 10. 阅读后的研究方向反思

本文最值得借鉴的是把“可泛化性”从口号变成 patch scope、golden tests、反例和静态成本的组合评测。对于本仓库的 Generator 分类，论文的最终产物是 LLVM 源码中的可复用优化实现 patch，因此比只输出 pass 序列的 Selector 更接近 Generator；但它仍是 patch-generation study，不应直接写成已经自动生成完整 compiler pass 的系统。

与 `Finding Missed Code Size Optimizations in Compilers using LLMs` 的关系尤其需要准确表述：

| 维度 | 本文 | Paper_ID 41：Finding Missed Code Size Optimizations |
|---|---|---|
| 论文标识 | arXiv:2607.02370；2026；Guan/Wang/Li | arXiv:2501.00655；DOI 10.1145/3708493.3712686；2025；正式库记录为 `41` |
| LLM 最终输出 | 修改 LLVM 优化实现的 compiler patch | C/C++ 编译器测试程序及变异样本 |
| 主要目标 | 让 patch 覆盖开发者 intended optimization scope | 用差分测试发现代码尺寸 missed optimization |
| 评估对象 | agent patch 与 developer golden patch 的作用域 | 测试程序触发的编译器尺寸异常、误报过滤和真实缺陷 |
| 关系 | 新工作，非同题版本；正文把 Finding Missed 列为相关工作 | 既有相邻工作，不是本文的版本或数据集 |

因此查重结论是：**非重复工作，但属于强相邻问题族；候选仍需 `Needs_Review=YES`。** 另外，仓库中还存在 Paper_ID `C41` 的 `LLVM-Bench`，它是 benchmark/supporting 条目，与本次所说的 Paper_ID `41` 不是同一记录，不能混淆。

## 11. 可进一步尝试的研究方向

### 11.1 跨架构 patch scope 与真实后端收益联合评估

#### 研究问题

同一个 LLVM 中端 patch 在 x86、RISC-V 和其他目标上是否具有一致的作用域和收益？

#### 与原论文的区别

原论文主要使用 LLVM IR、Alive2 和 llvm-mca 的静态成本；新方向增加目标架构维度和真实后端代码生成评估。

#### 可能的创新点

把 `A/G` 作用域关系扩展为架构条件作用域，并区分语义安全、静态成本和真实硬件收益。

#### 实验框架

```text
真实 missed-optimization issue
        ↓
LLM patch generation
        ↓
Alive2 semantic gate
        ↓
x86/RISC-V LLVM build + llvm-mca + hardware benchmark
        ↓
per-architecture scope and speedup matrix
```

#### 可行性

需要 LLVM、Alive2、RISC-V 交叉编译器、llvm-mca 和可重复的 RISC-V 测试板或模拟环境。

#### 主要风险

真实硬件噪声、目标后端差异和 patch 的跨架构有效性可能使结果难以归因。

### 11.2 由 patch scope 驱动的历史知识检索

#### 研究问题

能否根据待修复 IR 的语义结构，而不是只按文本相似度，检索能帮助 agent 泛化的历史 patch？

#### 与原论文的区别

原论文使用 IR-pair embedding RAG 和 pass 级蒸馏；新方向可以加入 ConstantRange、类型约束、value-tracking 和目标架构信息的结构化索引。

#### 可能的创新点

构建“语义模式—历史 patch—golden test—失败反例”四元组，并用 scope 反馈更新检索结果。

#### 实验框架

```text
LLVM issue/IR
  ↓
结构化语义特征提取
  ↓
历史 patch + golden test + 反例检索
  ↓
agent 生成 patch
  ↓
scope evaluation
  ↓
将失败模式回写知识库
```

#### 可行性

可以复用论文的 43 个实例和 LLVM 历史 PR；主要工程工作是抽取稳定的语义特征。

#### 主要风险

历史 patch 的开发者意图不总是显式记录，检索增强可能泄露测试或把不兼容的 pass 经验迁移过来。

### 11.3 将 patch 生成与可维护性检查分离

#### 研究问题

通过测试的 LLM patch 是否满足 LLVM 风格、复杂度、长期维护和回归测试要求？

#### 与原论文的区别

原论文重点是优化 scope；新方向增加 code review、静态分析、回归测试覆盖和 patch complexity 评估。

#### 可能的创新点

定义“性能作用域—语义安全—维护成本”的三目标 patch 质量指标。

#### 实验框架

```text
agent patch
  ↓
build/lit + Alive2
  ↓
scope and performance evaluation
  ↓
LLVM style/static analysis/review agent
  ↓
accept, revise, or reject
```

#### 可行性

不需要重新训练基础模型，可在现有 benchmark 上增加审查指标。

#### 主要风险

维护成本指标主观性较强，且风格合规不代表优化逻辑正确。

## 12. 与其他已读文献的关系

- 与 Paper_ID `41` 的 `Finding Missed Code Size Optimizations in Compilers using LLMs`：问题族相同，输出角色不同。Paper_ID 41 是 Generator/G4 的测试与 fuzz 生成；本文是候选 Generator/G1 的 LLVM patch 生成与 scope 评估。前者可以作为“发现 missed optimization”的上游，本文可以作为“将 issue 实现为 patch”的下游，但不能合并为同一版本。
- 与 LPO（仓库中已有的 peephole optimization 工作）：LPO 关注从具体优化实例泛化出更广的 peephole 规则；本文关注 coding agent 修改真实 LLVM pass，并以开发者 patch scope 为参照。两者都需要泛化，但输入、输出和评测对象不同。
- 与 LLM-VeriOpt：LLM-VeriOpt 直接生成 LLVM IR 变换，最终角色是 Translator；本文生成 LLVM 优化实现 patch，最终角色更接近 Generator。本文也使用 Alive2，但它用于评估 patch 产生的 IR refinement，不是本文的训练奖励。
- 与 Agentic Harness for Real-World Compilers：二者都使用 LLVM agent harness 和工具调用；Agentic Harness 的主任务是 compiler bug repair，本文专门研究 missed optimization 的 patch scope，因此本文不是该工作的版本。

在本次阶段2中只对本候选完成正文核验；上述关系使用仓库已有记录和本文相关工作内容，不把相邻工作误写成当前批次新阅读结果。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 评估 coding agent 为真实 LLVM missed-optimization issue 生成可泛化 compiler patch 的能力 |
| 核心问题 | 初始测试修复不等于覆盖开发者 intended optimization scope |
| 输入 | LLVM issue、初始 LLVM IR、LLVM 源码、LangRef；历史 PR 用于增强 |
| 输出 | 修改 LLVM 优化实现的 patch，以及离线评估中的 LLVM IR 测试 |
| 核心方法 | agent harness + golden patch/scope 对照 + golden tests + fuzz + RAG/蒸馏 |
| 使用的模型 | GPT-5.5、DeepSeek-V4-Pro、Qwen3.5-Plus、Kimi K2.5；历史摘要蒸馏使用 DeepSeek-V4-Pro |
| 使用的编译器工具 | LLVM、lit、Alive2、llvm-mca、LLVM LangRef |
| 是否使用强化学习 | 否；论文没有 RL 奖励函数或策略更新 |
| 是否使用形式化验证 | 使用 Alive2 做语义 refinement 检查；不是对 developer intent 的形式化证明 |
| 数据集规模 | 43 个验证过的 LLVM missed-optimization 实例；历史摘要 869 个 |
| 主要指标 | 初始测试应用数、A/G 作用域关系、Alive2 refinement、llvm-mca cost、真实 IR optimization hits |
| 最重要实验结果 | GPT-5.5 默认设置 32/43 个实例成功应用 patch；蒸馏增强把真实 IR 累计命中从 20.2M 提高到 24.3M |
| 核心创新 | 将 patch 泛化作用域作为独立研究对象，并用历史 compiler knowledge 改善泛化 |
| 主要局限 | 43 个实例且集中于 LLVM 中端；scope 评估依赖测试代理；静态成本不等于真实硬件收益 |
| 与 RISC-V 研究的相关性 | 中：论文未研究 RISC-V，但其 patch scope 和验证流程可迁移；需要额外加入 RISC-V 后端/硬件实验才有架构贡献 |
| 最适合作为 | Generator 方向的候选方法参考与 scope-evaluation baseline；不是现成的跨架构优化系统 |

> 这篇论文最值得学习的是把“能修初始案例”和“能生成覆盖开发者意图的可复用 patch”分开评估；最主要的局限是 benchmark 小、组件集中且作用域只能经验观测；如果用于后续研究，最合理的使用方式是作为 patch 泛化与验证基线，而不是简单把 LLVM 替换成 RISC-V 就宣称完成了新的编译器生成方法。
