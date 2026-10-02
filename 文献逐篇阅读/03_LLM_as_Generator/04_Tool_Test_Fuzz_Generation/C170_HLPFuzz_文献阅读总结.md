# HLPFuzz 文献阅读总结

论文题目：**Hybrid Language Processor Fuzzing via LLM-Based Constraint Solving**

作者：Yupeng Yang、Shenglong Yao、Jizhou Chen、Wenke Lee

发表时间：2025

发表平台：USENIX Security 2025, pp.6299–6318

论文链接或编号：USENIX Security 2025
元数据核验来源：[USENIX 正式论文页](https://www.usenix.org/conference/usenixsecurity25/presentation/yang-yupeng)
代码/数据/工件：论文 Open Science 声明提供 HLPFuzz 源码和运行示例：[Zenodo 15606060](https://doi.org/10.5281/zenodo.15606060)（归档工件）。

关键词：语言处理器、编译器测试、灰盒 fuzzing、LLM 约束求解、控制流图中心性、迭代上下文构造

> 本文档只依据本地官方 PDF 正文记录事实；论文事实与阅读后的分析分开。

## 1. 研究背景

论文研究语言处理器 fuzzing，包括编译器和解释器。语言处理器通常经历词法分析、解析、语义检查、代码生成或执行等阶段，每一阶段都有约束；只有通过前序约束，输入才可能到达深层优化、代码生成或执行逻辑（第 2.2 节、图 1）。传统灰盒 fuzzing 依靠随机/语法感知变异与覆盖率反馈，但很难构造同时满足复杂语义、类型和数据依赖的输入。通用的污点分析、符号/合取执行在大型语言处理器上会遇到路径爆炸、语义建模不完整或过约束；目标专用方法有效但需要大量人工启发式，迁移性差（第 2.3 节）。

论文引入 LLM，是因为模型可能从源代码、函数名、已有触发样例和自然语言规格中理解约束，并生成能穿过深层检查的输入；但模型调用昂贵，且上下文过大或过小都会降低效果（第 2.4 节）。

## 2. 论文要解决的问题

### 2.1 复杂约束的可达性

如何让 fuzzing 输入通过语言处理器多阶段的复杂约束，触达传统变异难以到达的深层代码区域，尤其是优化、代码生成和执行阶段。

### 2.2 LLM 调用资源分配

如何选择最值得交给 LLM 求解的约束：既要考虑该约束可能带来的覆盖收益，也要考虑 LLM 对该类约束的历史求解难度。

### 2.3 上下文规模控制

如何向 LLM 提供足够但不过大的代码上下文。论文具体研究：当初始窄上下文无法求解时，如何用执行差异和语言服务器逐步补充信息。

> 本文主要研究：如何将 LLM 作为约束求解器嵌入可泛化的混合语言处理器 fuzzing 框架，以到达深层程序状态。

## 3. 核心方法概述

HLPFUZZ（Hybrid Language Processor Fuzzer）把传统灰盒 fuzzing runtime 与白盒 LLM 求解器结合。灰盒部分负责快速探索浅层区域、执行语法感知变异和收集覆盖率；LLM 部分负责选择并求解难约束。核心机制是 hybrid centrality prioritization（混合中心性优先级）和 iterative context construction（迭代上下文构造）。

```text
源代码 + 简短自然语言规格 + 语法文件 + 初始种子
        ↓
传统灰盒 fuzzing：执行、调度、语法感知变异、覆盖率/bug oracle
        ↓
CFG 中未覆盖约束提取与中心性排序
        ↓
LLM 根据窄上下文生成约束解或请求更多上下文
        ↓
验证解；失败时做 divergence analysis 或语言服务器取符号定义
        ↓
将成功解加入队列并继续变异，输出覆盖区域和发现的 bug
```

LLM 最终输出的是用于触发目标基本块的测试输入/约束解；可复用的系统输出是 HLPFUZZ fuzzing 工具。因此按 taxonomy v2 的最终系统角色归为 GENERATOR/G4，而非 SELECTOR：中心性分数只负责内部调度，系统交付的是测试/fuzz 能力。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用运行时混合 fuzzing 和提示式 LLM 推理。

### 4.1 灰盒 fuzzing runtime

runtime 包含目标执行器、种子调度器、变异器和 bug oracle。覆盖率反馈驱动浅层探索，语法感知变异生成候选输入。论文输入包括源代码、平均约 6 行的自然语言规格、语法文件及需要时的开源种子（图 3、第 7.1 节）。

### 4.2 约束优先级

从 CFG 中选取未覆盖基本块对应的约束，删除已覆盖节点后计算剩余图上的 Katz centrality；再用 LLM 历史成功/失败记录校准得分。约束分为只覆盖 enclosing function 之前路径的 SC1、通过前置约束但未通过完整约束的 SC2、通过完整约束的 SC3（第 4.2 节）。

### 4.3 迭代求解与验证

先向 LLM 提供目标基本块所在函数的窄上下文，并给出可触发 enclosing function 的 learning example。若失败，debug 模式下用 LLDB 做 divergence analysis，定位第一处未命中的调用帧并补入对应函数；若仍缺少外部符号定义，则由 LLM 通过 clangd 等语言服务器请求符号。每次候选解都通过实际执行/覆盖率验证，超过迭代阈值则放弃。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有 SFT、PPO 或 GRPO。关键目标是用 CFG 影响力和历史求解结果排序约束。

论文给出的中心性基础是 Katz centrality（式 (1)，第 4.1 节），其具体矩阵展开在当前 PDF 的文字提取中排版不完整；核心含义是衡量节点对下游未覆盖节点的影响。混合校准公式为：

```text
SF_i = (1 + γ · PCsuccess(F_i)) /
       (1 + γ · PCsuccess(F_i) + FCfailure(i) + PCfailure(F_i))
Score_i = SF_i · C_i
```

其中 `C_i` 是基本块 `i` 的中心性，`F_i` 是其 enclosing function，`PCsuccess/PCfailure` 统计函数级前置约束成功/失败，`FCfailure` 统计目标完整约束失败，`γ` 为正参数。成功时缩放因子保持为 1；失败越多，相关函数或基本块的优先级越低。实验中设置 `α = 0.5`、`γ = 3`（第 6 节实现说明）。这不是训练损失，而是 fuzzing 约束调度目标。

## 6. 实验设置

### 6.1 数据集来源

论文没有使用传统意义上的训练/验证/测试数据集。真实 bug-hunting 覆盖 9 个语言处理器、7 种编程语言；比较实验使用从开源仓库收集的相同初始输入（第 7.1 节）。作者还提供了包含规格、grammar 和互联网种子的 Docker 示例 artifact（第 13 节 Open Science）。

### 6.2 模型与工具

主要 LLM 是通过 OpenAI API 调用的 GPT-4o-mini，temperature=0。另测 Llama 3.1 70B Instruct 和 8B Instruct。运行环境为 Ubuntu 22.04.2 LTS、两颗 AMD EPYC 7452 32-core 处理器、1024 GB RAM；多数对比实验每个 Docker 容器限制单 CPU 核。工具包括 AFL++、LLDB、clangd、graph-tool、SanitizerCoverage 和目标语言处理器；论文未明确给出所有工具版本。

### 6.3 对比方法

消融与 AFL++（相同语法感知变异）比较；外部比较包括通用约束求解 SymQEMU、REDQUEEN，通用语法 fuzzers POLYGLOT、Grammarinator，以及 Clang++ 上的 YARPGen、Hermes 上的 Fuzzilli（第 7.1、7.4 节）。

### 6.4 评价指标

主要指标是 SanitizerCoverage collision-free instrumentation 记录的 edge coverage、约束求解成功率、发现/确认/修复 bug 数，以及 24 小时运行结果。edge coverage 越高越好；bug 数以论文报告的发现、开发者确认和修复口径区分，不能等同于全部已验证安全缺陷。

## 7. 实验结果与结论

### 7.1 主要结果

HLPFUZZ 在 9 个语言处理器、7 种语言上发现 52 个 bug，其中 37 个获开发者确认、14 个已修复（第 7.2 节、表 7）。真实 bug-hunting 总计约三个月；LLM 查询平均成本约 0.2 美元/小时。

### 7.2 与传统方法的比较

在 Clang++、Flang、Hermes、PHP 的 24 小时、重复 5 次平均实验中，相对 AFL++ syntax baseline，HLPFUZZ 平均多发现 109%、45.8%、34.3%、31.0% 的边，顺序对应四个目标（第 7.3 节）。与外部工具相比，HLPFUZZ 的 edge coverage 在四个目标均最高；相对第二名，Clang++、Flang、PHP、Hermes 的提升分别为 90.34%、40.71%、35.28%、7.18%（第 7.4 节、图 7）。

### 7.3 与其他 LLM 方法的比较

本文的对比重点是传统/目标专用 fuzzers，并未把另一个 LLM compiler fuzzer 作为主实验 baseline。换用 Llama 3.1 70B 时相对 baseline 的提升为 24.7%–85.1%，8B 时为 6.1%–49.3%；作者观察到 8B 更常出现格式错误和忽略反馈（第 7.3 节、表 5）。

### 7.4 消融实验

去掉 hybrid centrality prioritization、iterative context construction、hybrid calibration 或 language server integration 的变体整体下降。完整系统相对无语言服务器变体的覆盖增加范围为 PHP 2.65% 到 Hermes 8.01%；divergence analysis 的增加范围为 Hermes 4.21% 到 Clang++ 19.3%；hybrid calibration 相对只做中心性分析的变体增加范围为 Hermes 8.02% 到 Clang++ 20.51%。平均成功解需要 2.64 次迭代，中位数为 2–3 次；单次迭代成功率仅 5%（Flang）到 9.1%（Hermes）。

### 7.5 案例分析

论文展示了 Flang 的 `pack-mask` assertion failure 和 Clang++ 模板参数包/ lambda 组合导致的 stack overflow。前者需要依赖关系和正确参数类型，HLPFUZZ 两次上下文迭代后求解；后者三次迭代后生成模板函数、lambda 和参数包结构（第 7.2 节、图 5、图 11–12）。

## 8. 主要创新点

### 8.1 创新点一：混合中心性优先级

将 CFG 中未覆盖区域的中心性与 LLM 历史求解成功/失败结合，使 LLM 预算优先用于潜在覆盖收益高且相对可求解的约束。消融结果支持该组合优于仅中心性或随机选择。

### 8.2 创新点二：迭代上下文构造

不是一次性提供大规模代码，而是从 enclosing function 的窄上下文开始，结合执行分歧定位和语言服务器按需取符号定义。实验中的多轮成功率与覆盖率变化表明该机制有独立贡献。

### 8.3 创新点三：通用混合 fuzzing 工具

HLPFUZZ 将上述机制嵌入传统灰盒 runtime，在 7 种语言的 9 个目标上运行，仅需短自然语言规格，减少了针对每个语言处理器手写约束启发式的依赖。贡献是工具与系统组合，不是单独提出新的 LLM 训练算法。

## 9. 局限性

### 9.1 论文明确承认的局限

静态分析缺少过程间数据流信息，可能导致优先级不精确；divergence analysis 只沿 learning example 的执行路径提供上下文，可能漏掉其他可行路径（第 8.1 节）。对二进制输入、约束较简单的 Binutils 等目标，LLM 缺乏二进制训练与检索经验，泛化能力尚未验证（第 8.2 节）。

### 9.2 阅读后的潜在局限

HLPFUZZ 的变异不保证语义保持，缺少逻辑错误 oracle，因此没有发现某些 Clang++ 循环优化错误；YARPGen 在该类目标上更有优势（第 7.5 节）。报告的 coverage 是边覆盖，不等于语义缺陷覆盖；LLM 求解和调试/语言服务器调用也带来成本与工程依赖。将平台直接替换为 RISC-V 还需要 RISC-V 语言/编译器输入约束、目标后端 oracle 和真实硬件验证，论文没有做这项实验。

## 10. 阅读后的研究方向反思

值得借鉴的是“便宜的灰盒探索 + 昂贵的 LLM 深层解锁”分工，以及用历史失败校准 LLM 调度、用按需上下文控制 token 成本。其核心贡献是 HLPFUZZ 的两种调度/上下文机制，不能仅通过换成 RISC-V 或换一个模型声称复现出新贡献。对本仓库方向而言，它更适合作为 GENERATOR/G4 工具模块或 compiler fuzzing baseline；若研究 RISC-V，应新增 ISA/后端特定约束、跨架构差分 oracle 或真实硬件反馈，而不是简单平台替换。

## 11. 可进一步尝试的研究方向

### 11.1 面向 LLVM/RISC-V 后端的语义保持深层 fuzzing

#### 研究问题

如何让 LLM 求解后端约束，同时保留可用于 miscompilation 检测的程序语义。

#### 与原论文的区别

增加 translation validation 或差分执行 oracle，并专门覆盖 LLVM RISC-V instruction selection、legalization 和 RVV lowering。

#### 可能的创新点

将中心性/历史难度与语义保持分数联合调度；区分 crash、wrong-code 与性能回归。

#### 实验框架

```text
LLVM IR/C 测试种子 → HLPFUZZ 约束求解 → RISC-V 编译/模拟或硬件执行
                   → RVV/标量差分与语义 oracle → 覆盖率、缺陷、成本
```

#### 可行性

需要 LLVM、QEMU 或 RISC-V 板卡、RVV 测试集和可重复的差分 oracle。

#### 主要风险

模拟器与真实硬件差异、未定义行为和语义保持约束可能显著降低有效样本率。

### 11.2 跨语言处理器的检索增强约束上下文

#### 研究问题

能否从历史成功约束、错误日志和符号依赖图中检索更短且更准确的上下文。

#### 与原论文的区别

把当前按需语言服务器取符号扩展为可审计的结构化检索，而非只依赖 LLM 请求。

#### 可能的创新点

以约束类型和 CFG 位置为键的上下文缓存，以及跨版本/跨语言的失败迁移模型。

#### 实验框架

```text
目标约束 → 依赖图/历史库检索 → LLM 生成 → 执行验证 → 更新成功/失败库
```

#### 可行性

需要 HLPFUZZ artifact、语言服务器、版本化编译器和历史运行日志。

#### 主要风险

检索到的旧上下文可能过时，且历史库可能造成训练/评测泄漏。

## 12. 与其他已读文献的关系

本 staging 子任务只通读并交付 HLPFUZZ 一篇论文，未在本轮独立通读其他候选论文，因此不建立未经确认的横向事实比较。就正式语料去重审计而言，HLPFUZZ 的题名、作者、DOI/arXiv 标识和本地 PDF 路径均未命中正式 taxonomy、角色索引、逐篇目录、年份清单、候选缓存或本轮其他 staging；它与已有 compiler fuzzing 条目的关系应由主代理在最终联合去重时再次复核。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | 语言处理器深层约束 fuzzing |
| 核心问题 | 通用方法难穿透复杂约束，目标专用方法难泛化 |
| 输入 | 源代码、自然语言规格、语法文件、种子 |
| 输出 | 可复用 HLPFUZZ 工具、测试输入、发现的 bug |
| 核心方法 | 混合中心性优先级 + 迭代上下文构造 |
| 使用的模型 | GPT-4o-mini；Llama 3.1 70B/8B 对照 |
| 使用的编译器工具 | AFL++、LLDB、clangd、SanitizerCoverage、graph-tool |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 不是核心机制；论文中未明确说明形式化验证流程 |
| 数据集规模 | 无训练集；9 个语言处理器、7 种语言 |
| 主要指标 | edge coverage、求解成功率、bug 数、查询成本 |
| 最重要实验结果 | 发现 52 个 bug，37 个确认，14 个修复；相对第二名覆盖最高提升 90.34% |
| 核心创新 | 用历史难度调度 LLM，并按执行分歧/符号需求增补上下文 |
| 主要局限 | 缺少过程间分析、语义保持和逻辑错误 oracle；二进制目标未验证 |
| 与 RISC-V 研究的相关性 | 中：可迁移 fuzzing 框架，但论文未做 RISC-V 实验 |
| 最适合作为 | GENERATOR/G4 工具模块、compiler fuzzing baseline |

> 这篇论文最值得学习的是把昂贵的 LLM 推理限制在高收益深层约束，并用可执行反馈逐步补齐上下文；最主要的局限是它更擅长覆盖/崩溃发现而不是语义保持的 wrong-code 检测；如果用于后续研究，合理方式是加入 RISC-V/LLVM 后端专用 oracle 和真实执行验证，而不是简单替换平台。
