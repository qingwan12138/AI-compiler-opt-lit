# Protean Compiler 文献阅读总结
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）
元数据核验来源：[论文/来源](https://doi.org/10.1145/3831596)

论文题目：**Protean Compiler: An Agile Framework to Drive Fine-grain Phase Ordering**

作者：Amir H. Ashouri、Shayan Shirahmad Gale Bagi、Kavin Satheeskumar、Tejas Srikanth、Jonathan Zhao、Ibrahim Saidoun、Ziwen Wang、Bryan Chan、Tomasz S. Czajkowski

发表时间：2026（arXiv 首次提交 2026-02-05；ACM TACO 正式版本 2026-08）。

发表平台：ACM Transactions on Architecture and Code Optimization（TACO），23(3), 2026, pp. 1–26；DOI [10.1145/3831596](https://doi.org/10.1145/3831596)。arXiv v3 标注为该接收版本的预印本。

关键词：LLVM、细粒度 phase ordering、IR2Score、Simulated Annealing、模块级优化、AArch64

> 本文档基于 29 页预印本 PDF 全文整理。

## 1. 研究背景

多数 phase-ordering 系统是包在编译器外的 Python/工具链，难以进入正常构建流程；同一全程序序列也不一定适合每个 module/function/loop。MLGO/ACPO 等虽集成 LLVM，但主要替换单个 pass 的 profitability，而不是控制整个 pass manager（第1–2节）。

## 2. 论文要解决的问题

如何把细粒度 phase ordering 直接集成进 Clang/LLVM，按代码分区并行搜索不同优化 recipe，同时用低开销的静态特征与预测模型减少真实运行评估和构建时间。

## 3. 核心方法概述

Protean 给 Clang 增加 `-OP` 优化级别和 agile driver，把程序按 module 等粒度分区。每次迭代从 5 组人工聚类的 pass subsequences 组成 recipe，应用到分区 IR，再由 IR2Score 预测速度。搜索以 Simulated Annealing 为主，GA recommender 在第 20 轮后利用已有优良 recipe 加速探索。IR2Score 可吃自研 140+ 维 Protean Feature Set（PFS）或 IR2VEC；模型 AOT 编译并通过 MLGO/ACPO 接口在 LLVM 内调用（第3节）。

## 4. 实验框架与训练流程

```text
源码 → Clang -OP → LLVM IR 分区
→ 组合 subsequence recipe → opt 变换
→ PFS/IR2VEC 特征 → IR2Score 预测
→ SA 接受/拒绝 + GA 推荐
→ 每分区最优 recipe → 链接 → 可执行程序
```

IR2Score 使用 12K 个 PolyBench、CORAL-2、CoreMark C/C++ 函数训练；真实运行时间用于构造模块级速度标签。

## 5. 奖励函数、损失函数或关键公式

SA 对更优分数直接接受，对更差分数按温度相关指数概率接受（第3节，式1），温度逐步下降。IR2Score 用 weighted MSE，额外提高预测速度小于 0.8 或大于 1.5 样本的权重；训练目标为预测 recipe 相对 `-O3` 的模块速度。

## 6. 实验设置

### 6.1 编译器、模型与训练

LLVM 19；IR2Score 为 4 层 Transformer Encoder、10 个 attention heads、FFN 512，20 epochs、10-fold、batch 32、学习率 1e-4、Adam。PFS 140+ 静态特征，另支持 IR2VEC（第3.5、4节）。

### 6.2 硬件与测量

Kunpeng 920 AArch64/ARMv8.2，2.6GHz；`perf` 单线程计时，`numactl` 绑核/绑 NUMA，运行 5 次，方差超过 1% 则重测。评测实际采用 module granularity。

### 6.3 基准

25 个 cBench 程序，搜索 10/50/100/500 轮；SPEC CPU2017 Integer 中 9 个 C/C++ 应用，搜索 30 轮，测 1-copy/64-copy。另演示 ACPO inlining 和 LLM 源码优化集成。

## 7. 实验结果与结论

cBench 上平均相对 `-O3` 加速从 2.2% 提升到 4.1%，个别程序最高约 15.7%；PFS 的 500 轮几何平均为 1.041×（第4.1节、表4）。SPEC 1-copy/64-copy 平均加速 1.0%/1.1%，但平均二进制增大约 1.4%；`557.xz` 在 64-copy 下最高约 6.5%（表5）。PFS 平均约 145 轮早停，IR2VEC 约 310 轮；64 核并行能显著降低构建时间。与 MiCOMP 的间接比较中，500 轮为 1.041× 对 1.031×（表9）。与 ACPO/LLM 两级组合在选定 Jpeg/Susan 案例达 8.5%/10.1%。

## 8. 主要创新点

### 8.1 编译器内生的 phase-ordering driver

不是外部 wrapper，而是正常 Clang/LLVM 流程中的新优化级别。

### 8.2 分区级不同 recipe

框架声称支持 module/call graph/function/loop，避免全程序一刀切；论文重点验证 module。

### 8.3 AOT 预测模型与编译并行

静态特征、模型函数调用、早停和多核构建共同控制调优开销。

## 9. 局限性

论文虽然声称支持多种粒度，但主要实证是 module-level；函数/循环级的收益与组合正确性未充分展示。模型训练标签仍依赖 `perf` 动态测量，跨架构复用没有验证。MiCOMP 因工具版本问题采用代理式间接比较，不是完整同条件复现。部分收益伴随代码尺寸膨胀；ACPO/LLM 组合只测试选定个案。预印本称代码计划未来开源，当前可复现性受限。

## 10. 阅读后的研究方向反思

Protean 已占据“编译器内、细粒度、预测模型+SA/GA”的系统位置。新的框架若仍是 module 切分+模型打分+经典搜索，创新会高度重合。可考虑把重点转向可插拔证据协议、跨设备成本校准、验证门控或极低预算部署，而非再造 agile driver。

## 11. 可进一步尝试的研究方向

### 11.1 分层预算与跨设备停止策略

#### 研究问题

不同 module 对最终性能贡献不同，如何联合决定“优化哪个 module、在 x86/K3 上各测几次、何时停止”。

#### 与原论文的区别

Protean 每个分区迭代；扩展工作将 module 价值、设备测量成本和不确定性纳入统一预算分配。

#### 可能的创新点

分层 bandit、跨设备成本转移、收益下界早停。

#### 实验框架

```text
模块热点 → 快速模型评分 → 分层预算器
→ x86/K3 选择性测量 → 更新置信度 → 提前停止
```

#### 可行性与风险

可复用 LLVM 构建系统；难点是模块独立优化与 LTO/跨模块效应冲突。

## 12. 与其他已读文献的关系

与 CITROEN 都做 module-specific phase ordering：CITROEN 是外部 BO+statistics，Protean 是 LLVM 内 IR2Score+SA/GA；与 ECCO 都采用模型+GA，但 Protean 模型预测性能、ECCO LLM 输出语义意图；与 ACPO/MLGO 的关系是将其作为可集成的单 pass 决策层。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLVM 内生的细粒度 phase ordering |
| 核心问题 | 外部 wrapper、全程序统一序列、构建成本 |
| 输入/输出 | C/C++ / 分区特定 recipe 与二进制 |
| 核心方法 | `-OP` driver、subsequence、IR2Score、SA+GA |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否 |
| 实验平台 | Kunpeng 920 AArch64 |
| 最重要结果 | cBench 平均最高 4.1%，SPEC 平均约 1% |
| 核心创新 | 直接集成 LLVM 的分区级 agile phase ordering |
| 主要局限 | 实证粒度有限、跨架构未验证、代码未开源 |
| 与 RISC-V 相关性 | 中高，框架可移植但需重新训练/校准 |
| 最适合作为 | 系统级强竞品与开销对比基线 |

> Protean 的门槛在“真正进入编译器构建流程”，而不仅是得到更高搜索收益。
