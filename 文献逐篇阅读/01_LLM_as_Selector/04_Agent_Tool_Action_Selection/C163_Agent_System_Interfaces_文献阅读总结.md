# Agent-System Interface 文献阅读总结

论文题目：**Improving Parallel Program Performance with LLM Optimizers via Agent-System Interfaces**

作者：Anjiang Wei、Allen Nie、Thiago S. F. X. Teixeira、Rohan Yadav、Wonchan Lee、Ke Wang、Alex Aiken

发表时间：2025

发表平台：ICML 2025, Proceedings of Machine Learning Research 267, pp.66155–66177

论文链接或编号：arXiv:2501.01589
元数据核验来源：[PMLR 正式论文页](https://proceedings.mlr.press/v267/wei25j.html)；[PMLR 官方 PDF](https://raw.githubusercontent.com/mlresearch/v267/main/assets/wei25j.pdf)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：LLM agent、generative optimization、parallel programming、Legion mapper、DSL、AutoGuide、runtime feedback、mapping strategy

> 本笔记基于本轮 staging 中保存的 23 页正式 PDF 全文。论文事实与阅读后的分析分开记录。按本轮 SELECTOR/S4 口径，重点关注 agent 对既有映射动作与策略的选择，以及外部编译器/运行时的执行反馈；该论文同时让 LLM 生成 DSL mapper，因此这是一个需要明确边界的 SELECTOR 候选。

## 1. 研究背景

本文研究任务型并行程序中的 mapper 自动开发。Legion 等系统把程序计算拆成任务，mapper 决定任务放在哪类处理器上、数据放在哪类内存上，以及数据分区和任务索引如何映射到处理器。高质量 mapper 会显著影响通信、内存访问、GPU 启动和端到端吞吐。

论文第 1 节指出，手写高性能 mapper 需要熟悉应用、硬件和底层系统 API，通常要经过数天人工调优；而搜索空间又随任务、数据参数和处理器数量快速膨胀。传统自动调优方法通常依赖数值性能反馈，难以利用错误解释、系统启发式和可读的修改建议。作者因此引入 LLM agent、结构化 DSL 和自然语言反馈，把 mapper 生成与搜索结合起来。

## 2. 论文要解决的问题

### 2.1 降低 mapper 代码生成的系统复杂度

原始 C++ mapper 暴露大量底层 API 与实现细节，LLM 生成时容易编译失败或产生不能执行的映射。论文要解决的是如何提供更适合 LLM 的高层表示，同时保留关键映射决策能力（第 3、4.1 节）。

### 2.2 在离散的大搜索空间中高效寻找 mapper

任务处理器选择、数据内存放置、布局、实例限制和索引映射共同构成离散搜索空间。论文要解决如何利用应用元数据、服务器规格和执行反馈，在较少迭代中找到高吞吐 mapper（第 3、4.2 节）。

### 2.3 让反馈真正能指导下一次动作

运行时原始错误往往只有 assertion、非法参数或执行时间，不能直接告诉 agent 应怎样修改。论文设计 AutoGuide，把原始输出转换成错误解释和 mapper 修改建议。

> 本文主要研究：如何用 Agent-System Interface（DSL + AutoGuide）把 LLM agent 接入 Legion mapper 的策略搜索，并通过运行时反馈迭代选择更高性能的映射方案。

## 3. 核心方法概述

论文提出 Agent-System Interface（ASI），由两部分组成：

1. Domain-Specific Language（DSL，领域专用语言）：以声明式语句描述任务到处理器、数据到内存、布局约束和索引映射；其 compiler 再把 DSL 翻译成底层 C++ API。
2. AutoGuide：对运行时输出做关键词匹配，生成错误解释和修改建议，例如把 stride assertion 解释为布局问题，或建议把任务移到 GPU。

整体数据流：

```text
服务器规格 + 应用任务/数据元数据
          ↓
定义 DSL 中的结构化 mapper 决策空间
          ↓
LLM agent 按模块生成 DSL mapper
          ↓
DSL compiler 翻译为 Legion/C++ mapper
          ↓
与应用一起编译、执行和测试
          ↓
获得吞吐、执行时间或错误输出
          ↓
AutoGuide 生成解释/建议
          ↓
Trace optimizer 更新可训练决策模块
          ↺ 重复若干轮，保留最佳 mapper
```

LLM 的主要作用是根据硬件与应用上下文生成和修改映射策略。外部 compiler、Legion runtime 和应用执行真实的映射、正确性检查与性能测量。Trace 把 mapper 拆成 task、region、layout、instance-limit、index-task-map 和 single-task-map 等相对独立的决策模块；LLM 更新这些模块，而不是直接修改底层 Legion 实现。

与传统 RL/autotuning 相比，本文把文本形式的程序策略、错误解释和建议作为反馈，不把所有信息压缩为单一标量 reward。按 SELECTOR/S4 观察，最终搜索行为是选择既有映射动作及动作组合；但由于 LLM 输出的是 DSL mapper 文本，而非单独的离散 action token，分类边界应在正式入库时人工复核。

## 4. 实验框架与训练流程

### 4.1 初始化与搜索空间

输入包括服务器规格（CPU/GPU 数量、每节点配置、节点数）和应用元数据（任务名、任务访问的数据参数）。DSL 预先限定可表达的映射维度。论文未采用预训练或专项 SFT 来学习 DSL；文中称 DSL 没有训练语料中的示例。

### 4.2 agent 生成与迭代更新

Trace 中的 `MapperAgent` 以多个 `@bundle(trainable=True)` 模块生成 mapper。每轮先根据应用生成 mapper，再由 evaluator 执行应用并返回性能或异常。若出现 `TraceExecutionError`，错误节点与反馈被送入 optimizer；optimizer 通过 `backward` 和 `step` 更新相应决策模块（附录 A.6）。

### 4.3 反馈增强与停止

AutoGuide 对原始执行输出添加 explain/suggest 两类自然语言内容。主实验每个应用运行 10 次迭代，并重复 5 次处理随机性；最终报告 Trace 的平均轨迹与跨运行找到的最佳 mapper。论文还将 OpenTuner 延长到 1000 次迭代进行比较。论文没有使用 PPO、GRPO 或其他模型参数强化学习训练；OpenTuner 是对照方法，Trace 的更新是基于执行图和文本反馈的生成式优化。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数，也没有报告用于更新 LLM 参数的传统数值损失。论文把 mapper 优化形式化为三元组 `(Theta, omega, T)`：`Theta` 是候选 mapper 集合，`omega` 是文本形式的优化目标，`T(theta)` 执行 mapper 并返回性能反馈 `f` 与生成过程图 `g`（第 4.2 节）。

可概括为：

```text
(f, g) = T(theta)
```

| 符号 | 论文中的含义 |
|---|---|
| `Theta` | 所有可能 mapper 构成的候选集合 |
| `theta` | 一个具体的 mapper 程序 |
| `omega` | 以文本表达的优化目标 |
| `f` | 执行 mapper 后得到的性能反馈 |
| `g` | mapper 生成过程的 trace graph |

主实验指标是相对于 expert-written mapper 归一化的 throughput；语法或运行时错误的 mapper throughput 记为 0。这个指标用于评估和搜索，不等同于论文提出了一个显式 reward shaping 公式。运行环境被作者控制为尽可能确定性，但附录显示非法配置仍可能导致失败和较高方差。

## 6. 实验设置

### 6.1 数据集来源

论文没有使用传统机器学习意义上的训练集、验证集和测试集，而是直接在 9 个并行程序 benchmark 上进行在线/离线 mapper 搜索：Circuit、Stencil、Pennant，以及 Cannon’s、SUMMA、PUMMA、Johnson’s、Solomonik’s、COSMA 六个矩阵乘法算法。作者还设计了 10 个自然语言描述的 mapping strategies，用于 DSL 与 C++ 代码生成消融。

论文未报告统一的训练/验证/测试规模，也没有把这些 benchmark 称为一个可独立发布的数据集。数据泄漏风险未明确讨论；由于 mapper 依赖固定应用、输入和硬件，论文明确把找到的最佳 mapper 视为可复用的部署结果。

### 6.2 模型与工具

- LLM：`gpt-4o-2024-08-06`。
- agent/优化框架：Trace；代码生成评估使用 DSPy。
- 编译/运行：DSL compiler、Legion mapper/runtime、应用执行环境。
- RL 对照：OpenTuner，以执行时间作为标量反馈，并对失败施加高惩罚。
- 硬件：一台节点，两个 Intel 10-core E5-2640 v4 CPU、256G 主存、四块 NVIDIA Tesla P100 GPU。

### 6.3 对比方法

对比包括 expert-written mappers、随机生成 mapper、Trace、OPRO 和 OpenTuner。Trace/OPRO 的主实验为每应用 10 次迭代、重复 5 次；随机 mapper 使用 10 个随机种子。

### 6.4 评价指标

主要指标是 normalized throughput，基准为 expert-written mapper；同时报告执行正确性、代码生成成功率、平均/标准差/最差/中位数/最佳 normalized throughput，以及搜索迭代数和调优时间。throughput 是端到端应用性能，不是静态 cost model 预测值。

## 7. 实验结果与分析

### 7.1 性能主结果

根据第 5.1 节图 4 和附录表 A1，Trace 找到的最佳 mapper 在 9 个 benchmark 上都能匹配或超过 expert-written mapper；相对于 expert mapper，Circuit 的最高 speedup 为 1.34×，COSMA 为 1.31×。Trace 平均 normalized throughput 为：Circuit 1.33×、Stencil 1.01×、Pennant 1.03×、Cannon 1.09×、SUMMA 0.86×、PUMMA 0.57×、Johnson 0.98×、Solomonik 0.52×、COSMA 1.25×；这些是 5 次运行的均值，不是每个 benchmark 的最佳值。

Trace 与 OpenTuner 的平均轨迹比较显示：10 次迭代限制下，Trace 相对 OpenTuner 的优势约为 11×；将 OpenTuner 扩展至 1000 次迭代后，Trace 仍有 3.8× 优势（图 5，跨 9 个 benchmark 的平均相对 throughput）。论文报告 Trace 每个应用约 10 分钟完成整个优化流程，把人工数天调优降到分钟级。

### 7.2 正确性与失败

随机 mapper 中 90 个样本有 74 个（82.2%）因无效映射决策触发运行时错误。附录表 A2 显示，OpenTuner 在相同 10 次迭代预算下对 9 个 benchmark 中 8 个没有产生有效 mapper。Trace 在 SUMMA、PUMMA、Solomonik 上仍可能出现 0.00× 最差结果，原因包括违反 cuBLAS layout 约束；作者建议通过多次运行并选择最佳结果缓解。

### 7.3 DSL 代码生成消融

第 5.2 节表 2 使用 10 个自然语言 mapping strategies。相同提示材料下，C++ 单次生成成功率为 0%，DSL 单次为 80%；C++ 允许迭代 refinement 后仍为 0%，DSL 迭代后为 100%。成功定义为代码能编译并通过预定义执行测试，不是形式化语义等价证明。DSL 平均减少约 14× 行代码；作者将差距归因于声明式表达与较小的自然语言到代码语义间隔。

### 7.4 AutoGuide 消融

第 5.3 节在 3 个 benchmark 上比较 0-shot、5-shot、仅执行原始输出、Execution + Explain、完整 Execution + Explain + Suggest。完整 AutoGuide 配置始终优于删减版本；0-shot 和 5-shot 最差，说明迭代执行反馈而非单纯增加示例是关键。AutoGuide 本身用关键词匹配实现，并非学习型错误诊断器。

## 8. 论文创新点

以下是论文正文明确提出或实验直接支持的创新：

1. 设计 ASI，把高层 DSL 和反馈解释层放在 LLM agent 与低层并行系统之间。
2. 用声明式 DSL 表达处理器选择、内存放置、布局和索引映射，既缩短代码又结构化搜索空间。
3. 用 AutoGuide 将运行时错误和性能输出转换为可执行的自然语言解释/建议。
4. 将 generative optimization 引入系统 mapper 优化，展示丰富文本反馈相对于标量 RL autotuning 的搜索效率优势。
5. 在 9 个 benchmark、DSL 生成消融和反馈消融上给出端到端证据。

这些创新不应扩写为通用 LLVM pass selector、RISC-V 优化器或形式化验证系统；论文实验对象是 Legion mapper 与固定 CPU/GPU 节点。

## 9. 局限性与失败边界

### 9.1 论文明确暴露的局限

- DSL 受表达能力约束。作者称尚未遇到无法表达的优化，但没有证明 DSL 覆盖所有 Legion mapper 或所有并行系统。
- AutoGuide 依赖关键词匹配，面对未覆盖的错误类型或复杂性能瓶颈时可能不能给出有效建议。
- 10 次迭代内仍可能生成非法 layout 或 index mapping；附录报告多个 benchmark 的最差 throughput 为 0。
- 最佳结果依赖重复运行和选择 best mapper，平均性能和最差性能并不总是稳定。
- 实验只覆盖一个节点配置、四块 P100、Legion 任务型程序和 9 个 benchmark，跨硬件、跨输入和跨应用泛化未验证。
- 论文没有使用形式化验证；正确性主要由编译、预定义测试和 runtime rejection 检查。

### 9.2 论文中未明确说明的事项

训练/验证/测试数据集划分、LLM token 成本、每轮调用次数、完整 prompt、系统性数据泄漏审计、不同 LLM 的敏感性和跨版本 compiler 兼容性，论文中未明确说明。

## 10. 对后续研究方向的启发（阅读后的分析）

### 10.1 对 SELECTOR/S3/S4 的启发

可以把 compiler action 设计成结构化、可验证的 DSL 或 schema，让 LLM 选择动作参数，而不是自由生成底层实现。动作执行后应保留原始诊断、性能计数器和结构化 trace，再生成给 agent 使用的自然语言摘要；这样既保留搜索信息，又避免只依赖单一 reward。

### 10.2 对 LLVM/RISC-V 的启发

类似接口可用于 LLVM pass/flag/phase 选择或 RISC-V/RVV 调优，但必须重新定义合法动作空间、目标硬件反馈和验证门。本文没有 RISC-V 实验，因此不能直接声称其 DSL、P100 结果或 Legion 规则能迁移到 RVV。

### 10.3 对失败边界建模的启发

附录中的失败案例适合形成“错误类型 → 可能原因 → 可尝试动作”的可审计记忆表。应同时保留失败率、最差结果和重复运行方差，避免只报告 best-of-N 而掩盖搜索不稳定性。

## 11. 可借鉴内容

| 可借鉴 | 具体做法 | 借鉴边界 |
|---|---|---|
| 结构化 action space | 用 DSL/JSON schema 分离 task、layout、memory、index 等决策 | 需按目标 compiler 重新定义合法性 |
| 丰富反馈 | 将编译错误、运行时错误、性能指标转换为解释和建议，同时保存 raw feedback | AutoGuide 的关键词规则不能直接视为通用诊断模型 |
| agent 模块化 | 为不同动作设独立可更新模块，减少决策间耦合 | 模块独立性需由目标 IR/编译器依赖验证 |
| best-of-N 与稳定性并报 | 同时报告均值、方差、最差、中位数、最佳 | 不能只用最佳结果代表部署风险 |
| 端到端验证 | 让 compiler/runtime/test gate 参与每轮候选筛选 | 有限测试不等于形式化证明 |

## 12. 不可直接照搬的内容

1. 不能把 OpenTuner 的失败或 Trace 的 3.8×/11×优势直接外推到 LLVM、GCC、MLIR 或 RISC-V；论文比较的是特定 Legion mapper 搜索空间。
2. 不能把 DSL 生成成功率 100% 写成语义正确率或形式化等价率；它表示通过预定义编译和执行测试。
3. 不能把 AutoGuide 的关键词匹配当作 RL reward、可学习 policy 或因果归因模型。
4. 不能忽略非法配置导致的 0.00× 最差结果，也不能只复现 best-of-5 而不报告重复运行统计。
5. 不能直接采用 P100/Legion 的硬件启发式指导 RVV、GPU 或其他加速器；必须重新测量硬件效应。
6. 不能把本文称为 SFT/PPO/GRPO 训练论文；论文使用现成 GPT-4o 和 Trace 生成式优化，未报告模型参数强化学习训练。

## 13. 总结

这篇 ICML 2025 论文展示了一个清晰的 SELECTOR/S4 agent-tool 交互模式：LLM 在结构化 DSL 中选择并迭代修改并行 mapper 策略，compiler 和 runtime 执行候选并提供反馈，AutoGuide 把低层错误转成 agent 可用的自然语言。正文实验证明，在固定 Legion/CPU/GPU 环境中，Trace 在 10 次迭代内可以找到匹配或超过专家 mapper 的方案，并以较少迭代优于 OpenTuner。

最值得学习的是 ASI 的接口设计、反馈分层和失败统计；最重要的限制是实验范围窄、DSL/关键词规则依赖人工设计、搜索仍有非法候选，而且输出 DSL mapper 使其处在 SELECTOR 与直接代码生成的边界。若用于后续 compiler/RISC-V 研究，合理做法是借鉴“结构化动作 + 可执行反馈 + 审计统计”的框架，再针对目标 IR、硬件和正确性门重新设计，而不是直接搬用其 mapper 语法或性能结论。
