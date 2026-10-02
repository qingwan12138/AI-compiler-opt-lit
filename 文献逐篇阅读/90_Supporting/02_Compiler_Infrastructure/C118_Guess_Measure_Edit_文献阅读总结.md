# Guess, Measure & Edit 文献阅读总结

论文题目：**Guess, Measure & Edit: Using Lowering to Lift Tensor Code**

作者：José Wesley de Souza Magalhães、Jackson Woodruff、Jordi Armengol-Estapé、Alexander Brauckmann、Luc Jaulmes、Elizabeth Polgreen、Michael F. P. O’Boyle

发表时间：2025 年（论文正文标注 PACT 2025；University of Edinburgh Research Explorer 记录的正式出版日期为 2025-12-16）

发表平台：PACT 2025, pp.216–228

论文链接或编号：DOI 10.1109/PACT65351.2025.00029
元数据核验来源：[IEEE DOI](https://doi.org/10.1109/PACT65351.2025.00029)；[PACT 官方日程](https://pact2025.github.io/program/)；[Edinburgh 作者机构页](https://www.research.ed.ac.uk/en/publications/guess-measure-amp-edit-using-lowering-to-lift-tensor-code/)
代码/数据/工件：作者公开实现 KONRUL：[JWesleySM/konrul](https://github.com/JWesleySM/konrul)；模型卡：[KONRUL Guesser](https://huggingface.co/jwesleysm/konrul-guesser)。

关键词：代码提升（lifting）、语言模型、程序相似度、引导式编辑、einsum、LLVM IR、张量程序、异构硬件

> 本文档依据论文正文 PDF（13 页）整理。论文事实、阅读后的分析和后续建议分开书写；论文没有明确说明的内容不作推断。

---

## 1. 研究背景

本文研究的是面向异构并行硬件的程序提升（program lifting）。近年来，GPU、专用 AI 加速器等硬件不断增加，程序员通常需要使用平台专用 API 或领域特定语言（Domain-Specific Language，DSL）才能获得编译器针对目标硬件生成的高性能代码。张量代数和 Einstein summation（einsum，使用索引表示张量收缩）是许多机器学习工作负载以及张量 DSL 的共同表达形式。

问题在于，已有 C/C++ 代码要迁移到这些 DSL，往往需要人工重写。论文引言指出，程序合成方法通常具有较好的可验证性，但底向上枚举的搜索空间随程序规模和张量维度快速膨胀；语言模型可以给出初始翻译，却容易产生语义错误或幻觉，而且新 DSL 的训练数据很少。论文希望结合两者：让语言模型负责提供可扩展的初始猜测，让编译器 lowering（将高层表示降到低层表示）提供可测量的反馈，再修复错误猜测。

## 2. 论文要解决的问题

### 2.1 从遗留 C 张量代码提升到高层张量 DSL

给定 C 语言中的张量计算程序 `P`，目标是找到等价的 einsum 表达式 `E`。论文正文将有效解定义为：对所有输入张量 `x`，`P(x) = E(x)`。在实现中，einsum 表达式还可以导出到支持该表示的工具，例如 PyTorch 的 einsum 模式。

### 2.2 在保证正确性的同时扩展到高维张量

现有底向上枚举方法在张量维度和程序规模增加时难以扩展；单独依赖语言模型又无法保证语义正确。本文研究如何利用已有编译器把原程序和候选程序降到共同的 LLVM IR，并用低层程序相似度引导高层候选编辑。

### 2.3 一句话概括

> 本文主要研究：如何用语言模型产生高层 einsum 初始猜测，再通过 LLVM lowering、程序相似度、I/O 测试和有界模型检查，自动把 C 张量程序提升为可迁移到 CPU/GPU 的正确张量表达式。

## 3. 核心方法概述

论文提出 Guess, Measure & Edit 方法，并实现 KONRUL。KONRUL 先让一个训练好的 encoder-decoder Transformer 猜测 einsum 表达式；随后将 C 原程序和猜测都降到 LLVM `-Oz` IR；再分别计算变量、索引和算术操作相似度；最后根据不匹配的相似度选择参数化编辑规则，迭代修复猜测。候选接近原程序后，系统用自动生成的 I/O 样例进行观察等价检查，并对通过测试的候选使用 CBMC 做有界模型检查。

```text
C 张量程序 P
        ↓ 预处理、抽取计算 kernel
254M 参数 Transformer 生成 einsum 初始猜测 Ê
        ↓ einsum 编译器生成 C，再与 P 一起降到 LLVM -Oz IR
计算变量 / 索引 / 算术操作三类相似度
        ↓
按不匹配指标选择 einsum 编辑规则，循环搜索候选
        ↓
自动生成 I/O 输入输出样例，检查观察等价
        ↓
CBMC 对通过测试的候选进行有界模型检查
        ↓
导出 PyTorch einsum，执行 CPU/GPU 性能评估
```

与传统枚举合成相比，候选不是完全从空白开始枚举，而是从神经模型的猜测出发；与单独使用语言模型相比，候选会经过编译器反馈和正确性筛选。论文的关键洞见是：虽然直接在低层语言和高层 DSL 之间做提升很难，但如果两者能被 lowering 到共同表示，则低层相似度可以作为高层编辑的指导信号。

## 4. 实验框架与训练流程

### 4.1 合成训练数据

论文支持的 einsum 语法由张量赋值、索引表达式、加减乘除、取负、常量和张量等构成。作者对该语法进行自底向上枚举，生成 einsum 表达式，再通过 einsum 编译器生成等价 C 代码，得到约 800,000 对 `<C 程序, einsum 表达式>`。由于自动生成的 C 代码包含包装代码且命名模式不接近人工遗留代码，作者删除包装代码、按词典序把标识符规范化为单字符，并把常量和循环边界替换为 `CONS`、`DIM` 等符号。

### 4.2 Transformer 训练

800,000 对数据中划出 5,000 对验证集和 5,000 对测试集。作者声明评估用的 81 个 benchmark 程序不在训练数据中。模型是 254M 参数的 encoder-decoder Transformer，包含 6 个 encoder 层、6 个 decoder 层、16 个 attention heads，embedding 和 context size 均为 1024；使用 BPE 分词和 Adam 优化器。论文将这一阶段称为训练，但没有使用 SFT、PPO、GRPO 或其他强化学习流程。

### 4.3 推理与候选搜索

测试时，对未见过的 C 程序执行相同预处理，用 beam size 为 5 的 beam search 生成初始 einsum 猜测。之后，KONRUL 反复执行：lower 原程序和候选、计算相似度、根据指标选编辑规则、重新 lowering。候选达到相似度阈值后才进入 I/O 检查；候选集合中未立即成功的高分候选在搜索结束时再次检查。

### 4.4 测试和有界验证

系统根据原始 C kernel 自动生成输入输出样例，检查候选 einsum 的观察等价。随后把原始 C 降到 MLIR，并使用 JAX 把 einsum 降到 MLIR，生成包含两份输入和断言的 C 文件，用 CBMC 检查输出不相等的断言是否可被违反。由于等价性问题不可判定，论文固定输入矩阵尺寸；浮点语义验证超时时，少数情况下改用实数表示进行验证。因此这里是有限边界下的模型检查，不是对任意输入的无界形式化证明。

## 5. 奖励函数、损失函数或关键公式

本文没有强化学习奖励函数，也没有报告 RL 训练。关键目标是让候选的低层相似度接近原程序，并通过测试和验证。

### 5.1 总体相似度

论文使用三个分数：

```text
score = Simvars × Simindex × Simops
```

当三个分数都为 1 时，候选的变量、索引和算术操作结构与目标 IR 匹配，系统跳过编辑，进入检查阶段。分数高于阈值但未达到 1 的候选会被保存为备选。

### 5.2 变量相似度

```text
Simvars = max(1 - |Var(IR_P) - Var(IR_Ê)| / Var(IR_P), 0)
```

`Var` 统计非循环归纳变量、非循环边界的相关变量。变量名会规范化，以便比较变量数量而不是原始命名。分数反映候选是否具有接近的变量规模；论文指出，lowering 可能引入辅助变量，所以语义等价程序的变量数仍可能不同。

### 5.3 索引和算术操作相似度

索引相似度把矩阵访问转换为索引元组序列，再使用 Levenshtein 距离：

```text
Simindex = 1 - D(Indices(IR_Ê), Indices(IR_P)) / |Indices(IR_P)|
Simops   = 1 - D(Ops(IR_Ê), Ops(IR_P)) / |Ops(IR_P)|
```

索引相似度由 Polly 的多面体分析提取矩阵索引；算术操作相似度抽取最内层循环中的操作及其操作数顺序。编辑顺序是先修复变量规模，再修复索引，最后修复算术操作，因为作者认为算术操作在 einsum 与 LLVM IR 之间的表示差距最大。

### 5.4 正确性目标

论文的语义目标是 `∀x. P(x) = E(x)`。实际验证分两层：自动 I/O 样例用于快速筛选，CBMC 在固定边界内检查等价断言。样例通过并不等于无界语义等价；论文通过 CBMC 给出更强但仍受边界限制的证据。

## 6. 实验设置

### 6.1 数据集来源

评估集共 81 个 benchmark，来自以往张量代码提升工作，覆盖图像处理、数字信号处理、数学函数、数组编程、深度学习等类别，也包含循环展开和后增量指针寻址等程序风格，以及高维张量收缩程序。训练数据是作者按 einsum grammar 自动生成的约 800,000 对程序；验证集和测试集各 5,000 对。作者说明评估 benchmark 与训练数据不重合，但合成训练分布与真实手写遗留代码之间仍可能存在分布差异。

### 6.2 模型与工具

| 类别 | 论文明确给出的设置 |
| --- | --- |
| 模型 | 254M 参数 encoder-decoder Transformer；6 encoder 层、6 decoder 层、16 heads、1024 embedding/context |
| 推理 | BPE；beam size 5 |
| 编译器/IR | TACO 0.1；LLVM 14.0；LLVM `-Oz` IR；Polly；MLIR |
| 深度学习/导出 | PyTorch 2.3.0；JAX 用于将 einsum 降到 MLIR |
| 训练/分词框架 | Fairseq 0-12.2；Google SentencePiece |
| C 编译器 | gcc 9.4 |
| 验证器 | CBMC；浮点或内部用有理数表示的实数 |
| CPU | 36-core Intel Xeon W-2285，3.00 GHz，125 GB DDR4-2666 |
| GPU | NVIDIA RTX A6000；driver 510.47.03；CUDA 11.6 |

### 6.3 对比方法

主要对比包括 TF-Coder、C2TACO、Tenspiler 和 GPT-4。另有两个 KONRUL 变体：`LM` 只使用 Transformer 初始猜测，`Greedy` 每轮随机编辑并选择相似度最近的候选，用于隔离 measure/edit 的作用。替代方法包含随机成分时，作者取 10 次运行中的最佳性能作为竞争性 baseline。所有方法单个程序的时间预算为 2 分钟；Tenspiler 使用其可复现性 artifact 中的手写 grammar。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
| --- | --- | --- |
| Lifting coverage | 81 个程序中成功得到候选的比例 | 越高越好 |
| Candidate explored | 搜索过程中探索的候选数 | 通常越少越好，但需结合 coverage |
| I/O tested | 实际执行 I/O 样例的候选数 | 越少表示筛选更有效 |
| Lifting time | 搜索、测试和验证耗时 | 越低越好 |
| CBMC verification | 在固定边界下证明等价的比例 | 越高越好 |
| Geomean speedup | 论文定义为 lifted einsum 程序相对原始 gcc `-O3` 实现的运行时间比；未提升程序按 1 计 | 按论文报告值比较，越高表示收益更大 |

## 7. 实验结果与结论

### 7.1 主要结果：覆盖率

在 81 个 benchmark 上，KONRUL 成功提升 98% 的程序，并在 9 个类别上达到 100% 覆盖；失败的 2 个程序与 LLVM common sub-expression elimination 在 lowering 时消除引用有关。对比方法的总覆盖率为：C2TACO 72%、Tenspiler 70%、TF-Coder 62%、GPT-4 47%。GPT-4 的失败中 86% 是语义错误、14% 是语法无效；TF-Coder 的失败中 94% 是超时。

### 7.2 按张量维度比较

最高维度为 1 时，KONRUL 覆盖率为 98%；维度为 2 时，KONRUL 仍为 96%，而 TF-Coder 仅 24%、C2TACO 52%。对于维度为 3 和 4 的程序，KONRUL 成功提升全部相应程序；论文报告 GPT-4 在这两类分别为 80% 和 50%，而枚举方法无法处理这些高维程序。

### 7.3 搜索效率

KONRUL 平均探索 11.2 个候选，平均实际用 I/O 测试 2.4 个候选；70% 的案例只测试 1 个候选。当三个相似度都为 1 时，论文报告候选在所有案例中都被验证为正确。KONRUL 在 2 分钟预算内提升 79 个程序，Tenspiler 提升 57 个；延长到 24 小时后，Tenspiler 没有增加成功数，C2TACO 只增加 5 个且没有提升高维张量收缩程序。

### 7.4 时间与验证

KONRUL 平均每个程序耗时 23 秒，其中约 10 秒用于搜索、约 13 秒用于测试。CBMC 验证平均耗时 0.27 秒，最长类别平均约 0.83 秒；高维张量收缩个别验证最多可达 5.2 秒。浮点表示下，79 个已提升程序中 69 个（87%）完成证明，10 个超时；改用实数表示后，论文报告 100% 完成验证。这里的“完成验证”仍受固定矩阵大小和有界模型检查限制。

### 7.5 CPU/GPU 性能

作者把 lifted einsum 导出到 PyTorch，并在指定 CPU、GPU 上执行。论文的整体几何平均结果是：KONRUL 在多核 CPU 上 4.07×、GPU 上 38.30×；GPT-4 为 2.31× 和 6.42×；C2TACO 为 1.80× 和 8.83×；Tenspiler 为 1.32× 和 5.78×；TF-Coder 为 1.29× 和 5.17×；LM 变体为 1.11× 和 1.53×；Greedy 变体为 1.26× 和 2.53×。这些结果受到不同方法可成功提升的程序集合影响，论文将未提升程序按 speedup 1 计入总体几何平均。

### 7.6 消融与失败分析

`LM` 仅使用初始模型，覆盖率和性能明显低于完整 KONRUL；`Greedy` 说明只用相似度做选择、但不按指标决定编辑方向，扩展性较差。KONRUL 的两个失败案例说明 LLVM 优化可能把区分候选所需的引用消除掉，表明当前相似度度量不是对所有 lowering 结果都稳健。论文没有报告独立的“去掉 CBMC”或“去掉 I/O 测试”的完整消融表。

## 8. 主要创新点

### 8.1 创新点一：用 lowering 后的低层相似度修复高层猜测

现有语言模型可以直接翻译但可能产生语义错误；传统枚举合成较准确但难以扩展。本文让原程序和候选高层程序共享 LLVM IR 表示，并利用变量、索引、算术操作三类低层差异决定高层编辑。这一设计的价值在于：候选错误时不必完全丢弃，而是根据结构反馈进行定向修复。覆盖率、候选数量和与 `LM`/`Greedy` 的比较支持这一机制有效，但不表示相似度对任意 DSL 都有效。

### 8.2 创新点二：KONRUL 的神经猜测、编译器度量和验证组合

KONRUL 把 Transformer 的可扩展初始猜测、LLVM/Polly 分析、参数化编辑、I/O 检查和 CBMC 有界验证串成一个完整 lifter。论文的贡献不是“使用 LLVM”或“使用语言模型”本身，而是把 compiler lowering 作为反馈通道来指导语义可能不保持的编辑搜索。

### 8.3 创新点三：针对高维张量的可扩展 C 到 einsum 提升

在论文 81 个 benchmark 上，KONRUL 能处理高维张量收缩，而底向上枚举方法在高维程序上严重退化。论文以 98% 总覆盖率和 3/4 维程序的成功率证明了可扩展性；这是一项针对张量 DSL 场景的实验证据，不能泛化为所有源语言翻译任务。

## 9. 局限性

### 9.1 论文明确承认的局限

论文结论明确指出，当前实现的相似度指标和编辑空间专门面向张量领域，未来希望推广到更通用的相似度和编辑空间。因此它目前不是通用的 C 到任意 DSL lifter。

论文还指出两个失败案例与 LLVM common sub-expression elimination 有关，说明 lowering 可能移除用于度量的结构信息。

### 9.2 阅读后发现的潜在局限

1. 训练数据来自 grammar 枚举和编译器生成的 C 代码，尽管做了规范化，但与真实人工遗留代码仍可能存在分布差异。
2. 正确性先依赖自动生成的 I/O 样例；CBMC 只在固定矩阵尺寸和有界条件下验证，浮点证明还有 10 个超时，不能等同于无界形式化等价证明。
3. 三种相似度主要反映变量、索引和算术操作的语法/结构接近程度，语法相似不必然意味着语义等价；论文也需要用 I/O 和 CBMC 作为最终过滤器。
4. 性能评估依赖 PyTorch、gcc、指定 CUDA/硬件和 einsum 后端，几何平均 speedup 受“不同方法成功提升的程序集合”影响；因此不能直接解释为每个程序都获得相同加速。
5. 训练模型和搜索规则对目标 einsum grammar、TACO 风格和张量领域强绑定；迁移到 MLIR 其他 dialect、RISC-V 特定 intrinsic 或非张量控制流需要重新设计相似度和编辑空间。

## 10. 阅读后的研究方向反思

### 10.1 值得借鉴的思想

最值得借鉴的是“高层生成 + 编译器反馈 + 正确性过滤”的闭环。对于 LLVM IR 或 MLIR，低层 canonical form、数据流结构和 verifier 结果可以作为模型候选修复信号。这样可以把语言模型的作用限定为提出候选或编辑建议，而不是把其一次性输出直接视为正确翻译。

### 10.2 不能直接照搬的部分

不能只把 einsum 换成 RISC-V 汇编或 RVV intrinsic 就声称形成新工作。本文的核心贡献依赖张量 grammar、LLVM `-Oz` 的规范化效果、Polly 的索引分析以及 CBMC 对小尺寸数组的检查。换目标语言后，变量和索引相似度的语义、lowering 归一化程度、浮点/向量等价性都必须重新证明。

### 10.3 与 RISC-V/LLVM 研究的关系

相关性为中高：论文直接使用 LLVM IR、LLVM 14 和编译器 lowering，并且最终代码面向 CPU/GPU 异构执行；但论文没有 RISC-V 硬件实验，也没有 RVV 指令级代码生成。它更适合作为“编译器反馈引导的翻译/提升”方法参考，不能直接作为 RISC-V 后端优化结果。

### 10.4 最适合扮演的研究角色

它适合作为 TRANSLATOR 类方法参考、程序提升 baseline，以及“低层 IR 相似度反馈”工具模块。若研究主线是 RISC-V，较合理的组合是把它的反馈闭环用于 C/DSL 到 RVV-friendly IR 的翻译验证，再加入真实 RVV 后端性能和跨硬件泛化实验；这属于新的研究问题，不是本文已实现的内容。

## 11. 可进一步尝试的研究方向

### 11.1 面向 MLIR 多层 dialect 的跨层提升

#### 研究问题

能否把源级张量程序提升到多个候选 MLIR dialect，并用跨层 lowering 的结构反馈修复错误 dialect 程序？

#### 与原论文的区别

原论文目标是 C 到 einsum；该方向研究多层 IR、dialect 合法性和可重定向 lowering，而不是简单替换目标字符串。

#### 可能的创新点

设计跨 dialect 的相似度表示，结合 verifier 诊断和 lowering 失败原因，形成结构化编辑策略。

#### 实验框架

```text
C/张量程序 → Transformer 生成 MLIR dialect 候选
          → 多层 lowering / verifier
          → IR 结构差异与诊断反馈
          → 编辑候选 → 语义测试与有界验证
```

#### 可行性

需要 MLIR、若干稳定 dialect、测试生成器和至少一种真实 CPU/GPU 后端；不一定需要重新训练大模型。

#### 主要风险

不同 dialect 的 lowering 可能产生不稳定或信息丢失的 IR；通用相似度可能无法可靠预测高层语义。

### 11.2 面向 RISC-V RVV 的等价提升与后端反馈

#### 研究问题

能否把 C 张量/循环代码提升为 RVV-friendly 表达式，并用 LLVM-RISCV/RVV lowering 和真实硬件性能共同指导候选选择？

#### 与原论文的区别

不是把 PyTorch GPU 后端替换成 RISC-V，而是增加向量长度无关代码、尾部处理、LMUL/寄存器压力和真实 RVV 运行反馈。

#### 可能的创新点

把 LLVM-RISCV 汇编特征、向量化合法性和硬件测量纳入候选评价，同时保持 CBMC/差分测试的正确性门槛。

#### 实验框架

```text
C/DSL kernel → 初始 RVV-friendly 候选
             → LLVM RISC-V/RVV lowering
             → IR/汇编结构反馈 + RVV 实机性能
             → 编辑与候选选择 → 差分测试/CBMC
```

#### 可行性

需要 LLVM RISC-V 后端、RVV 模拟器或真实开发板、性能计数器和受控 benchmark。

#### 主要风险

模拟器与实机性能可能不一致；浮点向量语义、向量长度变化和编译器版本会使验证与复现更困难。

### 11.3 将相似度反馈与编译器诊断结合

#### 研究问题

相较于只使用 lowered IR 的结构相似度，加入编译器 verifier、优化 remark、未向量化原因等诊断，是否能减少无效编辑？

#### 与原论文的区别

原论文的反馈主要是变量、索引、算术操作序列；新方向使用语义/优化诊断作为额外反馈，不局限于文本结构距离。

#### 可能的创新点

建立诊断到编辑规则的映射，并比较“仅相似度”“仅诊断”“联合反馈”三种搜索策略。

#### 实验框架

```text
候选程序 → 编译/lowering/verifier → IR 相似度 + 诊断消息
        → 结构化编辑策略 → 正确性检查 → 性能测量
```

#### 可行性

可从 LLVM remarks、MLIR diagnostics、Alive2/CBMC 或差分测试日志开始，逐步增加硬件反馈。

#### 主要风险

诊断消息可能不稳定、冗余或与最终性能不一致；需要严格定义不同编译器版本下的可比性。

## 12. 与其他已读文献的关系

本 slot-4 本轮只完成这一篇 PACT 2025 论文，因此没有另一篇“当前批次已读论文”可进行事实级横向比较，也不虚构其他文献的实验结果。

从论文正文给出的 related work 可确认：Tenspiler、C2TACO、TF-Coder 和 GPT-4 是本文实验中的直接对比对象；本文与它们的关系是“以语言模型提供初始猜测，再用 lowering 相似度和验证进行修复”，而不是完全依赖底向上枚举或一次性神经生成。论文还提到 MLIR、LLVM、CBMC、Polly 等可作为基础设施或验证/分析模块，但本笔记不把这些被引用工作当作本批次已通读论文。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | 将 C 张量程序提升到高层 einsum 表达式 |
| 核心问题 | 语言模型易错、枚举合成难扩展，如何兼顾可扩展性和正确性 |
| 输入 | C 张量计算 kernel |
| 输出 | 等价 einsum 程序，可导出到 PyTorch |
| 核心方法 | Transformer Guess + LLVM lowering/Measure + guided Edit + I/O/CBMC Verify |
| 使用的模型 | 254M 参数 encoder-decoder Transformer；beam size 5 |
| 使用的编译器工具 | TACO、LLVM 14 `-Oz`、Polly、MLIR、PyTorch、JAX、gcc |
| 是否使用强化学习 | 否；没有 RL 奖励函数 |
| 是否使用形式化验证 | 是，但为固定边界下的 CBMC 有界模型检查 |
| 数据集规模 | 合成约 800k 训练对；5k 验证、5k 测试；81 个评估 benchmark |
| 主要指标 | lifting coverage、候选数、I/O 测试数、CBMC 验证、CPU/GPU 几何平均 speedup |
| 最重要实验结果 | KONRUL 覆盖率 98%；平均探索 11.2 个候选；平均耗时 23 秒；CPU 4.07×、GPU 38.30×（按论文定义的几何平均报告） |
| 核心创新 | 用 lowered LLVM IR 的结构相似度引导高层 einsum 编辑，而不是丢弃错误猜测 |
| 主要局限 | 相似度和编辑空间专门面向张量；验证受边界限制；存在 lowering 信息消失失败案例 |
| 与 RISC-V 研究的相关性 | 中高：LLVM/异构优化方法相关，但无 RISC-V/RVV 实验 |
| 最适合作为 | TRANSLATOR 方法参考、程序提升 baseline、编译器反馈工具模块 |

> 这篇论文最值得学习的是把语言模型限制在“生成初始候选”，再用 compiler lowering 提供可计算反馈并用测试/有界验证兜底；最主要的局限是方法深度绑定张量 einsum 和固定边界验证。如果用于后续研究，最合理的使用方式是把它作为 LLVM/MLIR 或 RVV 提升的反馈闭环 baseline，并重新设计目标 IR 的相似度、验证和真实硬件评测，而不是简单替换目标平台。
