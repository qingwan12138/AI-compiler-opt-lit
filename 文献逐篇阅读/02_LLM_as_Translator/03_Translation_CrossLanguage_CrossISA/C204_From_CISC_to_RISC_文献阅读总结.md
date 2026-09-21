# From CISC to RISC 文献阅读总结

论文题目：**From CISC to RISC: Language-Model Guided Assembly Transpilation**

作者：Ahmed Heakl；Chaimaa Abi；Rania Hossam；Abdulrahman Mahmoud

发表时间：2024（arXiv v1，2024-11-25）

发表平台：arXiv，cs.PL / cs.AR

论文链接或编号：[arXiv:2411.16341](https://arxiv.org/abs/2411.16341)；[官方 PDF](https://arxiv.org/pdf/2411.16341)；arXiv DOI：`10.48550/arXiv.2411.16341`

关键词：CISC、RISC、x86、ARM、RISC-V、汇编转译、语言模型、二进制翻译

> 本文档依据本轮下载的 arXiv v1 PDF（15 页）撰写。论文事实与阅读后的分析分开记录。本文最终 LM 输出是目标 ISA 汇编代码，因此按角色判定为 `TRANSLATOR / T3_Translation_CrossLanguage_CrossISA`，不是 Selector。

---

## 1. 研究背景

本文研究 CISC（Complex Instruction Set Computer，复杂指令集计算机）到 RISC（Reduced Instruction Set Computer，精简指令集计算机）的汇编级跨 ISA 转译。论文以 x86 到 ARM 为主，并扩展到 x86 到 RISC-V64。

传统解决方案包括重新编译源代码、虚拟化和动态二进制翻译。重新编译需要源代码和完整软件构建链；但现实中可能只拥有可执行文件。QEMU 具有通用性但引入运行时开销，Rosetta 2 在 Apple 平台上效率较高却是封闭且平台专用的翻译层。论文因此尝试让语言模型直接学习 x86 汇编与目标 RISC 汇编之间的映射。

论文强调，CISC 与 RISC 在指令复杂度、寄存器使用、寻址模式、比较与跳转语义以及指令数量方面存在差异。汇编转译不能只做字符串替换，还要保持寄存器依赖、内存访问和控制流语义。

## 2. 论文要解决的问题

### 2.1 无源代码条件下的跨 ISA 转译

论文希望在不依赖原始源代码的情况下，把 x86 汇编转换为可执行的 ARM 汇编，并尽量避免完整虚拟化层的开销。

### 2.2 语义保持而非单纯文本相似

目标汇编可能采用不同的寄存器分配、指令顺序、常量加载方式或等价指令序列。因此论文同时考察编辑距离、精确匹配和运行测试正确性，避免把文本相似误当作语义等价。

### 2.3 面向 RISC-V 的可迁移性

论文进一步检验同一类方法是否能用于 x86 到 RISC-V64 的转译，并将结果与 x86 到 ARM 的难度进行比较。

> 本文主要研究：如何使用经过汇编数据微调的语言模型，在没有源代码的条件下，将 x86 汇编转译为功能上可执行的 ARM 或 RISC-V 汇编。

## 3. 核心方法概述

论文提出 CRT（CISC-to-RISC Transpiler），使用自回归语言模型直接生成目标 ISA 汇编。其核心不是选择已有编译器 pass、schedule 或配置，而是输出新的目标汇编文本。

```text
AnghaBench 中的 C 程序
        ↓
分别用 x86 与 ARM 工具链编译，形成配对汇编
        ↓
扩展汇编 opcode / register tokenizer
        ↓
微调 DeepSeek-Coder、Yi-Coder 或 BART
        ↓
输入 x86 汇编，模型自回归生成 ARM/RISC-V 汇编
        ↓
用 QEMU 执行测试并比较功能正确性
        ↓
在 Apple M2 上与原生 ARM、Rosetta 2 比较时间、能耗和内存
```

输入是 x86 汇编，输出是 ARM 或 RISC-V 汇编。语言模型承担跨 ISA 代码生成/翻译；GCC、ARM GNU 工具链、QEMU、Clang 和 Apple `powermetrics` 负责数据生成、编译、执行或测量。论文没有把 LM 限定为“选择一个配置后交给现有编译器”，所以最终角色是 Translator。

## 4. 实验框架与训练流程

本文不涉及强化学习，主要采用监督微调、模型选择和推理阶段评测。

### 4.1 数据准备阶段

作者从 AnghaBench 的约 1,000,000 个可编译 C 程序中随机抽取 500,000 个程序。每个程序分别使用 GCC 编译为 x86 汇编，并使用 `ARM-gnueabi-gcc` 交叉编译为 ARMv5 汇编，形成配对训练样本。论文将该训练集描述为约 8 billion tokens。

### 4.2 模型与超参数实验阶段

作者在 AnghaBench 的 100,000 样本子集上比较模型和超参数，涉及 batch size、gradient accumulation、warmup、优化器、学习率和 epoch 等设置。比较的模型包括 DeepSeek-Coder 1.3B、Yi-Coder 2B 和 BART-Large 300M；此外还报告了 GPT-4o、DeepSeekCoder2-16B、Yi-Coder-9B 等直接比较结果。

### 4.3 全量训练与推理阶段

作者选出配置后在 500,000 样本上训练最佳模型。训练使用 4 张 A100 40 GB GPU、bfloat16、有效 batch size 4、2 个 epoch、16k context window 和 paged AdamW。推理时使用缓存并关闭 sampling，以得到确定性输出；另用 llama.cpp 探索 bfloat16、int8 和 int4 量化。

### 4.4 验证与部署阶段

目标汇编通过 QEMU 执行测试。主实验使用 HumanEval 的 C 转换版本；Apple M2 case study 则比较原生 ARM64、Rosetta 2 执行 x86 二进制和 CRT 直接生成的 ARM64 汇编。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习，因此不存在强化学习奖励函数。

论文将转译建模为自回归条件生成：

```text
P(Y | X; θ) = ∏t P(yt | y<t, X; θ)
```

其中 `X` 是输入 x86 汇编，`Y` 是目标 ARM/RISC-V 汇编，`θ` 是模型参数，`y<t` 是已经生成的目标 token。该表达式说明模型按序生成目标汇编 token，而不是从固定的编译动作集合中选择 action。

论文没有明确给出单独的损失函数公式或训练损失分解。训练本质上是配对汇编上的监督式语言建模微调。

## 6. 实验设置

### 6.1 数据集来源

- AnghaBench：约 1,000,000 个从公开 GitHub C 项目挖掘的可编译程序；训练随机抽取 500,000 个。
- 训练对：C 程序分别编译为 x86 与 ARMv5 汇编。
- 超参数探索：使用 100,000 样本子集。
- 评测：HumanEval 的 C 转换版本，包含 164 个编程问题；论文报告平均代码行覆盖率为 98.81%。
- RISC-V 扩展：对 x86 到 RISC-V64 重新训练/评测，论文未在正文中给出与 ARM 训练集完全对应的独立样本数。

### 6.2 模型与工具

- 模型：DeepSeek-Coder 1.3B、Yi-Coder 2B、BART-Large 300M；比较 GPT-4o、DeepSeekCoder2-16B、Yi-Coder-9B。
- 训练硬件：4 × NVIDIA A100 40 GB；AMD Ryzen 7 用于数据编译。
- 编译/执行：GCC、`ARM-gnueabi-gcc`、QEMU、Apple Clang 14.0.3、llama.cpp、gcov。
- 部署硬件：Apple M2 Pro、16 GB RAM、macOS 13.7。
- Apple case study 编译目标：`arm64-apple-darwin22.6.0`，`-O0`。
- 代码、模型、数据和 benchmark：论文与 arXiv 摘要页指向 [作者项目页](https://ahmedheakl.github.io/asm2asm/)。本轮未确认独立 GitHub 仓库。

### 6.3 对比方法

- GPT-4o、DeepSeekCoder2-16B、Yi-Coder-9B：未按本文任务微调的较大模型。
- Yi-Coder-1.5B、DeepSeek-Coder 1.3B：作者训练的模型配置。
- int8/int4 量化版本：用于衡量低资源部署影响。
- Apple case study：原生 ARM64、Rosetta 2 动态二进制翻译、CRT 生成的 ARM64 汇编。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
| --- | --- | --- |
| Average Edit Distance | 生成汇编与 ground truth 的 Levenshtein 编辑距离 | 越小越好 |
| Exact Match | 生成汇编与参考汇编完全相同的比例 | 越大越好 |
| Test Accuracy | 生成汇编通过全部测试用例的比例 | 越大越好 |
| Line Coverage | 测试执行到的源代码行比例 | 越大越好 |
| Execution Time | Apple M2 上的执行时间 | 越小越好 |
| Energy Consumption | `powermetrics` 测得的 CPU 能耗 | 越小越好 |
| Memory Usage | 执行期间的内存占用 | 越小越好 |

## 7. 实验结果与结论

### 7.1 x86 到 ARM

表 2 中，DeepSeek-Coder 1.3B + 扩展 tokenizer 的 ARM 结果为平均编辑距离 165、精确匹配 50.32%、测试正确率 79.25%。其测试正确率高于 GPT-4o 的 8.18%、DeepSeekCoder2-16B 的 7.36% 和 Yi-Coder-9B 的 6.33%。这些数字是 HumanEval C 转换评测上的 test accuracy，不是形式化语义等价率。

### 7.2 x86 到 RISC-V64

表 3 中，DeepSeek-Coder 1.3B + 扩展 tokenizer 的 x86 到 RISC-V64 结果为平均编辑距离 27、精确匹配 69.81%、测试正确率 88.68%。GPT-4o 和 DeepSeekCoder2-16B 的测试正确率分别为 7.55% 和 6.29%。作者将 RISC-V 结果更高的现象归因于 RISC-V 指令集相对更简单、一致性更强，但这是论文中的解释，不是独立因果验证。

### 7.3 tokenizer 与量化

扩展 opcode/register tokenizer 后，ARM 测试正确率相对未扩展的 DeepSeek-Coder 1.3B 提升约 2%，token 数量减少 7.27%。ARM 上 int8 与 float32 相比下降约 3.8%，int4 下降接近 2.5%；RISC-V 上 int8 仅下降约 0.63%，而 int4 出现约 19.5% 的明显下降。

### 7.4 推理与编译器比较

beam size 从 1 增加到 8 时，论文观察到正确率提高，但推理成本增加。在 A100 上，16k context 的样本平均生成时间为 18.3 秒，约 437.2 tokens/s；Ryzen 7 上，int8 和 int4 的 Ollama 推理速度分别为 18.51 和 87.23 tokens/s。GCC 与 Clang 训练实验的编辑距离和精确匹配结果相近，作者因此报告 GCC 结果。

### 7.5 Apple M2 case study

在 Apple M2 Pro 上，每个程序执行 100 次并报告几何平均。CRT 相对 Rosetta 2 达到 1.73× speedup、1.47× energy efficiency 和 2.41× memory efficiency。论文给出的内存示例为原生执行 1.03 MB、CRT 1.034 MB、Rosetta 2 2.49 MB。ARMv8 的测试正确率为 75.0%，低于 ARMv5 的 79.25%。

### 7.6 失败模式

论文将错误归为 register allocation errors（14.11%）、addressing errors（62.03%）和 other errors（23.86%）。常见问题包括错误的立即数、寄存器过早覆盖、内存偏移错误、非法跳转地址和浮点异常。小编辑距离不代表小语义风险；例如除以 4 的算术右移被错误生成为除以 2，可能导致循环提前终止。

## 8. 主要创新点

### 8.1 创新点一：直接的 x86 汇编到 ARM/RISC-V 汇编转译

与依赖源代码重编译或虚拟化的方式不同，CRT 直接以 x86 汇编为输入并生成目标 RISC 汇编。该设计使其能处理没有源代码的场景，但同时把寄存器、寻址和控制流正确性全部交给生成模型与测试流程。

### 8.2 创新点二：面向汇编的扩展 tokenizer

作者把高频 opcode 和寄存器名称加入 tokenizer，使模型不必把诸如 `ldr`、`r1` 等汇编元素拆成过细的普通文本 token。论文实验显示该设计带来约 2% 的 ARM 测试正确率提升和 7.27% 的平均 token 数减少。

### 8.3 创新点三：小模型、数据和部署优化的组合

论文没有主要依赖扩大模型规模，而是结合 500k 配对样本、16k 上下文、扩展 tokenizer、确定性解码和量化，使 1.3B 模型在该任务上超过若干更大的通用模型。该结论只适用于论文的汇编转译设置，不能直接推广到所有跨 ISA 任务。

## 9. 局限性

### 9.1 论文明确或实验中承认的局限

- ARM 结果低于 RISC-V；ARMv8 又低于 ARMv5，说明 ISA 版本和寻址/寄存器复杂度会影响转译难度。
- 地址错误占论文统计错误的 62.03%，说明数值 token、内存地址和寻址模式仍是主要风险。
- int4 在 RISC-V 上出现明显正确率下降。
- 测试正确性来自有限测试用例，不是形式化语义等价证明。
- 论文主要以生成汇编与 ground truth 的比较为核心，未展示广泛的真实生产二进制分布或跨编译器泛化实验。

### 9.2 阅读后发现的潜在局限

- 训练数据由同一类 C 程序经过编译生成，可能使模型学习到特定编译器、优化级别和汇编格式，而不等价于覆盖任意真实 x86 二进制。
- RISC-V 结果使用 RISC-V64 扩展实验，但论文没有系统覆盖 RV32、RVV、压缩指令或自定义扩展，因此不能直接称为完整 RISC-V 后端迁移方案。
- QEMU 测试通过只能说明测试输入上的行为一致，不能排除未覆盖输入、异常路径或系统调用语义差异。
- Apple M2 case study 只比较有限的 ARM64 部署设置，不能将 1.73× speedup 解释为跨平台普遍加速。
- 论文没有把编译器/验证器作为不可绕过的语义门控；对于需要安全保证的二进制迁移，测试反馈仍弱于形式化验证或可信翻译验证。

## 10. 阅读后的研究方向反思

### 10.1 可借鉴之处

最值得借鉴的是将功能测试、编辑距离和真实硬件测量分开报告，并明确展示“文本不同但功能正确”和“编辑距离很小但语义错误”的案例。扩展 tokenizer、长上下文和量化也说明低层代码任务的表示设计可能比单纯增加模型参数更重要。

### 10.2 不应直接照搬之处

不能把 x86→RISC-V 汇编转译直接改写成 RISC-V 编译优化 Selector。本文的核心贡献是目标代码生成，最终 LM 角色属于 Translator；将其改为“LM 选择 RVV pass/config”需要重新定义动作空间、合法性约束和性能反馈。

### 10.3 对 RISC-V 研究的启发

本文可作为跨 ISA Translator baseline，特别适合研究 RV64GC、RVV 或自定义扩展上的表示、寄存器映射和验证问题。若仅把 ARM 换成 RISC-V，不足以形成新贡献；更有价值的方向是把 RISC-V ABI、扩展指令、向量长度、调用约定和后端性能纳入条件化转译与验证。

### 10.4 最合适的语料库定位

本文最适合作为 `TRANSLATOR / T3_Translation_CrossLanguage_CrossISA` 的跨 ISA 汇编转译 baseline 或数据/表示方法参考，不适合作为 Selector、schedule 搜索策略或 compiler pass 选择 baseline。

## 11. 可进一步尝试的研究方向

### 11.1 RVV 感知的跨 ISA 汇编转译

#### 研究问题

如何将 x86 SIMD/标量代码转译为具有正确向量长度和尾部处理的 RVV 汇编。

#### 与原论文的区别

不只把目标 ISA 替换为 RVV，而是显式建模 `vtype`、VL、掩码、尾部策略和 ABI 约束。

#### 可能的创新点

建立 RVV 指令/寄存器 tokenizer，加入可检查的向量语义约束，并比较不同 VLEN 的迁移稳定性。

#### 实验框架

```text
x86 SIMD 汇编
  ↓
向量语义抽取与 RVV 约束提示
  ↓
LM 生成 RVV 候选
  ↓
汇编器 + 仿真器/真实 RVV 硬件验证
  ↓
功能、向量长度迁移和性能评估
```

#### 可行性

需要 x86 SIMD/RVV 配对数据、LLVM 或 GCC、Spike/QEMU 及至少一个 RVV 实机或可重复模拟环境。

#### 主要风险

向量长度无关代码的语义覆盖、内存对齐和尾部处理可能使有限测试难以发现错误。

### 11.2 验证门控的跨 ISA 汇编转译

#### 研究问题

如何在模型输出进入执行评测前，利用 CFG、符号执行或等价检查过滤明显错误的寄存器和内存映射。

#### 与原论文的区别

原论文以测试正确性为主；该方向增加机器可检查的中间验证层，而不是只增加训练数据。

#### 可能的创新点

将错误定位反馈结构化为寄存器活跃性、地址范围、调用约定和控制流约束，并用于候选重排或修复。

#### 实验框架

```text
LM 生成目标汇编
  ↓
语法、CFG、寄存器和地址约束检查
  ↓
通过候选进入 QEMU/真实硬件测试
  ↓
失败诊断反馈给下一轮生成
```

#### 可行性

可复用 LLVM MC、Capstone、QEMU 和现有 C/RISC-V 测试集；不必先训练大模型。

#### 主要风险

跨 ISA 的内存模型、系统调用和异常语义难以完全由局部检查覆盖；验证器误拒绝也会降低可用候选率。

### 11.3 编译器配置 Selector 与汇编 Translator 的分层协作

#### 研究问题

能否让上层 LM 先选择目标 ISA、优化级别或后端配置，再由下层 Translator 生成汇编，并用统一的真实硬件反馈协同优化。

#### 与原论文的区别

把本文的直接汇编生成限制在受控候选空间内，新增一个明确的 Selector 层，而不是把所有决策都交给单个生成模型。

#### 可能的创新点

设计“配置选择—代码生成—验证—性能”跨层反馈，并研究配置错误与生成错误的责任分离。

#### 实验框架

```text
源程序/输入 ISA/硬件特征
  ↓
Selector 输出后端配置与约束
  ↓
Translator 输出目标汇编
  ↓
编译、验证、运行和性能测量
  ↓
联合反馈更新候选配置与生成提示
```

#### 可行性

可将本文的配对汇编数据与 LLVM/RISC-V 后端配置数据结合，先做小规模 RV64GC 任务集。

#### 主要风险

Selector 与 Translator 的收益可能相互混淆；需要严格区分配置选择收益、生成质量收益和后端本身收益。

## 12. 与其他已读文献的关系

本轮阶段2按“成功后停止队列”只完成本文，因此没有其他新候选的正文笔记可用于事实级横向比较。

从角色定位上，本文与本地已有的 RISC-V/跨 ISA Translator 类工作属于相邻方向，但本文的独特点是 x86 汇编直接转为 ARM/RISC-V 汇编，并以测试执行和 Apple M2 部署为主要验证方式。它不能与 Selector 类论文的 pass/config/schedule 结果直接比较，也不能把其 RISC-V accuracy 当作编译器优化或形式化验证指标。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | x86 汇编到 ARM/RISC-V 汇编的跨 ISA 转译 |
| 核心问题 | 无源代码条件下保持功能并降低虚拟化开销 |
| 输入 | x86 汇编 |
| 输出 | ARM 或 RISC-V 汇编 |
| 核心方法 | CRT；配对汇编监督微调 + 扩展 tokenizer + QEMU 测试 |
| 使用的模型 | 主要为 DeepSeek-Coder 1.3B；比较 Yi-Coder、BART、GPT-4o 等 |
| 使用的编译器工具 | GCC、ARM-gnueabi-gcc、QEMU、Clang、llama.cpp、gcov |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；主要是有限测试执行 |
| 数据集规模 | AnghaBench 约 100 万程序；训练抽取 50 万；HumanEval C 评测 164 题 |
| 主要指标 | 编辑距离、精确匹配、测试正确率、时间、能耗、内存 |
| 最重要实验结果 | ARMv5 test accuracy 79.25%；RISC-V64 88.68%；Apple M2 上相对 Rosetta 2 达到 1.73× speedup |
| 核心创新 | 直接汇编转译、汇编专用 tokenizer、小模型量化部署组合 |
| 主要局限 | 地址/寄存器错误、有限测试、无形式化等价证明、RISC-V 扩展覆盖有限 |
| 与 RISC-V 研究的相关性 | 高：直接评测 x86→RISC-V64，但尚未覆盖 RVV/自定义扩展 |
| 最适合作为 | TRANSLATOR/T3 baseline、跨 ISA 数据与表示方法参考 |

这篇论文最值得学习的是针对低层汇编设计表示和评测闭环，而不是单纯扩大模型；最主要的局限是测试正确性不能替代跨 ISA 语义证明，且地址与寄存器错误仍然集中。如果用于后续研究，最合理的使用方式是作为 RVV/跨 ISA Translator baseline，并在其上增加验证门控或配置选择层，而不是把直接汇编生成误称为 schedule/config Selector。

