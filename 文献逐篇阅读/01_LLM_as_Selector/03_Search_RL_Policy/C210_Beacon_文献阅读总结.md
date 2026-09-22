# Beacon 文献阅读总结

论文题目：**Beacon: LLM Multi-Agent Driven Hardware Design Space Exploration for Heterogeneous Multi-Chiplet Deep Learning Accelerators**
作者：Boyu Li、Zongwei Zhu、Qianyue Cao、Xi Li、Xuehai Zhou
发表时间：2026；arXiv 首次公开 2026-08-31。
发表平台：arXiv 预印本；PDF 未给出正式会议/期刊。
论文链接或编号：[arXiv:2608.30932](https://arxiv.org/abs/2608.30932)；DOI：未找到。
PDF：`Beacon_2608.30932.pdf`，14 页，`%PDF-` 签名有效，pypdf 可解析。
代码：Code_Status=NOT_FOUND；Code_URL=NOT_FOUND；Code_Checked_At=2026-09-23。

> 本文档事实来自本地 PDF；分类和研究建议属于阅读后的分析。

## 1. 研究背景

异构多芯粒加速器需要同时决定芯粒数量、芯粒类型、数据流、片上缓冲、DRAM/NoP 带宽、tensor parallelism 和 micro-batch。搜索空间与仿真成本高，单纯随机、贝叶斯优化或 RL 难以从详细执行报告中定位瓶颈。Beacon 使用三个层次的 LLM agent，从“哪里慢、为什么慢、如何改硬件”逐层生成下一步合法配置。

## 2. 论文要解决的问题

### 2.1 从复杂报告中定位可优化区域

模型级 agent 先比较搜索状态并选择分析基准，再定位高影响 layer/operator group。

### 2.2 解释瓶颈根因

layer-level agent 使用计算、内存和通信工具生成 Bottleneck State Description（BSD）。

### 2.3 在约束下提出硬件动作

solution agent 将 BSD、历史案例和合法硬件空间转为候选配置，由工具归一化后交给 evaluator。

> 本文主要研究：如何用层次化 LLM agent、工具调用和 RAG 记忆，在有限评估预算下选择异构多芯粒加速器的硬件配置。

## 3. 核心方法概述

Beacon 的 LM 不直接输出 RTL 或 kernel；它输出瓶颈判断、修改意图、参数动作和候选选择。确定性 Analysis Toolbox 与 Evaluator Adapter 将这些动作物化为合法硬件配置并运行评估。

```text
workload + hardware search space + 初始配置
        ↓
Evaluator 产生 latency/energy/cost 和详细报告
        ↓
Model-Level Agent：选择分析基准、候选层
        ↓
Layer-Level Agent：生成 BSD 和根因
        ↓
RAG Memory 检索相似历史案例
        ↓
Solution Agent：提出参数动作/硬件候选
        ↓
工具归一化合法配置 → Evaluator
        ↓
指标与轨迹写回，开始下一轮
```

## 4. 实验框架与训练流程

### 4.1 层次化 ReAct

三个 agent 都采用 ReAct 风格：LLM 决定工具调用，工具返回结构化证据，达到条件后生成严格 JSON。模型级 agent 不提出硬件改动，只选分析区域；layer-level agent 诊断根因；solution agent 生成最终硬件修改意图。

### 4.2 RAG Memory

每轮将 BSD、前后硬件配置、动作和指标组成历史案例。检索键同时包含 BSD embedding 和 hardware embedding，返回 top-5 案例给 solution agent。

### 4.3 评估与候选物化

Evaluator Adapter 负责硬件候选格式归一化、仿真调用和报告标准化。最终候选不是自由文本直接执行，而是由硬件分析工具转换为合法配置。

### 4.4 RL 对照流程

论文实现 RL baseline：以硬件资源和反馈为状态，以合法修改为动作，以 `r = log(C_prev / C_new)` 为奖励，并采用 clipped policy objective。RL 是对比方法，不是 Beacon 的核心训练流程。

## 5. 奖励函数、损失函数或关键公式

Beacon 的 RL baseline 奖励为：

```text
r = log(C_prev / C_new)
```

其中 `C_prev` 是动作前目标值，`C_new` 是动作后目标值；目标值下降时奖励为正。Beacon 本身主要是在线提示、工具调用和历史案例检索，不依赖该 RL 奖励训练 LLM。总目标联合 latency、energy 和 monetary cost，具体归一化组合按实验设置计算。

## 6. 实验设置

### 6.1 数据集来源

论文没有使用传统固定数据集；评估 workload 是 LLM inference 场景，搜索过程中动态产生硬件配置、执行报告和历史案例。实验覆盖 64 TOPS、512 TOPS 和 2048 TOPS 三种计算规模，每种方法从相同初始配置搜索 100 轮。

### 6.2 模型与工具

- LLM backend：DeepSeek-V4-Pro；另比较 Gemini-3.1-Pro、GPT-5.5、Claude-Opus-4.8。
- RAG embedding：BGE-small-en-v1.5，top-5 检索。
- 硬件模拟/评估：Evaluator Adapter 及多芯粒加速器 evaluator。
- 服务器：2 个 Hygon 7285 处理器、128 logical cores、4 张 A100 GPU；主要仿真跑 CPU，embedding 和 RL 更新使用 GPU。

### 6.3 对比方法

比较 Random Search、Bayesian Optimization 和 RL baseline；还做 single-agent、去除 RAG Memory、去除 ReAct 的消融。

### 6.4 评价指标

| 指标 | 含义 |
|---|---|
| Latency | 推理执行延迟，越低越好 |
| Energy | 能耗，越低越好 |
| MC | monetary cost，越低越好 |
| Composite objective | 三项归一化联合目标，越低越好 |
| Search runtime/API cost | 搜索和调用开销 |

## 7. 实验结果与结论

### 7.1 主比较

相对每个规模下最强传统 baseline，Beacon 的联合目标在 64 TOPS、512 TOPS、2048 TOPS 分别降低 79.1%、77.0%、25.1%。论文强调优势来自报告驱动的层次诊断，而不是单纯增加随机样本。

### 7.2 配置结果

64 TOPS 选择 2 个异构芯粒；512 TOPS 选择 8 个芯粒，其中 7 个 WS、1 个 OS；2048 TOPS 选择 32 个芯粒，其中 25 个 WS、7 个 OS。配置没有随计算规模简单线性扩张，而是联合考虑数据流、带宽、TP 和 micro-batch。

### 7.3 LLM backend 比较

在 512 TOPS 实验中，Gemini 找到约为 DeepSeek 0.90 倍的目标值；GPT 与 DeepSeek 接近；Claude 目标值约为 DeepSeek 的 1.28 倍但延迟较低、成本较高。DeepSeek 估算 API 成本约 4.94 美元，Gemini 约 48.98 美元，GPT 约 149.03 美元，Claude 约 224.47 美元。

### 7.4 消融实验

512 TOPS 归一化目标：完整 Beacon 为 1.000；去除 ReAct 为 1.159；去除 RAG Memory 为 1.301；single-agent 为 1.499。结果支持层次化 agent、RAG 记忆和动态工具调用的作用。

### 7.5 开销

512 TOPS 主运行包含约 2270 次 LLM API 调用、约 35.9M tokens；evaluator 占运行时间 74.5%，solution agent 平均调用时间最高。该开销说明 Selector 决策本身不是唯一瓶颈，底层硬件评估仍然昂贵。

## 8. 主要创新点

### 8.1 “哪里—为什么—如何”层次化决策

Beacon 不让一个 agent 直接读完整报告并输出配置，而是把全局定位、局部诊断和硬件修改拆开，提升可追踪性。

### 8.2 BSD 作为稳定中间表示

BSD 将详细报告压缩为目标、根因、layer diagnosis 和推荐关注参数，并作为 RAG 检索键，连接当前诊断与历史案例。

### 8.3 工具约束下的候选物化

LLM 提供高层修改意图，工具保证参数合法、格式规范并交给 evaluator，明确区分语言推理和硬件执行。

## 9. 局限性

### 论文正文显示的局限

- 评估依赖特定多芯粒模拟器和预定义硬件空间。
- 100 轮搜索和大量 API 调用仍有明显成本。
- 未报告公开代码或可直接复现实验包。
- 目标是硬件配置，不是 HLS pragma 或 LLVM schedule。

### 阅读后的潜在局限

候选动作最终由工具物化，因此论文适合作为 Selector，而非 Generator；但其与 HLS 的直接关系较弱。迁移到 RISC-V/RVV 需要重新定义硬件参数、性能报告和合法动作，而不能直接沿用多芯粒字段。

## 10. 阅读后的研究方向反思

最有价值的是 BSD 这种可审计的“反馈到动作”中间层。它可作为 RISC-V 后端 Selector 的证据契约：将 cache miss、向量利用率、stall 和编译失败映射到 pass/config 动作。其多 agent 架构本身不应直接照搬，必须证明分层是否减少错误并控制调用成本。

## 11. 可进一步尝试的研究方向

### 11.1 RISC-V 后端 BSD Selector

研究问题：从 LLVM 优化报告、硬件计数器和运行时间构造 BSD，再选择 pass、RVV 参数或调度动作。区别是动作由 LLVM/后端执行并可验证。风险是反馈跨平台不稳定。

### 11.2 预算感知的动作门控

研究问题：让 solution agent 根据编译/运行预算决定是否继续工具调用。区别是把评估成本纳入动作选择。风险是节省调用可能牺牲搜索质量。

## 12. 与其他已读文献的关系

与 LUMINA 相同点是 LM 选择硬件 DSE 动作；LUMINA 从 simulator code 构造架构启发式知识，Beacon 更强调详细执行报告、层次化 agent 和 RAG 历史案例。与 LLM-DSE、AI4DSE、ChatHLS 相比，Beacon 不是 HLS pragma 选择，而是多芯粒架构配置 Selector。与传统 RL baseline 的差异是 Beacon 用自然语言模型和结构化工具直接利用报告语义。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 异构多芯粒加速器硬件 DSE |
| 核心问题 | 有限预算下从详细报告中选择有效硬件动作 |
| 输入 | workload、硬件空间、搜索状态、评估报告 |
| 输出 | 合法硬件配置和搜索轨迹 |
| 核心方法 | 三层 agent、Analysis Toolbox、BSD、RAG Memory、Evaluator Adapter |
| 使用的模型 | DeepSeek-V4-Pro；比较 Gemini/GPT/Claude |
| 使用的工具 | 硬件 evaluator、结构化分析工具、向量检索 |
| 是否使用强化学习 | Beacon 否；RL 仅作 baseline |
| 是否使用形式化验证 | 否；依赖 evaluator |
| 数据集规模 | 三种 TOPS 规模，每方法 100 轮搜索 |
| 主要指标 | latency、energy、MC、联合目标、API/运行开销 |
| 最重要实验结果 | 联合目标相对最强传统 baseline 降低 25.1%–79.1% |
| 核心创新 | 层次化“哪里—为什么—如何”与 BSD/RAG 闭环 |
| 主要局限 | 模拟器依赖、成本高、缺公开代码、非 HLS 直接任务 |
| 与 RISC-V 研究的相关性 | 中；反馈契约可迁移，硬件空间需重建 |
| 最适合作为 | SELECTOR/S3 搜索策略与反馈契约 baseline |

这篇论文最值得学习的是把 LLM 的硬件决策限制在工具可执行、可审计的动作上；最主要的局限是评估器和硬件空间高度特定；用于后续研究时应迁移 BSD 与预算门控，而不是简单替换芯粒参数为 RISC-V 参数。
