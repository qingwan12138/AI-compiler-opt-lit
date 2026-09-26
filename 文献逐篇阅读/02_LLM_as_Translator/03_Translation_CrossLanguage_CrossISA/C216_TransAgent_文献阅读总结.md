# TransAgent 文献阅读总结

论文题目：**TransAgent: Enhancing LLM-Based Code Translation via Fine-Grained Execution Alignment**

作者：Zhiqiang Yuan、Weitong Chen、Hanlin Wang、Xin Peng、Zhenpeng Chen、Yiling Lou

发表时间：2026（arXiv 首稿 2024；FSE 正式版 2026）

发表平台：Proceedings of the ACM on Software Engineering，Vol. 3，FSE，Article FSE071，2026 年 7 月

论文链接或编号：DOI [10.1145/3797099](https://doi.org/10.1145/3797099)；arXiv:2409.19894v5

关键词：代码翻译、程序修复、大语言模型、多智能体、控制流图、运行时对齐、语义错误定位

> 本笔记依据 FSE 2026 正式 PDF（24 页）整理。TransAgent 的最终 LM 输出是跨语言 target program 及其修复补丁，因此按 Taxonomy v2 建议归入 TRANSLATOR/T3。

## 1. 研究背景

代码翻译把一种编程语言的程序转换为另一种语言，同时保持功能一致。规则式 transpiler 需要人工维护大量转换规则，学习式方法需要大规模平行代码并面临训练成本；LLM 能直接生成 target code，但生成结果可能同时包含语法错误和语义错误。

论文指出，已有 LLM 翻译方法多采用“先翻译、再修复”流程，但通常只根据编译器报错或函数级测试结果做端到端修复，难以定位发生语义偏差的具体代码区域。TransAgent 将程序结构分析与两端运行状态对齐结合起来，为修复 agent 提供更细粒度证据。

## 2. 论文要解决的问题

### 2.1 语法错误修复

初始翻译结果可能无法编译。系统需要将编译器错误消息转换为更具体的修复建议，再让 LLM 修改 target program。

### 2.2 语义错误定位与修复

target program 可能编译并运行，但输出与 source program 不一致。论文研究如何利用 source/target 两端对应代码块的中间运行状态定位根因，并只修复错误代码块。

### 2.3 对齐粒度问题

逐语句顺序映射会受到代码行移动、一对多翻译和跨语言结构差异影响。论文使用 source 控制流图（CFG）划分出的原子代码块，再用 LLM 将代码块映射到 target。

> 本文主要研究：如何利用 CFG 引导的代码块映射和跨程序运行时状态比较，提升 LLM 跨语言代码翻译及错误修复的正确率。

## 3. 核心方法概述

TransAgent 是由四个协作 agent 组成的多智能体翻译与修复系统：Initial Code Translator、Syntax Error Fixer、Code Aligner 和 Semantic Error Fixer。其最终产物仍是 target source program 或该程序的修复版本，编译器、测试和运行时执行器提供外部证据。

```text
source program
      ↓
Initial Code Translator 生成测试与初始 target program
      ↓
编译 target，Syntax Error Fixer 根据错误消息修复
      ↓
CFG 将 source 划分为代码块，Code Aligner 用 LLM 建立跨语言块映射
      ↓
并行执行 source/target，比较对应块的运行时变量状态
      ↓
Semantic Error Fixer 定位错误块并生成局部补丁
      ↓
测试全部通过的 target program
```

Initial Code Translator 还为程序生成测试用例。Syntax Error Fixer 将编译器信息整理为更具体的提示。Code Aligner 结合 CFG 结构与 LLM 对齐，而不是仅按行号对应。Semantic Error Fixer 有 value-aware 和 vanilla 两种修复策略：前者提供变量运行时差异，后者提供更一般的错误信息。

## 4. 实验框架与训练流程

### 4.1 运行阶段

本文不训练 TransAgent 专用模型，主要采用提示驱动的多 agent 推理和外部工具反馈。Initial Code Translator 先翻译并生成测试；Syntax Error Fixer 循环处理编译错误；之后系统进行代码块映射和运行时比较，最后由 Semantic Error Fixer 生成局部修复。

### 4.2 代码块对齐

系统用 CFG 将 source 划分为结构化代码块，再通过 LLM 将每个 source block 映射到 target block。对齐结果用于决定哪些 target block 的中间状态需要比较。

### 4.3 语义修复

系统收集两端对应块的变量值和执行轨迹，发现偏差后向 Semantic Error Fixer 提供错误块、运行时差异和测试上下文。修复后重新编译、运行和检查；表 3 显示六种语言方向中超过 92% 的样例在不超过两次迭代内完成修复。

### 4.4 模型泛化

默认实验使用 Deepseek-Coder-6.7B-Instruct；泛化实验还使用 Llama-3-8B-Instruct、ChatGLM2-6B 和 Deepseek-Coder-33B-Instruct。作者选择较小模型并要求其知识截止时间早于新基准，以降低数据泄漏风险。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数或模型训练损失。核心评价依据是测试驱动的计算准确率：

```text
CA = 1  当 target program 通过全部测试用例
CA = 0  否则
```

论文没有把 CA 写成训练目标，而是把它作为翻译和修复是否成功的评价指标。运行时状态比较用于定位差异，不构成形式化等价证明。

## 6. 实验设置

### 6.1 数据集来源

作者从 LeetCode、GeeksforGeeks 等竞赛网站收集 2023 年 8 月之后发布的 Java、Python、C++ 解答，以避免与模型旧知识重叠。利用 gpt-4o-mini 为每个解答额外生成 10 个测试用例；执行不同语言版本，删除行为不一致的任务；两位作者进一步人工检查。

最终新基准包含 210 对 Python–Java、200 对 Python–C++、204 对 Java–C++ 翻译任务。平均行覆盖率为 Python 98.4%、Java 98.7%、C++ 98.4%。

### 6.2 模型与工具

使用的模型包括 Deepseek-Coder-6.7B-Instruct、Llama-3-8B-Instruct、ChatGLM2-6B 和 Deepseek-Coder-33B-Instruct。工具链包含目标语言编译器/运行时、CFG 分析、测试执行器和运行时状态采集器。具体编译器版本论文中未明确说明。

### 6.3 对比方法

- UniTrans：LLM 代码翻译与迭代修复方法。
- TransCoder：代表性学习式代码翻译方法。
- Agentless：用于程序修复的 LLM 方法。
- TransAgent-TM：将 TransAgent 的 Code Aligner 替换为 TransMap 顺序式对齐策略的变体。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| CA | target 是否通过全部测试用例，代表功能正确率 | 越大越好 |
| CodeBLEU | target 与 source 的代码相似度 | 越大越好 |
| Repair Accuracy | 生成修复后通过全部测试的比例 | 越大越好 |
| Block Mapping Accuracy | source/target 代码块正确对应的比例 | 越大越好 |
| 平均时间 | 每个样例完成翻译/修复的平均耗时 | 越小越好 |

## 7. 实验结果与结论

### 7.1 主要结果

在六种语言方向上，TransAgent 的总体翻译效果均优于 TransCoder 和 UniTrans。Python→Java 场景中，TransAgent CA 为 89.5%，比 TransCoder 的 2.4% 高 87.1 个百分点，比 UniTrans 的 56.2% 高 33.3 个百分点；相对两种基线的 Wilcoxon 检验均为 p << 0.001。

### 7.2 迭代成本

超过 92% 的样例在不超过两次修复迭代内完成。每个样例平均耗时约 19 秒，UniTrans 约 24 秒；该比较在论文的相同实验环境和实现设置下进行。

### 7.3 消融实验

Syntax Error Fixer 与 Semantic Error Fixer 均带来额外收益。C++→Java 场景中，完整系统相对 Initial Code Translator 提升 31.0%；Python→Java 场景提升 40.0%。在 Python→Java 中，value-aware Semantic Error Fixer 相对仅加入 Syntax Error Fixer 的配置增加 6.3%，vanilla 策略再增加 1.9%。

### 7.4 修复准确率

六种方向上，TransAgent 的 Repair Accuracy 均高于 Agentless 和 TransAgent-TM。各方向数值分别为 Java→Python 59.4%、Java→C++ 45.4%、C++→Java 83.5%、C++→Python 76.5%、Python→C++ 50.0%、Python→Java 79.2%。论文报告平均比 Agentless 高 56.7%，比 TransAgent-TM 高 14.5%。

### 7.5 对齐与泛化

六种语言方向的代码块映射准确率为 95.8%–100%，TransMap 为 64.6%–93.8%。在不同模型上，完整系统均比 Initial Code Translator 更好；例如 Python→Java 中，Llama-3-8B-Instruct 提升 31.5 个百分点，ChatGLM2-6B 提升 9.5 个百分点，Deepseek-Coder-33B-Instruct 提升 19.7 个百分点。

## 8. 主要创新点

### 8.1 CFG 引导的跨语言代码块对齐

论文把 source 程序先按 CFG 划分为结构化代码块，再交给 LLM 建立跨语言映射，减少行移动和一对多翻译造成的错位。对齐准确率实验支持该设计有效。

### 8.2 运行时状态驱动的细粒度语义修复

与只看最终输出或函数级错误的修复方法相比，TransAgent 比较对应代码块的中间变量状态，并把差异用于局部补丁生成。Repair Accuracy 和 TransAgent-TM 对比表明，细粒度定位是实质性贡献。

### 8.3 语法修复与语义修复的分工

系统将编译器错误处理和运行时语义偏差处理拆为两个 agent，使每个 agent 获得不同类型的证据；消融实验显示两者都能提升 CA。

## 9. 局限性

### 9.1 论文明确承认的局限

- 系统假设 source program 可执行；缺少依赖、无法编译或运行环境不完整时，无法获得可靠运行时信号。
- 评测仍可能存在数据泄漏风险，虽然作者通过 2023 年 8 月后数据和模型知识截止日期进行缓解。
- 实现错误、基线复现和指标计算可能影响内部有效性，作者通过调试和统一实现设置缓解。

### 9.2 阅读后的潜在局限

- 测试驱动的 CA 不是形式化语义等价证明，测试覆盖不足时仍可能漏掉错误。
- 代码块对齐和状态采集增加了运行时与工程依赖；论文未给出在大型多文件项目上的成本。
- 评测集中在 Java、Python、C++ 竞赛程序，不能直接推断对 LLVM IR、汇编、GPU kernel 或 RISC-V ISA 翻译同样有效。
- 论文没有明确给出编译器版本和完整硬件配置，可能降低严格复现能力。

## 10. 阅读后的研究方向反思

本文最适合归为 **TRANSLATOR/T3_Cross-language_ISA_Translation**：LLM 最终直接生成或修改 target source code，CFG 和运行时工具只是辅助证据。它对编译器优化的相关性主要来自“编译器/执行器反馈引导代码修复”，而不是 pass 或 schedule 选择。

值得借鉴的是将结构分析、动态证据和 LLM 生成拆开。仅把 Java/Python/C++ 换成 RISC-V 汇编或 LLVM IR，不足以形成新贡献；需要解决 IR 块映射、寄存器/内存状态对应、ISA 特有未定义行为或跨架构性能目标等新问题。该论文更适合作为跨语言翻译和反馈修复 baseline。

## 11. 可进一步尝试的研究方向

### 11.1 面向 LLVM IR 的块级语义对齐

#### 研究问题

如何把源 IR 与目标 IR 的 CFG、值依赖和内存状态对齐，并定位翻译后的错误基本块。

#### 与原论文的区别

从源语言代码块扩展到带 SSA、类型和内存别名信息的 IR 块，不能只依赖自然语言映射。

#### 可能的创新点

将 CFG、支配树、SSA def-use 链作为结构约束，与 LLM 的候选映射结合。

#### 实验框架

```text
C/C++ → LLVM IR → LLM IR 翻译 → CFG/SSA 对齐 → Alive2/测试验证 → 局部修复
```

#### 可行性与风险

LLVM 工具链和 Alive2 可提供基础设施；风险是 IR 级状态比较和未定义行为判定较复杂。

### 11.2 面向 RISC-V 的运行时反馈翻译

#### 研究问题

如何利用 RISC-V 与参考 ISA 的执行轨迹差异定位翻译错误，同时区分 ISA 语义错误与 ABI/运行时错误。

#### 与原论文的区别

加入 RISC-V 向量扩展、调用约定和内存模型证据，而不是只做语言间 source mapping。

#### 可能的创新点

设计指令块到源代码块的双向映射和寄存器/内存快照对齐协议。

#### 实验框架

```text
源程序 → LLM 翻译到 RISC-V → Spike/QEMU 与参考执行 → 状态差异定位 → 局部修复
```

#### 可行性与风险

可用 LLVM RISC-V 后端和 Spike；风险是模拟器轨迹与真实硬件行为存在差异。

### 11.3 语义正确与硬件性能联合修复

#### 研究问题

如何在保持功能正确的同时，让 LLM 根据真实硬件计时选择局部代码修复。

#### 与原论文的区别

TransAgent 主要优化翻译正确率；新方向将性能反馈纳入候选排序，并限制性能回归。

#### 可能的创新点

把测试通过、状态对齐和性能预算组合成可审计的修复接受准则。

#### 实验框架

```text
翻译候选 → 编译/测试 → 状态对齐 → 硬件 profiling → LLM 局部修复或回退
```

#### 可行性与风险

适合 GPU、RISC-V 或 FPGA 小规模 kernel；风险是硬件测量噪声和修复搜索成本。

## 12. 与其他已读文献的关系

本轮仅完成 TransAgent 的正文阅读，不能把其他队列论文的未读内容当作事实比较。就角色边界而言，TransAgent 与 Selector 类 pass/schedule 论文的区别是其 LM 直接输出跨语言 target program 和修复 patch；与 Generator 类测试/规则生成论文的区别是其主要产物是当前任务的翻译结果，而不是长期可复用的 compiler pass 或 heuristic。它可作为跨语言 Translator baseline，并可与编译器反馈修复方法组合。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLM 跨语言代码翻译与错误修复 |
| 核心问题 | 语法/语义错误难定位，跨语言行级映射不稳 |
| 输入 | source program、测试、编译器与运行时反馈 |
| 输出 | 修复后的 target source program |
| 核心方法 | CFG 块级对齐 + 跨程序运行时状态比较 + 多 agent 修复 |
| 使用的模型 | Deepseek-Coder-6.7B、Llama-3-8B、ChatGLM2-6B、Deepseek-Coder-33B |
| 使用的编译器工具 | 目标语言编译器、CFG 分析、测试和运行时执行器；版本未明确 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；使用测试和运行时比较 |
| 数据集规模 | Python–Java 210、Python–C++ 200、Java–C++ 204 对 |
| 主要指标 | CA、CodeBLEU、Repair Accuracy、映射准确率、平均时间 |
| 最重要实验结果 | Python→Java CA 89.5%；比 UniTrans 高 33.3 个百分点；平均修复准确率比 Agentless 高 56.7% |
| 核心创新 | CFG 引导的块级跨语言映射与运行时状态驱动局部修复 |
| 主要局限 | 依赖可执行 source 和测试；不是形式化等价证明；语言/项目规模有限 |
| 与 RISC-V 研究的相关性 | 中低；可借鉴反馈修复框架，但论文未研究 RISC-V |
| 最适合作为 | TRANSLATOR/T3 baseline 与运行时反馈修复方法参考 |

这篇论文最值得学习的是把翻译、结构对齐和错误修复拆成可验证的步骤；最主要的局限是依赖可执行程序和测试覆盖，不能直接保证语义等价；如果用于后续研究，最合理的使用方式是作为跨语言翻译与反馈修复 baseline，而不是简单替换目标语言或硬件平台。
