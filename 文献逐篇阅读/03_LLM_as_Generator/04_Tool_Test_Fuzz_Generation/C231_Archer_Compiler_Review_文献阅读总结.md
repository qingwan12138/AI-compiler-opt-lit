# Archer 文献阅读总结

- 论文题目：**Archer: Towards Agentic Review for Compiler Optimizations**
- 作者：Yunbo Ni；Shaohua Li
- 发表时间：2026
- 发表平台：arXiv 预印本（arXiv:2607.01808，2026-07-02）
- 论文链接或编号：<https://arxiv.org/abs/2607.01808>
- 关键词：LLVM、编译器优化审查、LLM agent、语义 obligation、Alive2、LLUBI、确定性验证

> 本笔记依据 arXiv v1 PDF（12 页）逐页阅读。论文事实与阅读后的分析分开记录。

## 1. 研究背景

Archer 关注 LLVM 优化补丁的代码审查。编译器优化必须保持源程序语义，同时还会与分析、规范化和后续优化发生交互。普通代码审查 agent 主要依赖补丁文本和仓库检索，容易遗漏 poison、符号位、溢出标志等只在特定语义条件下出现的问题。论文将编译器优化审查与传统编译器测试区分开：审查对象是具体补丁，报告必须解释该补丁引入的语义问题，并提供可执行证据。

论文提出把历史正确性修复转化为可复用的 pass-level semantic obligations（优化 pass 应保持的语义关系），再用确定性验证 guard 约束 agent 只能报告有可执行证据支持的问题。

## 2. 论文要解决的问题

### 2.1 从补丁文本恢复编译器语义关系

历史 bug 修复包含专家经验，但原始 issue、patch 和 IR 往往与具体实现绑定。论文研究如何把这些材料抽象成高层、可读且精确的语义 obligation，并按优化 pass 组织。

### 2.2 让审查结论具有补丁特定的可执行证据

单纯的自然语言怀疑可能是误报。论文研究如何从 agent 给出的 mutation strategy 生成 IR 对，并验证该证据确实触发当前补丁引入的语义差异。

### 2.3 在真实 LLVM PR 上评估审查能力

论文在 LLVM middle-end optimization PR 上评估 Archer，并比较不同 LLM、通用代码 agent、商业审查工具和定向测试工具。

> 本文主要研究：如何利用历史修复抽取的可复用语义知识和确定性验证，把 LLM agent 的 LLVM 优化审查结论约束为补丁特定的可执行证据。

## 3. 核心方法概述

Archer 由动态 obligation 构建、引导式 agent 分析和确定性验证 guard 三部分组成。它把历史修复中的具体案例转化为 pass 级语义约束；审查新 PR 时，agent 输出带语义理由的可执行 mutation strategy；guard 再用补丁前后编译器、Alive2 和 LLUBI 验证，只有确认由当前补丁引起的差异才报告。

```text
历史 LLVM 正确性修复
        ↓
按优化 pass 分组
        ↓
LLM A 抽取候选 obligation
        ↓
LLM B 生成 IR 源/目标对并由 Alive2 验证
        ↓
LLM C 总结为 pass-level obligation base
        ↓
新 LLVM PR + 受影响 pass + obligation
        ↓
agent 检索本地源码、LangRef 和测试，生成 mutation strategy
        ↓
guard 从 PR 测试种子实例化 IR
        ↓
post-patch OPT 转换；Alive2 检查，必要时 LLUBI 差分执行
        ↓
与 pre-patch 编译器比较是否为 patch-triggering
        ↓
输出 IR reproducer、语义分析和审查报告，或拒绝报告
```

LLM 的最终角色是编译器优化审查工具中的 obligation/strategy/evidence 生成能力。它不直接替代 LLVM pass，也不把一个用户源程序改写成最终程序。工具包括 LLVM `opt`、Alive2、LLUBI、源码搜索和 LLVM Language Reference 查询。

## 4. 实验框架与训练流程

本文不涉及模型预训练、SFT 或强化学习，主要采用多阶段提示式 agent 工作流和编译器工具反馈。

### 4.1 动态 obligation 构建

论文收集 2017 年至 2026 年 2 月的 LLVM miscompilation bug 及对应修复，先按优化 pass 分桶。LLM A 根据 issue 和 patch 提出候选 obligation；LLM B 将其具体化为源 IR/目标 IR 对；若 proof check 能确认语义差异，则保留该候选；LLM C 将通过验证的候选归纳成 pass-level obligation。

### 4.2 引导式 agent 分析

