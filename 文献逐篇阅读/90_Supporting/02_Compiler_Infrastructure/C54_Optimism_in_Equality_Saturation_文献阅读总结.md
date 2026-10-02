# C54 Optimism in Equality Saturation 文献阅读总结
发表平台：PLDI 2026 / PACMPL 10(PLDI), Article 224
论文链接或编号：DOI 10.1145/3808302
元数据核验来源：[PLDI 官方论文页](https://pldi26.sigplan.org/details/pldi-2026-papers/59/Optimism-in-Equality-Saturation)；[ACM DOI](https://doi.org/10.1145/3808302)；[作者代码工件](https://github.com/RArbore/pldi26-artifact)；[Zenodo 工件](https://zenodo.org/records/19581777)
代码/数据/工件：作者代码：[PLDI artifact](https://github.com/RArbore/pldi26-artifact)；[Zenodo 工件](https://zenodo.org/records/19581777)

论文题目：**Optimism in Equality Saturation**
作者：Russel Arbore、Alvin Cheung、Max Willsey。
发表：PLDI 2026，Proceedings of the ACM on Programming Languages 10(PLDI)，Article 224；DOI [10.1145/3808302](https://doi.org/10.1145/3808302)。首次公开于 2025-11，arXiv:2511.20782。
来源：[PLDI 官方论文页](https://pldi26.sigplan.org/details/pldi-2026-papers/59/Optimism-in-Equality-Saturation)、[arXiv 记录](https://arxiv.org/abs/2511.20782)、[作者 camera-ready PDF](https://www.russelarbore.com/PLDI2026_OptimismInEqsat_CameraReady.pdf)。阅读最终稿 PDF 24 页。
代码与工件：[作者代码仓库](https://github.com/RArbore/pldi26-artifact)；[Zenodo 工件记录](https://zenodo.org/records/19581777)。
venue 核验来源：[PLDI 官方论文页](https://pldi26.sigplan.org/details/pldi-2026-papers/59/Optimism-in-Equality-Saturation)。发表信息：PLDI 2026，Proceedings of the ACM on Programming Languages 10(PLDI)，Article 224，DOI [10.1145/3808302](https://doi.org/10.1145/3808302)。

## 1. 研究背景

Equality saturation 使用 e-graph 同时保存多个等价程序，再由 cost model 抽取实现。传统数据流分析常在 CFG/SSA 上利用控制流与 def-use 关系；若将分析直接提升到 e-graph，等价重写可能形成循环或不良结构，使朴素的乐观分析出现不健全结果。论文探讨如何在 equality saturation 中利用乐观数据流信息，同时保持 soundness。

## 2. 论文要解决的问题

核心问题是：如何对 e-graph 表示的程序实施有用的流敏感/迭代分析，而不把 e-graph 中的循环等价结构误当成普通程序控制流并推出错误事实？作者还研究分析与重写如何交替执行，以及该方法能否支持区间、全局值编号和可达性等分析。

## 3. 核心方法概述

方法将程序的 CFG/DFG SSA 语义与 e-graph 表示区分开来。作者识别并标记控制流图的 WTO（weak topological ordering）回边，对乐观 e-class 分析限制回边遍历次数，在受控的路径展开上计算 sound 的分析结果；分析与 equality saturation 阶段交替，直到稳定。论文给出 soundness 定理，论证结果对 e-graph 所表示的良构程序成立。实现包含 interval、global value numbering（GVN）及 reachability 等分析。

## 4. 实验框架与训练流程

无机器学习训练或奖励信号。论文实现 Rust 原型，在两个人工构造程序上展示能力，并在 100 个随机生成程序上运行分析；比较包括标准 abstract interpretation、标准 e-class analysis 与 optimistic e-class analysis，也将示例交给 GCC 15.2 和 Clang 21.1 `-O2` 检查优化结果。评测记录运行时间与迭代数，时间取 25 次运行的平均值；硬件为 Intel Core Ultra 7 155H 1.4GHz、32GB 内存的笔记本。随机程序结果只代表所构造生成器覆盖的样本，不等同于生产软件分布。

## 5. 奖励函数、损失函数或关键公式

本文没有奖励函数或损失函数。关键机制是限制 WTO 回边的分析展开次数，并在 equality saturation 与 analysis 间迭代到稳定点；正确性主张由 soundness theorem 给出，而不是从实验观察归纳。实验中迭代数 `n` 最大为 6，99% 随机程序的 `n≤3`。该 `n` 是收敛行为统计，不是理论复杂度上界。

## 6. 实验设置

评估数据为 2 个手工案例和 100 个随机生成程序；最大 e-graph 规模为 11,375 个 e-nodes，最大循环数为 184。比较三种分析变体：标准 abstract interpretation、标准 e-class analysis、optimistic e-class analysis。正文报告每种方法重复 25 次的时间统计，设备配置如第 4 节所述。GCC/Clang 对比针对两个设计案例，而不是完整 100 程序上的系统性编译器基准。原文主要采用非关系型分析，随机评估也是 flow-insensitive 设定。

## 7. 实验结果与结论

Table 1 的运行时间（微秒，median / mean / max）为：标准 abstract interpretation 252.95 / 755.35 / 7244.56；标准 e-class analysis 160.87 / 478.59 / 2320.57；optimistic e-class analysis 448.64 / 1480.11 / 7723.35。由此可见，提出的 optimistic 变体在该实现和样本上较慢，不能声称性能优于基线。两个手工程序展示了其他方法或 GCC 15.2、Clang 21.1 `-O2` 未完成相同优化的情形；100 个随机程序均满足论文所述 soundness 检查。结果支持方法可行性与能力示例，但作者将工作定位为偏理论的定性能力扩展，定量精度优势和普遍性能收益尚未证明。

## 8. 主要创新点

1. 指出朴素 optimistic e-class analysis 可能被 e-graph 的循环/非良构表示破坏 soundness。
2. 利用 CFG/DFG SSA 语义及 WTO 回边受限遍历，构造 sound 的乐观分析方法。
3. 将分析与 equality saturation 交替运行，并给出针对良构所表示程序的 soundness 定理。
4. 用 interval、GVN、reachability 和小规模原型展示框架覆盖面。

## 9. 局限性

1. 实验只有 2 个定制案例和 100 个随机程序；程序生成器及小型原型限制了外推范围。
2. 本方法时间开销在所报样本上高于两个基线：平均 1480.11 µs 对 755.35 µs、478.59 µs；性能优化仍是开放问题。
3. 实验以 flow-insensitive、非关系型分析为主，尚未显示复杂 flow-sensitive/relational domains 的效果。
4. 作者指出工作偏理论、能力提升主要为定性展示；还需找到能显示精度或优化收益的分析领域。
5. soundness 定理的范围是论文定义的良构表示与语义，不应扩展为对任意 e-graph 或任意编译器集成的保证。

## 10. 阅读后的研究方向反思

该文最重要的工程启示是：e-graph 的等价关系结构不能无条件替代程序的控制流语义；分析必须明确区分表示中的循环与源程序循环。其时间数据也提醒，理论上更强的分析能力会引入迭代成本。用于真实编译器时，应联合衡量抽取质量、分析精度、e-graph 膨胀、收敛轮数和总体编译时间，而不能只报告能否找到某个优化。

## 11. 可进一步尝试的研究方向

1. 在真实编译器优化基准上量化新增分析带来的代码质量收益和编译时开销。
2. 扩展至 relational 或 flow-sensitive domain，并比较精度增益是否抵消额外成本。
3. 研究增量式回边处理、缓存与停止准则，降低最多 6 轮交替迭代及大 e-graph 下的成本。
4. 将 soundness 条件接入现有 equality-saturation 框架，并对 rewrite、分析和抽取边界做机器可检验验证。

## 12. 与当前批次文献的关系

同批 C53 研究控制流分支重排下函数合并的参数化算法；C54 研究 equality-saturation e-graph 上数据流分析的 soundness 与运行代价。二者共同关注编译优化算法在控制流/等价表示中的正确性边界，但 C53 无 benchmark，C54 有小规模原型测量且报告了性能开销。两篇都归 SUPPORTING/B2，不以 LLM 为系统核心角色。

## 13. 一页式总结

本文为 equality saturation 引入一种通过 WTO 回边受限遍历实现的 optimistic e-class analysis，并以形式化定理保障论文语义范围内的 soundness。两个手工程序展示潜在优化能力，100 个随机程序上运行 interval、GVN、reachability 分析；最大图为 11,375 e-nodes、最多 184 个循环。25 次运行的平均时间显示提出方案（1480.11 µs）慢于标准 abstract interpretation（755.35 µs）及 e-class analysis（478.59 µs），所以主要贡献是分析能力与正确性框架，而非速度提升。下一步应在真实优化基准中检验精度、收益与代价。
