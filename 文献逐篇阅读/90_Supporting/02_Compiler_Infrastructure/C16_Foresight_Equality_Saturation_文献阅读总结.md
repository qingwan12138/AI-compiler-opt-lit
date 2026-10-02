# Foresight Equality Saturation 文献阅读总结

论文题目：**Parallel and Customizable Equality Saturation**

作者：Jonathan Van der Cruysse、Abd-El-Aziz Zayed、Mai Jacob Peng、Christophe Dubach

发表时间：2026

发表平台：CC 2026；DOI: 10.1145/3771775.3786266
代码/数据/工件：作者主仓库：[Foresight](https://github.com/jonathanvdc/foresight)；[CC 2026 Zenodo 评测工件](https://zenodo.org/records/17955956)
元数据核验来源：[ACM CC 2026 论文 DOI](https://doi.org/10.1145/3771775.3786266)

关键词：Equality Saturation、e-graph、并行重写、饱和策略、元数据、Foresight

> 本文档基于 12 页论文 PDF 全文整理。

## 1. 研究背景

Equality Saturation（等式饱和，EqSat）用 e-graph 同时保存大量等价程序，最后由代价模型抽取最优表达式，从结构上规避传统 pass 顺序过早承诺的问题。但主流引擎多采用单线程、固定饱和循环，定制多阶段调度或增量算法往往要 fork 引擎；元数据也通常被限制为格结构的 e-class analysis（第 1–2 节）。

## 2. 论文要解决的问题

论文针对三个引擎级问题：如何安全并行 e-matching 与重写；如何把固定饱和循环改造成可组合策略；如何让元数据支持分析之外的版本、证明等用途。

> 本文主要研究：如何构建兼顾并行性与可扩展性的 EqSat 引擎，使新策略可由库用户组合而非修改核心实现。

## 3. 核心方法概述

```text
输入表达式与重写规则
→ 线程安全 union-find 并行 e-matching
→ 每个匹配生成延迟 add/union 命令
→ 命令并行简化、依赖分析与批处理
→ addMany / unionMany 更新 e-graph
→ 策略组合器控制规则集、预算、重复与 rebase
→ 元数据 observer 响应批量插入/合并
→ 代价函数抽取结果
```

Foresight 用原子更新的 union-find 保留路径压缩；把重写副作用先表示成 SSA 风格命令，再按依赖分批；用策略高阶函数表达最大匹配、退避、超时、迭代限制和 rebase（第 4–5 节）。

## 4. 实验框架与训练流程

本文不涉及模型训练。系统实现于 Scala，评估四类案例：Horner 法与矩阵乘法结合顺序；LIAR 数组 idiom 识别；复现 Isaria/SymPy 风格定制调度；增量 EqSat。目标是分别验证正确性、并行速度、策略表达力和广义元数据（第 7 节）。

## 5. 奖励函数、损失函数或关键公式

本文没有强化学习奖励函数。抽取使用各案例的代价函数，例如 Horner 法优先减少指数和乘法，矩阵链以标量乘法次数为成本。并行命令的核心不是新数学目标，而是把匹配结果转成 `add`/`union` 命令并形成无内部依赖的最大批次。

## 6. 实验设置

Foresight 支持 Scala 2/3；编译机为 Ubuntu 24.04、Xeon Gold 6254，运行机为 i7-12700K，默认 8 线程。对比 egg 0.10.0、slotted 0.0.35、hegg 0.5.0.0、特定提交的 egglog/egglog-experimental。LIAR 使用相同 PolyBench/BLAS 任务。软件与数据在 Zenodo 公开（论文数据声明，DOI: 10.5281/zenodo.17955956）。

## 7. 实验结果与结论

Foresight 能在 Horner 与 20/40/80 矩阵链上找到预期最优结果。并行 Foresight 快于其串行版本、hegg 与 slotted，并在部分矩阵链上快于 egglog；但高优化 Rust `egg` 在全部这些基准上仍更快。表 1 显示 Foresight JVM 内存明显更高，例如 Horner 为 40 MB，而 egg 为 4 MB。

在 LIAR 复现中，Foresight 的饱和时间几何平均提速接近 16×；对 `2mm` 和 `gemm` 找到更高质量的 BLAS 组合，运行时分别为原方案的 3× 与 10×。并行总收益受串行 hash-consing/合并限制：`stencil2d` 8 线程约 1.7×，`80mm` 约 2.6×（图 8）。Isaria 与 SymPy 策略均可用不超过 10 行组合器表达；增量 EqSat 可复用先前 e-graph 状态并优于逐个独立饱和。

## 8. 主要创新点

### 8.1 延迟命令驱动的并行重写

将匹配、命令生成、简化和可并行批次从实际 e-graph 更新中解耦，减少固定点附近的大量无效顺序写入。

### 8.2 可组合饱和策略

把调度、预算、重复、rebase、日志和分析表示为基础策略与组合器，避免为每个新算法 fork 引擎。

### 8.3 Observer 式广义元数据

`onAddMany`/`onUnionMany` 同时覆盖传统格分析和非格的 e-class 版本，为增量 EqSat 等场景提供统一接口。

## 9. 局限性

论文明确指出 `egg` 的绝对饱和时间仍更快，Foresight 的 JVM 对象表示造成更高内存；hash-consing 和部分 union/metadata 阶段仍限制并行扩展；生产编译器集成不是本文主要目标。阅读后还可见：主要实证集中在表达式/数组 idiom，尚未覆盖 LLVM/MLIR 中复杂控制流；策略可表达并不自动保证策略优；元数据未来可做证明重建，但论文尚未实现完整证明链。

## 10. 阅读后的研究方向反思

Foresight 更像研究基础设施，而非独立优化算法。它与 phase ordering 的关系在于推迟承诺和显式控制搜索，而不是使用 RL 选择 LLVM pass。对 RISC-V 的价值主要来自可把标量—RVV 规则、语义/成本元数据和多阶段向量化策略装入统一引擎；仅换成 RISC-V 规则集仍不足，需证明真实后端收益、可伸缩性或验证能力的新问题。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RVV 的证明与真实性能双元数据

#### 研究问题

EqSat 是否能同时记录规则来源、语义证明状态和 RVV 真机代价，避免只按静态成本抽取。

#### 与原论文的区别

原文只提出证明重建前景；新方向增加 RVV 语义/合法性与 PMU 反馈闭环。

#### 可能的创新点

证明 provenance 元数据、目标合法性元数据、可变 VLEN 代价、失败候选过滤。

#### 实验框架

```text
标量/向量 IR → EqSat 规则探索
→ 记录规则与合法性元数据
→ 抽取候选 → 形式/差分验证
→ RISC-V 真机测量 → 更新代价
```

#### 可行性与风险

xDSL/MLIR、Alive2/SMT 与 RVV 工具链可组合；风险是循环/内存语义和可变 VLEN 的证明复杂度。

## 12. 与其他已读文献的关系

与 Protean 都把优化过程拆成更细的可控单元，但 Foresight 操作重写空间而非 LLVM pass；与 LPO/Alive2 互补，可为规则应用附加验证来源；与 MLIR Transform Dialect 都将变换控制显式化；C17 EggMind 可直接把 Foresight 式策略对象进一步交给 LLM 自动合成。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 构建并行、可定制 EqSat 引擎 |
| 输入/输出 | 表达式、规则与策略 / 抽取后的低成本等价程序 |
| 核心方法 | 线程安全 e-graph、延迟命令、策略组合器、广义元数据 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | EqSat 保持规则给定的等价关系；未实现完整证明重建 |
| 主要指标 | 饱和时间、并行加速、内存、抽取方案运行时 |
| 最重要结果 | LIAR 饱和几何平均近 16×；部分方案运行 3×/10× |
| 核心创新 | 引擎级并行与策略/元数据可扩展接口 |
| 主要局限 | egg 仍更快、JVM 内存高、串行阶段限制扩展 |
| 与 RISC-V 相关性 | 中；适合作为 RVV 规则探索基础设施 |
| 最适合作为 | EqSat 工具模块和策略实验平台 |

> 最值得学习的是把“重写机制”和“搜索策略”解耦；最合理的用法是作为可验证优化研究底座，而不是把引擎速度本身等同于后端性能收益。
