# C42 POLO 文献阅读总结

论文题目：**POLO: An LLM-Powered Project-Level Code Performance Optimization Framework**
作者：Jiameng Bai、Ruoyi Xu、Sai Wu、Dingyu Yang、Junbo Zhao、Gang Chen
发表时间：2025
发表平台：IJCAI 2025
元数据核验来源：[IJCAI 2025 正式论文页](https://doi.org/10.24963/ijcai.2025/814)；[IJCAI 2025 Paper 814](https://www.ijcai.org/proceedings/2025/814)
论文链接或编号：[IJCAI 2025 Paper 814](https://www.ijcai.org/proceedings/2025/814)
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）
关键词：项目级性能优化、运行时剖析、程序结构图、LLM agent、源码重写

> 阅读标记：gpt-5.6-luna，2026-09-05；依据 IJCAI-25 PDF（10 页）。POLO 是源码项目级优化，不能冒充 LLVM Pass 优化。

## 1. 研究背景

性能缺陷通常不破坏功能，却因低效数据结构、重复分配或不必要操作造成高开销。单函数/单文件 LLM 优化无法处理调用图、类关系和跨文件签名变化；传统 profiling 能找热点但不直接完成修复。POLO 模拟专家流程，把运行时热点、全项目结构和源码重写连接起来。

## 2. 论文要解决的问题

核心是如何在真实 C/C++ 项目中：用运行时数据定位真正热点，用全局结构理解热点影响的函数/类/变量，再由 LLM 生成跨函数、跨文件优化并用执行反馈决定接受或继续。论文不研究 LLVM IR pass 排序，也不生成 RISC-V 汇编。

## 3. 核心方法概述

POLO 三阶段为 Runtime Local Hotspot Detection、Global Correlation Detection、Code Rewrite with LLM Agents。Callgrind 经 FSM 解析成 Program Call Graph（PCG），Func Rank 综合函数自身时间和调用影响；Clang LibTooling/AST 构建 Program Structure Graph（PSG）；Generator Agent 改写热点，Decision Agent 重跑程序、接受/拒绝并提出下一策略。

```text
项目+工作负载 → Callgrind/FSM → PCG+Func Rank定位热点
 → LibTooling AST → PSG及邻域上下文 → Generator Agent重写
 → 编译/正确性测试/运行时测量 → Decision Agent接受或继续
 → 更新PSG/下一轮 → 终止并输出源码项目
```

## 4. 实验框架与训练流程

本文不训练模型，不涉及 SFT、RL、PPO/GRPO 或奖励函数。默认使用两个独立 GPT-4o 实例分别扮演 Generator/Decision，temperature=0.2；每个优化步骤生成 N=5 个结果并选最好者。若生成编译错误会重新询问。每轮修改后重新运行、更新 PSG，Decision Agent 判断接受/拒绝与是否继续。

## 5. 奖励函数、损失函数或关键公式

本文没有强化学习奖励。Func Rank 使用：

```text
w_ji = A^c_ji[time] / Σ_{u∈caller(j)} A^c_ju[time]
I_i = (1-α) A^c_i[time] + α Σ_{j∈callee(i)} w_ji I_j
```

`A^c_i[time]` 是函数自身运行时间，`w_ji` 是被调用函数向调用者传播的时间权重，`α` 平衡自身时间与传播影响，默认 0.5，迭代至收敛。性能提升指标 `PI=t_origin/t_optimized`，越大越好；每个项目执行五次取平均。该公式是热点排序，不是学习损失。

## 6. 实验设置

### 6.1 数据集来源

六个 C/C++ 项目：Quant（315 行）、AStar（484）、SkipList（886）、AES（1409）、KGraph（3937）、MiniSQL（15915），分 easy/medium/hard。既有开源也有作者私有项目。正确性测试覆盖率多数超过 80%；KGraph 使用 Audio、Sift1M 工作负载；论文没有传统训练/验证集。

### 6.2 模型与工具

Callgrind 做动态分析，Clang LibTooling/AST 做静态分析；编译比较 O0 和 O3。GPT-4o 做两个 agent，GPT-4 Turbo 用于 GPT4-hot/file baseline；Supersonic 是训练的 Seq2Seq C++ 优化模型。论文用 objdump 检查 AStar 中函数是否被 O3 内联。

### 6.3 对比方法

Supersonic（在线评测数据训练）、GPT4-hot（只给热点）、GPT4-file（给完整 C++ 文件但不显式标热点）。由于没有端到端项目级系统，作者向这些 baseline 手工注入热点。

### 6.4 评价指标

实际执行时间（ms）与 `PI=t_origin/t_optimized`；每项目每配置执行五次取均值。另比较 O0/O3、是否加入 PSG/PCG context、多轮 agent 及热点选择策略。

## 7. 实验结果与结论

### 7.1 主要结果

O3 下 POLO 的 PI：SkipList 3.19×（140.7→44.1 ms）、Quant 21.5×（178.7→8.3）、AStar 1.46×（8527.6→5844.6）、MiniSQL 4.23×（4133.8→977.6）、KGraph Audio 2.08×（78.5→37.8）、Sift1M 1.56×（14222.5→9096.5）。AES 的 ECB/CBC/CTR 分别 1.34×；论文摘要给出的范围是 1.34×–21.5×。

### 7.2 与传统方法的比较

POLO 是源码项目级系统，比较对象不是 LLVM Pass。相对 Supersonic、GPT4-hot、GPT4-file，图 6 显示 POLO 总体更快；具体数字需以图中项目/编译级别为准，不能写成统一平均提升。

### 7.3 与其他 LLM 方法的比较

Supersonic 多数项目改动很小；GPT4-hot 能改函数逻辑但难处理跨函数；GPT4-file 在热点及上下文同文件时有效，AStar 等跨文件项目受限。POLO 的优势来自 PSG/PCG 上下文与执行反馈，而非单纯更换模型。

### 7.4 消融实验

多轮交互对 SkipList、AES、AStar、KGraph 继续带来收益；Quant 第一轮已高效，MiniSQL 第二轮收益很小。某些第三轮 O3 结果下降，Decision Agent 会拒绝。加入 context 对跨函数 SkipList/AStar 明显有利，对 Quant/AES 这类函数内优化优势小。Func Rank 相比仅选最长执行函数，能识别调用链上真正热点。

### 7.5 案例分析

SkipList 将 `std::list` 换为 `std::vector`、调整搜索循环，并将高频临时 vector 改为 `static thread_local` 复用；AStar 用定长数组减少动态分配、对 unordered_map 使用 reserve、封装队列并手工 inline `getNeighbors`，objdump 证实 O3 未自动 inline。上述是源码级改写，不是 LLVM Pass 生成。

## 8. 主要创新点

### 8.1 PCG+Func Rank 的运行时热点定位

不同于仅按函数自耗时排序，传播被调用函数影响，减少把不可优化库函数或调用者误判为热点的风险。

### 8.2 PSG 全局上下文

将函数、类/结构体、全局变量及静态/动态关系编码为图，使函数签名变化能同步影响模块，支持跨文件修改。

### 8.3 双 agent 迭代执行反馈

Generator 负责提出改写，Decision 用真实执行结果控制接受和下一策略，形成项目级闭环。

## 9. 局限性

论文数据只有六个项目、工作负载和测试覆盖有限；性能测试基于特定机器/输入，不能等价为所有部署环境。依赖 Callgrind、LibTooling、可编译测试和 GPT-4o；源码修改可能有未覆盖语义问题。论文没有 LLVM IR、RISC-V/RVV 或跨架构实验，也没有形式化验证。

## 10. 阅读后的研究方向反思

可借鉴“真实运行时热点+全局结构+可拒绝反馈”的闭环。不能把 POLO 的源码改写直接描述成 LLVM Pass 优化，更不能仅把编译器换成 RISC-V 就声称创新；真正的新问题应是后端约束、跨架构性能与语义验证。POLO 适合作为源码项目级 baseline/上游候选生成器。

## 11. 可进一步尝试的研究方向

### 11.1 源码—LLVM IR—RVV 联合反馈
#### 研究问题
项目级源码改写如何在不破坏语义的前提下诱导 LLVM/RVV 生成更快代码？
#### 与原论文的区别
增加 IR/汇编质量与真实 RVV 执行反馈，不止看源码运行时间。
#### 可能的创新点
PSG 热点与 IR 反汇编、向量化指标联合决策。
#### 实验框架
`C/C++项目 → PCG/PSG → LLM源码改写 → LLVM-RVV → 板卡测量/验证 → Decision`。
#### 可行性与风险
需要 LLVM/RVV 工具链和板卡；风险是源码改写与后端优化相互干扰。

### 11.2 覆盖引导的安全项目优化
#### 研究问题
如何以覆盖率和差分测试约束跨文件性能改写？
#### 与原论文的区别
将有限功能测试扩展为覆盖/模糊测试/差分验证。
#### 可能的创新点
性能收益、覆盖增长和语义回归联合决策。
#### 实验框架
`候选改写 → 编译 → 单测/模糊/差分 → 性能测量 → 接受或回滚`。
#### 可行性与风险
可复用 POLO agent；风险是测试生成成本和性能噪声。

## 12. 与其他已读文献的关系

与 LiteCoOp 相比，POLO 操作 C/C++ 项目源码并依赖运行时，LiteCoOp 操作 TVM 张量 schedule；与 LLVM-Bench 相比，POLO 目标是性能改写而非 Issue 维护。POLO 可作为上游源码优化 baseline，LiteCoOp 作为张量调度 baseline，LLVM-Bench/LLVM-Gym 作为大型编译器维护验证模块，三者指标不应混用。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | C/C++ 项目级源码性能优化 |
| 核心问题 | 热点、跨模块关系和可验证改写的统一处理 |
| 输入/输出 | 项目+工作负载 / 优化后的源码项目 |
| 核心方法 | PCG/Func Rank、PSG、双 agent 多轮反馈 |
| 使用模型 | 两个 GPT-4o agent |
| 编译器工具 | Callgrind、Clang LibTooling、O0/O3、objdump |
| 强化学习/形式验证 | 无 RL；无形式化验证 |
| 数据集规模 | 6 个项目，测试覆盖多数 >80% |
| 主要指标 | 执行时间、PI，五次平均 |
| 最重要结果 | O3 PI 1.34×–21.5×；Quant 21.5× |
| 核心创新 | 运行时热点+全局结构+执行反馈闭环 |
| 主要局限 | 项目/输入有限，无 LLVM/RISC-V 实验 |
| RISC-V 相关性 | 中：源码级上游参考，未验证 RVV |
| 最适合作为 | 项目级源码优化 baseline/候选生成模块 |

这篇论文最值得学习的是让 LLM 看到热点的调用与结构上下文，并由真实执行结果决定是否保留改写；主要局限是项目规模、测试覆盖和硬件范围有限。后续使用应明确它是源码项目级优化，而不是直接照搬为 LLVM Pass 或 RISC-V 后端方法。
