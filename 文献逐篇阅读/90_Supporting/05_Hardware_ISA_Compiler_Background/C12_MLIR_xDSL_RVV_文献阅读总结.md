# MLIR-xDSL RVV Lowering 文献阅读总结
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

论文题目：**Enabling RISC-V Vector Code Generation in MLIR through Custom xDSL Lowerings**

作者：Jie Lei、Héctor Martínez、Adrián Castelló

发表时间：2026

发表平台：arXiv:2603.17800
元数据核验来源：[arXiv:2603.17800](https://arxiv.org/abs/2603.17800)

关键词：RISC-V、RVV、MLIR、xDSL、GEMM、微内核

> 本文档基于论文 PDF 全文整理。

## 1. 研究背景

MLIR 适合表示多层次编译过程，但论文使用的 LLVM/MLIR 22 尚缺少把动态 `memref` 和 RVV 操作完整落到可部署 C/C++ 的现成路径。xDSL 允许用 Python 快速定义 dialect 与 pass，因此可作为 MLIR 生态的轻量扩展层，在不修改上游 MLIR 的情况下补齐 RVV 代码生成。

## 2. 论文要解决的问题

如何从高层矩阵乘微内核表示出发，经由自定义 xDSL lowering 生成使用 RVV 1.0 intrinsic 的 C++，并在真实 RISC-V 开发板上获得超过通用 OpenBLAS 的性能。

## 3. 核心方法概述

系统先根据向量长度、数据类型和微内核尺寸生成 xDSL IR；随后通过自定义 `MemRefToEmitCPass` 与 `RVVToEmitCPass` 把内存操作和 RVV dialect 降到 EmitC。RVV 操作被改写为 `emitc.call_opaque` 对应的标准 intrinsic 调用，再用 `mlir-translate` 输出可独立编译的 C++ 函数。系统枚举不同 `mr×nr` 微内核，在目标板实测后为 GEMM 层选择合适配置。

## 4. 实验框架与训练流程

```text
向量长度、数据类型、kernel 尺寸
→ xDSL 生成高层微内核
→ MemRefToEmitC + RVVToEmitC
→ MLIR EmitC
→ mlir-translate 生成 C++
→ RISC-V 交叉/本机编译
→ 开发板搜索微内核并测试 GEMM
```

论文不训练机器学习模型；“搜索”指枚举微内核尺寸并用真实性能选择。

## 5. 奖励函数、损失函数或关键公式

无奖励函数或训练损失。GEMM 计算量按常见的 `2MNK` 浮点操作计，性能以 GFLOPS 报告。论文的核心选择依据是目标硬件实测时间，而非学习型代价模型。

## 6. 实验设置

### 6.1 软件环境

Python 3.13.9、xDSL 0.54.3、LLVM/MLIR 22、g++ 14，编译参数包含 `-march=rv64gcvzfh -mabi=lp64d`；对照库为 OpenBLAS 0.3.31。

### 6.2 硬件环境

- K230 CanMV：C908 1.6GHz、RVV 1.0、128-bit 向量、32KB L1D、256KB L2。
- BananaPi F3：SpaceMiT K1、8 核 1.6GHz、RVV 1.0、256-bit 向量、每核 32KB L1D、每四核共享 512KB L2。

两平台实验均为单核 FP32。

### 6.3 工作负载

微内核搜索覆盖 `mr=1…32`、`nr=1…16`，每个配置运行 1500 次；GEMM 运行 200 次。矩阵包括边长 1000–5000 的五组方阵，以及五组 BERT-Large 形状。

## 7. 实验结果与结论

K230 的最佳微内核为 `20×6`，达到 8.1 GFLOPS；BananaPi F3 的最佳配置为 `16×15`，达到 16.2 GFLOPS。方阵上，K230 的生成代码约 5.1–5.4 GFLOPS，较 OpenBLAS 的 4.5–4.9 GFLOPS 高约 10%–15%；F3 上约 8.3–8.6 GFLOPS，较 6.1–7.0 GFLOPS 高约 20%–35%。BERT 形状收益更明显，论文报告最高约 2.4×；结果说明面向层形状选择微内核比固定通用内核更有效。

## 8. 主要创新点

### 8.1 用 xDSL 补齐 MLIR 的 RVV lowering

在上游支持不完整时，以外部 dialect/pass 快速打通 RVV 到 EmitC 的路径。

### 8.2 生成可独立部署的标准 intrinsic 代码

输出不是仅能在编译器内部执行的 IR，而是可由普通 RISC-V C++ 工具链编译的函数。

### 8.3 将微内核生成与目标板实测选择结合

依据 RVV 宽度和矩阵形状选择内核，在两块真实开发板上验证性能。

## 9. 局限性

实验只覆盖单核 FP32 GEMM 与两种开发板，尚未验证卷积、归约、混合精度或端到端模型。微内核空间仍以枚举和实测为主，缺少可迁移代价模型。生成标准 intrinsic 并不等价于控制最终汇编质量；数据 packing、缓存分块和多核调度也未形成完整自动生成流程。

## 10. 阅读后的研究方向反思

这篇论文与现有 RISC-V 板卡条件高度契合：它给出了“MLIR/xDSL 变换—RVV intrinsic—真机测量”的短闭环。不过，复现论文只能证明工程能力，论文创新仍需放在自动选择、跨板迁移或验证反馈上。用户的 K3 板卡不是论文中的 K1/F3，移植时必须重新确认核心、向量长度、缓存和工具链，不能直接沿用论文最优微内核。

## 11. 可进一步尝试的研究方向

### 11.1 面向多款 RVV 开发板的可迁移微内核选择

#### 研究问题

能否用少量 K3 实测数据，从已有 K230/K1 数据快速找到不同 GEMM 形状的近最优微内核。

#### 与原论文的区别

原论文逐板枚举；新方向关注硬件特征驱动的跨板迁移和低样本适配。

#### 可能的创新点

把 VLEN、缓存、矩阵形状、寄存器压力与实测 PMU 组合成可解释的代价模型，并保留真实性能校准。

#### 实验框架

```text
微内核参数 + 硬件特征
→ 代价模型给出候选
→ K3 少量实测
→ 校准模型
→ 生成 RVV C++
→ 正确性与性能验证
```

#### 可行性与风险

已有多块 RISC-V 板卡可提供数据；风险是不同芯片 PMU、编译器和频率控制不一致，需要固定测量协议。

## 12. 与其他已读文献的关系

与 RVV 概率程序/MetaSchedule 工作一样依赖真机搜索，但本文强调 xDSL/MLIR lowering 与可部署代码；与 TCL 的跨硬件代价模型互补，可把本文的微内核空间作为 TCL 式迁移学习对象；与 MLIR Transform Dialect 结合后，可把离散 kernel 尺寸变换显式化并交给统一调优器。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 为 MLIR/xDSL 打通 RVV 代码生成并优化 GEMM |
| 核心问题 | 上游 lowering 不完整，通用库未针对层形状选内核 |
| 输入/输出 | 微内核参数与 MLIR/xDSL IR / RVV intrinsic C++ |
| 核心方法 | 自定义 dialect lowering、EmitC、真机微内核搜索 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否，使用数值测试 |
| 实验平台 | K230/C908 与 BananaPi F3/SpaceMiT K1 |
| 最重要结果 | 相对 OpenBLAS 最高约 2.4× |
| 核心创新 | 轻量补齐 MLIR→RVV 可部署路径 |
| 主要局限 | 单核 FP32 GEMM、两块板、枚举成本高 |
| 与 RISC-V 相关性 | 高，直接面向 RVV 1.0 真机 |
| 最适合作为 | K3 真机 RVV 编译实验的工程底座 |

> 论文最有价值之处，是给出了一条足够短、能够在真实 RVV 硬件上闭环验证的 MLIR 扩展路线。