对于待审查 PR，agent 获得补丁源码树、受影响 pass、对应 obligation、LLVM LangRef 和本地搜索工具。agent 输出 mutation method、semantic rationale 和 expected validation result 三元组组成的 strategy。

### 4.3 确定性验证 guard

guard 使用 PR 自带或关联测试作为种子，生成候选 IR。post-patch 编译器先通过 `opt` 产生目标 IR；优先调用 Alive2 做 source-target 检查，若结果 inconclusive，则调用 LLUBI 进行 UB-aware 差分执行。最后比较 pre-patch 与 post-patch 编译器，确保差异由当前 PR 引入。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有报告模型训练损失函数。核心是两个算法流程。

动态 obligation 构建可概括为：

```text
O_P = SUMMARIZE({(o, ir_src, ir_tgt) | PROOFCHECK(ir_src, ir_tgt) 通过})
```

其中 `P` 是优化 pass，`o` 是候选语义 obligation，`ir_src` 与 `ir_tgt` 是复现该 obligation 的源/目标 IR。论文要求保留的候选能被编译器感知的 proof checker 验证。

确定性 guard 的接受条件为：

```text
接受 c = (ir_src, ir_tgt, r)
当且仅当 r 暴露语义差异，且该差异只在 C+（补丁后）出现而不在 C-（补丁前）出现
```

`r` 可以来自 Alive2 的等价检查或 LLUBI 的定义行为/未定义行为差分执行。该条件的目的，是避免只触发旧 bug 或与当前补丁无关的测试被误报为审查结果。

## 6. 实验设置

### 6.1 数据集来源

论文构建两个评测集：

| 数据集 | 来源与规模 | 用途 |
|---|---|---|
| Real-world dataset | 2025-12-31 至 2026-02-28 的 LLVM middle-end optimization PR，共 398 个，其中 open 70、closed 328 | 真实 PR 审查 |
| Regression dataset | 2024-01-01 至 2025-10-01 的 LLVM miscompilation issue，经 commit-level bisection 得到 47 个案例 | 已知 bug 回归评测 |

obligation 构建使用 2017 年至 2026 年 2 月收集的 317 个候选修复补丁，最终覆盖 45 个优化 pass、188 个 validated cases。作者明确排除了与评测实例重叠的历史案例，以降低数据泄漏风险。

### 6.2 模型与工具

- LLM：Gemini-3.1-Pro-Preview-Custom-Tools、DeepSeek-V3.2、Qwen3.5-Plus；RQ1 的流程分析使用 Gemini-3.1-Pro。
- agent 框架：mini-SWE-agent V2。
- 编译器与验证：LLVM、`opt`、Alive2、LLUBI。
- 运行环境：Ubuntu 20.04，AMD EPYC 7742 64-core CPU，256 GB RAM。
- 每次 review 最多 500 agent rounds、10M tokens；每个工具最多 250 次调用。

### 6.3 对比方法

- Direct：DeepSeek-V3.2 直接审查，没有 Archer 的 obligation、agent 框架和验证 guard。
- MSWE：mini-SWE-agent + DeepSeek-V3.2，但没有 Archer 的专用机制。
- Codex、GitHub Copilot、CodeRabbit、Greptile：通用或商业代码审查工具。
- Optimuzz：LLVM 定向测试工具。
- Archer 的 `base`、去除 obligations 的 `wo`、静态 top-3 历史案例检索的 `rag`、提供全部 validated cases 的 `all`。

### 6.4 评价指标

论文主要统计发现的 semantic bugs、bug 状态、bug 症状、受影响组件、不同模型的发现数、误报情况、token/时间/费用和工具调用分布。发现数越高通常越好，但报告是否被开发者确认、是否由当前 PR 触发决定实用性；费用、token 和时间越低越好。

## 7. 实验结果与结论

### 7.1 主要结果

在 398 个真实 LLVM PR 上，Archer 报告 51 个 semantic bugs：15 个来自 open PR、36 个来自 closed PR。open PR 中 4 个 confirmed、7 个 fixed；closed PR 中 7 个 confirmed、26 个 fixed；另有 3 个 Not Planned、4 个 open Unconfirmed。论文摘要据此报告 open PR 的 21% 和 closed PR 的 11% 存在 bug。

51 个 bug 中，34 个是 miscompilation，17 个是 compiler crash。受影响组件最多的是 peephole optimizations（10）、vectorization optimization（9）和 loop transformations（7）。

### 7.2 与传统方法的比较

