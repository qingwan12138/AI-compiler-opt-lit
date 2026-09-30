# FLEX 文献阅读总结

论文题目：**Interleaved Learning and Exploration: A Self-Adaptive Fuzz Testing Framework for MLIR**

作者：Zeyu Sun, Jingjing Liang, Weiyi Wang, Chenyao Suo, Junjie Chen, Fanjiang Xu

发表时间：2025 年；ASE 2025 Research Papers；同时有 arXiv:2510.07815v1。

发表平台：ASE 2025；arXiv 预印本。

论文链接或编号：[arXiv:2510.07815](https://arxiv.org/abs/2510.07815)，[ASE 2025 官方页面](https://conf.researchr.org/details/ase-2025/ase-2025-papers/90/Interleaved-Learning-and-Exploration-A-Self-Adaptive-Fuzz-Testing-Framework-for-MLIR)。

关键词：MLIR、编译器模糊测试、CodeGen、LoRA、测试生成、反馈驱动、多层中间表示。

> 以下事实依据论文正文和官方代码仓库。FLEX 的最终角色是由 CodeGen-2B 生成可复用 MLIR 测试程序和 fuzzer 工具，因此建议归入 GENERATOR/G4。

## 1. 研究背景

MLIR 通过 dialect、operation 和 pass 支持多层中间表示及领域专用编译器。它的可扩展性也带来测试困难：不同 dialect 可以组合，既有语法约束，也有跨层语义约束。论文指出，面向 MLIR 的既有模糊测试器主要依赖人工模板、预定义变异算子或已有测试样本，生成多样且语义有效的程序仍然困难。

论文选择神经代码生成模型处理这一问题，但 MLIR 专用训练数据有限。因此核心思路是让模型从少量官方测试种子开始，持续学习新生成且通过编译的程序，并用随机扰动扩大探索范围。

## 2. 论文要解决的问题

### 2.1 MLIR 测试程序多样性不足

人工 grammar 和 mutation rule 难以覆盖 MLIR 众多 dialect、嵌套结构和 pass 组合，容易错过深层 bug。

### 2.2 专用训练数据稀缺

MLIR 程序不像通用源代码那样大量存在，直接对固定小语料微调模型会限制生成能力。论文研究如何在小种子集上进行自适应学习。

### 2.3 需要有效而合法的故障触发输入

生成结果必须先通过语法检查，再运行 MLIR pass；只有能触发崩溃的测试程序才进入最终测试集，同时无崩溃的有效程序继续用于训练。

> 本文主要研究：如何用 CodeGen-2B、受控随机采样和编译反馈，构建能持续生成有效且多样 MLIR 测试程序的自适应 fuzzer。

## 3. 核心方法概述

FLEX 是一个自适应循环。它从 MLIR 官方测试套件提取种子，用 LoRA 微调 CodeGen-2B；模型随后以温度采样方式生成变体。程序通过 `mlir-opt` 语法检查后，依次运行 237 个 MLIR pass。有效程序和 pass 变换后的程序加入训练集，触发崩溃的程序及 pass、错误信息加入最终测试集。

```text
MLIR 官方回归测试种子
        ↓
LoRA 微调 CodeGen-2B
        ↓
从短前缀进行温度采样，生成 MLIR 程序
        ↓
mlir-opt 语法检查
        ↓
依次运行 237 个 compiler pass
        ↓
记录 crash、pass、stack trace
        ↓
有效程序与变换程序加入训练集
        ↓
继续微调并生成下一轮测试
```

LLM 的最终输出是 MLIR 测试程序；系统输出则是可复用的测试生成工具和 crash-inducing test suite。它不生成新的 MLIR 优化 pass，而是测试已有 pass。

## 4. 实验框架与训练流程

本文不使用强化学习，主要采用监督式参数高效微调、随机生成和编译反馈。

### 4.1 种子与模型初始化

作者从 llvm-project/mlir/test 的 revision `c641fc3` 收集并按函数切分测试，共 15,344 个 seed programs。初始模型为 CodeGen-2B，使用 LoRA 只训练低秩矩阵；每轮训练 5 个 epoch。

### 4.2 受扰动生成

每个种子取前 3 个 token 作为前缀，生成 4 条候选。模型在每个 token 位置按温度 1.0 采样，最大长度为 600 token。与贪心解码相比，这一步提供了可控多样性。

### 4.3 编译与 pass 执行

先用 `mlir-opt` 验证语法。论文从官方文档获得 237 个 pass，对每个有效输入逐一运行；若某个 pass 崩溃，记录测试程序、pass 和错误信息。

### 4.4 多样性增强

未触发崩溃但通过检查的程序，以及成功变换后的程序，都回收到训练集合。流程重复至预设最大迭代次数。RQ2 的比较实验先进行 30 轮自适应训练，再用最终 checkpoint 做 24 小时测试。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数。论文没有给出独立的任务奖励或 RL loss，而是使用 CodeGen 的微调目标和模糊测试指标。

LoRA 更新可写为：

```text
W' = W + B A
```

其中 `W` 是冻结的原权重，`A` 和 `B` 是可训练低秩矩阵，rank `r=8`。论文给出 LoRA alpha=32、dropout=0.1、学习率 `5×10^-5`。

## 6. 实验设置

### 6.1 数据集来源

训练种子来自 MLIR 官方回归测试目录 `llvm-project/mlir/test`，具体 revision 为 `c641fc3`，共 15,344 个按函数切分的程序。自适应过程中新增的有效程序和 pass 变换结果成为后续训练数据。论文没有报告独立训练/验证/测试集划分。

### 6.2 模型与工具

基础模型是 CodeGen-2B；微调方法是 LoRA。目标编译器是 MLIR，执行工具是 `mlir-opt`。实验服务器为 Intel Xeon Gold 6354 3.00GHz、8 张 NVIDIA RTX 4090（每张 24GB VRAM）、500GB 内存、Ubuntu 20.04；RQ1 使用 3 张 GPU，其余配置按正文说明。

### 6.3 对比方法

对比工具为 MLIRSmith、MLIRod、SynthFuzz 和 Ratte。所有工具使用相同种子集、公开实现和推荐参数，在固定 MLIR revision 上运行 24 小时。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| Unique bugs | 按 crash message/stack trace 去重后的 bug 数 | 越大越好 |
| Line coverage | MLIR 源码行覆盖率 | 越大越好 |
| Generated files | 生成测试程序数量 | 需结合质量解释 |
| Average tokens/file | 每个测试程序平均 token 数 | 越小通常越易定位 |

## 7. 实验结果与结论

### 7.1 主要结果

30 天活动中 FLEX 发现 80 个此前未知 bug，其中 58 个已确认或修复。已标注的 58 个 bug 分布为 28 个 conversion、7 个 bufferization、9 个 general transformation、26 个 dialect-specific transformation、10 个 parser 相关实例；分类总数包含论文按不同统计口径列出的发现集合，已确认/修复子集需以表格为准。

### 7.2 与传统方法比较

固定 revision、24 小时实验中，FLEX 发现 53 个 bug、行覆盖率 28.22%；MLIRSmith 为 8/17.71%，MLIRod 为 15/19.87%，Ratte 为 3/16.26%，SynthFuzz 为 2/11.00%。论文在 273,187 行口径下报告 FLEX 比 MLIRod 的覆盖率高 42.0%。

### 7.3 生成规模与效率

FLEX 生成 19,898 个测试程序，平均 146.44 token；MLIRSmith、MLIRod、SynthFuzz、Ratte 分别生成 3,913、2,273、11,353、7,613 个。论文报告 FLEX 约每分钟执行 13.8 个测试，明显高于 MLIRSmith 的 2.72 和 MLIRod 的 1.58。

### 7.4 消融实验

在四轮训练、24 小时固定 revision 实验中，完整 FLEX 发现 25 个 bug、覆盖率 23.23%；去掉 perturbed generation 后为 14/15.44%；去掉 diversity augmentation 后为 20/16.98%。这说明随机探索和训练集回收都对结果有贡献。

### 7.5 案例分析

论文将已确认 bug 归为 Unsupported Type、Incomplete Verifier、Incorrect Pattern、Unregistered Dialect、Incorrect Rewrite Logic、Invalid Memory Access 和 Incorrect Assertion 七类，其中 Unsupported Type 和 Invalid Memory Access 被报告为此前未出现的根因类别。

## 8. 主要创新点

### 8.1 创新点一：面向 MLIR 的自适应生成闭环

FLEX 将模型微调、程序生成、编译执行和数据回收放入循环，不依赖每个 dialect 的人工 mutation rule。实验显示回收有效程序能持续扩大生成空间。

### 8.2 创新点二：扰动生成与多样性增强协同

短前缀、多分支温度采样负责探索；编译通过程序和 pass 变换程序负责扩大训练数据。消融实验支持两个模块的必要性。

### 8.3 创新点三：面向已有编译器能力的可复用 fuzzer

模型输出不是一次性样例，而是可持续生成 MLIR 输入的工具流程；最终输出包含触发具体 pass 崩溃的测试程序及错误上下文，适合开发者复现和定位。

## 9. 局限性

### 论文明确承认的局限

论文的训练数据来自 MLIR 回归测试，模型需要依赖现有种子；实验针对 MLIR，不能直接推断到其他 IR。bug 根因标签主要依据开发者讨论和补丁，因此只有确认或修复的 bug 可标注根因。FLEX 主要检测崩溃，不能证明所有未崩溃程序或优化结果语义正确。

### 阅读后的潜在局限

CodeGen-2B 和 LoRA 的生成质量仍受训练语料、token 长度和采样温度影响。对每个输入逐一运行 237 个 pass 可能成本较高。论文报告的是 crash/coverage，未提供跨硬件性能、误编译率或真实用户 workload 评估；RISC-V/RVV 适用性需另行验证。

## 10. 阅读后的研究方向反思

值得借鉴的是把编译器接受的有效 IR 作为下一轮训练数据，并把 pass 级 crash 信息保留为测试资产。FLEX 的核心贡献是 MLIR fuzzing 的自适应生成闭环，不能仅把 MLIR 替换为 LLVM IR 或 RVV 指令就视为新贡献。

对 RISC-V 的相关性为中等：其生成器结构可用于 RVV intrinsic 或 MLIR-RVV dialect 测试，但必须补充 ISA 约束、寄存器状态、VTYPE/VL 语义和跨编译器差分执行。FLEX 最适合作为 GENERATOR/G4 工具基线和测试生成模块。

## 11. 可进一步尝试的研究方向

### 11.1 方向名称：面向 RVV dialect 的自适应测试生成

#### 研究问题

如何让模型从少量 RVV/MLIR 回归测试学习向量类型、VL/VTYPE 和内存别名约束，并生成能触发编译器错误的合法输入。

#### 与原论文的区别

加入 RVV 状态机和差分执行约束，目标从 MLIR 通用 crash 扩展到 RVV 语义错误与误编译。

#### 可能的创新点

设计 RVV 状态感知的 token 采样、语义等价变体和编译器交叉验证反馈。

#### 实验框架

```text
RVV/MLIR 种子 → LoRA 微调 → 状态约束生成 → LLVM/GCC/XuanTie 编译
→ QEMU/硬件执行 → 差分反馈 → 回收有效程序
```

#### 可行性

需要 LLVM/GCC RVV 工具链、QEMU 或 RVV 硬件、MLIR/RVV 测试集及公开 bug 数据。

#### 主要风险

模拟器与硬件差异、未定义行为和向量状态初始化可能造成误报。

### 11.2 方向名称：面向 pass 覆盖的主动样本回收

#### 研究问题

如何按未覆盖 pass、dialect 和错误类型选择回收样本，而不是将所有有效程序等量加入训练集。

#### 与原论文的区别

把多样性目标从 token/程序层面扩展到 pass 覆盖和根因覆盖，并引入显式样本优先级。

#### 可能的创新点

构建 pass-aware replay buffer 和覆盖增益评分。

#### 实验框架

```text
生成程序 → pass 覆盖/错误标签 → 样本评分 → 选择回放集 → 微调 → 新一轮生成
```

#### 可行性

可复用 FLEX 代码、MLIR pass instrumentation 和公开基线。

#### 主要风险

覆盖率提升可能偏向浅层代码路径，导致深层语义 bug 被忽略。

## 12. 与其他已读文献的关系

FLEX 与 Ratte 都面向 MLIR 编译器测试，但 Ratte 以可组合 dialect 语义和确定性、无未定义行为的生成作为核心；FLEX 以 CodeGen-2B 微调、随机扰动和反馈回收扩大输入多样性。Ratte 可作为语义安全生成 baseline，FLEX 可作为神经生成模块；二者组合可能形成“语义约束过滤 + 自适应神经生成”系统。FLEX 与 RVISmith 的共同点是生成可执行编译器测试并进行差分/崩溃检查，区别是 FLEX 使用 LLM/CodeGen 生成 MLIR，RVISmith 使用 RVV intrinsic 规格驱动随机生成。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | MLIR 编译器自适应模糊测试 |
| 核心问题 | 既有模板/规则难以产生多样且有效的 MLIR 输入 |
| 输入 | MLIR 回归测试种子 |
| 输出 | MLIR 测试程序、crash 测试集与 fuzzer 工具 |
| 核心方法 | CodeGen-2B + LoRA、扰动采样、编译反馈、数据回收 |
| 使用的模型 | CodeGen-2B |
| 使用的编译器工具 | MLIR、mlir-opt、237 个 pass |
| 是否使用强化学习 | 否；使用 LoRA 微调和反馈回收 |
| 是否使用形式化验证 | 否；使用语法检查、编译和 crash 分析 |
| 数据集规模 | 初始 15,344 个 MLIR seed programs |
| 主要指标 | 唯一 bug 数、行覆盖率、生成规模、平均 token 数 |
| 最重要实验结果 | 24 小时固定 revision 发现 53 bug、28.22% 行覆盖率；30 天发现 80 个未知 bug |
| 核心创新 | 自适应生成与多样性增强闭环 |
| 主要局限 | 专注 MLIR；主要检测 crash，缺少跨硬件误编译验证 |
| 与 RISC-V 研究的相关性 | 中；可迁移到 RVV/MLIR-RVV，但需 ISA 状态和差分执行约束 |
| 最适合作为 | GENERATOR/G4 工具与测试生成 baseline |

这篇论文最值得学习的是将语言模型生成、编译器反馈和有效样本回收组成持续改进的 fuzzer；最主要的局限是它对 MLIR 和 crash 检测的依赖较强。如果用于后续研究，合理方式是将其作为自适应测试生成框架，再加入 RVV 语义约束与跨编译器差分验证，而不是只更换输入格式。

## 元数据与分类核验

- Paper_ID：C224；Primary_Category：GENERATOR；Secondary_Category：G4_Tool_Test_Fuzz_Generation。
- DOI：无；arXiv：2510.07815。
- Code_Status：PUBLIC_REPO；Code_URL：https://github.com/zys-szy/FLEX；Code_Checked_At：2026-10-01。
