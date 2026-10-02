# AutoCFS-SLMs 文献阅读总结

论文题目：**Automating Compiler Flags Selection for SW Processors via Small Language Models**

作者：Chaowei Zhao, Jinyang Yao, Bo Zhao, Lili Liu, Haoran Wang, Yadian Cheng, Ping Zhang, Jinlong Xu, Kai Nie

发表时间：2026（Research Square 预印本，Posted Date: 2026-06-08）

发表平台：Research Square 预印本（2026）

论文链接或编号：DOI 10.21203/rs.3.rs-9831666/v1
元数据核验来源：[Research Square DOI](https://doi.org/10.21203/rs.3.rs-9831666/v1)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：编译器优化、编译 flag、Small Language Model、GCC、Gimple IR、SW64

> 本文档只记录正文可确认的事实；第 10—12 节另行标明阅读后的分析。论文 PDF 位于同目录的 `Automating_Compiler_Flags_Selection_SLMs_ResearchSquare2026.pdf`。

## 1. 研究背景

论文研究 GCC 编译器优化 flag 组合选择。工业编译器通常使用与程序无关的 `-O2`、`-O3` 等固定组合；它们在平均意义上稳定，但会错过面向具体程序和目标处理器的优化机会。迭代编译或遗传搜索能够探索更多组合，却需要反复编译和运行，成本很高；传统机器学习又依赖人工特征，可能泛化不足。

论文将 RISC-V、SW64 等新 ISA 和异构计算的发展作为背景，指出目标硬件差异进一步放大固定 flag 的适应性问题。作者讨论的 LLM 能理解代码和编译策略，但也指出大模型推理延迟、内存、上下文和数据成本较高，因此提出在受限资源环境使用小型语言模型（SLM）。

## 2. 论文要解决的问题

### 2.1 高维 flag 组合的推理成本

给定程序和目标处理器，如何从超高维离散 flag 空间中快速得到接近最优的组合，而不是为每个程序反复进行昂贵搜索。

### 2.2 语言模型与工业编译器的部署可靠性

模型推荐的激进或冲突 flag 可能导致编译失败，甚至产生运行时错误。论文研究如何在 GCC 接口处回退高风险 flag，维持可编译性。

### 2.3 小模型是否足以完成 flag 选择

论文比较专门化的 110M 参数 CodeBERT/SLM 与大模型，考察其在 SW64 上的性能、可靠性和推理资源。

> 本文主要研究：如何让一个面向 Gimple IR 的小型语言模型输出 GCC 优化 flag 组合，再由 GCC 执行实际编译，并用安全回溯处理失败。

## 3. 核心方法概述

AutoCFS-SLMs 把编译 flag 优化建模为多标签分类：输入 GCC Gimple IR，输出 85 维二进制 flag 向量；GCC 根据该向量编译，失败时按风险等级逐步删除 flag 并重试。

```text
C/C++ 源程序
      ↓
GCC 前端生成 Gimple IR
      ↓
CodeBERT/SLM 预测 85 个 GCC flag 的开关
      ↓
GCC 使用推荐组合编译
      ↓ 成功                    ↓ 失败
生成目标文件/可执行文件     删除高风险→中风险→低风险 flag 后重试
                                  ↓ 全部失败
                              回退到空 flag 集（论文称等价于 -O2）
```

LLM/SLM 的最终角色是 SELECTOR：它只输出现有编译器 flag 配置，GCC 执行变换。论文没有让模型直接输出 LLVM IR、源码或汇编变换。

## 4. 实验框架与训练流程

本文包含领域继续预训练和任务微调两个阶段，不使用强化学习训练语言模型。

### 4.1 领域继续预训练

作者以 110M 参数 CodeBERT 为基础，在 714 万行 Gimple IR 上进行掩码语言模型训练；数据来自 CodeNet、ExeBench、LLM 生成代码及同一程序的 `-O0` 到 `-O3` 版本。词表增加约 200 个 Gimple/SSA 相关 token，掩码率为 15%，训练 50,000 steps，batch size 128，AdamW 学习率为 `5×10^-5`。

### 4.2 任务特定微调

自适应遗传算法从 `-O3` 起点搜索 85 个附加 flag，保留经真实编译、运行且优于 `-O3` 的组合；约 5,000 个程序产生超过 100,000 个 `<Gimple IR, flags>` 样本。CodeBERT 加入 85 维分类头，使用 LoRA 微调 query/value 投影；预训练参数冻结，LoRA 与分类头约更新 0.4M 参数。

### 4.3 推理和编译器集成

GCC 插件在生成 Gimple IR 后加载量化模型，使用阈值 `τ=0.5` 将概率转为 flag 开关，再交给 GCC 后端。编译失败后按高、中、低风险顺序逐步撤回 flag。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数。关键目标和公式如下：

```text
F(C_i) = T(-O3) / T(C_i)
```

其中 `C_i` 是候选 flag 组合，`T` 是运行时间；遗传算法以相对 `-O3` 的 speedup 作为 fitness，三次稳定执行后测量。

```text
Pθ(y|G) = ∏(i=1..85) Pθ(y_i|G)
y_hat_i = sigmoid(W h_[CLS] + b)_i
```

`G` 为 Gimple IR，`y_i` 表示第 `i` 个 flag 是否启用，`h_[CLS]` 是 CodeBERT 的序列表示。分类损失是带不平衡权重的 binary cross-entropy：`w_i ∝ 1/sqrt(p_i + ε)`，其中 `p_i` 是 flag 的经验频率。

LoRA 更新形式为：`h = W0 x + (α/r) B A x`，论文给出 `r=8`、`α=16`。

安全回溯的期望额外编译次数公式使用初次成功概率和各风险级失败概率；论文根据实验估计 `p_success≈0.67`、高风险失败概率约为 1，得到 `E[extra]≈1.33`。这不是训练奖励，而是部署开销估计。

## 6. 实验设置

### 6.1 数据集来源

训练代码来自 CodeNet、ExeBench，并加入 LLM 生成代码；去重、过滤低质量代码及编译/运行失败样本。Gimple 预训练语料共 714 万行。任务数据约来自 5,000 个程序，保留超过 100,000 个优于 `-O3` 的 `<IR, flags>` 对。作者没有在 PDF 中完整给出训练/验证/测试程序的逐项规模划分，数据泄漏风险也没有专门量化说明。

评测使用 SPEC CPU 2017 integer speed 的 9 个程序：600.perlbench_s、602.gcc_s、605.mcf_s、623.xalancbmk_s、625.x264_s、631.deepsjeng_s、641.leela_s、648.exchange2_s、657.xz_s。

### 6.2 模型与工具

实验服务器为双 SW64 WX-H8000 2.00 GHz 处理器、NVIDIA A800 80GB GPU；编译器是 GCC 12.3.0。模型包括 CodeBERT 110M、CodeLlama-7B，以及 API 对比的 DeepSeek-V3.2、Qwen3-Max、GPT-5.1 和 Claude 4.5 Sonnet。论文说明 SW64 平台不支持 CUDA，因此 SLM 在 SW64 上做 CPU 推理；A800 用于内存测量。

### 6.3 对比方法

主要对比是 GCC `-O3`、AutoCFS-SLMs、四个通用 LLM 的相同 flag 推荐任务，以及 CodeLlama-7B 的推理效率。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
| --- | --- | --- |
| speedup | 相对 GCC `-O3` 的运行时间比 | 越大越好 |
| geometric mean | 9 个 SPEC 程序的几何平均 speedup | 越大越好 |
| 编译/运行成功率 | 推荐 flag 能否生成并运行有效程序 | 越大越好 |
| 推理时间 | 模型完成推理的时间 | 越小越好 |
| 峰值 GPU 内存 | 推理所需显存 | 越小越好 |
| 额外编译次数 | 安全回溯引入的编译尝试 | 越小越好 |

## 7. 实验结果与结论

### 7.1 主要结果

AutoCFS-SLMs 在 9 个 SPEC CPU 2017 integer speed 程序上相对 `-O3` 均达到优于或相等的结果，几何平均 speedup 为 7.6%。表 1 的逐项 speedup（相对 `-O3`）为：600 6.64%、602 4.83%、605 25.14%、623 8.58%、625 0.41%、631 12.44%、641 11.17%、648 0.49%、657 1.32%；表中几何平均为 7.65%。

### 7.2 与传统方法的比较

论文的直接传统 baseline 是固定 `-O3`。遗传算法用于构造训练标签，不是最终部署时的 baseline。论文没有报告与 OpenTuner 的完整同表结果。

### 7.3 与其他 LLM 方法的比较

Qwen3-Max 的几何平均为 7.03%，SLM 为 7.65%；DeepSeek-V3.2、GPT-5.1 和 Claude 4.5 Sonnet 因编译或运行错误存在缺失项，因此论文没有为它们计算完整几何平均。SLM 通过安全回溯在 9 个评测程序上达到 100% 编译和运行成功率；这些结果来自论文表 1 与第 6.2 节的描述。

### 7.4 消融实验

论文没有给出独立的模块消融表。它报告了与 CodeLlama-7B 的效率比较：CodeLlama-7B CPU 推理 29 分 11 秒，CodeBERT 36.38 秒，前者约为后者 48.1 倍；A800 峰值显存分别为 11.04 GiB 与 0.43 GiB，约 25.4 倍。该比较不是严格的 AutoCFS 模块消融。

### 7.5 案例分析

论文举例说明高风险 flag（如 `-ffinite-math-only`、`-fdelete-null-pointer-checks`）可能影响标准遵循或未定义行为风险；中风险例子包括 `-fmodulo-sched`、`-fsched-spec-load`，低风险例子包括 `-mcpu=sw8a`、`-fgcse-las`。论文没有提供逐程序的 flag 序列案例解释。

## 8. 主要创新点

### 8.1 创新点一：面向 Gimple IR 的轻量 flag 选择

论文把 85 个 GCC 优化 flag 作为多标签输出，使用继续预训练的 CodeBERT 学习 Gimple IR 到 flag 组合的映射；价值在于将在线搜索转为一次模型推理。实验以 SW64/SPEC CPU 2017 支持这一设计，但仅证明于论文给出的实验设置。

### 8.2 创新点二：编译安全回溯

论文在 GCC 接口对 flag 风险分级，编译失败时按风险等级删除并重试，最后回退基线。该机制是部署工程与可靠性设计，论文报告平均增加 1.33 次额外编译。

### 8.3 创新点三：真实硬件驱动的 flag 数据构造

自适应遗传算法在真实硬件上搜索并保留优于 `-O3` 的组合，再形成微调数据。论文将其作为专门化 SLM 的数据基础，但标签仍是近似最优而非全局最优。

## 9. 局限性

### 9.1 论文明确或正文可见的局限

论文把实验集中在 SW64 处理器、GCC 和 SPEC CPU 2017 integer speed；尚不能据此断言适用于 RISC-V 或其他 GCC target。训练标签来自有限预算遗传搜索，是 near-optimal 组合。论文没有报告完整训练/验证/测试划分、公开代码或数据下载地址，也没有同行评审平台信息。

### 9.2 阅读后发现的潜在局限

85 维独立 sigmoid 输出忽略 flag 之间的依赖、冲突和顺序关系；安全回溯按风险等级撤回可能丢失有益的协同组合。训练数据只保留优于 `-O3` 的样本，可能造成选择偏差。论文称空 flag 集等价于 `-O2`，这一实现细节需要在复现时核查。实验只报告 9 个 SPEC 程序，跨编译器版本、跨 ISA 和真实产品代码的泛化仍未确认。

## 10. 阅读后的研究方向反思

本文最值得借鉴的是“模型输出受控配置、现有编译器执行变换”的角色边界，以及在配置失败时保留可执行回退路径。它适合作为 S1 flag selection baseline，不应被理解为 LLM 直接生成优化变换。把 SW64 直接替换成 RISC-V 只构成平台迁移；若要形成新问题，还需研究 RISC-V `-march/-mabi`、扩展可用性和微架构性能反馈如何进入选择模型，并验证跨核心迁移。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V 扩展约束的 flag 选择

#### 研究问题

模型能否在不同 `-march`/`-mabi` 和 RVV 配置下选择合法且高性能的 GCC flag。

#### 与原论文的区别

加入显式 ISA 能力约束和目标微架构条件，不是只替换 SW64 名称。

#### 可能的创新点

将 flag 依赖/冲突图、扩展可用性和硬件计数器联合编码，并对非法配置做编译前屏蔽。

#### 实验框架

```text
RISC-V 源程序→Gimple IR+ISA 元数据→受约束 S1 模型→GCC→真实板卡性能→校准/回退
```

#### 可行性

需要 GCC RISC-V 后端、QEMU 与至少一种真实 RV64/RVV 平台，以及 SPEC/PolyBench 子集。

#### 主要风险

真实硬件数量和测量噪声有限，模型可能学习到 benchmark/核心关联而非通用规律。

### 11.2 结构化 flag 依赖下的层次选择

#### 研究问题

能否先选 flag 家族，再在家族内选择具体选项，降低 85 维独立分类的冲突率。

#### 与原论文的区别

把输出从独立 bit vector 改为合法配置图上的层次策略，并做组合级验证。

#### 可能的创新点

以 GCC option metadata 和编译失败日志构建约束解码器，比较独立 sigmoid、图约束和序列策略。

#### 实验框架

```text
Gimple IR→flag 家族选择→依赖/冲突约束过滤→具体 flag 配置→GCC 编译/运行
```

#### 可行性

可复用本文的 Gimple IR 数据构造方式和 GCC 插件接口。

#### 主要风险

过强约束可能删除真实有益组合，且配置空间仍可能存在长程协同。

## 12. 与其他已读文献的关系

本 staging 子批次只完成本文一篇正文阅读，因此不存在可据正文建立的同批次横向比较。根据正式 corpus 去重检查，已有的 `Large Language Models for Compiler Optimization`（2023）也是模型选择 compiler options 的 SELECTOR/S1，但本候选是 2026 年、面向 Gimple IR/CodeBERT/SM64 GCC 的新工作；DeCOS（2025）已在正式 corpus 中，属于 LLM 点火的 RL 优化序列选择，但不是本候选的独立 staging 论文。上述关系仅用于去重与角色定位，不能替代对这些论文正文的本轮复读。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | GCC 编译 flag 组合选择 |
| 核心问题 | 固定 `-O3` 不能适应程序/硬件，迭代搜索昂贵 |
| 输入 | Gimple IR |
| 输出 | 85 维 GCC flag 二进制配置 |
| 核心方法 | Gimple 领域预训练 + CodeBERT/LoRA 多标签预测 + 安全回溯 |
| 使用的模型 | CodeBERT 110M；对比 CodeLlama-7B 和通用 LLM |
| 使用的编译器工具 | GCC 12.3.0、Gimple IR、GCC 插件 |
| 是否使用强化学习 | 否；遗传算法用于标签搜索 |
| 是否使用形式化验证 | 否；仅编译/运行成功检查与风险回溯 |
| 数据集规模 | 714 万行 Gimple IR；约 5,000 程序、超过 100,000 对任务样本 |
| 主要指标 | 相对 `-O3` speedup、成功率、推理时间、显存 |
| 最重要实验结果 | SPEC CPU 2017 integer speed 9 程序几何平均约 7.6% speedup；SLM 100% 编译/运行成功 |
| 核心创新 | 轻量 SLM 输出受控 flag 配置并接入 GCC，失败时风险分级回退 |
| 主要局限 | 单一 SW64/GCC/SPEC 设置；近似标签；未提供完整划分和跨 ISA 证据 |
| 与 RISC-V 研究的相关性 | 中：论文将 RISC-V 作为背景，但未在 RISC-V 上实验；可作为 flag 选择 baseline |
| 最适合作为 | S1 SELECTOR baseline、GCC flag 数据构造和安全回溯参考 |

这篇论文最值得学习的是把语言模型限制在可审计的编译 flag 配置输出，并由 GCC 负责实际变换与失败回退；最主要的局限是实验证据集中在 SW64 单一目标和小规模 benchmark。用于后续 RISC-V 研究时，合理方式是引入 ISA/微架构约束和跨平台测量，而不是简单替换目标处理器名称。
