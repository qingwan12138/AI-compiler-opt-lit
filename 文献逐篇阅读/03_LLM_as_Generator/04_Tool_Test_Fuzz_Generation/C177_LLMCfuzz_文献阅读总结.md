# LLMCfuzz 文献阅读总结

论文题目：**LLMCfuzz: A Fuzz Testing Method for Aero Engine Compilers Based on Large Language Models**
中文题目：LLMCfuzz：基于大语言模型的航空发动机编译器模糊测试方法
作者：陆炜、李伟湋、施彬彬、曾俊伟、黄志球
发表时间：2025
发表平台：《计算机学报》48(12):2875–2892（2025）
论文链接或编号：DOI 10.11897/SP.J.1016.2025.02875
元数据核验来源：[期刊 DOI](https://doi.org/10.11897/SP.J.1016.2025.02875)；[期刊正式 PDF](http://cjc.ict.ac.cn/online/onlinepaper/lv-20251222131358.pdf)
代码/数据/工件：未找到可确认的作者代码仓库或工件
关键词：航空发动机嵌入式系统、C 交叉编译器、LLM、编译器模糊测试、程序变异、静默误编译

## 1. 研究背景

论文研究航空发动机嵌入式系统中的 C 交叉编译器测试。该场景受 DO-178C 等安全约束影响，目标硬件通常具有专用优化和交叉编译流程。最危险的问题是 silent miscompilation：编译器生成了错误可执行文件，但编译和运行过程不出现显式错误。

传统 Csmith、YARPGen 等生成器受规则和程序结构限制；GrayC 等变异器容易积累小变异，难以产生能够触发后端优化错误的复杂数据流/控制流。通用 LLM fuzzing 方法没有充分适配航空嵌入式约束，也缺少跨主机/目标板的变量值 oracle。

## 2. 论文要解决的问题

### 2.1 生成复杂且合规的测试程序

测试程序需要保持前端可编译，同时具有复杂数据流、控制流，并适应航空发动机嵌入式代码约束。

### 2.2 检测 silent miscompilation

仅观察编译器 crash 或显式错误不能覆盖静默误编译，需要在目标板运行并比较变量结果。

### 2.3 核心总结

> 本文主要研究：如何利用 LLM 驱动程序变异、变量追踪和跨版本/优化级别差分测试，发现航空发动机 C 交叉编译器的后端和静默误编译缺陷。

## 3. 核心方法概述

```text
GCC/LLVM 测试套件 + 航空发动机嵌入式程序
        ↓
1200 个种子程序库
        ↓
多样性引导的 13 类变异算子选择
        ↓
LLM 根据变异 prompt 生成复杂变体程序
        ↓
前端错误反馈更新 few-shot prompt
        ↓
LLM 插入全局/局部变量输出或 UART 追踪语句
        ↓
Linux GCC-14.1 与航空发动机交叉编译器差分测试
        ↓
跨编译器版本、跨优化级别比较结果并人工分析
```

LLM 主要生成变异程序和监控语句；真正执行变换、编译和运行的是编译器与目标板环境。因此这是 GENERATOR/G4，不是 pass/transform generator。

## 4. 实验框架与训练流程

本文不涉及模型训练、SFT 或强化学习，直接使用 DeepSeek-Coder-V2.5 API 和提示词。

### 4.1 变异提示生成

论文设计 13 个变异算子，分为插入分支/循环/控制语句/结构体/表达式/死代码、替换常量/变量/运算符/赋值右值、移除限定符/修饰符/一元运算符。模板包含 Task、Instructions、Code 三部分。

### 4.2 前端错误反馈

若变体无法通过前端，记录程序和错误信息，加入对应变异算子的 few-shot 示例；最大反馈样本数为 3。

### 4.3 变量追踪与差分

LLM 在函数 return 前或程序末尾插入 `printf`/UART 输出，追踪全局变量及插入点作用域内的局部变量。交叉编译器版本使用 UART 传回目标板结果；Linux 参照环境使用 GCC-14.1。

### 4.4 变异算子调度

先按 Jaccard 距离和前端通过率计算 operator score，再使用 Metropolis-Hastings 式概率选择而非始终贪心选择最高排名算子。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数。

程序行集合之间的距离为：

```text
Dist(P1, P2) = 1 - |Stmt(P1) ∩ Stmt(P2)| / |Stmt(P1) ∪ Stmt(P2)|
```

变异算子得分为：

```text
Score(Mut) = (1/n) × Σ Dist(Pi, Pi-1) × Rate(Mut)
```

其中 `Rate(Mut) = #FrontPass_Mut / #All_Mut`。它同时鼓励变体多样性和前端有效率。

Metropolis-Hastings 接受概率使用算子排名差和参数 `p=0.3`；高排名候选更易被接受，但低排名算子仍有非零机会，避免固定贪心。

## 6. 实验设置

### 6.1 数据集来源

种子库包括 GCC/LLVM 测试套件中的 1,000 个程序，以及航空发动机嵌入式系统源码中的 200 个程序，共 1,200 个 C 程序，约 16 万行；平均程序长度约 135 行，平均圈复杂度约 8.2。论文称这些种子均已通过编译器前端且不触发待测编译器错误。

### 6.2 模型与工具

- 模型：DeepSeek-Coder-V2.5，论文写明参数规模 236B，使用官方 API，temperature=1。
- 编译器：航空发动机 GCC-based cross compiler、Linux GCC-14.1。
- 优化级别：`-O0, -O1, -O2, -Os, -O3`。
- 环境：Ubuntu 22.04，32 GB 内存；交叉编译结果在航空发动机嵌入式开发板上通过 UART 验证。
- 覆盖率：Gcov。

### 6.3 对比方法

Fuzz4All、GrayC、Clang-Fuzzer、universalmutator、Csmith。消融包括 LF-No-DFCF、LF-No-Feedback、LF-Random。

### 6.4 评价指标

| 指标 | 含义 |
|---|---|
| 覆盖行数 | 待测编译器被执行的代码行数 |
| 覆盖提升率 | 相对其他方法覆盖行数的百分比提升 |
| 有效率 | 通过编译器前端的测试程序比例 |
| 误编译数 | 通过差分测试和人工分析确认的 miscompilation 数量 |

## 7. 实验结果与结论

### 7.1 覆盖率对比

每种方法生成 10,000 个程序、运行 10 次取平均，编译超时 20 秒。LLMCfuzz 覆盖 165,843 行，有效率 97.73%；Fuzz4All 156,368 行、95.27%；GrayC 161,360 行、99.65%；universalmutator 145,774 行、98.54%；Csmith 147,305 行、99.65%。相对 GrayC 覆盖提升 2.78%，相对 Fuzz4All 提升 6.06%，相对 Clang-Fuzzer 提升 21.08%。

### 7.2 消融实验

LLMCfuzz 覆盖 165,843 行；去除数据流/控制流增强后为 162,334 行；去除前端反馈后为 165,269 行；随机选择算子为 163,560 行。结果支持三类模块分别贡献结构复杂度、有效率和多样性探索。

### 7.3 缺陷发现

24 小时差分测试中，LLMCfuzz 发现 5 种误编译错误：3 个 silent miscompilation、2 个显式误编译；其中 4 个是其独有发现。基线最多发现 1 个错误，Fuzz4All、Clang-Fuzzer 和 Csmith 在该实验中未发现错误。

错误包括隐式类型转换、常量折叠、非法代码移动、循环语句拆分和只读内存写入；论文对每个错误进行了最小化和原因分析。

### 7.4 上下文长度

10,000 次 API 调用统计中，N=3 few-shot 时前端错误信息长度平均 28 tokens，prompt 总长度平均 1,235 tokens，最大 1,643 tokens；论文据此认为没有超过 128K context limit。

## 8. 主要创新点

### 8.1 面向航空交叉编译器的 LLM 变异框架

将领域种子、13 类变异算子、前端错误反馈和复杂控制/数据流指令组合起来，针对通用 fuzzer 难以覆盖的后端优化路径。

### 8.2 变量追踪机制

相较只输出全局哈希值，LLMCfuzz 输出全局和局部变量的具体值，并通过 UART 适配目标板，使 silent miscompilation 可以被观察和定位。

### 8.3 多样性引导的算子调度

Jaccard 距离、前端有效率和 MH 采样共同避免算子选择退化为固定贪心。

## 9. 局限性

### 9.1 论文明确承认的局限

- 只在一种航空发动机嵌入式 C 交叉编译器上验证。
- 变量追踪、变异算子和提示模板需要根据具体领域约束适配。
- 依赖目标板/UART 运行验证，部署成本高。
- 当前没有发现编译器 crash 类错误；方法更偏向 miscompilation。
- 未来工作提出 RAG、强化学习和主动学习，但这些不是本文已实现模块。

### 9.2 阅读后发现的潜在局限

测试程序中插入输出语句可能改变优化行为，尤其是 volatile、IO 和别名关系；论文通过设计进行缓解，但不能把变量追踪结果视为形式化语义证明。覆盖率提升也不必然等价于所有后端缺陷覆盖。

## 10. 阅读后的研究方向反思

LLMCfuzz 是明确的 GENERATOR/G4，适合作为 compiler-testing component 或 baseline，不应与 pass/optimization-rule generator 混为一类。对 RISC-V 的迁移若只替换目标编译器，创新性有限；可增加 RISC-V 特权/向量扩展、目标特定 lowering、汇编级差分和 QEMU/FPGA/真实板卡多层 oracle。

值得借鉴的是将 domain seed、结构化变异、前端反馈和 runtime oracle 串成闭环；不能直接照搬的是航空发动机的 DO-178C/PowerPC/UART 约束和错误分类。

## 11. 可进一步尝试的研究方向

### 11.1 RISC-V 向量编译器的静默误编译差分测试

#### 研究问题

如何让 LLM 生成能触发 RVV mask、VL/VTYPE、别名和循环优化边界的 C/LLVM IR 程序，并在多种 RVV 后端/硬件环境中稳定比较结果。

#### 与原论文的区别

从航空发动机专用 C 编译器扩展到 RISC-V vector semantics 和多实现差分。

#### 可能的创新点

加入 RVV 状态追踪、汇编级 oracle 和跨 QEMU/真实硬件分层执行。

#### 实验框架

```text
RISC-V/RVV seed corpus → LLM mutation → cross-compiler/QEMU/board execution → value and assembly differential oracle
```

#### 可行性与主要风险

可使用 GCC/LLVM/RVV、QEMU 和 Spike；风险是硬件环境差异、未定义行为和 IO 插桩改变优化。

### 11.2 基于 pass remarks 的定向误编译生成

将覆盖率和运行值反馈扩展为 LLVM pass remarks、优化开关和 IR before/after 轨迹，直接诱导模型生成对特定 pass 敏感的程序。

### 11.3 变量追踪与 translation validation 联合 oracle

将运行时结果差分与 Alive2/等价性验证结合，区分真实编译错误、程序未定义行为和目标硬件语义差异。

## 12. 与其他已读文献的关系

与 Germinator 同属 GENERATOR/G4，均使用 LLM 生成可复用测试输入并配合传统 fuzzing；Germinator 解决 MLIR dialect 低资源 seed 冷启动，LLMCfuzz 解决航空 C 交叉编译器的复杂变异和 silent miscompilation oracle。ACT 是非 LLM backend generator，属于 SUPPORTING/B2 边界对照。

本篇与仓库已有 Fuzz/Test Generator 类条目可能存在方法近邻，正式入库前应按 DOI、中文/英文题名、作者、种子库与变量追踪机制查重。当前 DOI、卷期、页码和官方 PDF 已核验。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 航空发动机 C 交叉编译器模糊测试 |
| 核心问题 | 复杂后端错误和 silent miscompilation 难以发现 |
| 输入 | 1,200 个 C 种子程序 |
| 输出 | LLM 变体程序、变量追踪测试程序、差分测试结果 |
| 核心方法 | 13 类变异 + 前端反馈 + 变量追踪 + 差分测试 |
| 使用的模型 | DeepSeek-Coder-V2.5，236B |
| 使用的编译器工具 | 航空发动机交叉编译器、GCC-14.1、Gcov、UART |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；采用差分测试与人工分析 |
| 数据集规模 | 1,200 个种子，约 16 万行 |
| 主要指标 | 覆盖行数、提升率、有效率、误编译数 |
| 最重要实验结果 | 165,843 行覆盖；发现 5 个误编译，其中 3 个静默误编译 |
| 核心创新 | 变量追踪 oracle 与领域化 LLM 变异闭环 |
| 主要局限 | 单一专用交叉编译器、依赖目标板和领域适配 |
| 与 RISC-V 研究的相关性 | 中高：方法可迁移，但需要重建 RVV/后端 oracle |
| 最适合作为 | G4 compiler-testing baseline、测试工具模块 |

> 这篇论文最值得学习的是把 LLM 变异与可观测的运行时 oracle 结合；最主要的局限是强依赖特定航空平台和插桩环境；如果用于后续研究，合理方式是建立 RISC-V/RVV 多层差分 oracle，而不是只更换目标架构名称。
