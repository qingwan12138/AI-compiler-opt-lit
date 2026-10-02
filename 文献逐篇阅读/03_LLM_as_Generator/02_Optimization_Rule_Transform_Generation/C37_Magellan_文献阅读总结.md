# C37 Magellan 文献阅读总结
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

论文题目：**Magellan: Autonomous Discovery of Novel Compiler Optimization Heuristics with AlphaEvolve**
作者：Hongzheng Chen、Alexander Novikov、Ngân (NV) Vũ、Hanna Alam、Zhiru Zhang、Aiden Grossman、Mircea Trofin、Amir Yazdanbakhsh（Google/Google DeepMind/Cornell）
发表时间：2026-01-28；本次阅读日期：2026-09-05
发表平台：C4ML@CGO 2026 Workshop（作者声明录用；非 CGO 主会）；arXiv:2601.21096v1（PDF v1）
元数据核验来源：[arXiv:2601.21096](https://arxiv.org/abs/2601.21096)；[C4ML@CGO 2026 官方议程](https://sites.google.com/view/c4ml)
关键词：编译器启发式、AlphaEvolve、LLM coding agent、演化搜索、自动调参、LLVM/XLA

> 本文档为 Luna 阅读（2026-09-05）。

## 1. 研究背景

现代编译器中的内联、寄存器分配、图提取等问题常为 NP-hard，生产实现依赖多年人工维护的启发式。启发式能在大规模程序上快速决策，却难适应不断变化的软件、硬件和目标函数。已有 LLM 工作多生成目标代码，或为每个输入搜索 Pass 序列；神经策略（如 MLGO）则需把模型集成进编译器。Magellan 选择直接演化 Pass 内的 C++ 决策逻辑，使结果能像人工代码一样编译、部署和复用（第 1 节，第 1—2 页）。

## 2. 论文要解决的问题

### 2.1 启发式设计自动化

能否让 LLM 在现有 LLVM/XLA API 和真实宏基准上，直接产生可执行、可部署的 Pass 决策策略，而不是每个程序单独生成优化序列？

### 2.2 搜索效率

LLM 同时搜索逻辑和大量数值阈值会浪费昂贵的编译/运行评测。论文研究把高层策略模板和低层超参数分离是否更高效。

### 2.3 生产级泛化

生成策略能否跨时间戳、应用域和不同编译任务保持收益？

> 本文主要研究：如何用 AlphaEvolve 驱动 LLM 演化可读、可集成的 C++ 编译器启发式。

## 3. 核心方法概述

```text
LLVM/XLA Pass 中标记 EVOLVE-BLOCK
        ↓
AlphaEvolve + LLM 提出带显式超参数的 C++ policy template
        ↓ 编译器重编译
真实宏基准（binary size / perf / 任务目标）评测
        ↓
Vizier 黑盒自动调参（固定模板，仅改 flags）
        ↓ 分数、调参日志、profiling traces
AlphaEvolve 选择/变异下一模板，循环搜索
```

Magellan 不是生成汇编或 kernel，也不是训练一个在线神经模型；它演化现有编译器的决策逻辑。内联合法性由 LLVM 既有 MLInlineAdvisor 基础设施检查，故集成成功的策略不会因非法内联破坏编译器正确性（第 3.1 节，第 3 页）。

## 4. 实验框架与训练流程

### 4.1 策略提议

LLM 修改 EVOLVE-BLOCK 中的源文件，并把可调数字暴露成 compiler flags。内联实验有 feature-based partial heuristic（组合 38 个预定义特征）与 API-level full heuristic（从 CallBase 访问周围 LLVM IR 自行构造逻辑）两种空间。

### 4.2 局部评测

每个候选都重编译内部 clang/LLVM 工具链，在宏基准上测最终 binary size 或 runtime。论文使用接近 LLVM main tip-of-tree 的 Google 内部工具链，代表版本为 2025 年夏季提交；不是公开 release snapshot（第 3 节，第 3 页）。

### 4.3 自动调参与演化反馈

Vizier 对模板超参数做 box-constrained 黑盒搜索。调参结果、最优分数、日志和 profiling trace 返回 AlphaEvolve，由其选择、交叉/变异式生成下一模板。正文没有 SFT、PPO 或 GRPO；LLM 是搜索中的 coding agent。性能内联任务从 Gemini-2.5-Pro 运行到 Gemini-3-Pro continuation。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数或神经网络损失。目标由任务直接定义：

```text
Binary-size task: maximize size reduction relative to upstream LLVM heuristic
Performance task: maximize end-to-end macro-benchmark speedup
Register allocation: maximize workload performance
XLA extraction: minimize extracted graph cost
XLA auto-sharding: minimize compute + communication/resharding cost under memory budget
```

这些是实际测量/竞赛目标，不等同于指令数或静态估计。对 clang 性能实验，baseline 与候选均使用相同 PGO、分布式 ThinLTO 和 -O3（第 3.2 节，第 5—6 页）。

## 6. 实验设置

### 6.1 数据集来源

没有一个公开训练集；输入是 Google 内部宏基准、生产二进制和 XLA IOPDDL contest 实例。XLA auto-sharding 使用公开训练实例 5 个、私有评测实例 20 个，遵循竞赛划分。论文中未给 LLVM 内部内联/寄存器任务的完整程序数量与公开数据集规模。

### 6.2 模型与工具

默认 LLM 为 Gemini-2.5-Pro；部分性能实验使用 Gemini-3-Pro。AlphaEvolve 负责进化搜索，Vizier 负责黑盒调参。工具为内部 clang/LLVM、llvm-size、perf stat、PGO、ThinLTO；XLA 案例涉及 Enzyme-JAX/e-graph、auto-sharding 和 TPU 目标。

### 6.3 对比方法

主要 baseline 是 upstream LLVM 手工内联启发式；时间泛化还比较约两年前训练、部署于 MLGO 的神经内联模型。寄存器分配与 XLA 任务对比各自人工设计策略/竞赛基线；没有统一的 LLM 或 RL baseline。

### 6.4 评价指标

binary size reduction 是相对 baseline 的最终二进制大小减少比例；performance improvement 是宏基准端到端性能相对变化；寄存器和 XLA 任务使用其工作负载/竞赛 objective。论文报告的是实测宏基准结果，不是 instruction count。

## 7. 实验结果与结论

### 7.1 LLVM 内联：二进制大小

1.5 天顺序搜索后，partial heuristic 比 upstream 减小 4.27%；API-level full heuristic 达 5.23%，且超过 feature-based 方法（图 3，第 4 页）。加入 Vizier 后，约 10 个外层迭代（100 个配置）即可超过 5%；LLM 直接提议完整启发式的无效率超过 65%，调参后降至 13%，浪费样本约减少 5×（第 3.1.1 节）。

### 7.2 泛化与可维护性

同一策略在四个时间快照上带来 5.75%–5.95% size reduction，略优于旧神经模型；十多个生产二进制平均减少 8.79%，神经模型为 8.52%（图 4—5，第 4—5 页）。生成可执行逻辑 143 行，而去掉注释和空行的人工实现约 2,115 行；作者提醒代码短不等于长期可维护。

### 7.3 LLVM 性能内联

从全 false 策略开始，Gemini-2.5-Pro 最终约退化 2.4%，Gemini-3-Pro 独立搜索 100 次迭代仍未超过 0%。以 2.5-Pro 的最好策略作为 Gemini-3-Pro continuation 种子后，最终超过人工 baseline 0.61%（图 6，第 5—6 页）。这说明初始化和搜索空间对稀疏、噪声性能目标很关键。

### 7.4 其他任务

RegAllocGreedy 的 live-range priority 队列搜索收敛到常数策略，性能从 -0.55% 改善到 -0.15%，匹配复杂人工启发式。XLA e-graph extraction 比人工策略提高 7%；auto-sharding 一周演化后与顶级竞赛提交相当，但全文明确说完整 XLA pipeline 集成仍在进行（第 3.3 节，第 6 页）。

## 8. 主要创新点

### 8.1 Pass-level policy evolution

创新在于演化可部署 C++ 决策逻辑，区别于每程序 Pass 序列或直接目标代码生成。

### 8.2 高层模板/低层调参分离

LLM 负责结构性策略，Vizier 负责数字参数；这降低重新编译和生成成本，实验证明收敛更快、无效率更低。

### 8.3 真实宏基准闭环

用最终链接二进制大小和端到端运行时间作反馈，避免只优化对象文件 `.text` 或静态 surrogate；同时展示跨时间、域和 LLVM/XLA 的迁移性。

## 9. 局限性

**论文明确承认：**编译/评测每候选成本高，性能目标尤其昂贵；尚不清楚何时优于神经策略；搜索预算、模型和目标噪声的规模规律未厘清（第 5 节，第 7 页）。部分 XLA 结果仍未完成完整 pipeline 集成。

**阅读后潜在局限：**内部 clang、宏基准、程序数量和硬件环境并非全部公开，复现性受限。内联 size 的强结果与 performance 的 0.61% 结果来自不同搜索设置，不能混为统一指标。LLVM 合法性检查保证的是该 Pass 决策的编译器约束，不是整个生成启发式的形式化最优性证明。

## 10. 阅读后的研究方向反思

可借鉴“源码级启发式 + 真实宏基准 + 自动调参”的架构；核心不是把 LLM 接到编译器，而是利用其探索 API 和结构决策。仅把 LLVM 替换成 RISC-V backend 不足以构成创新，必须研究 RISC-V 特有代价（向量长度、寄存器压力、扩展选择、代码尺寸/功耗）和跨核心泛化。Magellan 更适合作为 compiler decision policy baseline/搜索工具，不是现成的 RISC-V 方案。与 PassNet 的区别是 Magellan 演化已有 Pass 内逻辑，PassNet 生成图级 matcher/rewriter；与 SuperCoder 的区别是 Magellan保留编译器流水线，SuperCoder直接改汇编。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V 向量后端的启发式演化

#### 研究问题

能否演化同时考虑 RVV 向量化收益、VL/VLMAX、寄存器压力和代码尺寸的 C++ 后端决策？

#### 与原论文的区别

不只迁移 LLVM，而是加入 RISC-V 特有多目标代价与多核心测试。

#### 可能的创新点

硬件计数器/仿真器反馈、跨 VLEN 泛化、size-performance-energy Pareto 策略。

#### 实验框架

```text
LLVM-RISCV EVOLVE-BLOCK → LLM模板 → Vizier参数 → QEMU/真实RVV板卡宏基准 → 演化反馈
```

#### 可行性

可使用 LLVM RISC-V backend、QEMU 与公开 SPEC/MLIR workload；但真实 RVV 板卡需准备。

#### 主要风险

硬件样本少、运行噪声大、策略可能只适应单一 VLEN。

### 11.2 编译器启发式的可解释验证

让演化候选自动生成决策覆盖报告、边界 case 和差分测试；与原文的真实性能搜索相比增加安全证据层。风险是证明成本和性能目标冲突。

## 12. 与其他已读文献的关系

Magellan 与 PassNet 都用真实编译器反馈，但前者生成 Pass 的 decision logic，后者生成图变换代码；两者可串联为图模式选择后再由后端启发式决定是否内联/分配。与 SuperCoder 都以运行时间为目标，但 Magellan 的输出是可部署 C++ heuristic、一次生成后跨程序复用；SuperCoder 的输出是针对每个 C/汇编实例的 x86-64 程序。Magellan适合作为编译器后端 heuristic baseline，PassNet适合作为图级 Pass 数据/评测，SuperCoder适合作为汇编级性能对比。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 自动发现可部署编译器优化启发式 |
| 核心问题 | 人工启发式难维护，搜索成本和泛化未知 |
| 输入/输出 | EVOLVE-BLOCK与宏基准；C++ Pass policy |
| 核心方法 | AlphaEvolve 演化 + Vizier 黑盒调参 |
| 模型/工具 | Gemini-2.5/3-Pro、LLVM/XLA、perf/llvm-size |
| 强化学习/形式验证 | 无 PPO/GRPO；非形式化最优性证明 |
| 数据规模 | 内部宏基准；XLA 5 public/20 private contest实例 |
| 主要结果 | 在 LLVM 内联任务中，相对 upstream 启发式的 binary-size reduction 为 5.23%；十多个生产二进制平均减少 8.79%；在相同 PGO/ThinLTO/-O3 设定下端到端性能超过人工 baseline 0.61%；XLA extraction 相对人工策略提高 7% |
| 核心创新 | Pass-level executable heuristic evolution |
| 主要局限 | 评测昂贵、内部资源不全公开、任务规模规律未明 |
| RISC-V相关性 | 中：方法可迁移，但需 RVV/硬件代价新证据 |
| 最适合作为 | 编译器 heuristic baseline、搜索方法、后端工具模块 |

这篇论文最值得学习的是把启发式搜索落在可编译、可维护的 Pass 源码上；最主要局限是评测依赖内部宏基准且性能收益较小、昂贵。如果用于后续研究，应引入目标架构特有代价与可复现硬件，而不是只更换后端名称。
