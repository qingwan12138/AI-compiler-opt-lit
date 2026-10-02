# OML-vect 文献阅读总结
代码/数据/工件：作者公开实现及 artifact：[CAPS-UMU/onnx-mlir-caps](https://github.com/CAPS-UMU/onnx-mlir-caps/tree/oml-vect-cgo-2026)，位于 `oml-vect-cgo-2026` 分支。

论文题目：**Enabling Automatic Compiler-Driven Vectorization of Transformers**

作者：Shreya Alladi、Alberto Ros、Alexandra Jimborean

发表时间：2026

发表平台：CGO 2026，DOI: 10.1109/CGO68049.2026.11395226
元数据核验来源：[CGO 2026 官方议程](https://2026.cgo.org/details/cgo-2026-papers/29/Enabling-Automatic-Compiler-Driven-Vectorization-of-Transformers)；[DOI](https://doi.org/10.1109/CGO68049.2026.11395226)

关键词：RISC-V、自动向量化、MLIR、ONNX-MLIR、归约识别、数据布局

> 本文档基于 15 页作者公开 PDF（含工件附录）全文整理。

## 1. 研究背景

Transformer 在边缘设备上的部署常依赖为 x86/ARM 手工优化的算子库，移植到 RISC-V 代价高。ONNX-MLIR 虽能逐级 lowering，但默认不直接暴露 LLVM IR/生成可独立运行的 RV64 二进制；更关键的是，MLIR affine super-vectorizer 依赖归约映射且会拒绝非连续内存访问，导致看似规则的 MatMul 也无法自动向量化（第 1–3 节）。

## 2. 论文要解决的问题

论文要打通 ONNX→MLIR→LLVM→RISC-V 可执行文件的静态编译链，并回答两个具体问题：如何跨 dialect 识别 FMA、SoftMax 等复杂归约；如何在不增加运行时转置开销的前提下，把 MatMul 输入变成连续访问布局，使现有 MLIR/LLVM 向量器愿意且能够向量化。

> 本文主要研究：通过归约识别与编译期布局变换激活上游 MLIR 自动向量化，并在 x86 与真实周期精确的 RVV 核上比较手工优化基线。

## 3. 核心方法概述

```text
ONNX 模型
→ ONNX-MLIR 高层 lowering
→ 编译期转置/transpose 合并，使 MatMul 的 B 连续访问
→ 降至 affine/scf 等中层 dialect
→ OML-reduction 跨 arith/math 分解 def-use 链并构造 reductionMap
→ 调用 MLIR affine super-vectorizer
→ vector/LLVM dialect → LLVM IR
→ LLVM -O3、RVV 后端与已移植 runtime 库
→ 静态链接的 x86 或 RV64 可执行文件
```

对静态权重，直接在编译期改写权重布局；对运行时输入，利用上游 transpose 节点并组合 permutation，避免新增运行时复制。归约 pass 识别 `math.fma`、`arith.maxnumf` 等默认测试遗漏的模式（第 4 节）。

## 4. 实验框架与训练流程

本文不训练模型。作者扩展 ONNX-MLIR 形成 OML，再叠加 reduction 与 data-layout 两个 pass 形成 OML-vect；同时交叉编译 ONNX-MLIR runtime，修复 RV64 不兼容库并静态链接。实验工件提供 LLVM/MLIR 20、ORT 1.23.2、Docker、交叉编译和 CSV 生成脚本，估计需要约 80 GB 磁盘（工件附录）。

## 5. 奖励函数、损失函数或关键公式

没有 RL 奖励或学习损失。核心判定来自编译器分析：OML-reduction 沿 innermost loop 的 iter-arg/def-use 链定位 combiner 与 reduction position，构造 `reductionMap`；布局 pass 组合原 transpose permutation 与交换最后两维的 permutation。性能以相对 ONNX-MLIR-default（启用手工优化）的归一化运行时间和提升百分比计算。

## 6. 实验设置

工作负载实际为 8 个：7 个 Transformer/注意力模型（auto Opset18、bert google、bgesmallenv、deberta、distilbert、phonemizerbig、roberta）和 1 个 RNN 降噪模型 nsNet2。论文正文有一句误写为“eight Transformer models”，但清单与前句均对应 7+1。平台为 Intel Xeon E5-2630 v4 2.20 GHz，以及 Xilinx U55C FPGA 上 100% cycle-accurate 仿真的 Atrevido 423 RV64 核（512-bit 向量单元）。因 RVV 仿真较慢，主要测最常见 attention layer/nsNet2 的 MatMul+GRU 层，并说明完整模型趋势一致。

比较五种配置：无手工优化的 ONNX-MLIR、仅打通 LLVM 的 OML、带手工优化的 ONNX-MLIR baseline、自动 OML-vect，以及 ONNX Runtime。指标包括归约覆盖、运行时间、指令数、二进制大小、内存和编译时间（第 5 节）。

## 7. 实验结果与结论

OML-reduction 的归约覆盖最高达到默认 MLIR 的 2.5×。只增加归约识别仍无性能收益，因为非连续访问使 MLIR/LLVM 代价模型拒绝向量化；与编译期布局变换组合后，OML-vect 相对手工优化 ONNX-MLIR baseline 在 x86 平均提升 5%、RISC-V 提升 59%，并在 RVV 上 8 个模型中 4 个超过 ORT（第 6.1–6.2 节）。

相对手工 ONNX-MLIR，RVV trace 中 OML-vect 的 vector load 减少 18.83%、store 减少 99%；二进制大 2.67%，但指令数少 2.2%。x86 VTune 显示内存占用少 35%，论文明确说明 RVV 缺少相同 profiling 工具，不能外推该内存结果。RISC-V 编译时间从 11 秒增至 23.1 秒，x86 从 9.5 秒降至 8.2 秒。

需要特别记录元数据差异：CGO 网页摘要一度显示“94%/91%”及“2%/8%”，而作者 PDF 的摘要、正文和结论一致报告 **5%/59%**；本笔记以 PDF 正文为准。

## 8. 主要创新点

### 8.1 面向生产 lowering 的 OML 静态编译链

暴露 MLIR/LLVM IR、移植 runtime 并生成独立 RV64 可执行文件，使自动中低层优化能真正到达 RISC-V 后端。

### 8.2 跨 dialect 的复杂归约识别

不局限于 `arith` 的简单二元模式，能分解三元 FMA、SoftMax max 等归约并向 affine super-vectorizer 提供完整映射。

### 8.3 编译期布局变换与现有向量器协同

针对静态权重和已有 transpose 分别改写/合并布局，在零运行时转置开销下把非连续访问转为连续访问，而非另写一个专用向量代码生成器。

## 9. 局限性

RISC-V 平台是 U55C 上的周期精确 FPGA 仿真核，并非量产板的完整系统评测；主要测 attention layer 而非所有端到端模型。512-bit 向量宽度与具体 cache/展开参数影响结果，跨 VLEN 泛化未验证。baseline 含手工算法、ORT 还存在无法完全关闭的 x86 多线程，比较并非严格同等并行度。布局优化依赖静态权重或恰有可合并 transpose；卷积即使识别归约也未必可向量化。论文没有形式化证明 pass 等价性，也没有报告 RVV 内存占用。

## 10. 阅读后的研究方向反思

论文最强的证据不是“自动向量化胜过手工库”，而是证明归约识别与布局必须联合：单独增加可识别模式无法越过代价模型。对 RVV 研究，应进一步问同一变换在不同 VLEN、LMUL、cache 和 `vsetvli` 策略下何时有益，并用真机 PMU 校准 MLIR/LLVM 的拒绝理由，而不是只在固定 512-bit 核上报告均值。

## 11. 可进一步尝试的研究方向

### 11.1 跨 VLEN 的证据驱动布局—向量化联合决策

#### 研究问题

如何让 MLIR 在编译期联合选择 transpose、tile、unroll、LMUL 与归约向量化，并在多种 RVV VLEN/缓存配置上保持收益。

#### 与原论文的区别

原文用规则固定地准备布局后调用现有向量器；新方案把代价模型拒绝原因、真机 PMU 和多硬件测量纳入决策，允许选择“不转置”或不同 tile。

#### 可能的创新点

参数化布局收益模型、跨 VLEN 鲁棒目标、编译器解释的失败证据、在线/离线校准，以及 pass 等价性差分验证。

#### 实验框架

```text
ONNX/MLIR 模型 → 候选布局与归约映射
→ LLVM/RVV 合法化与代价解释
→ 多 VLEN 仿真/真机 PMU 测量
→ 更新联合代价模型 → 选择配置
→ 端到端正确性与性能验证
```

#### 可行性与风险

可先复用公开 OML-vect 工件并限制 8 个 attention kernel；风险是硬件稀缺、测量噪声、搜索成本和浮点重排语义。

## 12. 与其他已读文献的关系

与 C12 xDSL→RVV lowering 都补齐 MLIR 到 RVV 的缺口：C12 以自定义 xDSL lowering 生成 intrinsic C，OML-vect 尽量复用上游 affine/vector/LLVM 链；与 C20 形式规格驱动指令选择互补，本文解决中层布局/归约，C20 解决 gMIR→ISA 规则；与 Closer in the Gap 的真机成本模型诊断形成后续验证路线；与 IntrinTrans/VecIntrinBench 的手工 intrinsic 迁移不同，本文目标是避免源级架构专用代码。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | Transformer 的 MLIR 自动向量化与独立 RISC-V 二进制生成 |
| 输入/输出 | ONNX 模型 / 自动向量化的 x86、RV64 可执行文件 |
| 核心方法 | OML lowering、跨 dialect 归约识别、编译期布局转置、MLIR super-vectorizer |
| 使用的模型 | 无学习模型 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；依赖编译器与运行测试 |
| 数据集规模 | 7 个 Transformer + 1 个 RNN，共 8 个 workload |
| 主要指标 | 归约数、运行时间、load/store、体积、指令数、内存、编译时间 |
| 最重要结果 | 相对手工 ONNX-MLIR：x86 +5%、RISC-V +59%；归约覆盖最高 2.5× |
| 核心创新 | 归约识别与零运行时布局变换联合激活上游自动向量器 |
| 主要局限 | 固定 512-bit RVV 仿真核、层级评测、比较并行度不完全一致、无形式验证 |
| 与 RISC-V 相关性 | 很高；生成 RV64 静态二进制并在 Atrevido 423 RVV 上实测 |
| 最适合作为 | MLIR→RVV 自动向量化与布局协同基线 |

> 最值得复用的是对“向量器为什么拒绝”的分层诊断与联合修复；最值得补强的是跨 VLEN 真机验证和更严格的同等并行度比较。
