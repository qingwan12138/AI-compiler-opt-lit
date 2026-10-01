# WhiteFox 文献阅读总结

论文题目：**WhiteFox: White-Box Compiler Fuzzing Empowered by Large Language Models**<br>
作者：Chenyuan Yang、Yinlin Deng、Runyu Lu、Jiayi Yao、Jiawei Liu、Reyhaneh Jabbarvand、Lingming Zhang<br>
发表时间：2024<br>
发表平台：Proceedings of the ACM on Programming Languages, Volume 8, OOPSLA2, Article 296<br>
论文链接或编号：DOI `10.1145/3689736`；arXiv `2310.15991`；正式 PDF：[OOPSLA24-WhiteFox.pdf](https://yangchenyuan.github.io/files/OOPSLA24-WhiteFox.pdf)<br>
关键词：白盒编译器模糊测试、LLM、深度学习编译器、优化触发、测试生成、反馈循环、Thompson Sampling

> 本文档仅依据阶段2下载并解析的 27 页正式 PACMPL PDF 编写。论文事实、阅读后的分析和后续建议分开记录。WhiteFox 的最终 LM 输出是面向编译器测试的测试程序与可复用 fuzzing 流程，因此建议分类为 `GENERATOR / G4_Tool_Test_Fuzz_Generation`。

## 1. 研究背景

编译器把高级程序转换为高效机器代码，错误优化可能导致错误结果甚至安全问题。传统编译器 fuzzing 通常分为黑盒、灰盒和白盒路线：黑盒方法不使用实现内部信息，灰盒方法用覆盖率等反馈指导输入，白盒方法尝试分析内部路径。论文指出，随机或覆盖率导向的生成往往难以满足触发深层优化所需的精确条件；传统符号执行面对大型编译器代码库又会遇到路径爆炸和工程规模问题。

深度学习编译器进一步增加了难度。其优化实现可能跨越 Python、C++ 和后端库，输入还必须同时满足编程语言约束和张量/算子组合约束。论文以 PyTorch Inductor 的 `permute_linear_fusion` 为例：只有输入模型满足一组嵌套条件时，相关优化才会执行。论文的出发点是让语言模型读取优化实现，把低层实现中的触发条件转换成高层测试程序可用的需求，再用执行反馈持续改进生成。

## 2. 论文要解决的问题

### 2.1 深层优化难以被常规 fuzzing 触发

黑盒和灰盒方法通常生成大量输入，但缺少对优化实现逻辑的理解，难以构造同时满足多个严格条件的测试程序。论文希望提高被实际触发的优化数量，而不只追求前端崩溃或一般代码覆盖。

### 2.2 编译器源代码与测试输入之间存在表示鸿沟

优化实现使用低层 IR、内部对象和辅助函数，而测试输入通常是用户可见的 Python 模型或其他高层程序。直接把实现源代码交给生成模型会带来冗余信息、内部格式差异和上下文长度限制，论文因此引入中间的需求摘要。

### 2.3 触发样例如何用于后续生成

即使模型已经生成一个能触发优化的输入，不同触发样例的价值也可能不同。论文使用反馈循环和 Thompson Sampling 选择少量触发样例作为后续 few-shot 示例，以增加继续触发目标优化的概率。

> 本文主要研究：如何利用编译器优化实现的源代码和运行反馈，自动生成能够触发深层优化的高质量测试程序，并用这些程序发现优化错误。

## 3. 核心方法概述

WhiteFox 是一个面向深度学习编译器的白盒 fuzzing 框架。它首先收集目标编译器中的优化实现，并由分析 LLM 将实现转换为自然语言与伪代码混合的触发需求；随后生成 LLM 依据需求生成高层测试程序；测试程序经过编译、执行和优化触发检测后，成功触发的样例被加入反馈池，供下一轮 few-shot 生成使用。

```text
目标编译器的优化实现源码
        ↓
收集优化函数并插入触发日志
        ↓
分析 LLM：源码 → 高层输入的触发需求
        ↓
生成 LLM：需求 + few-shot 示例 → 测试程序
        ↓
编译、执行、优化日志与测试 oracle
        ↓
筛选触发优化的程序并发现 bug
        ↓
Thompson Sampling 选择反馈示例
        ↺ 进入下一轮生成
```

分析 LLM 的输出是可读的触发需求，生成 LLM 的输出是具体测试程序；这使最终系统产出成为可复用的编译器测试生成能力，属于 Generator/G4。论文还用差分执行和异常检查来识别 bug：测试程序以有优化和无优化模式编译/执行，并关注崩溃、错误结果、错误优化以及错误通过的优化等情况。

## 4. 实验框架与训练流程

### 4.1 优化收集与插桩

作者指定目标编译器中可能包含优化的目录，再用关键词如 `fusion` 或 `fuse` 搜索相关函数，并补充其调用的辅助函数。随后在优化函数入口插入日志，以便从测试执行中判断某个优化是否被调用。TensorFlow-XLA 的优化实现较长，实验只选择少于 400 行的实现，以适应 LLM 上下文窗口。

### 4.2 需求摘要生成

对于每个优化，分析 LLM 读取实现源码和少量人工构造的示例，输出自然语言和伪代码混合的触发条件。论文说明，混合表示同时保留语义解释和结构约束；直接把实现源码交给生成 LLM 的 `WF-Impl` 变体效果较差。

### 4.3 测试程序生成

生成 LLM 接收目标输入格式、触发需求和示例，输出符合公开 API 的测试程序。PyTorch Inductor 的目标输入是 PyTorch 模型；TensorFlow Lite 和 TensorFlow-XLA 的测试输入也以高层 Python 模型为主。默认设置为每个优化生成 1000 个测试，分成 100 轮，每轮 10 个。

### 4.4 反馈与运行时验证

若测试程序触发了目标优化，系统把它加入反馈候选池。默认使用 Thompson Sampling 选择 3 个触发样例作为下一轮 few-shot 示例；若尚未触发，则继续使用初始 few-shot 提示。测试程序由目标编译器执行，并依据优化日志、编译错误、运行错误和差分结果判断触发与 bug。

### 4.5 训练流程判断

本文不涉及模型预训练、SFT、PPO 或 GRPO。核心过程是提示词驱动的推理、编译器插桩、测试执行和在线 few-shot 反馈。论文使用 GPT-4 作为分析 LLM、StarCoder 作为生成 LLM；代码仓库提供了 GPT-4 请求和 StarCoder 本地生成脚本。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数或模型参数训练损失。反馈选择使用 Beta 分布和 Thompson Sampling。论文伪代码中，对每个示例维护 `alpha` 与 `beta`：触发成功次数增加 `alpha`，未触发次数增加 `beta`，然后从各示例的 Beta 分布采样并选择候选示例。

可将其概括为：

```text
触发成功：alpha_example ← alpha_example + NumTrigger
未触发：  beta_example  ← beta_example  + NumNotTrigger
选择：从 Beta(alpha_example, beta_example) 采样并选取高样本值示例
```

这里的反馈目标是选择更可能帮助后续生成触发优化的示例，不是训练 LLM 的奖励优化。论文没有把该选择过程定义成 PPO 或其他策略梯度算法。

## 6. 实验设置

### 6.1 数据集来源

论文没有使用一个独立的传统训练数据集。输入来源主要包括目标编译器优化实现、人工准备的少量 few-shot 需求/测试示例，以及运行过程中产生的触发测试。测试生成结果在 fuzzing 过程中动态累积。论文没有报告固定的训练集、验证集和测试集划分，也没有给出独立数据泄漏审计。

### 6.2 模型与工具

| 项目 | 论文设置 |
|---|---|
| 分析模型 | GPT-4 |
| 生成模型 | StarCoder；代码仓库支持本地或服务模式 |
| 目标 | PyTorch Inductor、TensorFlow Lite、TensorFlow-XLA |
| 优化数量 | 61、13、49 |
| 源语言 | PyTorch Inductor 为 Python；其余两个主要为 C++ |
| 测试语言 | Python |
| 编译器版本 | PyTorch nightly 20230509；TensorFlow nightly 20230507 |
| 环境 | Ubuntu 20.04.5、64 核 CPU、256 GB RAM、NVIDIA RTX A6000 |
| 覆盖率工具 | Python 使用 Coverage.py；C++ 使用 GCOV |
| 其他 | 优化函数入口日志、差分执行、异常/结果检查 |

### 6.3 对比方法

- TitanFuzz：LLM 驱动的深度学习库 fuzzing 方法。
- NNSmith：基于符号规则的有效深度学习编译器测试生成器。
- WhiteFox-Mini：WhiteFox 的缩小预算版本，用于和较快的 NNSmith 比较。
- 消融变体：WF-Mix、WF-NL、WF-Code、WF-Impl、WF-No-Feedback、WF-Naive、WF-SC。

### 6.4 评价指标

| 指标 | 含义 | 方向 |
|---|---|---|
| 触发优化数量 | 日志显示被实际执行的不同优化数量 | 越大越好 |
| 触发测试数量 | 至少触发一个目标优化的测试数量 | 越大越好 |
| 测试数量/时间 | 生成并执行的测试量及耗时 | 需结合预算解释 |
| 行覆盖率 | 优化实现所在 Python/C++ 源码的行覆盖 | 通常越大越好，但只是代理指标 |
| 发现 bug 数 | 由崩溃、错误结果或错误优化等 oracle 发现的 bug 数 | 越大越好 |

## 7. 实验结果与结论

### 7.1 主要结果

默认预算下，PyTorch Inductor 的 61 个优化中，WhiteFox 触发 41 个；TensorFlow Lite 的 13 个中触发 12 个；TensorFlow-XLA 的 49 个中触发 20 个。对应触发测试数分别为 21,469、2,801 和 12,990。论文将 WhiteFox 与基线比较后报告，触发优化数量最高可达到基线的 8.2 倍。

24 小时比较中，WhiteFox 在 PyTorch Inductor 触发 41 个优化、TensorFlow Lite 触发 12 个、TensorFlow-XLA 触发 19 个。行覆盖率方面，WhiteFox 在 PyTorch Inductor 和 TensorFlow-XLA 高于基线，论文报告最高约 19.9% 的差异；在 TensorFlow Lite 上略低于 TitanFuzz 约 5.9%。论文提醒覆盖率并不总是与复杂系统的 bug 发现能力强相关。

### 7.2 与传统方法的比较

NNSmith 在 TensorFlow-XLA 上产生大量触发测试，主要因为其实现会重复生成容易触发 `IdentityReshapeRemoving` 的 reshape；这提高了触发测试数量，却不等价于覆盖更多不同优化。WhiteFox 在 PyTorch Inductor 和 TensorFlow Lite 触发了基线未覆盖的额外优化，并且通过白盒需求把测试生成指向实现中的深层条件。

### 7.3 与其他 LLM 方法的比较

TitanFuzz 能覆盖常见模型模式，但 WhiteFox 直接使用优化源码信息，因此在 PyTorch Inductor 和 TensorFlow Lite 上触发更多优化。TensorFlow-XLA 的部分优化较简单，TitanFuzz 在该目标上触发 22 个，略高于 WhiteFox 的 20 个；WhiteFox 仍触发了 4 个 TitanFuzz 未触发的独特优化。

### 7.4 消融实验

在 PyTorch Inductor 上，WF-Mix 触发 39 个优化、1,113 个触发测试；WF-NL 为 37 个和 940 个；WF-Code 为 32 个和 1,055 个；WF-Impl 为 32 个和 638 个；WF-SC 为 32 个和 745 个。论文据此认为自然语言与伪代码混合需求优于直接使用实现源码。

反馈消融中，WhiteFox 产生 21,469 个触发测试，WF-Naive 为 17,004 个，WF-No-Feedback 为 8,152 个；WhiteFox 的覆盖率为 55,857，WF-Naive 为 54,602，WF-No-Feedback 为 52,838。论文报告 Thompson Sampling 相比均匀随机选择使触发测试数约提高 1.3 倍。

### 7.5 案例与 bug 结果

WhiteFox 共报告发现 101 个 bug：PyTorch 79 个、TensorFlow Lite 11 个、TensorFlow-XLA 11 个；其中 95 个已确认，92 个被确认是此前未知，70 个已修复。PyTorch 的 79 个中有 14 个来自最新版本测试，10 个被开发者标记为高优先级。论文还报告 68 个 WhiteFox bug 基线无法发现。

## 8. 主要创新点

### 8.1 创新点一：从优化源码到高层触发需求

论文把低层优化实现和高层测试输入之间的鸿沟显式化，先让分析 LLM 生成需求，再让生成 LLM 生成程序。实验中 WF-Mix 比直接把实现源码交给生成模型的 WF-Impl 多触发 7 个优化，说明中间需求表示具有实际作用。

### 8.2 创新点二：面向优化触发的白盒 LLM fuzzing

WhiteFox 的白盒信息不是只用于静态路径探索，而是转化为模型可用的测试约束，直接围绕每个优化收集触发样例。它因此把评价重点从一般覆盖扩展到“多少不同优化被真正执行”。

### 8.3 创新点三：基于 Thompson Sampling 的反馈示例选择

系统把触发样例视为不同质量的 few-shot 候选，并根据历史触发反馈选择示例。相较无反馈和均匀随机变体，默认反馈循环产生更多触发测试和更高覆盖率。

### 8.4 工程实现与研究贡献的边界

GPT-4、StarCoder、日志插桩、Coverage.py 和 GCOV 是实现组件，本身不是独立创新。真正的研究贡献是把优化源码摘要、测试生成和触发反馈组织成可扩展的白盒编译器测试框架。

## 9. 局限性

### 9.1 论文明确承认或实验体现的局限

- 目标主要是三个深度学习编译器；TensorFlow-XLA 只选少于 400 行的优化实现，不能代表其全部优化。
- 需求收集依赖目录指定和关键词搜索，仍需要研究者识别优化相关函数及其辅助函数。
- WhiteFox 在 TensorFlow Lite 上的覆盖略低于 TitanFuzz，论文将其与优化数量少、可利用白盒信息有限联系起来。
- 代码覆盖率只是代理指标，不保证与 bug 发现能力强相关。
- GPT-4 和 StarCoder 的调用成本、模型版本细节及服务稳定性会影响复现，论文没有给出所有 API 参数。

### 9.2 阅读后的潜在局限

- 论文的差分 oracle 主要比较优化开关和异常/结果行为；这不是对所有生成测试程序的形式化等价证明。
- 白盒需求依赖可读的优化实现和目标输入映射；迁移到 LLVM/RISC-V 时，需要重新建立 IR 到源级输入的对应关系。
- 论文没有单独构造跨版本、跨架构的稳定 benchmark，因此当前结果不能直接推断到 RISC-V 后端。

## 10. 阅读后的研究方向反思

WhiteFox 最适合作为 Generator/G4 的方法参考和编译器测试工具模块。它对 LLVM/RISC-V 的启发是：可以让模型读取目标 pass、指令选择或后端合法性检查的实现，把实现条件转化为 IR/源程序测试约束，再通过编译反馈迭代生成。仅把目标编译器替换为 RISC-V LLVM 后端并不足以构成新贡献；需要处理 RVV 类型约束、指令选择路径、汇编合法性和跨编译器差分 oracle 等新的问题。

论文的核心贡献是白盒需求摘要与反馈生成的组合，后续工作应把它当作 baseline 或模块，不应将 WhiteFox 的 101 个 bug 结果直接当成其他架构可复现的保证。与 Selector 类方法相比，WhiteFox 输出的是测试程序和 fuzzing 能力；与 Translator 类方法相比，它不负责把一个给定程序翻译成目标 ISA。

## 11. 可进一步尝试的研究方向

### 11.1 RVV 后端优化触发测试生成

#### 研究问题

能否从 LLVM RVV 指令选择、向量化和合法化代码中生成能稳定触发目标后端路径的 LLVM IR 测试？

#### 与原论文的区别

目标从深度学习编译器高层模型改为 LLVM IR/RISC-V RVV 后端；输入条件还包括向量长度、LMUL、数据类型和 ABI 约束。

#### 可能的创新点

构造后端实现到 IR 约束的映射，并将编译器诊断、汇编反汇编和 QEMU/硬件差分结果纳入反馈。

#### 实验框架

```text
LLVM RVV pass/ISel 源码 → LLM 需求摘要 → LLVM IR 生成
→ llc/clang 编译 → 汇编与执行检查 → 触发反馈 → 下一轮生成
```

#### 可行性

需要 LLVM、RISC-V 交叉工具链、QEMU 或真实 RVV 设备、少量人工示例和本地代码模型。

#### 主要风险

静态触发不等于硬件语义正确；QEMU 与真实硬件差异、编译器版本变化和后端日志粒度可能影响复现。

### 11.2 面向 MLIR 方言的需求摘要与验证

#### 研究问题

能否将 WhiteFox 的低层实现摘要机制迁移到多个 MLIR dialect，并自动生成满足类型和 region 约束的 IR？

#### 与原论文的区别

MLIR dialect 的操作、类型和 region 约束由可扩展 IR 定义，输入不是单一深度学习框架的高层模型。

#### 可能的创新点

把 dialect verifier、canonicalizer 和 lowering 反馈组织成多级触发信号，并区分“通过 verifier”和“进入目标 lowering”两种成功条件。

#### 实验框架

```text
dialect/rewriter 源码 → LLM 生成操作需求 → MLIR module 生成
→ verifier/lowering 执行 → coverage/diagnostic 反馈 → 约束更新
```

#### 可行性

需要 MLIR、目标 dialect 测试集、verifier/lowering 日志和可运行的代码模型。

#### 主要风险

复杂 dialect 的合法性条件可能超出上下文窗口；仅通过 verifier 不能说明下游 lowering 或硬件执行正确。

## 12. 与其他已读文献的关系

- 与 C138 LegoFuzz 相同点：都让 LLM 参与编译器测试程序生成。LegoFuzz 重点是离线生成代码块并在线组合，WhiteFox 重点是读取优化实现并针对目标优化生成测试。
- 与 C180 ReFuzzer 的关系：ReFuzzer 对已有 LLM 测试程序做编译器/ sanitizer 反馈修复；WhiteFox 在初始生成阶段利用优化源码和触发反馈。二者可以组合，但 WhiteFox 的输出角色仍是测试生成器。
- 与 C152 Mut4All、C175 MetaMut 的关系：Mut4All/MetaMut 生成可复用 mutator；WhiteFox 直接生成优化触发测试程序并维护反馈示例，输出粒度不同。
- 与 C156 FeatureFuzz 的关系：FeatureFuzz 从 bug 报告提取语义 feature 并组合程序；WhiteFox 从优化实现提取触发需求。二者都把高层语义约束显式化，但信息来源不同。
- 与 Selector 类工作不同：WhiteFox 不选择 pass 顺序或编译配置，而是生成用于测试现有编译器能力的程序。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 面向深度学习编译器优化的白盒 LLM fuzzing |
| 核心问题 | 常规 fuzzing 难以触发需要复杂条件的深层优化 |
| 输入 | 优化实现源码、人工 few-shot 示例、运行反馈 |
| 输出 | 高层测试程序、可复用的优化触发测试生成能力 |
| 核心方法 | 分析 LLM 生成需求，生成 LLM 生成测试，Thompson Sampling 选反馈示例 |
| 使用的模型 | GPT-4 分析；StarCoder 生成 |
| 使用的编译器工具 | PyTorch Inductor、TensorFlow Lite、TensorFlow-XLA、Coverage.py、GCOV |
| 是否使用强化学习 | 否；使用 Thompson Sampling 进行示例选择 |
| 是否使用形式化验证 | 否；使用运行时/差分 oracle，不是形式化证明 |
| 数据集规模 | 无固定训练集；默认每个优化生成 1000 个测试 |
| 主要指标 | 触发优化数、触发测试数、行覆盖率、bug 数 |
| 最重要实验结果 | 触发 101 个 bug，其中 92 个此前未知、70 个已修复；默认设置下最多触发 41/61 个 PyTorch Inductor 优化 |
| 核心创新 | 优化源码到高层触发需求的双模型白盒生成与反馈循环 |
| 主要局限 | 目标编译器有限，依赖源码映射和昂贵模型调用，覆盖率不是正确性证明 |
| 与 RISC-V 研究的相关性 | 中；方法可迁移到 RVV/LLVM 后端，但需要新的 IR、ISA 和执行 oracle |
| 最适合作为 | Generator/G4 方法参考、编译器测试工具模块、RISC-V 后端测试 baseline |

这篇论文最值得学习的是把编译器内部优化逻辑转换为模型可用的高层测试约束，并把真实触发样例用于下一轮生成；最主要的局限是其验证对象集中在三个深度学习编译器，且运行反馈不等于形式化正确性证明。如果用于后续研究，合理方式是把它作为白盒测试生成 baseline，再针对 LLVM IR、RISC-V/RVV 后端或 MLIR dialect 增加新的约束和 oracle，而不是只更换目标平台。
