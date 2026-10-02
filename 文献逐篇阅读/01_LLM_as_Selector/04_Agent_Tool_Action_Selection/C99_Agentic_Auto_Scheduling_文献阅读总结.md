# Agentic Auto-Scheduling 文献阅读总结

论文题目：**Agentic Auto-Scheduling: An Experimental Study of LLM-Guided Loop Optimization**

作者：Massinissa Merouani、Islem Kara Bernou、Riyadh Baghdadi

发表时间：2025

发表平台：PACT 2025, pp.186–200

论文链接或编号：DOI 10.1109/PACT65351.2025.00027；arXiv:2511.00592
元数据核验来源：[PACT 官方程序](https://pact2025.github.io/program/)；[IEEE DOI](https://doi.org/10.1109/PACT65351.2025.00027)；[Zenodo 工件](https://zenodo.org/records/16812384)
代码/数据/工件：作者工件：[Zenodo 16812384](https://zenodo.org/records/16812384)

研究工件：[PACT 2025 Artifact（Zenodo）](https://zenodo.org/records/16812384)。
来源：[PACT 2025 官方程序](https://pact2025.github.io/program/)；[Artifact 记录及可下载实验包](https://zenodo.org/records/16812384)。
关键词：编译器优化、循环变换、多面体编译、LLM、智能体、Tiramisu、PolyBench、经验反馈

> 本文档基于 staging 中已核验的 19 页 PDF 正文，论文事实、阅读分析和后续建议分开描述。该条目仅为 batch-6/slot-3 交付物，未分配正式 Paper_ID，也未修改 taxonomy 或正式索引。

---

## 1. 研究背景

论文关注复杂循环嵌套的自动性能优化。现代处理器的多级缓存、流水线和并行执行资源相互作用，使人工调优成本高；GCC、LLVM 等传统启发式编译器以及 Pluto 等多面体编译器可以进行融合、交换、分块和并行化，但很难在不同程序和硬件上始终找到高性能方案（第 1 节）。

循环嵌套是科学计算、图像处理和机器学习中的常见瓶颈。已有 LLM 方法大致有两类：直接生成优化代码，或选择编译器 pass/flag。前者需要额外的正确性检查，后者通常难以组合高层源代码循环变换，并且部分工作依赖编译器领域微调或主要优化代码大小。本文因此研究让通用、无需任务专用微调的 LLM 通过编译器反馈探索循环变换。

## 2. 论文要解决的问题

### 2.1 复杂循环变换的搜索问题

给定包含一个或多个循环嵌套的 C/C++ 程序，如何在融合、交换、并行化、分块、展开、错位和反转等变换的组合空间中找到高性能 schedule。

### 2.2 LLM 建议的正确性与可执行性问题

LLM 可能提出语法无效、语义不合理或违反数据依赖的 schedule。论文研究如何把 LLM 的高层探索与编译器的合法性检查、代码生成和真实运行时间反馈结合起来，避免让 LLM 直接承担低层代码生成与正确性保证。

### 2.3 反馈驱动的实际收益问题

论文的中心问题是：在没有任务专用微调的情况下，通用 LLM 是否能在经验编译器反馈的约束下有效指导复杂循环优化；反馈、硬件上下文、初始程序分析、显式推理和多次运行分别有多大作用（第 1、3 节及附录）。

## 3. 核心方法概述

论文提出 COMPILOT（Compiler Pilot），其中 LLM 不直接输出变换后的 C 代码，而是输出 Tiramisu API 可执行的循环变换序列。Tiramisu 使用多面体依赖分析检查合法性，编译并在目标 CPU 上运行候选程序；反馈生成器把无效、非法、求解器失败、编译器崩溃或成功运行及 speedup/slowdown 返回给 LLM。对话历史作为上下文记忆，LLM 据此修改下一轮 schedule。

```text
输入 PolyBench 循环嵌套
        ↓
上下文初始化：标准化 C/C++ 循环、comp_ID、匿名化变量、初始运行时间
        ↓
LLM 分析循环并提出 schedule
        ↓
响应解析与轻量 validity 检查
        ↓
Tiramisu 依赖分析与 legality 检查
        ↓
合法候选编译并在目标 CPU 上运行
        ↓
反馈 validity / legality / 错误类型 / speedup
        ↓
LLM 更新策略，继续迭代或停止
        ↓
输出达到最佳实测性能的程序变体
```

本文中 LLM 的最终系统角色是选择和排序循环变换动作；变换本身由已有编译器后端执行。论文实现使用 Tiramisu，未实现新的多面体变换算法。

## 4. 实验框架与训练流程

本文不涉及模型训练、SFT、PPO、GRPO 或其他参数更新，主要采用提示词推理、编译器工具调用和运行时反馈。

### 4.1 上下文初始化阶段

系统提示词定义任务、输入格式、输出格式、支持的变换、动作空间、目标硬件和崩溃处理。上下文初始化器提取循环嵌套，用 `comp_ID` 标记 computation block，并将循环迭代器和缓冲区名称匿名化。LLM 先分析循环结构、依赖和潜在机会，再进入优化阶段。

### 4.2 迭代优化阶段

LLM 每轮输出带有 reasoning 和 `<schedule>...</schedule>` 的结构化响应。响应解析器做语法、标识符和基本前置条件检查；通过后转换为 Tiramisu API 调用。Tiramisu 对依赖合法性进行检查，必要时使用内部求解器计算 skewing 或 fusion 相关参数，然后编译、运行并测量 speedup。

### 4.3 反馈与停止

反馈分为 Invalid Schedule、Illegal Schedule、Solver Failure、Compiler Crash 和 Successful Execution 五类。成功反馈给出相对于原始程序的运行时间比值。对话在 LLM 发出 `no_further_transformations` 且框架不再要求继续，或达到预设迭代上限时停止。为减轻随机性，论文重复从头运行多个独立对话，并选取 best-of-K 结果。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有训练损失函数。核心性能反馈定义为：

```text
speedup = 原始程序执行时间 / 变换后程序执行时间
```

speedup 大于 1 表示变快，小于 1 表示变慢。论文以 40 次独立运行得到的每个 benchmark 的中位 speedup，再对 150 个实例取几何平均；95% 置信区间来自对每个实例的 40 个 speedup 有放回重采样 1000 次的 bootstrap（附录 B）。

需要注意，speedup 是运行时性能反馈，不是 LLM 的显式可微目标；LLM 通过上下文中的成功/失败信息进行 in-context refinement。论文没有把该过程表述为强化学习，也没有报告模型参数更新。

## 6. 实验设置

### 6.1 数据集来源

使用 PolyBench/C 4.2.1，共 30 个多面体编译 benchmark，覆盖线性代数、stencil 等领域；每个 benchmark 使用五种标准数据规模和默认数据类型，共 150 个 benchmark instances。论文没有报告训练集、验证集或测试集划分，因为本文不训练模型。其主要数据来源是现有 PolyBench 程序及目标机器上的执行测量。

### 6.2 模型与工具

| 项目 | 论文设置 |
| --- | --- |
| 主 LLM | gemini-2.0-flash |
| 对比 LLM | gemma3 (27B)、gpt-4o、llama3.3 (70B)、gpt-o3-mini、qwq (32B)、qwen2.5-coder (32B)、codestral-2501 (22B) |
| 编译器 | Tiramisu，commit `041afad` |
| 后端能力 | 多面体依赖分析、Tiramisu 内部 skewing/shifting 求解器、代码生成 |
| 硬件 | 双路 Intel Xeon E5-2695 v2 @ 2.40GHz；共 48 threads；128GB RAM |
| 主要缓存 | L1d/L1i 总计各 768 KiB，L2 6 MiB，L3 60 MiB |
| 运行预算 | 主结果使用每轮最多 T=30、best-of-K 中 K=5 |

### 6.3 对比方法

主要对比 Pluto 多面体优化器；另在其支持的 8 个 PolyBench benchmark 上比较深度学习 Tiramisu autoscheduler。论文还设置了 COMPILOT 的消融版本：无反馈、直接生成 C 代码、无硬件上下文、移除初始程序分析、移除每轮 reasoning，以及不同 T/K 和继续探索策略。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
| --- | --- | --- |
| speedup | 原始或对比程序运行时间与候选运行时间之比 | 越大越好 |
| 几何平均 speedup | 跨 benchmark 的相对性能聚合 | 越大越好 |
| runnable ratio | schedule 通过检查、编译并成功运行的比例 | 越大越好 |
| invalid ratio | 解析或基础规则判定为无效的比例 | 越小越好 |
| illegal ratio | 依赖合法性检查未通过的比例 | 越小越好 |
| token consumption | 输入和输出 token 的累计消耗 | 越小越省成本 |
| wall-clock time | 完成一次 benchmark 优化的实际耗时 | 越小越省成本 |

## 7. 实验结果与结论

### 7.1 主要结果

在 150 个实例、每个实例 40 次独立运行、T=30 的设置下，主 LLM 的单次典型结果 `COMPILOT@30` 相对原始未优化代码的几何平均 speedup 为 2.66×，95% CI 为 [2.60, 2.77]。best-of-5 的 `COMPILOT 5@30` 为 3.54×，95% CI 为 [3.45, 3.58]。单次结果的实例分布中，至少 50% 的实例达到 1.24×，25% 达到 3.6×，最高 10% 超过 23.65×；best-of-5 对应数字为 1.59×、4.8×和 53.65×。

个别大规模实例获得很高 speedup，例如 `correlation_XLARGE` 的中位单次 speedup 为 339×，`trmm_XLARGE` 为 183×；论文将其与外层并行化、分块和展开组合以及 48 线程机器联系起来。相反，`cholesky`、`durbin`、`ludcmp` 的中位 speedup 约为 1×，论文推测复杂循环携带依赖超出当前变换集合的有效处理范围。

### 7.2 与传统方法的比较

best-of-5 COMPILOT 相对 Pluto 优化代码的几何平均 speedup 为 2.94×，95% CI 为 [2.88, 2.97]；在 150 个实例中胜过 Pluto 119 个、近似相同 9 个、低于 Pluto 22 个。按输入规模聚合时，COMPILOT/Pluto 在 MINI、SMALL、MEDIUM、LARGE、XLARGE 上分别约为 16.35×、5.19×、未在正文表格中单独给出、1.06×和 0.82×。论文指出大规模时两者都能利用并行化，COMPILOT 相对优势减弱；小规模时经验反馈帮助避免 Pluto 的规模无关启发式导致的回归。

与“不会比原始代码更慢”的 capped Pluto 比较时，COMPILOT 的整体几何平均相对 speedup 从 2.94×降为 1.78×，说明部分优势来自避免 Pluto 的性能回归，而不是所有场景都产生同等程度的绝对优化。

在 Tiramisu 深度学习 autoscheduler 支持的 8 个 benchmark 上，COMPILOT 单次相对其 speedup 为 2.65×，best-of-5 为 3.23×。论文将差异归因于更丰富的变换集合、可迭代修改的 schedule，以及直接使用实测反馈而非离线 cost model 预测。

### 7.3 与其他 LLM 方法的比较

不同 LLM 在 T=30 的单次几何平均 speedup 为：gemini-2.0-flash 2.66×、gpt-4o 2.63×、gpt-o3-mini 在该列未评估、llama3.3 (70B) 2.47×、qwq 2.36×、qwen2.5-coder 2.14×、gemma3 (27B) 2.03×、codestral-2501 (22B) 1.75×。论文观察到通用模型总体不差于专门的 coding 模型；代码生成能力不直接等价于高层 schedule 规划能力。

### 7.4 消融实验

无反馈时，gemini-2.0-flash 在 T=30 的单次 speedup 约为 2.01×，比有反馈的 2.66×低约 23%；best-of-5 也低约 28%。gpt-4o 的单次差距约 40%。

直接生成 C 代码的变体在 25 轮后比 COMPILOT 低约 14–16%，且初始输出比较通过的 schedule 中，随机输入二次检查发现 17.9% 在全部通过初检的 schedule 中实际非法；只看实现 speedup 大于 1 的候选，错误比例为 17.6%。直接生成还消耗约 5.3× token。该对比不是形式化证明，但说明有限输出比较可能产生假阳性。

去掉初始程序分析后，T=30 单次几何平均由 2.66×降到 2.42×，约低 8%；best-of-5 约低 4%，GPT-4o 单次约低 14%。去掉每轮 reasoning 对 GPT-4o 单次约低 11%，对两种模型 best-of-5 低约 4–7%。去掉硬件上下文没有观察到统计显著差异。

### 7.5 成本、合法性与探索分析

T=30 的一次 benchmark 优化平均约 8.9 分钟；XLARGE 约 16 分钟，MEDIUM/SMALL/MINI 约 5–6 分钟。主设置中约 78.5% 时间消耗在后端检查合法性、编译和运行，而不是 LLM 通信。gemini-2.0-flash 的通信约占每个 benchmark 1–3 分钟，token 消耗随对话历史非线性增长。

截至 T=30，gemini-2.0-flash 的 schedule 平均约 31.4% invalid、32.5% illegal、36.1% runnable；初始迭代的 illegal 比例接近 60%，后续下降，runnable 比例逐步接近约 36%。T 从 1 增到 75 时单次几何平均从 1.41×到 3.06×，T=30 已达到 2.68×，呈现递减收益；K 从 1 增至 5 时由 2.66×到 3.54×，K=10 为 3.75×、K=13 为 3.82×，因此作者选择 K=5 作为代表性折中。

## 8. 主要创新点

### 8.1 创新点一：把 LLM 作为编译器动作选择器

论文不是让 LLM 直接生成完整优化代码，而是让它选择和排序一组受限的高层循环变换命令，再由 Tiramisu 执行。这种接口把搜索策略与变换实现分开，实验上比直接代码生成表现更好、token 成本更低。真正创新是交互式动作接口和实验反馈闭环，不是单独使用 LLM 或 Tiramisu。

### 8.2 创新点二：把合法性与性能反馈放进持续对话

每轮都将编译器检查结果、错误类型和实测 speedup/slowdown 反馈给 LLM，让其通过 in-context learning 更新 schedule。无反馈消融显著下降，支持反馈是该系统有效性的核心组成。

### 8.3 创新点三：系统研究通用 LLM 的探索行为

论文不仅报告最终 speedup，还比较模型、T/K、token、失败比例、初始分析、显式 reasoning、硬件上下文和继续探索。best-of-K 结果显示利用模型随机性可以逃离局部最优，但收益递减。

## 9. 局限性

### 9.1 论文明确承认或实验中显示的局限

论文原型依赖 Tiramisu 和其多面体表示；当前变换集合不能有效处理 `cholesky`、`durbin`、`ludcmp` 等复杂依赖程序。大量 schedule 无效或非法，主设置只有 36.1% 可运行。一次优化平均约 8.9 分钟，且多次运行增加成本。LLM 随机性造成不同局部最优，必须用 best-of-K 缓解。论文未来工作还提出更详细的依赖失败原因、硬件性能计数器、系统搜索混合和对话摘要。

### 9.2 阅读后的潜在局限

实验仅在单一双路 Intel Xeon 平台上测量；论文没有给出 RISC-V、GPU 或异构硬件实测，因此不能把结果直接外推到这些平台。硬件上下文消融无显著差异，说明当前 LLM 未必能把具体缓存和核心数字稳定地转成 tile size 等决策。Tiramisu 依赖分析给出了其支持范围内的合法性保障，但这不等价于对任意编译器、任意程序或全栈语义的形式化证明。对 Pluto 的优势还部分来自 Pluto 在某些规模上的性能回归，不能简单解读为所有 benchmark 上的绝对优势。

## 10. 阅读后的研究方向反思

论文最值得借鉴的是“受限动作空间 + 编译器验证 + 真实硬件反馈”的角色分工。它适合作为 SELECTOR 类 baseline 或编译器智能体模块，而不是直接照搬成 RISC-V 论文。若仅把 CPU 换成 RISC-V，主要是平台迁移，创新性不足；需要新增可验证的 RISC-V/RVV 后端效应建模、跨硬件迁移或硬件计数器解释机制，才能形成新的研究问题。

论文还说明，直接让 LLM 改写低层代码会带来验证假阳性和较高 token 成本；因此对于 LLVM/MLIR 或 RISC-V，较稳妥的接口是让模型选择 pass、schedule、pragma、RVV 相关配置或编译器动作，由后端负责变换与验证。本文没有实验 LLVM/MLIR 或 RISC-V，相关启发应视为阅读分析，不是论文结果。

## 11. 可进一步尝试的研究方向

### 11.1 方向一：面向 RISC-V/RVV 的硬件计数器反馈调度

#### 研究问题

在 RISC-V/RVV 后端中，speedup 之外加入 cache miss、向量利用率、分支和访存事件，能否减少非法探索并提高跨输入规模的 schedule 迁移能力。

#### 与原论文的区别

不是只替换 CPU，而是把可解释的硬件效应反馈纳入动作选择和失败分析。

#### 可能的创新点

建立“变换—硬件事件—收益”的结构化反馈协议，并比较仅 wall-clock 与多计数器反馈。

#### 实验框架

```text
LLVM/MLIR 或 Tiramisu 循环
  → LLM 选择受限 schedule
  → RISC-V/RVV 编译与真实运行
  → 性能计数器 + 正确性/合法性证据
  → LLM 继续探索或回退
```

#### 可行性

需要可用的 RISC-V/RVV 硬件或仿真器、编译器后端、计数器采集和小规模循环基准。

#### 主要风险

计数器可比性、测量噪声、仿真器与真实硬件差异，以及后端对变换合法性的支持范围。

### 11.2 方向二：跨硬件的 schedule 迁移与安全回退

#### 研究问题

在 CPU、RISC-V/RVV 和 GPU 等平台之间，如何复用已探索的 schedule 知识，同时用少量目标平台测量修正代价模型。

#### 与原论文的区别

原论文固定单一 CPU；该方向研究 schedule 表示、迁移和目标平台适配，而不是单平台 best-of-K。

#### 可能的创新点

用平台无关的循环结构特征与平台相关的代价残差分离 schedule 选择。

#### 实验框架

```text
源平台反馈轨迹 → 结构化 schedule/特征库
                         ↓
目标平台少量候选采样 → 编译、验证、测量
                         ↓
迁移策略更新 → 安全回退到基线
```

#### 可行性

需要多平台编译链、相同 benchmark 的可重复构建和明确的正确性检查。

#### 主要风险

不同后端可表达的变换集合不一致，跨平台 speedup 不能直接比较，且少样本迁移可能过拟合。

### 11.3 方向三：把合法性失败原因结构化为可审计证据

#### 研究问题

依赖冲突、完美嵌套不满足、求解器失败和编译器崩溃能否转成结构化错误证书，以减少重复的非法尝试。

#### 与原论文的区别

原论文主要把失败类别和消息反馈给 LLM；该方向进一步抽取依赖边、违反的前置条件和可修复动作。

#### 可能的创新点

设计可回放的 legality evidence schema，并评估它对 runnable ratio、token 和最终 speedup 的影响。

#### 实验框架

```text
LLM schedule → 编译器检查
      ├─ 合法：运行并返回性能证据
      └─ 非法：返回依赖/前置条件证书
                    ↓
              约束下一轮动作空间
```

#### 可行性

需要编译器依赖分析接口、错误消息解析器和现有 PolyBench/Tiramisu 基线。

#### 主要风险

证书可能过于复杂，或只适用于特定后端；减少探索错误也可能抑制有价值的探索。

## 12. 与其他已读文献的关系

当前 slot-3 本轮只完成这一篇论文，因此本节不虚构与其他已读论文的实验关系。

就方法定位而言，本文与 pass/flag 选择工作共享“LLM 输出编译器动作”的接口思想，但本文输出的是 Tiramisu 源级循环 schedule，并以目标机器实测 speedup 为反馈；它与直接代码生成、LLM-Vectorizer 等路线的主要差异是由编译器负责变换和依赖合法性检查。与传统 Pluto、Tiramisu autoscheduler 相比，本文是反馈驱动的动作选择器，而不是新的多面体变换后端。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | 用通用 LLM 通过编译器反馈自动搜索循环优化 schedule |
| 核心问题 | 复杂循环变换的搜索、合法性和实测性能反馈 |
| 输入 | PolyBench/C C/C++ 循环嵌套、comp_ID、初始运行时间、硬件上下文 |
| 输出 | Tiramisu 可执行的循环变换序列及最佳程序变体 |
| 核心方法 | COMPILOT 闭环：LLM 动作提议—编译器检查/运行—反馈—迭代 |
| 使用的模型 | 主结果 gemini-2.0-flash；另比较 7 个 LLM |
| 使用的编译器工具 | Tiramisu、依赖分析、内部 skewing/shifting 求解器、PolyBench |
| 是否使用强化学习 | 否；没有参数训练或奖励函数 |
| 是否使用形式化验证 | 使用 Tiramisu 的多面体依赖合法性检查；不等同于全程序形式化证明 |
| 数据集规模 | 30 个 benchmark × 5 个标准规模 = 150 个实例 |
| 主要指标 | speedup、几何平均、95% bootstrap CI、runnable/invalid/illegal、token、时间 |
| 最重要实验结果 | 单次 2.66×；best-of-5 3.54×；相对 Pluto best-of-5 2.94× |
| 核心创新 | 受限编译器动作接口与实测反馈驱动的 LLM 迭代探索 |
| 主要局限 | 单平台、成本高、约三分之二 schedule 不可用、复杂依赖场景收益弱 |
| 与 RISC-V 研究的相关性 | 中：闭环和动作接口可借鉴，但论文没有 RISC-V/RVV 实验 |
| 最适合作为 | 编译器智能体/循环 schedule 选择的 baseline 与方法参考 |

这篇论文最值得学习的是让 LLM 负责高层策略探索，让编译器负责变换、合法性和代码生成；最主要的局限是搜索效率、单平台验证和复杂依赖覆盖不足。如果用于后续研究，合理方式是把它作为反馈驱动 selector baseline，再加入 RISC-V/RVV 的真实后端效应与可审计证据，而不是只替换目标 CPU。
