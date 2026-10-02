# C68 A Compiler for Fused Relational Operations on Multisets 文献阅读总结

论文题目：**A Compiler for Fused Relational Operations on Multisets**

作者：James Dong、Fredrik Kjolstad

发表时间：2026

发表平台：PLDI 2026，Proceedings of the ACM on Programming Languages，Vol. 10，Article 205，25 页
元数据核验来源：[Stanford Compilers Lab 论文页](https://compilers.stanford.edu/publications/pldi26relations/)；[ACM DOI](https://doi.org/10.1145/3808283)
代码/数据/工件：作者 Zenodo 论文工件：[PluS](https://doi.org/10.5281/zenodo.19635976)

论文链接或编号：DOI [10.1145/3808283](https://doi.org/10.1145/3808283)

关键词：关系代数、multiset、ALIR、loop fusion、sparse compilation、code generation、query compiler

> 本文档依据论文正文整理。论文事实、阅读后的分析和后续建议分开描述。

## 1. 研究背景

数据库系统通常把 SQL 降低为关系代数查询计划，再从固定的手写 operator 库中选择实现。论文第 1—2 节指出，固定实现难以覆盖大量 operator 组合、复杂 join 类型和多种关系存储结构；缺少跨 operator 的 fusion 会增加中间存储、降低局部性，并在某些图查询中失去更好的渐近复杂度。

问题在 multiset（允许重复 tuple 的关系）上更难：交、并、差、笛卡尔积的 multiplicity 语义不同；outer join 还需要在循环域和循环体中处理 NULL；non-equi join 不能简单套用只支持交集迭代的 worst-case optimal join。论文因此研究一个既能表达复杂关系代数，又能生成融合 C++ 代码的编译器。

## 2. 论文要解决的问题

### 2.1 通用关系代数融合

如何把 selection、projection、aggregation、inner/outer/non-equi join、difference、intersection、union 等操作降低到统一的循环 IR，并保持 set/multiset 语义。

### 2.2 存储结构可移植

如何让同一套高层关系表达适配 row/column store、hash map、trie、B-tree 等不同数据结构，而不把存储遍历细节硬编码到 operator 语义中。

### 2.3 生成高质量低层代码

如何从抽象 loop domain 生成能够 co-iterate 多个不规则数据结构的 C++，同时在融合查询上获得性能收益。

> 本文主要研究：如何用 ALIR、数据结构表示和 iteration machine，将含 multiset 语义的通用关系代数编译成融合且高效的 C++。

## 3. 核心方法概述

论文提出 ALIR（Abstract Loop IR）和围绕它的 lowering、iteration machine 与 C++ code generator。ALIR 把关系操作表示为访问关系属性的嵌套循环，把 loop domain 与物理存储结构分离；数据结构规范描述如何遍历实际 relation。iteration machine 再把抽象 domain 组合成 merge、intersection、union、difference 等可执行迭代。

```text
关系代数/物理查询计划
        ↓
关系操作 lowering
        ↓
ALIR：抽象循环、语句、loop domain
        ↓
数据结构规范 + iteration lattice/machine
        ↓
循环控制、预计算和优化
        ↓
C++ fused code
        ↓
TPC-H、LSQB、triangle query 等基准评测
```

本文不使用 LLM、SFT、强化学习或形式化验证器。系统是传统编译器/查询编译基础设施，当前 taxonomy 分类为 SUPPORTING / B2。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用编译器执行流程。

### 4.1 高层 lowering

第 6 节将关系代数操作降低为 ALIR。实现覆盖 non-join 操作、inner join、outer join、non-equi join、precomputation/permutation 和 set semantics；multiset 语义通过 multiplicity-aware loop domain 与语句表达。

### 4.2 抽象循环与迭代机器

ALIR 的 loop domain 描述关系属性的集合/多重集合表达式。第 7 节在 iteration lattice 基础上构造 iteration machine，决定各输入 iterator 的推进、匹配、补值和输出；同时允许聚合更新与预计算插入循环结构。

### 4.3 代码生成

第 8 节根据 loop domain 和 data structure specification 生成 C++，包括循环生成、case handling 和不同存储形式的 co-iteration。作者还实现了存储格式转换与动态选择，以比较 sort-merge、galloping 和 hash join。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有神经网络损失函数。论文的关键形式化对象是 multiset multiplicity。例如：

```text
#(A ∪ B)(x) = max(#A(x), #B(x))
#(A ∩ B)(x) = min(#A(x), #B(x))
#(A − B)(x) = max(#A(x) − #B(x), 0)
#(A + B)(x) = #A(x) + #B(x)
#(A × B)(a,b) = #A(a) · #B(b)
```

这些定义决定 lowering 和生成循环如何推进 iterator 与产生重复输出。编译器运行时的选择目标是生成正确且高效的 fused code；论文没有把它写成统一的可学习 reward。

## 6. 实验设置

### 6.1 数据集来源

主要实验使用 TPC-H、LSQB 图查询、triangle query，以及 Twitch Gamers network 上的层次迭代查询。论文还构造了不同密度和存储格式的关系 A/B 交集实验，`N = 10,000,000`，比较 sort-merge、galloping、hash join 和原生 `unordered_map`。

### 6.2 模型与工具

系统包含 Python-based ALIR DSL、关系代数到 ALIR 的 lowering、ALIR-to-C++ code generator、iteration machine 和多种 data structure specification。对比系统包括 Hyper、DuckDB、Free Join；论文还讨论 TACO、SDQL、Indexed Streams 和其他 worst-case optimal join 系统。论文正文未把所有编译器版本和硬件规格完整列出，不能据此补写未说明的版本。

### 6.3 对比方法

主要 baseline 是 Hyper；部分查询与 DuckDB 比较，支持的查询子集还与 Free Join 比较。Table 1 将本文与 EmptyHeaded、Indexed Streams、Free Join、SDQL 在 join、outer join、non-equi、multiset 和 data-structure portability 等能力上比较。

### 6.4 评价指标

| 指标 | 含义 |
|---|---|
| Speedup | 生成查询相对 baseline 的运行时间比，越大越好 |
| Runtime | 特定查询/存储配置的执行时间 |
| Scaling | 随输入规模增长的渐近或经验趋势 |
| Fusion coverage | 能否跨 operator 生成融合实现 |
| Data-structure portability | 同一抽象能否适配不同关系存储 |

## 7. 实验结果与结论

### 7.1 主要结果

论文摘要和第 1 节报告，在选定 LSQB 和 worst-case optimal triangle queries 上，相对 Hyper 的融合执行达到平均 3.8× speedup，范围为 0.8–12.1×。在 TPC-H 上，顺序执行平均约 1.0×（0.4–4.3×），并行执行约 0.6×（0.2–1.8×）；这说明系统的主要优势是更强的融合表达能力，而不是所有查询都更快。

### 7.2 与传统系统比较

在 LSQB 查询中，Q6/Q9 等查询受益于 multi-way join 和 hierarchical iteration，系统可以避免 Hyper/DuckDB 先生成中间 binary join 的路径。与 Free Join 的支持子集相比，Q2 上为 1.21×，Q4 上为 0.47×；将输出改成类似 factorized representation 后，Q4 可达 1.32×。这些数字只适用于论文给出的查询、数据结构和执行设置。

### 7.3 Triangle query

对 `A(a,b) ⋈ B(b,c) ⋈ C(c,a)`，论文构造了输入使传统 join-based strategy 呈二次复杂度，而生成的代码可达到线性 `Theta(n)` 级别。Heavy-Light 和 Generic Join 两个版本分别体现 lookup 成本与循环复杂度的取舍；在 arithmetic oracle 替换 lookup 后，Heavy-Light 为 Generic Join 的约 1.29×。

### 7.4 Hierarchical iteration 与数据结构

在 Twitch Gamers 查询中，过滤选择率约为 1/159.9，hierarchical iteration 可以跳过大量边。数据结构实验显示 sort-merge 在密度相近时更好，galloping 可处理一定稀疏性，hash join 在稀疏输入上更有优势但构建常数更高；存储转换有一次性成本，因此是否转换取决于查询重复运行次数。

## 8. 主要创新点

### 8.1 创新点一：覆盖扩展关系代数的 ALIR

ALIR 以抽象循环表达 inner/outer/non-equi join、difference、intersection、union 和 multiset，而不是只支持线性 pipeline 或 inner equi-join。Table 1 直接展示了覆盖范围相对既有系统的扩展。

### 8.2 创新点二：关系数据结构与循环域解耦

数据结构规范把 row/column/trie/B-tree/hash map 的遍历能力与关系语义分开，因此同一关系编译表示可以选择不同物理存储，支持 portability 和运行时成本权衡。

### 8.3 创新点三：面向融合的 iteration machine

论文将 generalized iteration lattice、iterator 控制、预计算、聚合更新和代码生成组织成 iteration machine，使复杂 multiset/NULL/non-equi 语义仍能形成融合循环。实验显示这一机制在 LSQB 和 triangle query 上有实质收益。

## 9. 局限性

### 论文明确承认的局限

- 论文没有显式处理 vectorized execution，作者将其留作未来工作。
- 某些与 Free Join 的比较只能覆盖 Free Join 支持的查询子集。
- 存储转换可能带来明显一次性开销，运行次数少时不一定值得转换。
- TPC-H 顺序性能接近 Hyper，但并行结果平均低于 Hyper，说明融合通用性会带来额外成本。

### 阅读后发现的潜在局限

论文主要面向关系/查询编译，不能直接推断对一般 LLVM 程序或 RISC-V 后端有效。其性能结果依赖查询计划、存储格式和数据分布；将 ALIR 迁移到其他硬件还需要新的向量化、并行调度和成本模型。

## 10. 阅读后的研究方向反思

最值得借鉴的是把“可融合的语义范围”和“物理数据结构”分层，再让中间表示承载两者之间的可组合接口。对 AI 编译器或 RISC-V，可以借鉴这种 domain-specific IR 与 storage/backend decoupling；但简单把关系 operator 改成 tensor operator 并不足以形成新颖性，因为 tensor fusion、sparse compilation 和 iteration lattice 已有丰富先行工作。本文更适合作为编译器 IR、lowering 和 code-generation 的方法参考，而不是 LLM 优化 baseline。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V 向量后端的关系/稀疏 IR lowering

#### 研究问题

如何把 ALIR 的 multiset co-iteration 和 hierarchical iteration 映射到 RVV 的向量循环、gather/scatter 与 cache-aware schedule。

#### 与原论文的区别

新增目标是具体 ISA 的向量化和内存层次成本，不只是复用 C++ generator。

#### 可能的创新点

建立关系迭代到 RVV 指令/向量长度的合法 lowering，并用真实 RISC-V 硬件测量。

#### 实验框架

```text
关系查询 → ALIR → RVV-aware loop transformation → RVV codegen → Q1/Q2/LSQB 与硬件测量
```

#### 可行性

需要 LLVM/RVV 后端、关系 benchmark、RVV 模拟器或开发板及数据结构实现。

#### 主要风险

不规则访问、NULL/multiset 语义和 RVV 向量化收益可能相互冲突。

### 11.2 融合编译器的可验证成本模型

研究问题是如何在保证 multiset/NULL 语义的同时，使用硬件计数器或静态模型选择 fusion、存储格式和并行策略。区别于原论文的手工/规则式选择，创新点应在跨查询分布泛化和可解释反馈；风险是模型预测的静态成本不等于真实端到端收益。

## 12. 与其他已读文献的关系

本轮另一篇 “Fungible Memories…” 同样强调中间表示、语义保持变换和多后端生成，但它处理硬件 memory technology mapping 与 netlist lifting；本文处理关系代数和查询执行。两者可以在“抽象语义—目标约束—后端 lowering”的方法层面互相借鉴，却不应把数据库查询 benchmark 或 memory mapping 结果直接混合。本文与仓库中传统 ML compiler 论文的关系是：它是非 LLM 的通用 DSL/query compiler，更适合作为 compiler infrastructure baseline。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 将扩展关系代数编译成融合 C++ |
| 核心问题 | 复杂 join、multiset、NULL 和多存储结构难以统一融合 |
| 输入 | 关系代数/物理查询计划与数据结构描述 |
| 输出 | 融合的低层 C++ 查询实现 |
| 核心方法 | ALIR、lowering、iteration machine、data structure specification |
| 使用的模型 | 无神经模型 |
| 使用的编译器工具 | Python ALIR DSL、C++ generator、Hyper/DuckDB/Free Join 对比 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 论文未报告独立形式化验证器 |
| 数据集规模 | TPC-H、LSQB、Twitch Gamers、triangle query 与合成密度实验 |
| 主要指标 | runtime、speedup、扩展性、融合覆盖和存储可移植性 |
| 最重要实验结果 | LSQB/triangle 选定基准相对 Hyper 平均 3.8×；TPC-H 顺序约 1.0× |
| 核心创新 | 支持复杂 multiset 关系代数的可融合 ALIR 与 iteration machine |
| 主要局限 | 未显式向量化，通用性与并行性能仍有代价 |
| 与 RISC-V 研究的相关性 | 中：可作为 RVV 稀疏/不规则循环 lowering 的 IR 参考，但论文未研究 RISC-V |
| 最适合作为 | DSL 编译器、融合 IR 和 code generation 的方法参考 |

这篇论文最值得学习的是用抽象循环域统一表达复杂关系操作和物理迭代；最主要的局限是尚未覆盖向量化且并行结果不总是占优；如果用于后续研究，最合理的使用方式是借鉴其 IR/lowering 分层，而不是把数据库查询结果直接当作通用编译优化结论。
