# AIVRIL 2 文献阅读总结

论文题目：**EDA-Aware RTL Generation with Large Language Models**

作者：Mubashir ul Islam、Humza Sami、Pierre-Emmanuel Gaillardon、Valerio Tenace

发表时间：2025（DATE 2025；arXiv 首次公开于 2024-11-21）

发表平台：DATE 2025

论文链接或编号：DOI 10.23919/DATE64628.2025.10992789；arXiv:2412.04485
元数据核验来源：[IEEE DOI](https://doi.org/10.23919/DATE64628.2025.10992789)；[DATE 官方 proceedings archive](https://past.date-conference.com/proceedings-archive/2025/START.pdf)；[arXiv](https://arxiv.org/abs/2412.04485)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：大语言模型、RTL 生成、电子设计自动化、 多智能体、语法修正、功能验证、Verilog、VHDL

> 本笔记依据本槽位保存的 arXiv 正文 PDF（8 页）及 DATE 2025/DOI 元数据撰写。论文事实与阅读后的研究思考分开描述。

## 1. 研究背景

本文属于大语言模型辅助电子设计自动化（Electronic Design Automation, EDA）和寄存器传输级（Register Transfer Level, RTL）代码生成。RTL 用硬件描述语言表达时序逻辑、数据通路和模块接口，是从设计规格走向综合、仿真和实现的重要中间层。

论文引言指出，LLM 可以根据自然语言需求生成 RTL，但零样本生成仍经常出现语法错误和功能错误。人工反复检查、修改、编译和仿真会增加验证负担，也可能引入新的错误。现有工作往往只处理某一个环节，例如语法修正、局部调试或单一 Verilog 流程，难以形成同时覆盖测试基准生成、语法检查和功能验证的闭环。

论文因此引入外部 EDA 工具反馈：编译器/编译工具提供语法日志，仿真器提供功能日志，LLM 代理将这些日志转换成下一轮修正提示。研究价值在于把概率式代码生成连接到可执行的编译与仿真证据，而不是只依赖模型内部推理。

## 2. 论文要解决的问题

### 2.1 语法正确性问题

LLM 生成的 RTL 可能无法通过编译。论文希望自动分析 EDA 编译日志，定位语法问题，并将其转化为 Code Agent 可以执行的纠正提示（第 2、3.2 节）。

### 2.2 功能正确性问题

即使 RTL 能编译，也可能不能满足设计行为。论文使用测试台和仿真日志检查功能失败，并让 Verification Agent 生成修正反馈（第 3.3 节）。

### 2.3 多语言和模型适应性问题

相关工作多集中于 Verilog。本文希望框架对 RTL 语言正交，并在 Verilog 与 VHDL 上验证，同时不依赖某一个固定 LLM。

> 本文主要研究：如何把 LLM、EDA 编译/仿真工具和多代理反馈循环组合成一个可迭代的 RTL 生成与自验证框架，以提高语法通过率和功能通过率。

## 3. 核心方法概述

AIVRIL 2 是一个 LLM-agnostic（与具体 LLM 解耦）的多代理 RTL 生成框架。Code Agent 负责生成测试台和 RTL；Review Agent 负责解析编译器日志并修正语法；Verification Agent 负责解析仿真日志并修正功能。框架先生成测试台，再用固定测试台持续验证后续 RTL 版本。

整体数据流如下：

```text
用户自然语言规格
        ↓
Code Agent 生成测试台
        ↓
Code Agent 根据规格和测试台生成 RTL
        ↓
RTL 编译器检查语法
        ↓
Review Agent 读取编译日志并产生语法纠正提示
        ↓
语法通过后，用同一测试台运行仿真
        ↓
Verification Agent 读取仿真日志并产生功能纠正提示
        ↓
Code Agent 更新 RTL，循环直到通过或达到最大迭代次数
```

与传统一次性生成相比，本文的区别是把 RTL 生成变成“生成—编译—解释日志—修正—仿真”的工具反馈过程。LLM 仍然直接输出 RTL，因此按最终系统角色建议归类为 `TRANSLATOR / T1_Source_Optimization_Refactoring`；这只是 staging 中的分类建议，不是正式 taxonomy 登记。

## 4. 实验框架与训练流程

### 4.1 测试台优先阶段

Code Agent 先读取用户需求，生成覆盖预期行为的测试台。论文强调测试台在后续迭代中保持不变，以便不同 RTL 版本使用相同标准进行比较（第 3.1、3.3 节）。如果用户规格不充分，Code Agent 会通过额外提示与用户交互补充信息。

### 4.2 RTL 生成阶段

Code Agent 以用户规格和测试台为条件生成初始 RTL。论文给出的示例是一个复位后连续四个时钟周期置位 `shift_ena` 的模块。

### 4.3 Syntax Optimization Loop

Review Agent 将 RTL 送入行业标准 RTL 编译器，读取编译日志，识别错误位置、代码片段和可执行修正建议。若发现语法错误，Code Agent 根据纠正提示重新生成 RTL；循环继续直到语法通过。

### 4.4 Functional Optimization Loop

当 RTL 和测试台均通过语法检查后，Verification Agent 调用仿真器执行测试台。它比较测试输出和预期行为，读取失败日志，并给 Code Agent 发送功能纠正提示。直到所有测试通过，或达到预设最大迭代次数。

### 4.5 训练方式与工具调用

本文不涉及模型预训练、SFT、PPO、GRPO 或其他强化学习训练。实验中使用现成 LLM，以提示和外部工具反馈驱动推理；论文明确说明没有 fine-tuning，也没有 RAG。工具调用包括 RTL 编译和混合语言仿真。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有报告模型训练损失函数。主要优化目标是可执行的离散通过条件：

```text
语法循环终止条件：RTL 通过编译器语法检查
功能循环终止条件：固定测试台中的全部测试通过
```

论文实验中的 `pass@1S` 表示一次生成/流程结果通过全部语法检查的比例，`pass@1F` 表示一次结果同时满足语法和功能测试的比例。它们是评价指标，不是训练奖励。功能通过依赖给定测试台，不能等同于对所有可能输入的形式化证明。

## 6. 实验设置

### 6.1 数据集来源

论文使用 VerilogEval-Human benchmark suite 的全部 156 个 benchmark。功能通过率由该基准提供的测试台执行得到。论文没有把这些样本描述为作者新构建的数据集，也没有给出独立训练集、验证集和测试集划分；因此数据泄漏风险的完整评估在论文中未明确说明。

任务覆盖两种 RTL 语言：Verilog 和 VHDL。论文指出，VerilogEval-Human 原始基准用于评测，且本文据此报告两种语言的结果；关于 VHDL 测试项如何从基准组织得到，论文未进一步完整说明。

### 6.2 模型与工具

| 类别 | 论文设置 |
|---|---|
| LLM | Claude 3.5 Sonnet、GPT-4o、Llama3-70B |
| 微调/RAG | 均未使用 |
| 温度 | 0.2 |
| top_p | 0.1 |
| 编译/仿真工具 | Vivado Design Suite - HLx Editions 2018.1 |
| 语言 | Verilog、VHDL |
| 代理 | Code Agent、Review Agent、Verification Agent |
| 评估规模 | VerilogEval-Human 全部 156 个 benchmark |

论文没有报告硬件平台、GPU 型号、LLM 参数量或推理框架版本。

### 6.3 对比方法

主要对比包括三个基础 LLM：Llama3-70B、GPT-4o、Claude 3.5 Sonnet；还与 CodeGen-16B、CodeV-CodeQwen、ChipNeMo-13B/70B、CodeGen-16B-Verilog-SFT、RTLFixer、VeriAssist 和作者先前的 AIVRIL 比较。表 2 的跨工作比较只报告 Verilog 的 `pass@1F`，因为作者认为此前工作缺少 VHDL 可比结果。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| `pass@1S` | 一次结果通过全部语法检查的比例 | 越大越好 |
| `pass@1F` | 一次结果通过语法检查并通过功能测试的比例 | 越大越好 |
| `ΔF` | AIVRIL 2 相对对应基础 LLM 的功能通过率提升 | 越大越好 |
| 平均延迟 | 生成与 EDA 工具执行的平均耗时 | 越小越好，但需结合正确率 |

作者采用 unbiased pass@k estimator，实验中 `k=1`。延迟图计入 EDA 工具执行时间。

## 7. 实验结果与结论

### 7.1 主要结果

表 1 在 156 个 VerilogEval-Human benchmark 上报告结果。AIVRIL 2 在除 Llama3-70B 的 VHDL 配置外均达到 100% 的 `pass@1S`；该配置达到 58.87%，而其基础模型的语法通过率为 1.28%。

功能通过率如下，数值均为百分比，且是对应语言和模型配置的一次通过率：

| 配置 | Verilog `pass@1F` | VHDL `pass@1F` |
|---|---:|---:|
| Llama3-70B 基线 | 37.82 | 0 |
| GPT-4o 基线 | 51.29 | 27.56 |
| Claude 3.5 Sonnet 基线 | 60.23 | 53.85 |
| AIVRIL 2 + Llama3-70B | 55.13 | 32.69 |
| AIVRIL 2 + GPT-4o | 72.44 | 59.62 |
| AIVRIL 2 + Claude 3.5 Sonnet | 77.00 | 66.00 |

作者将平均功能质量提升报告为 Verilog 38.28%、VHDL 至少 69.44%。这里的提升来自表 1 对应基线的汇总口径，不应解释成每个单独配置都提升相同幅度。

### 7.2 与传统方法的比较

本文不是与传统编译器优化 pass 或 autotuner 比较，而是与 LLM/LLM-agent RTL 生成系统比较。AIVRIL 2 的结果显示，外部编译和仿真反馈可以提升代码的可编译性和功能通过率；但它并没有证明生成 RTL 在综合面积、时序、功耗或真实芯片性能上优于传统设计流程。

### 7.3 与其他 LLM 方法的比较

表 2 中，Verilog `pass@1F` 为：CodeGen-16B 41.9%、CodeV-CodeQwen 53.2%、ChipNeMo-13B 22.4%、ChipNeMo-70B 27.6%、CodeGen-16B-Verilog-SFT 28.8%、RTLFixer 36.8%、VeriAssist 50.5%、GPT-4o 51.29%、Claude 3.5 Sonnet 60.23%、AIVRIL 67.3%。AIVRIL 2 + Claude 3.5 Sonnet 为 77%，相对表中 ChipNeMo-13B 的 22.4% 约为 3.4 倍；这是特定基准、语言和指标下的比较，不是通用硬件生成能力的倍数结论。

### 7.4 消融实验

论文没有给出标准意义上逐模块删除 Code Agent、Review Agent 或 Verification Agent 的完整消融表。它通过 syntax loop、functional loop 的分阶段结果和延迟分解展示组件作用，但不能据此声称完成了严格的独立消融。

### 7.5 延迟与案例

作者报告最差情况下的平均额外延迟不超过 42 秒。Claude 3.5 Sonnet 生成 Verilog 时，Syntax Optimization Loop 平均需要 2 步，Functional Optimization Loop 平均需要 3 步；Llama3-70B 生成 VHDL 时分别平均需要 3.95 和 4.7 个循环。论文示例中，Verification Agent 发现 `shift_ena` 在四个时钟周期后未清零，随后提示 Code Agent 增加计数条件，最终测试台输出 “All tests passed successfully!”。

## 8. 主要创新点

### 8.1 创新点一：测试台优先的两阶段反馈流程

已有流程通常直接生成 RTL 或只做语法修正。本文先生成测试台，再把流程分成 Syntax Optimization Loop 和 Functional Optimization Loop，使功能反馈具有固定的评估基准。论文通过语法与功能通过率提升证明该组合有效，但没有证明测试台能覆盖所有设计行为。

### 8.2 创新点二：专门化代理与 EDA 日志对接

本文将 Code Agent、Review Agent、Verification Agent 的职责分开，并把编译器日志和仿真日志转成纠正提示。价值在于让 LLM 的后续修改依赖可执行工具证据，而不是仅依赖自我批评。这个机制由论文的方法框架和示例支持，但并非形式化验证器。

### 8.3 创新点三：面向 Verilog 与 VHDL 的语言解耦设计

论文声称框架与目标 RTL 语言正交，并用 Verilog/VHDL 评测。VHDL 基线很弱时，AIVRIL 2 仍能提高功能通过率，这支持了框架对语言差异的适应性；但实验语言数量只有两种，不能直接推广到 SystemVerilog 或其他 HDL。

## 9. 局限性

### 9.1 论文明确承认或可直接从实验读出的局限

1. 多轮编译和仿真会增加延迟，尤其是 Llama3-70B 生成 VHDL 时循环次数和延迟明显增加。
2. 跨工作比较主要限于 Verilog，因为作者没有找到可比的 VHDL 结果。
3. 框架仍依赖测试台质量；论文没有给出对抗错误测试台或覆盖率不足的系统分析。
4. 功能正确性由基准测试台通过来定义，不是对所有输入的形式化语义证明。
5. 论文采用 Vivado Design Suite 2018.1，工具版本较旧，跨工具可复现性需要额外确认。

### 9.2 阅读后发现的潜在局限

1. 固定测试台可能使代理优化到测试台行为，而不是真正满足完整规格。
2. 论文未报告综合后的面积、频率、功耗、资源利用率或硬件质量，因此“代码质量”主要指语法和功能通过率。
3. 没有完整消融实验，难以分别量化测试台生成、语法代理和功能代理的边际贡献。
4. 论文未报告最大迭代次数的具体值、各轮 token 成本或失败重试成本，工程部署成本当前 PDF 内容不足以确认。
5. 将平台替换到 LLVM IR、RISC-V 或其他 HDL 不能自动继承本文结果；需要重新定义编译、语义验证和性能反馈。

## 10. 阅读后的研究方向反思

本文最值得借鉴的是“模型输出必须经过可执行工具反馈”的证据链：生成结果、编译日志、仿真日志和修正版本之间有明确关系。对于 LLVM/MLIR 研究，可以借鉴其把 compiler diagnostics、IR verifier、translation validation 或运行时测试作为反馈接口的思想。

不能直接照搬的部分包括：RTL 测试台驱动的功能检查与 LLVM IR 优化并不等价；RTL 的时序和硬件仿真失败日志也不能直接替代 IR 语义等价证明。仅把 Verilog 替换为 RISC-V 汇编或 LLVM IR，属于平台迁移，单独看不足以形成新颖性。

更合理的定位是：本文适合作为“工具反馈式 Translator/代码生成代理”的方法参考和实验基线；若用于 RISC-V/LLVM 方向，应增加 ISA 约束、ABI 约束、机器 IR 语义检查、差分测试和真实硬件性能证据。

## 11. 可进一步尝试的研究方向

### 11.1 面向 LLVM IR 的双证据优化代理

#### 研究问题

能否让 LLM 直接提出 LLVM IR 改写，同时由 verifier 检查语义、由目标机运行或仿真反馈性能？

#### 与原论文的区别

原论文处理 RTL 语法/功能；该方向同时约束 LLVM IR 语义等价和目标机性能，不只追求测试通过。

#### 可能的创新点

把 IR verifier、Alive2/translation validation 和性能测量统一成可审计反馈协议。

#### 实验框架

```text
LLVM IR 输入 → LLM 提出改写 → verifier/翻译验证 → 目标机编译运行
          → 语义与性能反馈 → 选择或修正候选
```

#### 可行性

需要 LLVM/MLIR、语义验证器、可重复 benchmark 和至少一个真实或可控的目标硬件平台。

#### 主要风险

验证器支持范围、性能噪声和候选改写导致的未定义行为可能限制闭环可靠性。

### 11.2 RISC-V 后端的编译器日志与硬件反馈协同

#### 研究问题

能否让代理根据 LLVM 后端诊断、指令合法性检查和 RISC-V 仿真性能反馈选择后端配置或修正低层代码？

#### 与原论文的区别

不是把 HDL 换成 RISC-V，而是把“语法—功能”双循环扩展成“合法性—语义—性能”三层证据。

#### 可能的创新点

将 RISC-V 扩展约束、寄存器分配失败、指令选择结果和硬件计数器统一表示为反馈状态。

#### 实验框架

```text
LLVM/MIR 或 C 输入 → 后端候选 → assembler/verifier → Spike/QEMU/真实板卡
              → 性能计数器与失败日志 → 代理修正或候选排序
```

#### 可行性

可从 RV64GC 或 RVV 基线开始，再增加一个明确的自定义扩展；需要 LLVM、汇编器、模拟器和硬件计数器。

#### 主要风险

模拟器结果不一定代表真实芯片，且自定义扩展的工具链支持成本较高。

### 11.3 测试台质量感知的编译器代码生成

#### 研究问题

如何检测自动生成测试是否覆盖了真正的语义边界，避免代理只针对弱测试台优化？

#### 与原论文的区别

原论文固定测试台并将其作为验证标准；该方向显式评估测试台覆盖率、变异杀伤率和规格一致性。

#### 可能的创新点

把测试生成、规格约束和变异测试纳入反馈循环，并区分“测试通过”和“语义可信”。

#### 实验框架

```text
规格与候选程序 → 生成测试 → 变异/覆盖分析 → 编译运行与失败归因
             → 改进测试或程序 → 独立隐藏测试集复评
```

#### 可行性

需要 LLVM/RTL 测试工具、覆盖率工具、变异框架和隐藏测试集。

#### 主要风险

覆盖率指标容易被投机，且生成高质量 oracle 的成本可能超过 LLM 节省的人工成本。

## 12. 与其他已读文献的关系

本槽位只完成 AIVRIL 2 一篇论文，没有同批次第二篇已通读论文可作事实级横向比较。因此本节不虚构跨论文实验差异。

就主题定位而言，AIVRIL 2 与正式 corpus 中的 RTL 生成、RTL 验证和硬件代码生成工作存在相邻关系，但本笔记没有重新通读那些条目的正文，不能在此处给出方法或数字比较。AIVRIL 2 最适合作为“带 EDA 工具反馈的 Translator/代码生成流程”参考，而不是 LLVM pass 选择器、传统 autotuner 或形式化验证器本身。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 使用多智能体和 EDA 工具反馈提高 LLM RTL 生成的语法与功能正确性 |
| 核心问题 | 零样本 RTL 生成错误多、人工调试成本高、现有方法环节割裂 |
| 输入 | 用户自然语言规格、生成测试台、LLM 反馈提示 |
| 输出 | Verilog/VHDL RTL 代码 |
| 核心方法 | Code Agent + Review Agent + Verification Agent；语法循环与功能循环 |
| 使用的模型 | Claude 3.5 Sonnet、GPT-4o、Llama3-70B |
| 使用的编译器工具 | Vivado Design Suite - HLx Editions 2018.1；RTL 编译与仿真 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；使用编译检查和测试台仿真，不等同形式化证明 |
| 数据集规模 | VerilogEval-Human 全部 156 个 benchmark |
| 主要指标 | `pass@1S`、`pass@1F`、`ΔF`、平均延迟 |
| 最重要实验结果 | AIVRIL 2 + Claude 3.5 Sonnet 达到 Verilog 77%、VHDL 66% 的 `pass@1F` |
| 核心创新 | 测试台优先、语法/功能双循环、专门化代理解析 EDA 日志、语言解耦 |
| 主要局限 | 依赖测试台和 Vivado；有额外延迟；缺少综合 PPA 与严格消融 |
| 与 RISC-V 研究的相关性 | 中低：可借鉴工具反馈闭环，但论文对象是 RTL 生成而非 RISC-V 编译 |
| 最适合作为 | 工具反馈式 Translator 方法参考、RTL 生成基线、验证闭环设计参考 |

这篇论文最值得学习的是把 LLM 的 RTL 输出放进可执行的编译和仿真闭环，并把日志变成下一轮修正证据；最主要的局限是通过测试台不等于形式化语义正确，也没有证明综合后的面积、时序和功耗收益。如果用于后续研究，最合理的使用方式是借鉴其反馈协议并替换为 LLVM/MLIR/RISC-V 的语义与性能证据，而不是简单把目标语言改成 RISC-V 后宣称完成了新的编译器方法。

