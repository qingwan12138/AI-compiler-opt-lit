# SK²Decompile 文献阅读总结

论文题目：**SK²Decompile: LLM-based Two-Phase Binary Decompilation from Skeleton to Skin**

作者：Hanzhuo Tan、Weihao Li、Xiaolong Tian、Siyi Wang、Jiaming Liu、Jing Li、Yuqun Zhang

发表时间：2026（论文 PDF 为 arXiv:2509.22114v1，提交日期 2025-09-26；官方会议页面标为 ICLR 2026）

发表平台：ICLR 2026

论文链接或编号：arXiv:2509.22114
元数据核验来源：[ICLR/OpenReview 论文来源](https://openreview.net/)；[匿名论文仓库](https://github.com/anonymous-git-paper/sk2decompile)；[作者相关项目仓库](https://github.com/albertan017/LLM4Decompile/tree/main/sk2decompile)；[arXiv](https://arxiv.org/abs/2509.22114)
代码/数据/工件：论文所列匿名仓库：[sk2decompile](https://github.com/anonymous-git-paper/sk2decompile)；相关作者项目组件：[LLM4Decompile/sk2decompile](https://github.com/albertan017/LLM4Decompile/tree/main/sk2decompile)（作者归属关系分别标注）

代码仓库：[SK²Decompile 官方匿名仓库](https://github.com/anonymous-git-paper/sk2decompile)；作者项目仓库也已发布对应模型/脚本：[LLM4Decompile/sk2decompile](https://github.com/albertan017/LLM4Decompile/tree/main/sk2decompile)。
来源：[ICLR OpenReview 录用稿中给出的匿名代码地址](https://openreview.net/pdf/35435c73fb909ee84c1d913d71d150b6897c5963.pdf)；[作者仓库公告](https://github.com/albertan017/LLM4Decompile)。
关键词：二进制反编译、LLM、结构恢复、标识符命名、中间表示、强化学习、编译器反馈

> 本笔记依据本 staging 目录中的 18 页 PDF（arXiv v1）逐页阅读。论文事实、阅读分析和后续建议分开描述；未在 PDF 中明确给出的信息不作推断。

## 1. 研究背景

二进制反编译（decompilation）要从编译后的机器码/伪代码恢复高层 C 源代码，服务于恶意软件分析、漏洞发现和遗失源代码恢复。传统工具 Ghidra、IDA 主要依靠控制流/数据流分析和模式匹配，通常能保留基本逻辑，但输出接近低层伪代码，难以阅读和重新编译。LLM 方法可以把伪代码改写得更像源代码，却常在控制流、数据布局和功能正确性上失败。

作者认为核心困难是同时推断三类信息：控制流结构、数据结构布局以及有语义的类型/变量/函数名。信息量和不确定性集中在一个生成任务中，导致“正确但难读”与“好读但不等价”的权衡。本文将任务划成“骨架（skeleton）到皮肤（skin）”两阶段：先恢复不含原始标识符的结构，再恢复有意义的标识符。

## 2. 论文要解决的问题

### 2.1 结构与可读性耦合

本文研究如何从低层伪代码恢复可编译、功能结构较可靠的高层程序结构，包括循环、条件分支、类型关系和字段访问。动机示例中，`while(1)` 加 `goto` 应被整理为高层条件循环，指针偏移应被整理为嵌套字段访问。

### 2.2 标识符和数据语义恢复不足

仅恢复语法结构不能让分析者理解程序。本文还研究如何为函数、类型、字段和变量生成反映程序语义的名称；例如把泛化的 `field1`、`var3` 恢复为更接近 `available`、`state` 等语义角色的名称。

> 本文主要研究：如何通过一个结构化中间表示，把二进制反编译拆为结构恢复和标识符命名两个可分别优化的 LLM 任务，从而同时提升功能正确性与源代码可读性。

## 3. 核心方法概述

SK²Decompile（Skeleton-to-Skin Decompile）包含两个顺序模型。Structure Recovery 模型把低层伪代码转换为去标识符化的源代码式 IR；Identifier Naming 模型再把 IR 中的占位符替换为语义更丰富的名称。两个阶段分别用监督微调（SFT）和强化学习（RL）训练，并使用不同奖励。

```text
二进制
  ↓ IDA Pro 生成伪代码
低层伪代码 u
  ↓ Structure Recovery 模型 + 编译器可编译性奖励
结构化、标识符匿名化的 IR i（骨架）
  ↓ Identifier Naming 模型 + 代码语义相似度奖励
高层 C 源代码 s（皮肤）
  ↓ 重新编译/单元测试或可读性评估
功能正确性与可读性指标
```

IR 是从源代码生成的、将用户定义的函数/类型/字段/变量名替换成 `func1`、`type1`、`field1`、`var1` 等占位符的代码；标准类型和库函数等由伪代码中提取的保留列表保护。该设计保留结构和数据访问模式，同时减少低层噪声以及不可从二进制直接获得的名称信息。

## 4. 实验框架与训练流程

本文是训练型二进制反编译系统，不是只做提示词推理的静态框架。

### 4.1 数据构造与 IR 生成

作者从 ExeBench 和 Decompile-Bench 的 C 程序收集训练语料，用 GCC 和 Clang 在 x86 Linux 上以 `-O0` 至 `-O3` 编译，再剥离二进制并用 IDA Pro 生成伪代码。源代码删除注释并用 clang-format 规范化，伪代码按 R2I 标准格式化；用 MinHash-LSH 去除近重复样本。源代码通过 AST 遍历完成标识符分类、映射和替换，生成 IR。

### 4.2 Structure Recovery

模型输入伪代码，输出结构化匿名 IR。先以序列到序列（S2S）方式做一轮 SFT，再使用 GRPO（Group Relative Policy Optimization，组相对策略优化）进行 RL。每个候选 IR 交给编译器检查；不能编译的候选得 0 分，能编译的候选再按占位符集合恢复程度加分。

### 4.3 Identifier Naming

第二个模型输入 IR，输出带语义名称的源代码。该阶段也先 SFT、后 RL，但奖励不要求逐字匹配参考名称，而是比较生成代码和参考源代码的嵌入余弦相似度，以容纳多个语义等价的命名方案。

### 4.4 推理和评测

两个模型均从 LLM4Decompile-6.7B checkpoint 初始化。SFT 使用 LLaMA-Factory，一轮、batch size 128、学习率 `3e-6`；RL 因计算约束使用随机 50,000 样本。RL 阶段使用 veRL 中的 GRPO。编译可行性奖励依赖 Psyche-C 生成头文件，语义奖励使用 `qwen-embedding-0.6B`。训练在 NVIDIA H800-80GB 集群进行；推理使用 vLLM 和 greedy decoding。

## 5. 奖励函数、损失函数或关键公式

### 5.1 SFT 的交叉熵

论文给出的 token 级交叉熵为：

```text
L_CE(θ) = -Σ(i=1..N) log P_θ(y_i | y_<i, x)
```

它惩罚逐 token 预测错误，但不能区分“导致编译失败的分号错误”和“语义等价但名字不同”的错误。

### 5.2 Structure Recovery 奖励

```text
r_placeholder = |I_gen ∩ I_IR| / |I_gen ∪ I_IR|

r_structure = 0.0                    if IR cannot be compiled
              1.0 + r_placeholder    if IR can be compiled
```

其中 `I_gen` 是生成 IR 的占位符集合，`I_IR` 是真实 IR 的占位符集合；Jaccard 相似度鼓励恢复正确的数据布局/占位符关系。编译器可编译性是硬门槛，编译失败时不因局部 token 正确而获得结构奖励。

### 5.3 Identifier Naming 奖励

```text
r_identifier = cos(e_gen, e_src)
              = (e_gen · e_src) / (||e_gen|| ||e_src||)
```

`e_gen` 与 `e_src` 分别是生成代码和参考源代码的嵌入。该目标鼓励整体语义接近，而不是机械地复制原始名称。论文未报告专门的奖励投机检测或独立奖励校准实验；这一点应视为未明确说明。

## 6. 实验设置

### 6.1 数据集来源

训练语料来自 ExeBench 与 Decompile-Bench；作者称处理后约有 500 万样本、约 20 亿 token 伪代码、15 亿 token IR 和 15 亿 token 源代码。二进制目标为 x86 Linux，编译器为 GCC/Clang，优化级别为 `-O0` 至 `-O3`。论文称使用 MinHash-LSH 去重，但 PDF 未给出训练/验证/测试的精确拆分比例，也未给出每个原始数据集各自的样本数。

评测使用 HumanEval、MBPP、ExeBench 和 GitHub2025，沿用相同编译流程。HumanEval 和 MBPP 支持执行评测；ExeBench 因剥离过程破坏其执行环境而不做 re-executability 评测。论文说明对 stripped binary 的输出会恢复原始函数名以便测试。

### 6.2 模型与工具

| 项目 | 设置 |
|---|---|
| 初始化模型 | LLM4Decompile-6.7B；结构和命名模型各自初始化 |
| SFT | LLaMA-Factory，一轮，batch size 128，学习率 `3e-6` |
| RL | veRL 中的 GRPO；随机 50,000 样本 |
| 可编译性反馈 | Psyche-C 生成头文件并检查编译 |
| 语义奖励 | `qwen-embedding-0.6B` |
| 推理 | vLLM，greedy decoding |
| 硬件 | NVIDIA H800-80GB 集群 |
| 二进制/伪代码工具 | IDA Pro；编译使用 GCC 与 Clang |

论文未明确给出 GCC、Clang、IDA Pro、Psyche-C、vLLM 和 H800 集群的完整版本/数量。

### 6.3 对比方法

主要 baseline 为 GPT-5-mini、LLM4Decompile 和 Idioms。Nova、Ref-Decomp、D-Lift 未纳入比较，作者给出的原因是其数据预处理细节不足或模型未公开，难以进行公平、可复现评估。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| Re-executability rate | 重新编译后的代码通过原始预定义单元测试的比例；不是穷尽语义证明 | 越大越好 |
| R2I | 基于 AST 特征的相对可读性指数；论文修改了失败样本计分方式 | 越大越好 |
| GPT-Judge | GPT-5-mini 对标识符质量 1–5 分的自动评价 | 越大越好 |

R2I 的改动包括用 Psyche-C 生成头文件提高解析成功率，并对仍无法解析的具体输出记 0 分，而不是丢弃整条样本。附录说明 re-executability 只运行预定义测试，所有测试通过才计为成功，因此不能表述为对所有输入的语义等价证明。

## 7. 实验结果与结论

### 7.1 主要结果

表 1 在 HumanEval/MBPP、`-O0` 至 `-O3` 上比较 re-executability。SK²Decompile 的平均值分别为 69.00% 和 59.63%；GPT-5-mini 分别为 56.75% 和 47.23%。因此相对 GPT-5-mini，平均提高 21.6 个百分点和 26.3 个百分点（论文摘要重点报告 HumanEval 的 21.6% 增益）。例如 HumanEval 各优化级别为 86.59%、70.59%、61.31%、57.52%；MBPP 为 69.76%、62.33%、54.83%、51.58%。这些是通过预定义测试的通过率，不是形式验证通过率。

表 2 的 R2I 平均值显示，SK²Decompile 在 HumanEval、MBPP、ExeBench、GitHub2025 上分别为 70.27、71.66、71.50、74.99。相对最佳 baseline Idioms，ExeBench 和 GitHub2025 分别提高 18.4% 和 29.4%（论文按相对可读性结果报告）。表 3 中 GPT-Judge 平均分为 HumanEval 4.24、MBPP 4.12、ExeBench 2.42、GitHub2025 3.06，满分 5 分。

### 7.2 与传统方法的比较

本文没有把 Ghidra 或 IDA 的传统反编译输出作为表 1–3 的主要数值 baseline；比较对象主要是 LLM decompiler。论文背景中将传统工具描述为逻辑恢复较强但可读性和可重新执行性不足，具体传统工具对比数字在当前 PDF 中未明确给出。

### 7.3 与其他 LLM 方法的比较

SK²Decompile 在所列三个指标和四个数据集上总体优于 GPT-5-mini、LLM4Decompile 和 Idioms。其优势集中在优化后二进制的结构恢复和真实二进制可读性；但 ExeBench/GitHub2025 上 GPT-Judge 绝对分数低于 HumanEval/MBPP，说明真实二进制命名仍更困难。

### 7.4 消融实验

表 4 比较五个变体。直接 `pseudo-src` 的 HumanEval/MBPP 平均 re-executability 为 54.86%/47.51%；仅 SFT 的 `pseudo-ir-src` 为 63.75%/52.83%；完整 `pseudo-ir-src-rl` 为 69.00%/59.63%。仅结构恢复的 `pseudo-ir` 在 HumanEval 已超过直接 baseline。对 `pseudo-ir` 加结构 RL 得 `pseudo-ir-rl`，HumanEval/MBPP 相比 SFT 结构模型分别提升约 10.0% 和 20.8%（按论文的相对比较表述）。附录表 5、6 对 R2I 与 GPT-Judge 观察到类似趋势：分解任务与专门奖励总体有效，结构模型本身不应在命名指标上被期待得分高。

### 7.5 案例分析

内存分配函数案例中，直接模型仍保留泛化 `void *` 和低层偏移；GPT-5-mini 也使用 `(char *)arena + 8` 等偏移，未恢复数据结构。SK²Decompile 的 Structure Recovery 恢复了对齐、边界判断和结构字段；Identifier Naming 随后给出更有语义的 `available`、`state` 等名称。该案例展示了结构恢复与命名阶段的互补性，但单个案例不能代表所有程序。

## 8. 主要创新点

### 8.1 创新点一：骨架到皮肤的两阶段反编译

现有 LLM 反编译通常直接从伪代码生成源代码，结构恢复与命名互相干扰。本文把功能结构和标识符语义拆开，并为两阶段分别训练/优化。表 4 的分解与完整模型结果支持这一设计有效；“使用 LLM”本身不是创新，关键是任务分解和阶段接口。

### 8.2 创新点二：以匿名源代码为 IR

本文把去除用户定义标识符的源代码作为桥接 IR，而不是另行设计复杂的低层 IR。它保留控制流、类型形状和字段访问，压缩难从二进制恢复的命名信息。作者用信息瓶颈目标解释其“压缩输入噪声、保留源代码相关结构”的选择，并给出 AST 驱动的自动生成算法。

### 8.3 创新点三：阶段专用的 RL 奖励

结构阶段用编译可行性加占位符 Jaccard 奖励，命名阶段用代码嵌入语义相似度奖励；这使奖励分别对齐“可编译结构”和“可读语义”，避免只使用 token 级 CE。消融结果显示奖励和任务分解都贡献了性能，但论文没有单独量化每个奖励项在真实硬件语义等价上的效果。

## 9. 局限性

### 9.1 论文明确或实验设计中可直接确认的限制

- 训练与评测主要基于 x86 Linux；对 ARM、RISC-V、GPU 或跨 ISA 迁移，论文没有实验支持。
- re-executability 依赖有限的预定义单元测试；附录明确说明无法穷举所有输入，因此不是形式化语义等价证明。
- ExeBench 不做重新执行率评测，因为剥离过程破坏其执行环境。
- RL 只使用随机 50,000 个样本，原因是计算约束；这可能限制 RL 覆盖度。
- 部分新近 baseline 因数据处理或模型不可得被排除，横向比较范围因此受限。
- 作者称商业软件仍会被混淆方法显著限制有效反编译；伦理部分也将适用范围限定为获授权的分析和恢复。

### 9.2 阅读后的潜在限制

- 两阶段接口的 IR 依赖源代码生成和命名类别规则；跨语言或跨 ISA 时，保留列表、类型系统和 ABI 规则需要重新定义，不能直接照搬。
- 使用 GPT-5-mini 做 GPT-Judge，且命名奖励依赖 embedding，可能把模型偏好的可读性当成语义正确性；论文没有报告人工盲评与自动评分的系统校准。
- 编译反馈只验证“能否编译”及头文件约束，并不自动保证与原二进制行为等价；错误但可编译的结构仍可能得到正奖励。
- 训练语料由公开基准和许可仓库构成，但 PDF 没有给出精确拆分和逐样本泄漏审计；MinHash-LSH 只能处理近重复，不等同于完整训练-测试去泄漏证明。
- 单一 LLM4Decompile-6.7B 初始化和 x86/IDA 处理链可能把系统优势与特定数据/工具生态耦合起来。

## 10. 阅读后的研究方向反思

最值得借鉴的是“先恢复可验证结构、再恢复可读名称”的接口思想，以及让编译器反馈参与结构阶段训练。对 LLVM/RISC-V 研究而言，它更适合作为二进制恢复的 baseline 或结构恢复/命名工具模块，而不是完整的跨 ISA 翻译方案。

把 x86 换成 RISC-V 本身创新性不足：那只是输入 ISA 替换，仍未解决 ABI、指令习惯、向量扩展和跨架构语义差异。若要形成研究问题，需要加入明确的多 ISA 结构不变表示、可验证的源-目标行为约束或跨架构数据稀缺条件。本文没有声称支持 RISC-V，也没有提供跨 ISA 实验，不能把它直接描述为 RISC-V 方法。

与“LLM 直接生成优化代码”的工作相比，SK²Decompile 的目标是恢复原程序结构和可读性，不是性能优化；与传统二进制翻译相比，它输出高层 C，而非直接输出目标 ISA 二进制。因此更适合用作反编译前端、源代码恢复基线或跨 ISA 翻译中的语义恢复模块。

## 11. 可进一步尝试的研究方向

### 11.1 多 ISA 共享骨架与目标 ABI 约束

#### 研究问题

能否把 x86/ARM/RISC-V 伪代码映射到统一、去命名但显式表达调用约定和内存语义的 IR，并在不同 ABI 下恢复可编译源代码？

#### 与原论文的区别

不是把输入架构简单替换，而是把 ISA 特有寄存器、调用约定、向量长度和内存模型显式纳入 IR。

#### 可能的创新点

跨 ISA 结构对齐、ABI-aware IR、架构条件化的结构奖励。

#### 实验框架

```text
x86/ARM/RISC-V 二进制 → 统一伪代码/IR → ABI 条件化结构恢复 → 源码/目标重编译 → 差分执行
```

#### 可行性

需要多架构交叉编译器、QEMU 或真实 RISC-V 板卡、反汇编/反编译工具以及带源代码的多 ISA 配对语料。

#### 主要风险

行为差异可能来自未定义行为、系统调用和浮点/向量语义；QEMU 通过不等于真实硬件性能或完整等价。

### 11.2 编译器反馈与等价性验证的分层奖励

#### 研究问题

如何在可编译性、单元测试和可扩展的形式化/符号验证之间分层，减少“可编译但语义错误”的奖励投机？

#### 与原论文的区别

原论文结构奖励的硬门槛是编译成功；新方向增加 IR 级翻译验证或符号执行，并测量验证成本。

#### 可能的创新点

按验证强度自适应分配奖励，区分语法正确、测试通过和受约束语义等价。

#### 实验框架

```text
候选 IR → 编译检查 → 单元测试 → 符号/翻译验证 → 分层奖励与候选排序
```

#### 可行性

可从小函数和 LLVM IR 开始，逐步加入 Alive2、符号执行或架构模拟器。

#### 主要风险

验证器覆盖有限且成本高；形式化通过的范围必须明确，不能外推为全程序正确。

### 11.3 面向 RISC-V 向量代码的骨架恢复

#### 研究问题

能否从 RVV 汇编恢复包含向量长度、掩码和尾部策略的结构化 C/C++ 或 RVV intrinsic 代码？

#### 与原论文的区别

重点从通用 x86 C 结构恢复转向 RVV 的可变向量长度和目标特有语义，并以真实 RVV 执行反馈校验。

#### 可能的创新点

显式化 `vl/vtype` 状态的骨架、掩码/尾部策略表示、编译器后端约束奖励。

#### 实验框架

```text
RVV 汇编 → 向量状态抽取 → 匿名结构 IR → RVV intrinsic 命名恢复 → LLVM/GCC 编译 → RVV 执行测试
```

#### 可行性

需要 LLVM RISC-V 后端、QEMU RVV 或真实硬件、RVV intrinsic 基准和向量语义测试。

#### 主要风险

不同 `VLEN`、编译器版本和尾部策略会改变可观察行为；只在一种模拟配置上通过不足以证明可移植性。

## 12. 与其他已读文献的关系

本轮 T2 仅完成这一篇论文，因此没有其他可依据正文横向比较的本批次论文。论文自身明确把 LLM4Decompile、Idioms、Nova、Ref-Decomp、D-Lift 等作为相关/对比研究：SK²Decompile 的区别是把结构恢复与标识符命名分离，并分别使用编译/语义奖励。

在本仓库已有语料的角色关系上，SK²Decompile 属于 TRANSLATOR：它直接把低层伪代码转换为高层源代码，输出不是可复用的编译器规则，也不是 pass 选择器。它可作为跨 ISA/低级代码翻译研究的 baseline 或语义恢复模块；与 RISC-V 后端优化、LLVM pass 生成和传统跨 ISA 二进制翻译工作存在接口关系，但当前 PDF 不支持把它们的具体实验结果合并或宣称互补已验证。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLM 驱动的二进制反编译与低级伪代码恢复 |
| 核心问题 | 同时恢复功能结构和有意义标识符导致正确性/可读性冲突 |
| 输入 | x86 Linux 二进制经 IDA Pro 生成的伪代码 |
| 输出 | 结构化匿名 IR，再输出带语义标识符的 C 源代码 |
| 核心方法 | Structure Recovery + Identifier Naming 两阶段；各自 SFT 后 RL |
| 使用的模型 | LLM4Decompile-6.7B 初始化；GPT-5-mini 作 Judge；qwen-embedding-0.6B 作语义奖励 |
| 使用的编译器工具 | GCC、Clang、Psyche-C；IDA Pro 生成伪代码；vLLM/veRL/LLaMA-Factory |
| 是否使用强化学习 | 是，GRPO；结构奖励和命名奖励不同 |
| 是否使用形式化验证 | 否；编译检查和单元测试不等于形式化证明 |
| 数据集规模 | 训练约 500 万样本；约 2B 伪代码、1.5B IR、1.5B 源代码 token |
| 主要指标 | re-executability、R2I、GPT-Judge |
| 最重要实验结果 | HumanEval/MBPP 平均 re-executability 69.00%/59.63%；相对 GPT-5-mini 分别高 21.6/26.3 个百分点 |
| 核心创新 | 去标识符源代码 IR、骨架到皮肤分解、阶段专用 RL 奖励 |
| 主要局限 | 主要是 x86；执行测试有限；无跨 ISA 和形式化等价实验 |
| 与 RISC-V 研究的相关性 | 中：可借鉴结构 IR/反馈闭环，但论文未支持 RISC-V，需重做 ABI/RVV 设计 |
| 最适合作为 | 低级代码翻译 baseline、反编译工具模块、跨 ISA 语义恢复参考 |

> 这篇论文最值得学习的是把“可验证的程序骨架”和“可读的标识符皮肤”分开优化；最主要的局限是其证据集中在 x86 C 反编译和有限测试上。如果用于后续研究，合理方式是把它作为结构恢复/命名基线并补充多 ISA 与更强等价性验证，而不是简单把输入架构改成 RISC-V 就宣称完成跨 ISA 翻译。
