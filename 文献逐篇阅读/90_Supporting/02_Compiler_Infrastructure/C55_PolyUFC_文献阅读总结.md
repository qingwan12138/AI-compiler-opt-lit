# C55 PolyUFC: Polyhedral Compilation Meets Roofline Analysis for Uncore Frequency Capping 文献阅读总结
发表平台：发表渠道待核验（本轮未找到可确认的正式出版记录）
论文链接或编号：论文编号待核验
元数据核验来源：本地收录论文 PDF；外部正式来源待核验
代码/数据/工件：未找到可确认的作者代码仓库或工件

论文题目：**PolyUFC: Polyhedral Compilation Meets Roofline Analysis for Uncore Frequency Capping**  
作者：Nilesh Rajendra Shah、M V V S Manoj Kumar、Dhairya Baxi、Ramakrishna Upadrasta。  
发表：CGO 2026 主会，pp. 563–576，DOI [10.1109/CGO68049.2026.11395211](https://doi.org/10.1109/CGO68049.2026.11395211)。来源：[CGO 官方论文页](https://2026.cgo.org/details/cgo-2026-papers/40/PolyUFC-Polyhedral-Compilation-Meets-Roofline-Analysis-for-Uncore-Frequency-Capping)、[作者项目页](https://compilers.cse.iith.ac.in/projects/polyufc/)。阅读本地 PDF 14 页。

## 1. 研究背景

数据中心 CPU 的 uncore（LLC、内存控制器和互连）频率会影响带宽、功耗和执行时间。传统编译器的仿射变换主要以性能为代价函数，不能在编译期判断某个程序阶段是否适合降低 uncore 频率来节能。PolyUFC 将 MLIR/Polygeist 的仿射分析与性能、功耗 roofline 模型结合，面向编译器驱动的频率封顶。

## 2. 论文要解决的问题

给定仿射程序、目标微架构和可用 uncore 频率，如何静态估计 operational intensity、执行时间、带宽和功耗，并为 compute-bound 与 bandwidth-bound 区域选择频率上限，使性能、能耗或 EDP 达到目标。关键难点是缓存冲突/容量未命中估计、频率相关的 roofline 参数，以及 MLIR 多层 dialect 上的插入粒度。

## 3. 核心方法概述

PolyUFC 由四部分组成：用 isl/barvinok 在仿射 IR 上计算操作强度和缓存未命中；用一次性微基准拟合不同 uncore 频率下的 DRAM、LLC 和功率参数；以频率和操作强度为变量建立性能/功耗/EDP 模型；在 linalg 或顶层 affine 操作前插入频率 cap 调用，并通过模式重写消除冗余设置。完整流程从 Polygeist/MLIR 降到 LLVM IR，再使用 Intel UFS 驱动执行。

## 4. 实验框架与训练流程

没有机器学习训练流程。模型参数来自硬件微基准和 roofline 拟合，优化目标通过离散频率搜索得到。论文实现了 MLIR analysis/pass，并在 Broadwell Xeon 1650-v4 与 Raptor Lake i5-13600 上测试。

## 5. 奖励函数、损失函数或关键公式

没有奖励或损失函数。核心量包括操作强度 `I=FLOPs/DRAM bytes`、频率参数 `fc`、性能/带宽 roofline、uncore 功耗模型和 `EDP=Energy×Time`。对每个候选频率计算性能损失、带宽变化和 EDP 变化，再按性能优先、能耗优先或 EDP 优先策略选择 cap。缓存分析假设操作具有单位 FLOP 权重，不区分 f16/f32 或具体算术操作代价。

## 6. 实验设置

编译器基线是默认 tile size 为 32 的 Pluto 并行 tiled kernel，硬件基线是 Intel 默认 uncore scaling driver。工作负载包括 PolyBench 的 gemm、mvt、2mm、stencil/solver 等，以及来自 AlexNet、ConvNeXt、WideResNet、BERT、Gemma 2、GPT-2 和 Llama 2 的 conv2d、matmul、SDPA。字符化阶段关闭预取和超线程，性能/能耗阶段按默认设置运行，并用 PAPI 和实际运行时间校验静态预测。

## 7. 实验结果与结论

静态 operational-intensity 分类与硬件结果一致，代表性 conv2d 的性能估计误差小于 7%。相对 Intel uncore scaling driver，compute-bound 工作负载的性能损失约 7% 以内且 EDP 最多改善约 42%；bandwidth-bound 工作负载的性能提升最高约 30%，EDP 改善最高约 54%。论文的证据是两种 x86 微架构和一组仿射内核，不能外推为所有 CPU 或非仿射程序的收益。

## 8. 主要创新点

1. 将性能与功耗 roofline 统一到 MLIR 仿射编译流程中，用静态操作强度指导 uncore cap。
2. 用 polyhedral cache analysis 补充 roofline，减少对运行时性能计数器的依赖。
3. 在多个 MLIR dialect 层级插入频率控制，并同时支持性能、能耗和 EDP 目标。

## 9. 局限性

1. 缓存和功耗模型依赖微架构专用微基准，跨 CPU 移植需要重新标定。
2. 单位 FLOP 假设忽略算术操作种类、数据类型和低层指令差异。
3. 目标主要是 affine/MLIR 可分析程序，复杂指针、动态控制流和 GPU 场景未覆盖。
4. 频率搜索和运行时 cap 调用自身有开销；论文没有给出长期数据中心部署成本。

## 10. 阅读后的研究方向反思

PolyUFC 的价值在于把“编译器变换”和“硬件功耗控制”放入同一个可解释模型。对 RISC-V 而言，可将 uncore cap 替换为 DVFS、内存控制器或片上网络的可编程旋钮，并把 ISA 扩展暴露为 MLIR 属性；但需要先建立平台特定的带宽、功率和频率测量接口。

## 11. 可进一步尝试的研究方向

1. 将模型扩展到 RISC-V RVV 的向量宽度、缓存层次和内存控制器频率。
2. 用硬件计数器在线校准 roofline 参数，并对模型误差提供置信区间。
3. 将频率 cap 选择与 tiling、fusion、prefetch 等变换联合优化。
4. 比较静态 cap、运行时反馈控制和学习型代价模型的编译时间及能耗收益。

## 12. 与当前批次文献的关系

PolyUFC 与 C51 的符号矩阵链编译都使用静态代价模型和多版本/多配置选择，但 PolyUFC 的选择变量是硬件频率与能耗目标，C51 的选择变量是运行时矩阵尺寸。与 C20/C47 的 ISA 后端论文相比，PolyUFC 位于 MLIR 中端与系统控制交界处，没有引入 RISC-V 指令选择。

## 13. 一页式总结

PolyUFC 把 polyhedral 分析、缓存估计和性能/功耗 roofline 组合成 MLIR 编译流程，为仿射程序选择 uncore 频率上限。14 页 CGO 论文在 Broadwell 与 Raptor Lake 上报告了约 42%（compute-bound）和 54%（bandwidth-bound）的最大 EDP 改善，同时保持有限性能损失。其主要贡献是可解释、可迁移的编译期硬件控制接口；主要风险是模型对微架构标定和仿射假设的依赖。
