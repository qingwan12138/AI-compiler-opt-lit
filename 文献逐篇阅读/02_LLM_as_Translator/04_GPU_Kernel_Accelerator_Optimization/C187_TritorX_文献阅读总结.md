# TritorX 文献阅读总结

论文题目：**Agentic Operator Generation for ML ASICs**

作者：Alec M. Hammond、Aram Markosyan、Aman Dontula、Simon Mahns、Zacharias Fisches、Dmitrii Pedchenko、Keyur Muzumdar、Natacha Supper、Site Cao、Haishan Zhu、Mark Saroufim、Joe Isaacson、Laura Wang、Warren Hunt、Kaustubh Gondkar、Roman Levenstein、Gabriel Synnaeve、Richard Li、Jacob Kahn、Ajit Mathews

发表时间：2026

发表平台：MLSys 2026, Proceedings of Machine Learning and Systems 8

论文链接或编号：arXiv:2512.10977
元数据核验来源：[MLSys 正式论文页](https://proceedings.mlsys.org/paper_files/paper/2026/hash/8c54e9bfed4119c873f575d1d1e2f0a0-Abstract-Conference.html)；[arXiv](https://arxiv.org/abs/2512.10977)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：TritorX、LLM、Triton、PyTorch ATen、MTIA、ASIC kernel generation、有限状态机、OpInfo、执行反馈

> 本文档只依据 MLSys 2026 正式 PDF；arXiv 版本作为同一版本族记录，不另立条目。论文事实与阅读后的研究思考分开描述。

## 1. 研究背景

本文研究自研机器学习加速器的编译器和软件栈启动问题。论文指出，定制 ASIC 可以提供成本或能效优势，但要让 PyTorch 等上层框架真正运行，需要为大量 ATen 张量算子建立后端实现。算子覆盖的人工维护成本高，而且新硬件的 Triton 方言、内存约束和专用 intrinsic 往往不能直接由通用 GPU 经验推出（第 1—2 节）。

论文关注的是“先让整个后端广泛可用并且正确”，而不是只为少数热点 kernel 做极限性能调优。TritorX 使用大语言模型（Large Language Model，能够根据算子文档和反馈生成代码）生成 Triton kernel-wrapper 对，并用 linter、JIT 编译器、设备执行和测试反馈反复修正。

## 2. 论文要解决的问题

### 2.1 大规模 ATen 算子覆盖

需要为 MTIA 兼容的 ATen 算子生成可注册到 PyTorch 后端的实现，并覆盖不同数据类型、张量形状和参数组合。单纯人工编写难以在新硬件上快速扩展。

### 2.2 非标准 Triton 方言下的正确性

MTIA 的 Triton 方言并不等同于 GPU 版本 Triton。生成代码可能使用不可用 intrinsic、违反对齐约束、调用尚未实现的其他算子，或在低精度下出现数值错误。因此系统需要把编译器、linter、调试器和测试结果转成可用于下一轮生成的反馈。

### 2.3 从 OpInfo 测试到生产输入的泛化

PyTorch OpInfo 能提供大量标准化测试，但真实模型输入可能不在其分布内。论文还要检验由 OpInfo 反馈生成的 kernel 是否能覆盖 NanoGPT、DLRM 和内部推荐模型的生产输入。

> 本文主要研究：如何用带工具约束和执行反馈的 LLM 有规模地生成 MTIA 上的 Triton ATen kernel-wrapper，并以全面测试实现可用的 PyTorch 后端覆盖。

## 3. 核心方法概述

TritorX 是一个以有限状态机（Finite-State Machine，明确规定执行阶段和状态转移的工作流）为骨架的推理时 kernel 生成系统。LLM 直接输出 Python wrapper 与 Triton kernel；其余状态负责检查、编译、执行、调试和整理反馈。

```text
ATen 算子集合 + docstring/signature + 三个手工示例
        ↓
LLM 生成 wrapper/kernel pair
        ↓
Triton MTIA linter 检查语法、方言和禁止的“cheating”调用
        ↓
JIT 编译 + MTIA/QEMU 执行 + OpInfo/生产输入测试
        ↓
编译日志、崩溃信息、输出差异或测试结果
        ↓
反馈整理器/LLDB debugger 形成下一轮 prompt
        ↓
通过全部测试则注册 kernel；否则继续迭代或失败
```

LLM 的最终输出是可执行的 kernel-wrapper 代码，而不是从既有优化动作中选择一个动作，因此本报告建议归类为 `TRANSLATOR / T4_GPU_Kernel_Accelerator_Optimization`。系统主要使用推理时多轮修复，不是只生成一次候选后由外部搜索器选择。

## 4. 实验框架与训练流程

### 4.1 模型输入与初始化

每个算子使用独立会话。初始 prompt 包含任务描述、输出格式要求、PyTorch 算子 docstring/signature 和三个手工 kernel-wrapper 示例。论文还把互相引用的 ATen docstring 组织成有向无环图，以便把嵌套算子说明加入上下文（第 3.1 节）。

### 4.2 生成、linter 与编译

LLM 生成一个 wrapper/kernel pair。自定义 Triton MTIA linter 检查：代码是否能接入 JIT harness；是否通过 wrapper 调用尚未实现的其他 ATen/CPU 算子；是否使用有效的 MTIA Triton 语法和库。linter 失败会产生结构化报告并回传 LLM。

### 4.3 执行测试与反馈状态

通过 linter 后，代码进入 JIT 编译和执行测试。设备输出与 CPU ATen reference 输出按 PyTorch 容差比较。编译失败、运行时错误、精度错误或 crash 会触发反馈状态；编译日志可由第二个 LLM 摘要，crash dump 可交给基于 LLDB 的 debugger。若全部测试通过则成功，达到最大迭代次数、上下文饱和或主进程异常则终止。

### 4.4 是否训练模型

本文不涉及 SFT、PPO、GRPO 或其他模型训练。作者使用开源或闭源现成 LLM，并通过 linter、compiler、debugger 和测试交互在推理时逐步获得 MTIA 方言信息。实验中的 kernel-generation LLM 包括 CWM、GPT-OSS 120B、Claude Sonnet 4.5 和 Claude Opus 4.5；Llama-4-Maverick 用于反馈摘要（第 4 节）。

## 5. 奖励函数、损失函数或关键公式

本文没有强化学习奖励函数，也没有报告模型训练损失。系统的“成功”是一个离散的测试判定：生成的 kernel-wrapper 必须通过该算子对应的全部测试。

可用流程表达如下：

```text
Success(operator) = 1，当所有对应 OpInfo/指定生产输入测试通过
Success(operator) = 0，否则
```

这不是论文提出的可微损失或强化学习 reward，而是状态机的终止条件。性能不是该系统的主要优化目标；论文只对一部分 NanoGPT kernel 记录了相对手写 kernel 的性能。

## 6. 实验设置

### 6.1 数据集来源

主要测试来自 PyTorch OpInfo。初始 629 个算子经过 MTIA 兼容性筛选，得到 568 个 unique operators；论文只使用 bfloat16、float16、float32、int32 和 int64，并为全部算子组织了 20,000 多个测试。复杂数、随机数等难以在设备与 CPU 之间一致验证的算子被排除（第 3.2 节）。

生产输入来自 NanoGPT、DLRM 和两个内部推荐模型（MM1、MM2）。作者用 `__torch_dispatch__` 记录 forward/backward 过程中的张量和标量输入，固定 batch size 为 1024；其中 DLRM、MM1、MM2 使用真实生产数据而非随机输入（第 4.1 节）。

论文未把这些输入称为独立训练集、验证集和测试集；它们主要用于生成期间的测试与泛化检查，因此不应按监督学习数据集解读。

### 6.2 模型与工具

| 类型 | 配置 |
|---|---|
| 生成模型 | CWM、GPT-OSS 120B、Claude Sonnet 4.5、Claude Opus 4.5 |
| 反馈摘要模型 | Llama-4-Maverick |
| 目标 DSL | Triton MTIA dialect |
| 编译/执行 | Triton JIT、MTIA 硅片；未来设备使用 QEMU simulator |
| 测试 | PyTorch OpInfo、生产模型输入、CPU ATen reference |
| 调试与约束 | 自定义 linter、LLDB-based debugger |
| 推理参数 | context length 131,072；temperature 1.0；CWM top-P 0.95，GPT-OSS top-P 1.0 |
| 扩展规模 | 200 个生产 MTIA 设备（硅片或模拟器） |

### 6.3 对比方法

主要对比不是外部 kernel 生成系统，而是 TritorX 自身的 harness 配置：完整 baseline、去掉 linter、去掉反馈摘要模型。模型之间还比较 CWM、GPT-OSS、Sonnet 和 Opus 的 operator coverage。

### 6.4 评价指标

| 指标 | 定义 | 方向 |
|---|---|---|
| Operator coverage | 生成的 kernel-wrapper 通过该算子全部指定测试的比例 | 越大越好 |
| Production/model-input coverage | kernel 对模型输入形状和参数测试全部通过的比例 | 越大越好 |
| Kernel performance ratio | 生成 kernel 相对手写 kernel 的性能比例 | 越大越好 |
| LLM calls per operator | 生成一个正确 kernel 所需调用数 | 越小越好，但需结合 coverage |

## 7. 实验结果与结论

### 7.1 主要结果

在聚合多次运行、模型和配置消融的结果中，TritorX 对 MTIA-compatible OpInfo operators 达到 84.0% coverage；论文将 coverage 定义为 kernel-wrapper 通过该算子 100% 对应 OpInfo sample tests。摘要和结论均报告超过 481 个 ATen operators 通过其对应的 20,000 多个测试（第 4 节、第 7 节）。

单次 baseline 配置中，最多允许每算子 3 个 dialog sessions，每次最多 15 次 LLM calls。按模型，图 3 给出单次运行 coverage：CWM 55.3%、GPT-OSS 72.0%、Claude Sonnet 72.2%、Claude Opus 78.7%；模型 ensemble 的 84.0% 是跨运行聚合结果，不是单个模型一次运行结果。

### 7.2 生产模型输入

对四类模型输入，OpInfo/MIS 表格报告的 full model operator coverage 为：NanoGPT 87.2%、DLRM 81.4%、MM1 79.8%、MM2 80.6%。对已有 OpInfo kernel 再使用 model-input-specific（MIS）反馈后，覆盖率分别为 100.0%、90.0%、91.9%、87.3%。论文据此总结，系统能够覆盖接近 80% 的端到端模型 kernel 需求，并在额外 MIS refinement 后提升 6–20%（第 4.1 节）。

### 7.3 按算子类别和性能

表 1 中，Shape Manipulation 类 coverage 约为 94.7%–96.0%，而 Deep Learning 类约为 64.4%–76.2%，说明复杂深度学习模式更难映射到 MTIA Triton semantics。失败算子的主要问题是低精度数值错误和对 MTIA Triton dialect 不熟悉。

在 300 多个 NanoGPT 的 kernel/unique tensor shape 对上，大多数生成 kernel 达到手写 kernel 至少 70% 的性能；只有少数超过手写实现。论文明确说明性能调优不是 TritorX harness 的主要目标。

### 7.4 消融实验

单次 baseline 与去除组件的 coverage 如下：

| 配置 | CWM | GPT-OSS | Sonnet | Opus |
|---|---:|---:|---:|---:|
| 完整 baseline | 55.3% | 72.0% | 72.2% | 78.7% |
| 去掉 linter | 35.7% | 46.7% | 51.2% | 71.3% |
| 去掉 summarization | 48.2% | 71.5% | 69.4% | 71.8% |

去掉 linter 对 CWM 和 GPT-OSS 影响最大，说明 linter 同时提供方言约束和禁止“cheating”的安全边界。去掉摘要模型对 CWM 下降明显，但对 GPT-OSS 没有同样幅度的下降，说明反馈压缩效果依赖生成模型。

### 7.5 QEMU 与失败模式

在未来硬件的 QEMU simulator 上，GPT-OSS 单次运行达到 73.1% coverage。未覆盖算子中，约 80% 仍能编译并通过至少部分测试，约 30% 通过超过 80% 的测试。论文没有把这些部分通过结果当作成功覆盖。

## 8. 主要创新点

### 8.1 创新点一：coverage-first 的 ASIC 后端生成

与只追求少数高频 kernel 的系统不同，TritorX 把整个 ATen operator set 的正确性和覆盖作为首要目标。实验报告 481+ 个通过全部对应 OpInfo 测试的算子，以及 84.0% 的聚合 coverage，说明该设计确实改变了 kernel generation 的目标函数。

### 8.2 创新点二：面向 MTIA 方言的受约束 FSM harness

论文将生成、linter、JIT 编译、执行测试、debugger 和反馈摘要组织成显式有限状态机。linter 不只是语法检查，还阻止 wrapper 调用其他未实现或 CPU 算子，降低通过测试漏洞投机的风险。去掉 linter 后 coverage 显著下降，支持其必要性。

### 8.3 创新点三：从标准 OpInfo 到生产输入的闭环

系统先用 OpInfo 生成广泛覆盖的实现，再用真实模型输入检查分布外的形状和参数；失败时把 MIS 反馈重新用于 kernel refinement。该设计把“可通过标准测试”与“能运行代表性模型”连接起来。

## 9. 局限性

### 9.1 论文明确承认的局限

- MTIA 不支持复数，随机数算子也难以与 CPU reference 对齐，因此部分算子被排除。
- OpInfo 的高质量测试套件本身限制了泛化；它不能覆盖整个输入空间。
- 性能调优不在主要范围内，当前 harness 主要追求功能正确性和 coverage。
- Triton 的表达粒度低于 C++，某些 kernel 细节不能同样精细地控制。
- 低精度数值错误和 MTIA 方言不兼容仍是主要失败模式。

### 9.2 阅读后发现的潜在局限

- 84.0% 是跨多次运行聚合得到的 coverage，不能直接解释为一次运行的成功率；重复采样成本和尾部运行时间需要单独核算。
- 200 台 MTIA 设备、生产容器和内部模型输入构成较高复现门槛；论文未提供一个完全独立的公开硬件复现实验环境。
- 测试通过证明的是指定样例集合上的功能正确性，不是形式化语义等价证明。
- 目标方言与 MTIA 强绑定，迁移到 GPU Triton、RISC-V NPU 或其他 ASIC 需要重建 linter、编译器反馈和测试接口。
- 论文把生成与性能调优分开，因而生成 kernel 达到手写性能的比例不能代表系统已经解决了 kernel optimization。

## 10. 阅读后的研究方向反思

TritorX 最值得借鉴的是“编译器反馈不是最后的验证器，而是生成过程中的语言接口”。它把方言错误、运行时崩溃和输出差异压缩成下一轮 prompt，使 LLM 承担的是直接代码生成和修复，而不是选择一个已有的优化动作。

对 RISC-V 的直接启发有限但明确：如果目标是 RISC-V 向量或自研 AI 加速器，单纯把 MTIA 换成 RISC-V 并不足以形成新贡献；必须研究 RISC-V 特有的向量长度、内存对齐、intrinsic 可用性、编译器诊断与跨版本 ISA 迁移如何共同约束生成代码。TritorX 更适合作为 Translator/T4 的 baseline 或工具框架，而不是可直接照搬的性能优化方法。

论文已经完成的核心贡献是 coverage-first 的 MTIA backend generation 和 FSM harness。后续工作应在新问题上增加可验证创新，例如把生成代码映射到 LLVM IR/向量 intrinsic 后进行等价检查，或研究跨硬件版本的 kernel migration，而不是仅替换目标设备。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V 向量扩展的诊断驱动 kernel 翻译

#### 研究问题

如何将 LLM 生成的高层 kernel 稳定地翻译为 RVV-aware intrinsic 或 MLIR/LLVM IR，并利用编译器诊断修复向量长度、掩码和对齐错误。

#### 与原论文的区别

目标从 MTIA Triton 方言变为 RVV/MLIR/LLVM 约束，并增加显式低层中间表示和 ISA 语义检查；不是简单更换硬件名称。

#### 可能的创新点

把编译器诊断分成语法、向量语义、内存安全和性能反模式四类，设计面向类别的反馈上下文；使用 SMT 或等价测试检查关键 kernel 的语义保持。

#### 实验框架

```text
算子规格 → LLM 生成 kernel/MLIR → RVV lowering
        → 编译诊断 + 运行时测试 + 等价检查
        → 分类反馈 → 下一轮翻译
```

#### 可行性

需要 LLVM/MLIR、RVV simulator 或真实开发板、可执行测试和若干公开算子；可从 elementwise、reduction 和 matmul 子集开始。

#### 主要风险

RVV 模拟器与真实芯片性能差异、浮点容差和跨向量长度泛化可能导致结果难以解释。

### 11.2 面向跨代加速器的 kernel-wrapper 迁移

#### 研究问题

给定旧硬件上通过测试的 kernel-wrapper，如何迁移到新 ISA/方言，同时保持 ATen 接口、测试覆盖和性能下界。

#### 与原论文的区别

原论文主要是从 docstring 生成新实现；该方向研究已有实现的版本迁移，并显式利用旧后端状态和差异分析。

#### 可能的创新点

使用差异化 compiler IR、旧/新后端的对照执行和跨版本约束，避免每个算子从零生成。

#### 实验框架

```text
旧版 kernel-wrapper + 新硬件约束
        → LLM 迁移候选
        → 双后端测试/编译/性能比较
        → 差异反馈 → 新版 kernel-wrapper
```

#### 可行性

可使用 Triton 方言或 LLVM IR 的两种后端、OpInfo 子集和模拟器；不需要训练新模型即可验证推理时迁移。

#### 主要风险

旧后端的行为可能依赖未公开硬件特性；跨后端的性能比较可能不公平。

### 11.3 覆盖率与性能联合的安全生成

#### 研究问题

如何在保持 TritorX 大规模正确性覆盖的同时，识别最值得做性能优化的 kernel，并避免性能改写引入正确性回归。

#### 与原论文的区别

原论文明确把性能调优留作未来工作；该方向把 profiling、性能反模式和回归检测纳入生成闭环。

#### 可能的创新点

构建 coverage、correctness、latency、资源占用的多目标候选管理，并为每一类失败保留可审计证据。

#### 实验框架

```text
正确 kernel 库 → profiler 定位瓶颈
        → LLM 生成优化候选
        → 编译/正确性/性能三重门禁
        → 保留 Pareto 候选并更新反馈
```

#### 可行性

需要硬件 profiling 接口、稳定的 benchmark harness 和已有正确 kernel；可先在少量 NanoGPT kernel/shape 对上测试。

#### 主要风险

硬件噪声、shape-specific 优化和多目标排序可能让收益不稳定；必须严格区分真实运行时间与静态估计。

## 12. 与其他已读文献的关系

本阶段当前只完成 TritorX 的全文笔记，尚未对 GEAK 或 CUDA-LLM 完成正文阅读，因此不把它们的摘要信息写成已读结论。TritorX 正文将 CUDA-LLM、GEAK 等工作作为相关的 inference-time kernel generation 方向，但这些引用关系不构成版本重复。

在本批次中，TritorX 的清晰定位是：直接生成加速器 kernel-wrapper 的 Translator；其独特侧重点是 custom ASIC 的全后端 coverage、MTIA 方言 linter 和 OpInfo/生产输入双重测试。后续阅读 GEAK 与 CUDA-LLM 后，重点应比较它们的目标硬件、代码输出、反馈粒度、是否进行性能搜索，以及是否把 benchmark 贡献误当成核心生成系统。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 为 Meta MTIA/custom ML ASIC 自动生成 PyTorch ATen Triton 后端 |
| 核心问题 | 大规模算子覆盖、非标准 Triton 方言适配、生产输入泛化 |
| 输入 | ATen docstring/signature、算子集合、示例、编译/执行反馈 |
| 输出 | Triton MTIA kernel-wrapper pair |
| 核心方法 | FSM 编排的 LLM 生成、linter、JIT、测试、debugger 和反馈摘要 |
| 使用的模型 | CWM、GPT-OSS 120B、Claude Sonnet 4.5、Claude Opus 4.5；Llama-4-Maverick 做反馈摘要 |
| 使用的编译器工具 | Triton MTIA JIT、custom linter、LLDB debugger、QEMU simulator |
| 是否使用强化学习 | 否；采用推理时多轮反馈，不涉及 RL 训练 |
| 是否使用形式化验证 | 否；使用 OpInfo、CPU reference 和生产输入测试，不是形式化证明 |
| 数据集规模 | 568 个 MTIA-compatible operators；20,000+ OpInfo tests；生产模型输入另行采集 |
| 主要指标 | operator coverage、model-input coverage、性能相对比、LLM calls |
| 最重要实验结果 | 聚合 coverage 84.0%；481+ operators 通过全部对应测试；模型输入 coverage 约 79.8%–87.2% |
| 核心创新 | coverage-first ASIC backend generation 与受约束 FSM harness |
| 主要局限 | 依赖 MTIA/OpInfo 生态；性能调优不在范围；测试不等于形式化证明 |
| 与 RISC-V 研究的相关性 | 中等：反馈驱动代码生成可迁移，但 RVV/LLVM/ISA 语义需重新设计 |
| 最适合作为 | Translator/T4 baseline、后端生成工具框架和安全测试流程参考 |

> 这篇论文最值得学习的是把编译器、linter、调试器和测试反馈组织成可扩展的 kernel 生成状态机；最主要的局限是 coverage 依赖特定硬件和测试生态，且尚未解决性能调优。用于后续研究时，最合理的方式是借鉴其反馈与门禁机制并加入 RVV/LLVM 语义或跨代迁移问题，而不是简单替换目标硬件。
