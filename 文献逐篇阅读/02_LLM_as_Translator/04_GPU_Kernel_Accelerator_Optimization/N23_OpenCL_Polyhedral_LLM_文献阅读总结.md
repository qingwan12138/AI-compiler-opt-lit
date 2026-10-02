# OpenCL Polyhedral + LLM 文献阅读总结
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

论文题目：**Automatic Generation of OpenCL Code through Polyhedral Compilation with LLM**

作者：Marek Palkowski、Mateusz Gruzewski

发表时间：2024

发表平台：FedCSIS 2024；pp. 671–676；DOI: 10.15439/2024F6469
元数据核验来源：[FedCSIS 2024 正式论文 PDF](https://annals-csis.org/Volume_39/drp/pdf/6469.pdf)

关键词：OpenCL、Polyhedral Compilation、ChatGPT、Nussinov、GPU

> 本文档基于 PDF 全文整理。

## 1. 研究背景

Traco、Dapt、Pluto 能为复杂非均匀依赖循环生成 OpenMP tiled 代码，但主要面向 CPU；手写 OpenCL 需要大量平台、内存和 kernel 调度样板。

## 2. 论文要解决的问题

能否让 ChatGPT-3.5 将三种多面体编译器产生的 Nussinov OpenMP 代码转换为跨 NVIDIA/AMD/Intel GPU 可运行的 OpenCL，并保持正确性和性能。

## 3. 核心方法概述

把 Traco/Pluto/Dapt 产生的 skewed/tiled OpenMP 循环交给 GPT-3.5，提示并行循环位置。模型生成 host 端平台/上下文/缓冲区/计时/释放代码和 GPU kernel，并将二维数组线性化、设置边界与串行外循环。

## 4. 实验框架与训练流程

```text
Nussinov C → Traco/Pluto/Dapt → OpenMP tiled code
          → GPT-3.5 源到源生成 OpenCL
          → 编译/人工修正小错误 → CPU 输出差分 → 多 GPU 计时
```

没有训练、RL 或自动多轮 Agent。

## 5. 奖励函数、损失函数或关键公式

无奖励/损失。正确性通过 OpenCL 输出数组与对应 CPU OpenMP 输出逐元素相等；性能用执行时间比较。

## 6. 实验设置

### 6.1 数据集来源

单一 Nussinov RNA folding 非串行多元动态规划 kernel，输入长度 1,000–30,000；三种多面体变换版本。

### 6.2 模型与工具

ChatGPT-3.5；Traco、Dapt、Pluto；ICC 2021、GCC 11.4、Clang 14 `-O3`。Xeon Gold 6326+A100 80GB、AMD Radeon RX 6700S，以及 i5-1235U+Iris Xe。

### 6.3 对比方法

顺序 CPU、三种 OpenMP tiled CPU 代码和相应 OpenCL GPU 代码；未比较其他 LLM。

### 6.4 评价指标

输出一致性和不同输入/平台执行时间。

## 7. 实验结果与结论

三个 OpenCL kernel 均在 NVIDIA/AMD/Intel GPU 上通过 CPU 差分。以 n=30,000 为例，Xeon 顺序约 60,565.98s，A100 上 Pluto OpenCL 约 307.12s；对应 Traco/Dapt 约 496.68/494.24s。不同多面体调度在不同 GPU 上排名会变化，Iris Xe 对 Dapt 较弱、对 Pluto/Traco 部分规模具有竞争力。

## 8. 主要创新点

### 8.1 多面体编译器负责合法调度，LLM 负责异构样板

把数学依赖变换与 API 代码生成分工。

### 8.2 从 OpenMP 结果扩展到 OpenCL 平台

利用 LLM 补齐多厂商 GPU 的 host/kernel 代码。

### 8.3 跨三家 GPU 的经验验证

展示同一 OpenCL 代码的可移植性。

## 9. 局限性

只有一个 kernel、一个 GPT-3.5 会话流程，生成中的小错误需要人工修正；无自动化成功率、无消融、无形式证明。正确性仅靠有限输入差分，性能对比也受调度参数和平台驱动影响。

## 10. 阅读后的研究方向反思

该工作更像可行性案例而非通用框架。值得延伸的是把多面体合法性证书、OpenCL 编译反馈和多设备性能选择自动化，而不是仅让更新模型重新生成代码。

## 11. 可进一步尝试的研究方向

### 11.1 证书约束的多后端 kernel 生成

#### 研究问题

LLM 能否在不破坏依赖合法性的前提下，将同一 polyhedral schedule 生成 OpenCL、CUDA、CPU/RVV 多后端代码。

#### 与原论文的区别

从单 kernel 手工提示改成自动证书、编译反馈和设备选择管线。

#### 可能的创新点

schedule 语义约束和跨设备性能可移植性评估。

#### 实验框架

```text
多面体调度+依赖证书 → LLM 后端生成 → 编译/差分 → 多设备实测 → 选择
```

#### 可行性与风险

Pluto/isl 可提供结构；RVV 与 OpenCL 的并行层次不同。

## 12. 与其他已读文献的关系

与 N12 零样本并行化都用 LLM 生成并行源码，但这里先由 polyhedral 工具保证调度结构；与 N14 MEP 都关注 GPU kernel 可执行验证，N23 更强调 OpenCL 可移植性。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 多面体 OpenMP 到 OpenCL 自动生成 |
| 核心问题 | 优化 CPU 代码难直接运行于 GPU |
| 输入/输出 | tiled OpenMP / OpenCL host+kernel |
| 核心方法 | 多面体调度 + GPT-3.5 源到源转换 |
| 使用的模型 | ChatGPT-3.5 |
| 使用的编译器工具 | Traco、Dapt、Pluto、ICC/GCC/Clang |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否，使用差分测试 |
| 数据集规模 | 1 个 Nussinov kernel、3 个调度版本 |
| 主要指标 | 输出一致性、执行时间 |
| 最重要实验结果 | n=30000 A100 Pluto 约 307s vs CPU 60566s |
| 核心创新 | 编译器合法调度与 LLM API 生成分工 |
| 主要局限 | 单案例、人工修错、无通用成功率 |
| 与 RISC-V 研究的相关性 | 中；可扩展多后端生成 |
| 最适合作为 | 多面体+LLM 可行性基线 |

> 论文证明了组合可行，但距离自动、通用、可验证的编译系统仍有明显差距。