在 47 个 regression cases 上，Optimuzz 成功发现 3 个 bug，且都被 Archer 覆盖；但 Optimuzz 在 47 个案例中的 23 个无法启动 fuzzing，因为需要匹配特定 CFG 结构。Archer 的 guard 通过 patch 前后编译器比较，将证据绑定到当前补丁。

### 7.3 与其他 LLM 方法的比较

在相同回归集上，Direct 没发现 bug，MSWE 只发现 1 个。Archer 的三种底层模型均发现非零数量的语义 bug，Gemini-3.1-Pro 发现数最多，并覆盖另外两个模型发现的 bug。论文没有在正文文本中给出 Figure 7 每个 base 柱的完整数值表，因此除相对排序外不补写具体数字。

### 7.4 消融实验

移除 obligations 的 `wo` 设置在三种模型上发现数均明显下降，论文表述为“接近减少一半”。同一底层模型和框架下，带 guard 的 Archer `wo` 发现 3 个，而没有专用 validation guard 的 MSWE 发现 1 个，说明工具调用本身不足以替代 guard。

`base` 一致优于 `rag` 和 `all`。这支持了 obligation 应该经过验证、抽象和按 pass 组织的设计：直接检索历史案例会保留过多具体细节，而提供全部低层案例会增加上下文负担。

### 7.5 案例分析

论文分析一个 InstCombine/循环不变量外提案例。补丁把 `IV + X == C` 改写成 `IV == C - X`，但保留 `icmp samesign` 标志。取 `iv=-1`、`x=1` 时，原始比较的操作数为 `0` 与 `1`，均为非负；变换后比较 `-1` 与 `0`，符号不同，产生 poison。Alive2 在该循环案例中错误返回正确，LLUBI 的差分执行报告了 `Branch on poison`，体现了双重验证的必要性。

## 8. 主要创新点

### 8.1 创新点一：动态构建可复用的 pass-level obligations

论文没有把历史 patch 作为静态 RAG 文本，而是让 LLM 抽象、生成复现 IR、验证，再总结成面向 pass 的语义模式。价值在于把具体 bug 的经验转成可以跨新 PR 复用的编译器审查知识。评测中的 `base` 优于 `rag` 和 `all` 支持了该设计。

### 8.2 创新点二：确定性 validation guard

guard 要求文本策略必须落成可执行、oracle-checkable 且 patch-triggering 的证据。它把自然语言审查报告和补丁前后编译器行为连接起来，降低只凭语言判断生成误报的风险。

### 8.3 创新点三：Alive2 与 LLUBI 的互补验证

论文观察到 Alive2 对循环、部分 intrinsic 或求解困难路径可能 inconclusive 或不准确，因此在无法完成 proof check 时用 LLUBI 做 UB-aware 差分执行。案例研究中 Alive2 与 LLUBI 的结论差异直接展示了该组合的价值。

## 9. 局限性

### 论文明确承认的局限

- 第 VI 节指出 Archer 在 regression dataset 上仍有明显 recall gap。
- agent 有时重复调用工具或进行低收益探索，消耗 review budget。
- obligation 可以继续通过失败案例反馈自动修订，说明当前 obligation base 并非最终形态。
- Alive2 不是完整验证器；循环、unsupported intrinsics 和 solver-hard 条件可能使 proof check 不完整。

### 阅读后发现的潜在局限

- 评测目标集中于 LLVM middle-end optimization PR，不能直接推断对 GCC、MLIR 或 RISC-V backend 的效果。
- 真实审查成本为 398 个案例总计 988.7 美元，平均每案 2.5 美元；平均每案 5,054,201 tokens、877 秒。大规模持续运行仍需预算控制。
- 论文将 semantic obligations 作为 GENERATOR 能力，但它生成的是审查工具知识和证据，不是 LLVM pass 或 rewrite rule 本身；在三角色 taxonomy 中更接近 compiler-facing tool，二级分类仍需人工确认。
- 评测中使用的 Gemini-3.1-Pro 等模型版本和未来可获得性可能影响复现，论文没有提供所有模型权重。

## 10. 阅读后的研究方向反思

Archer 最值得借鉴的是“生成候选 + 编译器 oracle 约束 + 可复用语义知识”的闭环。对 LLVM/RISC-V 研究而言，可把 RISC-V 特定的 ISA、寄存器约束和 LLVM backend lowering 规则组织成 obligation，但不能只替换目标架构就宣称形成新方法。论文更适合作为 compiler-review tool 或 correctness-evidence 模块的 baseline；它不是直接生成优化 pass 的训练框架。

需要保留的边界是：有限的 Alive2/LLUBI 或测试证据只能支持具体案例，不能表述成对所有输入的形式化证明；历史 obligation 也不能替代 pass 的完整语义规范。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V 后端 lowering 的 obligation 审查

