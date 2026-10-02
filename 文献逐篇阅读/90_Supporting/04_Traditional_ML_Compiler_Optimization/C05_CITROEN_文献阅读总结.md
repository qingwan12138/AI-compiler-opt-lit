# CITROEN 文献阅读总结
代码/数据/工件：作者公开代码与数据：[LLVMTuner](https://github.com/gloaming2dawn/LLVMTuner)
元数据核验来源：[论文/来源](https://doi.org/10.1109/IPDPS64566.2025.00054)；[论文/来源](https://github.com/gloaming2dawn/LLVMTuner)

论文题目：**Leveraging Compilation Statistics for Compiler Phase Ordering**

作者：Jiayu Zhao、Chunwei Xia、Zheng Wang

发表时间：2025（在线发表 2025-07-23）。

发表平台：2025 IEEE International Parallel and Distributed Processing Symposium（IPDPS ’25），pp. 533–545；DOI [10.1109/IPDPS64566.2025.00054](https://doi.org/10.1109/IPDPS64566.2025.00054)。作者研究主页列出的代码与数据工件：[gloaming2dawn/LLVMTuner](https://github.com/gloaming2dawn/LLVMTuner)。

关键词：编译阶段排序、编译统计、贝叶斯优化、Gaussian Process、多模块调优

> 本文档基于 13 页作者版 PDF 全文整理。

## 1. 研究背景

LLVM 有大量优化 pass，顺序与交互使搜索空间极大。传统搜索方法把 pass 序列当作离散参数，难以知道某个序列实际触发了哪些变换；多源文件程序还需要决定把有限测量预算分给哪个模块（第1–2节）。

## 2. 论文要解决的问题

如何利用编译过程中已经产生的 pass statistics 描述“序列对 IR 做了什么”，在不运行低收益二进制的情况下提前拒绝候选，并动态为多模块程序分配 phase-ordering 搜索预算。

## 3. 核心方法概述

CITROEN 先用 `perf` 找出累计占 90% 运行时间的 hot modules。每个候选 pass 序列通过 LLVM `-stats -stats-json` 提取最多 255 维统计特征；Gaussian Process 从统计特征预测相对 `-O3` 的加速均值和不确定性。定制 acquisition function 优先探索训练集中未出现的非零统计类别；GA 从每个模块的优良序列中 mutation/crossover 生成候选。全局模型拼接各热模块特征，在每轮决定下一步优化哪个模块（第3节）。

## 4. 实验框架与训练流程

```text
-O3 profiling → 识别 hot modules
→ GA 为每个模块生成候选 pass 序列
→ 只编译并收集 statistics
→ GP 预测收益/不确定性
→ acquisition 选择一个候选实测
→ 更新全局 GP 与 GA 种群 → 循环
```

模型是在线训练的，不需要跨程序离线数据集。

## 5. 奖励函数、损失函数或关键公式

GP 使用 Matérn-5/2 kernel（第3.3节，式1–4），通过最小化负对数边际似然学习 lengthscale。标准 Expected Improvement 被改为：

```text
α(x) = EI(x) + 10^8，若 x 含训练集中未见的非零统计特征；
α(x) = EI(x)，否则。
```

优化目标是相对 `-O3` 的运行加速。

## 6. 实验设置

### 6.1 编译器与搜索空间

LLVM 17.0.6，76 个 pass，序列最长 120；GP 由 GPyTorch 实现，系统约 5K 行 Python。初始随机样本 `n_init=20`，每轮每模块候选数 `q=500`（第4节）。

### 6.2 硬件与基准

搜索服务器为双路 20 核 Xeon Gold 5218R；目标为 Jetson TX2 Cortex-A57 和 64 核 AMD Threadripper PRO 5995WX。测试 26 个 cBench 与 16 个 SPEC CPU2017 C/C++ 程序。

### 6.3 对比与测量

Random、OpenTuner、Nevergrad、BOCA、BaCO，以及 IR2Vec、AutoPhase、ProGraML 特征。cBench 预算 100/300/1000，SPEC 100/300；每个方法每基准重复 5 次，搜索反馈的运行时间相对标准误差低于 1%。

## 7. 实验结果与结论

在 cBench 的 100 次预算下，CITROEN 相对 `-O3` 达到 1.096×，基线为 1.067–1.083×；300 次达到约 1.11×，基线通常需 1000 次才能接近（第5.1节、图7）。论文总结其以约三分之一测量预算达到最强基线的效果；小预算下最高比随机搜索好 17%、比最强基线好 10%。消融显示 compilation statistics、OOD acquisition 和多模块调度都不可缺；收集 statistics 的开销不到编译开销的 0.05%（第5.6节）。

## 8. 主要创新点

### 8.1 用变换结果而非 pass 名称建模

statistics 直接反映 unroll、vectorize、CSE 等实际发生次数，更接近性能因果链。

### 8.2 稀疏特征空间的 OOD 探索

显式优先未见统计类别，缓解初始样本覆盖不足。

### 8.3 全局模型调度多模块预算

允许每个源文件采用不同序列，并动态切换搜索目标。

## 9. 局限性

仍需为大量候选执行编译并为被选候选多次运行程序；GP 在更高维、更大模块数时可能扩展困难。实验仅覆盖 LLVM 17、两类 CPU 和单线程程序，未验证 GPU、RISC-V 或跨编译器迁移。统计特征依赖 LLVM pass 的计数器实现，版本变化会改变特征语义；目标只报告运行时间，没有同时约束代码尺寸、能耗与编译时延。

## 10. 阅读后的研究方向反思

CITROEN 给当前主线的启发是“证据应来自实际触发的变换”，这比只喂 AutoPhase 静态特征更可信。若直接复制 GP+statistics 会撞上该工作；创新应加入跨架构证据迁移、分层不确定性或多目标安全门控。

## 11. 可进一步尝试的研究方向

### 11.1 跨架构一致性驱动的 phase ordering

#### 研究问题

x86 上有利的 pass statistics 模式，在 RISC-V 上何时仍有效、何时出现后端特有退化。

#### 与原论文的区别

从单目标/单设备在线 BO 扩展为共享变换证据、设备特定性能头的双架构搜索。

#### 可能的创新点

statistics 因果分组、跨 ISA 不确定性、退化风险约束。

#### 实验框架

```text
候选序列 → LLVM statistics
→ x86 快速筛选 + K3 少量复测
→ 共享 GP/多任务模型 → 鲁棒序列
```

#### 可行性与风险

LLVM statistics 与 ISA 无关部分可复用；K3 测量吞吐和噪声是主要瓶颈。

## 12. 与其他已读文献的关系

与 AutoPhase/CompilerGym 相比，CITROEN 不只看初始 IR，而观察 pass 真实统计；与 Protean 类似都做模块级 phase ordering，但 CITROEN 是编译器外部 BO，Protean 是编译器内 SA+预测模型；与 ECCO 的“证据因果”方向接近，但 CITROEN 的证据直接用于在线 surrogate，而非蒸馏为 LLM 推理。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 高效搜索 LLVM phase ordering |
| 核心问题 | pass 交互难建模、多模块预算难分配 |
| 输入/输出 | 程序与 pass 空间 / 模块特定序列 |
| 核心方法 | statistics 特征、GP、定制 EI、GA、全局调度 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否 |
| 数据集 | cBench 26、SPEC CPU2017 16 |
| 最重要结果 | 约三分之一预算达到基线相近结果 |
| 核心创新 | 用 pass statistics 代理 pass 交互效果 |
| 主要局限 | 在线实测仍贵、平台与目标单一 |
| 与 RISC-V 相关性 | 高，适合加入 K3 跨架构实测 |
| 最适合作为 | 证据驱动 phase-ordering 强基线 |

> 最值得保留的是“先观察变换是否真的发生，再决定是否值得运行程序”。
