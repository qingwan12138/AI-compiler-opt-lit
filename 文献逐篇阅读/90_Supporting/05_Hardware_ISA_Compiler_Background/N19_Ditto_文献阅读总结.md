# Ditto 文献阅读总结
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

论文题目：**Compiling Code LLMs into Lightweight Executables**

作者：Jieke Shi、Junda He、Zhou Yang、Chengran Yang、Mykhailo Klymenko、Thong (James) Hoang、Sherry (Xiwei) Xu、Zhenchang Xing、David Lo

发表时间：2026

发表平台：Proceedings of the ACM on Software Engineering（FSE 2026）；第3卷，pp. 4299–4322；DOI: 10.1145/3808196
元数据核验来源：[ACM DOI 正式出版页](https://doi.org/10.1145/3808196)；[arXiv:2603.29813](https://arxiv.org/abs/2603.29813)

关键词：Code LLM 部署、量化、LLVM Pass、GEMV、BLAS、边缘推理

> 本文档基于 PDF 全文整理。

## 1. 研究背景

7B Code LLM 的全精度推理需要约 26GB 内存，普通笔记本难以本地部署。单纯低比特量化减少内存，却可能因解包和码本查找反而降低速度；推理程序中的朴素 GEMV 又占据主要时间。

## 2. 论文要解决的问题

如何同时压缩模型权重和优化执行程序，将 Code LLM 编译成在消费级 CPU 上可运行、低内存、低能耗且基本保持代码生成准确率的可执行文件。

## 3. 核心方法概述

Ditto 将权重按块 K-Means 聚类，以 4-bit 索引和码本存储，可保留少数敏感张量为 FP32；随后生成 C 推理程序，LLVM Pass 识别 GEMV 归约循环并替换为目标机优化的 `cblas_sgemv`/专用低比特 kernel。

## 4. 实验框架与训练流程

```text
HF Code LLM → 分块码本量化 → llama2.c 风格 C 推理
 → Clang -O1 生成 IR → Ditto Pass 匹配 GEMV → BLAS 调用
 → -O3/-ffast-math 链接 → 本地可执行文件
```

这是编译/部署系统，不使用 LLM 生成优化，也无 RL。

## 5. 奖励函数、损失函数或关键公式

无训练奖励。量化目标为块内 K-Means 重构误差；能耗按 `E=P/tokens_per_second` 估算，论文在 M4 上观察平均持续功耗约 18W。

## 6. 实验设置

### 6.1 数据集来源

HumanEval+、MBPP+；评测 Code Llama-7B、Magicoder-CL-7B、OpenCodeInterpreter-CL-7B。

### 6.2 模型与工具

Apple M4、24GB、macOS 13.2.1、Clang 14、Apple Accelerate BLAS；默认 4-bit、组大小 256。LLVM Pass 约为总计 4,000 行 Python/C/C++ 系统的一部分。

### 6.3 对比方法

llama2.c 全精度实现、PyTorch 2.8 动态 int8；消融为仅量化、仅 BLAS 替换和完整 Ditto。

### 6.4 评价指标

pass@1/pass@5、峰值内存、tokens/s、ms/token、GFLOP/s、J/token。

## 7. 实验结果与结论

Ditto 的 pass@1 相对全精度平均只下降 0.27%，最大下降 1.83%；相对 int8 最高提升 6.96 个百分点。内存从约 26GB 降至 4.1–4.2GB（6.4×），吞吐最高提升 10.5×，能耗最高降低 10.5×。Code Llama 消融中，仅量化速度为 0.8×且能耗更差；仅 BLAS 为 4.8×；两者结合 7.0×、内存 6.4×、能耗 7.1×，说明两个瓶颈互补。

## 8. 主要创新点

### 8.1 模型表示与推理程序联合优化

不把量化和编译优化当作两个孤立阶段。

### 8.2 LLVM 自动识别并改写 GEMV

将朴素 C 循环替换成硬件优化 BLAS，保持后端可移植性。

### 8.3 用消融揭示协同而非简单叠加

量化单独会变慢，只有与计算 kernel 优化结合才释放收益。

## 9. 局限性

仅 Apple M4、三种 7B Llama 系 Code LLM 和 llama2.c 风格程序；能耗以近似恒定 18W 估算而非逐配置精细积分。LLVM 匹配依赖规范 GEMV 模式，其他模型结构/运行时未验证，4-bit/组大小也未系统搜索。

## 10. 阅读后的研究方向反思

Ditto 研究的是“优化运行 Code LLM 的编译器”，不是“用 LLM 优化编译器”，方向需区分。它对 K3 的价值在于提供联合优化基线：内存压缩必须与 RVV/BLAS kernel 优化一起设计。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V 的量化—RVV 联合编译

#### 研究问题

在 K3 上，4-bit 解包、访存和 RVV GEMV 如何协同，何种块大小最优。

#### 与原论文的区别

从 Apple Accelerate 固定后端转为开源 RVV kernel，并让编译器根据硬件选择量化布局。

#### 可能的创新点

量化块布局—VLEN—cache 联合代价模型。

#### 实验框架

```text
不同量化布局 → LLVM IR → RVV GEMV 匹配/生成 → K3 时间/内存/能耗
```

#### 可行性与风险

7B 模型可能受 K3 内存和带宽限制；可先用小模型或单层 GEMV。

## 12. 与其他已读文献的关系

与 N02 能效代码相同点是关注能耗，但 Ditto 是固定推理系统的编译优化；与 N13/N17 都面向专用硬件向量 kernel；与旧读多硬件/代价模型文献可组合做后端选择。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 将 Code LLM 编译为轻量本地可执行文件 |
| 核心问题 | 量化省内存但可能更慢，GEMV 低效 |
| 输入/输出 | 模型 checkpoint / 优化本地可执行文件 |
| 核心方法 | 4-bit 码本量化 + LLVM GEMV→BLAS |
| 使用的模型 | 三种 7B Code LLM |
| 使用的编译器工具 | LLVM/Clang 14、Apple BLAS |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否 |
| 数据集规模 | HumanEval+、MBPP+ |
| 主要指标 | pass@k、内存、吞吐、能耗 |
| 最重要实验结果 | 最高 10.5× 速度/能效，6.4× 内存减少 |
| 核心创新 | 量化表示与编译 kernel 联合优化 |
| 主要局限 | 单 M4 平台、单运行时家族 |
| 与 RISC-V 研究的相关性 | 高；可验证 RVV 联合优化 |
| 最适合作为 | 端侧 Code LLM 编译部署基线 |

> 最关键的结论是量化并不自动带来速度，必须同时优化解码与 GEMV 执行路径。