#### 研究问题

如何从 RISC-V LLVM backend 的历史 miscompile 修复中抽取指令选择、寄存器约束和 ABI 相关 obligation，并自动审查新 lowering patch？

#### 与原论文的区别

目标从 LLVM middle-end optimization PR 扩展到 RISC-V backend，重点是 ISA/ABI 语义和机器指令合法性。

#### 可能的创新点

引入 MIR、TableGen 和 ISA 语义的联合 obligation 表示，并设计能区分 pre-RA/post-RA 差异的 guard。

#### 实验框架

```text
历史 RISC-V miscompile 修复 → obligation 构建 → 新 backend PR 分析
→ MIR/汇编候选生成 → LLVM verifier 与差分执行 → patch-triggering 证据
```

#### 可行性

需要 LLVM RISC-V backend、MIR 测试、QEMU 或真实 RISC-V 运行环境，以及可用的验证/差分执行工具。

#### 主要风险

MIR 到机器代码的语义 oracle 覆盖不足，且真实硬件差异可能使性能和功能结果难以统一。

### 11.2 Obligation 质量的自动回归维护

#### 研究问题

如何在 LLVM 持续演化时自动发现过时、过宽或重复的 pass-level obligation？

#### 与原论文的区别

重点从一次性构建 obligation 转向版本演化、失效检测和回归维护。

#### 可能的创新点

使用 LLVM 版本迁移测试和反例聚类，对 obligation 做版本化，并测量旧 obligation 对新 PR 的 recall/precision 变化。

#### 实验框架

```text
跨版本历史修复 → obligation 版本库 → 新旧 LLVM 回归集
→ 失效/重复检测 → Alive2/执行验证 → 更新或废弃 obligation
```

#### 可行性

可以复用 Archer 的三阶段构建和 LLVM 历史 PR 数据。

#### 主要风险

版本差异会导致复现环境构建困难，obligation 的“过时”标准也需要人工标注。

## 12. 与其他已读文献的关系

本批次只有 Archer 获得完整 PDF 并完成逐篇阅读；BePilot 正文不可得，MultiFork 与 VEGA 已通过中央查重判为已有候选或版本族，因此不把它们的未读内容当作事实比较。

从角色上看，Archer 与 pass 生成、rewrite 生成论文共享“输出可复用编译器能力”的 GENERATOR 方向，但 Archer 输出的是语义审查 obligations、validation strategies 和 evidence-first review tool；它不生成部署到编译流水线的优化 pass。它与 compiler fuzzing 的区别也在于审查必须绑定具体 PR，而不是任意探索编译器输入。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLVM 编译器优化补丁的 agentic 语义审查 |
| 核心问题 | 让 LLM 审查从文本怀疑转为 patch-specific 可执行证据 |
| 输入 | LLVM PR、补丁前后源码树、受影响 pass、历史 obligations、相关测试 |
| 输出 | 可复用 pass-level obligations、mutation strategies、验证过的 IR reproducer 和审查报告 |
| 核心方法 | 动态 obligation 构建 + 引导式 agent + 确定性 validation guard |
| 使用的模型 | Gemini-3.1-Pro、DeepSeek-V3.2、Qwen3.5-Plus |
| 使用的编译器工具 | LLVM、OPT、Alive2、LLUBI |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 使用 Alive2；不完整时结合 LLUBI 差分执行 |
| 数据集规模 | 398 个真实 PR；47 个回归案例；obligation 构建 317 个候选修复、45 个 pass、188 个验证案例 |
| 主要指标 | bug 发现数、状态、症状、组件、误报、成本、token、时间 |
| 最重要实验结果 | 真实 PR 中报告 51 个 semantic bugs；open 21%、closed 11%；33 个已修复 |
| 核心创新 | 将历史修复抽象成可复用 obligation，并强制生成 patch-triggering 证据 |
| 主要局限 | recall gap、验证覆盖不完整、成本高、依赖 LLVM middle-end |
| 与 RISC-V 研究的相关性 | 中：方法可迁移到 RISC-V backend，但论文未评测 RISC-V |
| 最适合作为 | compiler review tool / correctness-evidence 模块 baseline |

这篇论文最值得学习的是把 LLM 的开放式分析限制在可执行、可验证且绑定当前补丁的证据流程内；最主要的局限是 recall 和 oracle 覆盖仍不完整、运行成本较高；如果用于后续研究，最合理的使用方式是作为编译器正确性审查模块或 baseline，而不是简单替换架构后声称生成了新的优化 pass。
