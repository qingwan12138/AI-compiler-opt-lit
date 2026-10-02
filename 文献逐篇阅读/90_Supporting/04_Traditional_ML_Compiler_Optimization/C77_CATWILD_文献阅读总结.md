# CATWILD 文献阅读总结

论文题目：**CATWILD: Compiler Autotuning for TPU Workloads in the Wild**

作者：Ignacio Cano、Yu Emma Wang、Mike Burrows、Ziqiang Feng、Matheus Camargo、Chao Wang、David H. Liu、Tengyu Sun、Alexander Wertheim、Arissa Wongpanich、Christof Angermueller、Hyojun Kim、Wenqi Cao、Aleksey Orekhov、Amit Sabne、Emma Sevastian、Mehrdad Khani、Karthik Srinivasa Murthy、Berkin Ilbeyi、Subhankar Shah、Ryan Lefever、Arjun Khare、Ankit Sinha、Peter Ma、Matthew Bierbaum、Jeremiah Wilke、Emily Donahue、Sami Abu-El-Haija、Nikhil Sarda、Vineetha Govindaraj、Shobha Vasudevan、Kirill Gugaev、Idan Nachman、Jie Sun、Jose Baiocchi Paredes、Samrat Ghosh、Domagoj Babic、Zongwei Zhou、Naveen Kumar、Phitchaya Mangpo Phothilimthana。

发表时间：2026 年

发表平台：Proceedings of Machine Learning and Systems 8，MLSys 2026 Conference，Industry Track，第 1608–1625 页（论文 PDF 第 18 页）。
元数据核验来源：[MLSys 2026 官方 proceedings 论文页](https://proceedings.mlsys.org/paper_files/paper/2026/hash/2093ed77c549eda95bd6f7212b735b43-Abstract-Conference.html)；[官方 PDF](https://proceedings.mlsys.org/paper_files/paper/2026/file/2093ed77c549eda95bd6f7212b735b43-Paper-Conference.pdf)

论文链接或编号：[官方 proceedings 页面](https://proceedings.mlsys.org/paper_files/paper/2026/hash/2093ed77c549eda95bd6f7212b735b43-Abstract-Conference.html)；[官方 PDF](https://proceedings.mlsys.org/paper_files/paper/2026/file/2093ed77c549eda95bd6f7212b735b43-Paper-Conference.pdf)。官方页面未显示 DOI，论文正文也未明确给出 DOI。
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

关键词：编译器自动调优、XLA、TPU、搜索空间、代价模型、性能预测、反馈驱动优化、生产部署、多芯片训练、tile-size tuning。

> 本文档依据官方 18 页 PDF 正文整理。论文事实与阅读后的研究思考分开描述；论文不属于 LLM 编译器论文，也不涉及 SFT、PPO、GRPO 或形式化等价验证。

---

## 1. 研究背景

本文研究 Google TPU 机群中的机器学习编译器自动调优。论文指出，XLA 依靠启发式规则和分析型代价模型处理算子融合、布局、tile size 等复杂问题，但模型、数据集、硬件和编译器版本持续变化，使固定规则或一次性调优迅速过时（第 1 节、第 2.2 节）。

传统做法包括人工分析性能 trace、修改编译器参数、反复实验，以及在编译时在线搜索。在线调优会把搜索开销直接加到用户作业延迟上；论文中的生产搜索空间约为 2^N 量级，且多芯片工作负载直接运行真实硬件的成本很高（第 2.2.2–2.2.3 节）。

因此，论文把问题转化为一个持续反馈回路：从生产机群采集工作负载和性能信息，在离线环境中搜索配置，用低成本预测器减少 TPU 试验，再以版本化、可验证和可回滚的方式部署配置（第 1、3、6 节）。

## 2. 论文要解决的问题

### 2.1 大规模搜索空间

论文需要在大量图级编译 flag 和算子级 tile size 配置中寻找低执行时间配置，同时避免人工枚举和逐候选多芯片执行（第 2.2 节、第 3.3 节）。

### 2.2 生产环境中的持续变化

工作负载、TPU 类型、拓扑、XLA 版本和编译器实现不断变化，一次性找到的配置可能失效或变得陈旧；系统需要持续重新调优、检测过时配置并维护版本适用性（第 1、3.4、5.1 节）。

### 2.3 调优结果的安全部署

配置既要带来性能收益，也不能引入编译失败、运行时失败或数值质量回退；还要支持配置共享、复现、回滚和在生产中的实际采用率监测（第 3.1、3.3、3.4、5.2 节）。

> 本文主要研究：如何在 Google TPU 生产机群中，以离线搜索、低成本性能预测和持续验证部署的方式，自动选择 XLA 编译配置。

## 3. 核心方法概述

CATWILD 是围绕 XLA/TPU 构建的端到端自动调优系统。它不是一个单独的搜索算法论文，而是把 fleet profiling、图/算子符号采集、可扩展 autotuner、单芯片性能预测器、版本化配置存储和持续验证组合成生产反馈系统（第 3 节）。

整体数据流：

```text
TPU 生产作业
    ↓
Fleet Profiling + Symbols Service 收集运行时间、计算图、算子和环境元数据
    ↓
分析管线按资源消耗筛选优先级高的图/算子
    ↓
Autotuner 搜索图级 flags 或算子级 tile sizes
    ↓
CPU 编译 worker 编译候选；TPU worker 执行候选或单芯片预测器估计多芯片运行时间
    ↓
选择相对默认配置更快的配置
    ↓
Configurations KV Store 按 graph/op fingerprint 存储和版本化
    ↓
XLA 编译时透明注入；Validator 持续检查速度、数值输出和版本兼容性
    ↓
生产 profiling 重新测量，形成持续反馈
```

图级 flag tuning 选择影响融合、内存分配和张量布局等决策的编译 flags；算子级 tile tuning 选择适合 scratchpad memory 和数据局部性的 tile sizes。论文没有使用语言模型作为系统角色；最终系统输出的是现有 XLA 执行的配置/参数，因此若按 taxonomy 角色分类，属于传统编译器自动调优支撑材料，而非 LLM Selector/Translator/Generator。

## 4. 实验框架与训练流程

本文不涉及一个待训练的主模型，主要采用生产系统执行流程；其中性能预测器内部使用机器学习模型估计通信成本。

### 4.1 Fleet Profiling 与符号采集

轻量版 XProf 持续监控 TPU 作业，收集图的聚合运行时间和元数据。编译器在编译期间上传未优化图和优化图：前者用于图级调优，后者提供后期 tile-size 选择所需的算子信息。图数据进入内部 Blobstore，元数据进入 Spanner；分析管线把 profiling 与符号数据连接起来，识别资源消耗最大的图和算子（第 3.2 节）。

### 4.2 Autotuner 搜索

CATWILD 扩展并重构 XTAT。对每个候选，系统分别编译并执行候选配置和默认配置，再按终止条件或时间预算停止。CPU-only worker 负责编译，TPU worker 负责执行，通过队列解耦两类资源。每项任务在隔离子进程中运行；编译崩溃和运行时错误作为预期失败处理，并由消息系统重试、持久化结果（第 3.3 节）。

### 4.3 单芯片性能预测器

预测器在低层 IR 中把通信操作改写为 no-op stub，使多芯片模型可以在单个 TPU 上运行并保留较多编译器优化。单芯片执行得到计算 profile；通信时间由基于微基准和 fleet profiles 训练的 ML 模型预测，再合成为多芯片运行时间投影（第 3.3 节）。论文报告的经验目标是约 5% 误差足以正确排序候选，但这不是普遍保证。

### 4.4 部署与持续验证

最佳配置按图/算子 fingerprint 写入版本化 KV store，并可静态嵌入用户二进制。Validator 复用编译/执行基础设施，比较新编译器版本下的性能和数值输出；发现回退或数值问题时使配置失效。若 tuned 配置导致编译失败，XLA 透明回退到默认配置（第 3.4 节）。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有报告端到端模型训练损失。核心优化目标是最小化配置对应的执行时间；实验中使用相对默认编译配置的 speedup。

论文明确给出或描述的量包括：

| 量 | 含义 |
|---|---|
| `speedup` | tuned 配置相对于编译器默认配置的相对性能提升；图级 flag tuning 的部分结果由预测器估计，算子级 tile tuning 可直接单芯片测量。 |
| `prediction error` | 预测 wall-clock runtime 与全尺寸 TPU 部署测得的 ground-truth wall-clock runtime 之间的误差，越低越好。 |
| `chips saved` | 论文按 `used chips × (speedup - 1)` 估算的节省芯片量。 |
| `2^N` | 多个二值或离散调优参数形成的候选配置空间的量级表达。 |

论文还使用机器学习模型输出通信成本预测，但正文没有给出统一的训练损失公式、模型参数规模或具体模型逐项配置；这些信息论文中未明确说明。论文不涉及奖励稀疏、奖励冲突或奖励投机问题。

## 6. 实验设置

### 6.1 数据集来源

数据来自 Google TPU 生产机群，而不是公开数据集。输入包括生产中的 ML 计算图、算子、profiling trace、编译环境元数据，以及用于通信成本模型的全尺寸 TPU 微基准和 fleet profiles（第 3.2–3.3 节）。论文说明系统运行和评估了大量真实工作负载，但没有公开训练/验证/测试集的传统数据集划分或样本总数。

系统以每天代表约 70% TPU training chip-time 的工作负载为调优对象；约 10% 是短作业或低资源作业，收益不足以抵消调优成本，另有约 20% 选择退出或不在每日调优覆盖范围内（第 4.2.1 节）。

### 6.2 模型与工具

| 类别 | 论文明确内容 |
|---|---|
| 编译器 | XLA；支持图级 compiler flag tuning 和算子级 tile-size selection。 |
| 硬件 | Google TPU 机群；真实部署包含单芯片和多芯片拓扑。具体 TPU 型号以匿名化的 ACC-A/ACC-B 等标签呈现。 |
| Profiling | XProf 的轻量版、Fleet Profiling Service。 |
| 性能预测 | 单芯片执行 + 低层 IR 通信 stub + ML communication cost model。 |
| 基础设施 | Spanner、Blobstore、内部消息队列、Configurations KV Store、Google 版本控制系统。 |
| 机器学习 | 用于通信时间预测；论文未明确给出统一模型名称、参数量或训练框架。 |
| 语言模型/形式验证 | 没有 LLM；没有形式化等价证明。 |

### 6.3 对比方法

本文不是以公开 baseline 表为中心的算法比较论文。正文相关工作列举了 XTAT、Lorien、学习式 cost model、roofline 等方法，但 CATWILD 的主要评估是系统覆盖率、预测误差、speedup、配置采用和芯片节省。与哪一个具体搜索算法在完全相同预算下比较，论文中未系统列明。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| Fleet/Symbol coverage | 成功收集 profiling、图和算子信息的覆盖率 | 越高越好 |
| Average relative speedup | tuned 配置相对 compiler default 的平均提升 | 越高越好 |
| Predictor error | 预测 runtime 与真实 full-scale runtime 的差异 | 越低越好 |
| Configuration hits/adoption | 生产作业命中并使用 tuned 配置的次数/比例 | 越高通常越好 |
| Chip savings | 按 speedup 和使用芯片量估算的节省 | 越高越好 |
| Profiling overhead | 对生产训练 step time 的额外开销 | 越低越好 |

## 7. 实验结果与结论

### 7.1 主要结果

Fleet Profiling 可覆盖约 90% 的 fleet accelerators；在已 profile 的对象中，Symbols Service 对 ops 的覆盖约 99%，对 graphs 的覆盖约 90%（图 8，第 4.1.1 节）。每日 profiling ingest 达到数十 TB，Symbols Service 上传数百 GB（第 4.1.2 节）。

CATWILD 为代表约 70% 每日 TPU training chip-time 的工作负载生成 tuned candidates。论文在 60 天窗口展示图级 flags 和算子级 tile tuning 的平均相对 speedup；图 9 使用图表呈现不同匿名 accelerator 的时间序列，正文没有将每个柱/曲线对应的精确数值完整列成表格，因此无法从当前 PDF 文本可靠抄录每个平台的单独平均值。

### 7.2 预测器结果

表 1 在四类匿名 accelerator 上评估若干代表性模型：模型数量分别为 28、26、19、29。平均预测误差分别为 3.5%、3.1%、4.8%、2.0%；95 分位误差分别为 14.3%、5.6%、13.5%、6.9%。误差是预测与真实全尺寸 TPU wall-clock runtime 的差异；真实运行用于科学验证，但日常对所有大模型执行真实多芯片评估成本过高（第 4.2.3 节）。

### 7.3 配置复用与芯片节省

图 10 显示，在三个月观察窗口内，图级 flag tuning 贡献总芯片节省约 80%，tile 配置贡献约 20%。tile 配置在不同图之间复用率更高、命中更多，但单次收益较低；graph fingerprint 加入 TPU 拓扑和版本等细粒度环境信息，增强正确性却降低了复用性（第 4.3.1 节）。

Validator 更新后的 graph-level flag 配置占每日配置命中的约 20%–60%，说明持续适配编译器和其他环境变化是必要的（图 11、第 4.3.1 节）。Fleet Profiling 的生产训练 step overhead 小于 0.1%；论文根据 2025 年成本估算 tuning operation 的 savings-to-cost ratio 为 8:1–30:1（第 6.1 节）。

### 7.4 消融实验

论文中未报告传统的逐模块 ablation table。可视为系统设计对照的分析包括：单芯片预测器相对直接多芯片执行的资源/准确率权衡、嵌入式配置与远程配置的延迟和复现性权衡、图级粗粒度 flag 调优与单算子细粒度调优的收益差异，以及 Validator 对配置采用率的动态影响。不能把这些生产经验表述为严格控制变量的消融结论。

### 7.5 数值安全与失败案例

论文指出，tile 改变浮点运算的分组和顺序可能引入数值差异。单个 fused op 的 tile 变化曾导致 2% accuracy loss；四个类似变化组合后曾导致 20% accuracy loss，说明局部数值检查不能替代更高层模型质量验证（第 5.2 节）。系统因此支持 model-owner 参与的定制验证流程，并在检测到问题后失效配置、恢复有效 checkpoint。

## 8. 主要创新点

### 8.1 创新点一：生产规模的持续自动调优闭环

论文把 profiling、候选搜索、低成本评估、版本化配置、生产注入和再验证连成持续 FDO loop。价值不在于提出一种全新的搜索算子，而在于解决自动调优从研究原型到 datacenter fleet 的系统接口、资源隔离、容错和生命周期问题。论文用约 70% training chip-time 覆盖、配置命中和节省结果证明其生产可用性。

### 8.2 创新点二：单芯片执行与通信成本模型结合的性能预测

通过低层 IR stub 保留较多实际编译行为，再用 ML 模型补回通信时间，系统用单个 TPU 近似多芯片候选成本。该设计把昂贵的真实多芯片试验转化为更可扩展的预测，但预测误差和通信抽象带来的偏差仍需持续验证。

### 8.3 创新点三：面向版本、复现和回滚的配置交付机制

graph/op fingerprint、静态嵌入、远程配置、Validator 和默认配置 fallback 共同处理编译器演进、配置失效和生产回退。论文明确展示了远程配置可把 ingestion-to-deployment latency 从 1–3 周降到 2–3 天，但也承认它引入 non-hermetic compilation 并损害复现信任。

### 8.4 创新点四：用调优数据反哺搜索空间和默认启发式

论文用随机森林、ridge regression、permutation importance、Gini importance 和 Shapley values 分析 flag 贡献，并据此在某些硬件平台把搜索空间最多缩小 80000 倍；同时把高收益配置中的规律反向用于改进编译器默认 heuristics。这是从“找配置”走向“减少配置空间并改善默认编译器”的闭环。

## 9. 局限性

### 9.1 论文明确承认的局限

- 系统强依赖 Google 内部 XLA、TPU、Spanner、Blobstore、profiling 和版本控制基础设施，外部组织无法直接复现全部生产环境（第 6.2 节）。
- 单芯片预测器通过通信 stub 产生的可执行程序可能在 VLIW bundling、调度、互连芯片状态和电压/热行为上不同于真实多芯片执行（第 5.3 节）。
- 预测器使用 dummy input，不能准确覆盖 value-dependent 操作，例如 SelectAndScatter 和新型 Pallas TPU kernels（第 5.3 节）。
- 数值检查不能完全保证模型质量；输入相关的问题可能只有更高层模型验证才能发现（第 5.2 节）。
- 远程配置降低部署延迟，却引入 non-hermetic compilation、调试复杂度和硬件失败风险；论文报告 2025 年芯片节省中远程配置约占 36%，嵌入式约占 64%（第 5.1 节）。
- 配置 fingerprint 越细，跨环境复用性越差；而图级调优结果的解释仍困难（第 4.3.1、5.5 节）。
- 论文没有公开完整的候选搜索算法、公开数据集或端到端可复现实验脚本；与多种 autotuner 的严格同预算比较也未完整报告。

### 9.2 阅读后发现的潜在局限

- 论文对匿名 accelerator 和内部工作负载的报告限制了外部读者判断跨硬件、跨编译器迁移性的能力。
- 论文的 80000× 搜索空间缩减来自生产数据和解释工具，不能直接理解为所有编译器/平台都能获得同样比例的缩减。
- 图级 speedup 部分依赖预测器而非逐候选真实全尺寸执行，因而应把它理解为预测支持的生产评估，而非全部结果都是真实多芯片 wall-clock 测量。
- 论文把 coarse-grained flags 作为可维护接口，但复杂的跨层依赖、参数条件和候选交互可能不适合简单二值 flag 表征。

## 10. 阅读后的研究方向反思

论文最值得借鉴的是“搜索空间治理 + 成本模型验证 + 安全部署”三者必须一起设计。对于本仓库方向，它更适合作为传统自动调优基础设施和代价模型的高质量支撑文献，而非 LLM 编译优化方法 baseline。

不能直接照搬的部分包括：Google 内部 profiling/存储/队列、TPU 专用低层 IR、XLA flag 语义和 fleet 规模。仅把 TPU 替换成 RISC-V，不足以构成新贡献；需要研究 RISC-V 后端特有的扩展组合、真实硬件计时、编译器版本漂移、向量长度/微架构差异和安全回退机制。

若映射到 taxonomy v2，建议主类为 `SUPPORTING`，二级类为 `B4_Traditional_ML_Compiler_Optimization`：论文是传统编译器自动调优和 ML 性能预测系统，并非语言模型输出编译 pass、代码或重写规则。若主代理更强调其大规模基础设施属性，可在说明中加注 B2 方向，但不应把它分类为 LLM `SELECTOR`。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V/RVV 的可验证搜索空间裁剪

#### 研究问题

能否利用 RVV 向量长度、掩码、访存形态、调度 flag 和后端成本信号，学习可解释的候选重要性，从而在真实 RISC-V 硬件上缩小 autotuning 搜索空间？

#### 与原论文的区别

不复用 TPU 的通信 stub，而是把 RISC-V 后端合法性、RVV 语义和真实硬件计时纳入候选筛选。

#### 可能的创新点

将 flag 重要性、硬件计数器和编译器版本作为可迁移但受约束的特征；对被裁剪候选保留反例回放。

#### 实验框架

```text
LLVM/RVV 候选空间 → 静态合法性过滤 → 小预算真实硬件采样
→ 解释模型估计 flag/参数贡献 → 搜索空间裁剪 → 真实硬件复验
```

#### 可行性

需要 LLVM、RISC-V/RVV 编译器、至少一种真实开发板或模拟器、性能计数器和可重复 benchmark。

#### 主要风险

模拟器与真实硬件排序可能不一致；硬件噪声、向量长度差异和 flag 交互会破坏简单重要性排序。

### 11.2 预测器误差感知的多保真编译自动调优

#### 研究问题

如何把静态成本、模拟器、快速仿真和真实 RISC-V 执行组合为带不确定性的多保真搜索器？

#### 与原论文的区别

CATWILD 主要使用单芯片执行和通信成本模型；该方向显式建模不同 fidelity 的偏差和置信度，并在排序不确定时主动请求真实硬件。

#### 可能的创新点

把预测区间、排序风险和硬件预算统一进候选选择策略，而不是只使用点估计 speedup。

#### 实验框架

```text
候选配置 → 静态/仿真/真实硬件多级评估 → 误差校准
→ 不确定性驱动候选选择 → 预算内真实硬件确认
```

#### 可行性

可从少量公开 RISC-V benchmark 和 LLVM pass/后端参数开始，不需要训练大型模型。

#### 主要风险

校准数据不足会使置信区间失真；多保真层的成本模型本身可能超过编译执行成本。

### 11.3 编译器版本漂移下的配置生命周期管理

#### 研究问题

如何检测某个 autotuned 配置在 LLVM、GCC 或 RISC-V 后端更新后仍然适用，并自动回退或重新搜索？

#### 与原论文的区别

将 CATWILD 的 fingerprint、Validator 和 fallback 思路迁移到开放编译器，并公开漂移数据与判定标准。

#### 可能的创新点

设计结合 IR 结构、目标 ISA、后端版本和真实计时的配置指纹，以及性能/正确性双阈值失效策略。

#### 实验框架

```text
版本化编译器与配置库 → 定期重编译/回归验证
→ 性能和语义检查 → 配置保留、降级或触发重新调优
```

#### 可行性

需要多个 LLVM/GCC 版本、可重复构建和 RISC-V benchmark；可先在 CI 或模拟器上进行，再用真实板卡抽样确认。

#### 主要风险

性能回退可能来自硬件、输入或环境变化而非编译器版本；过于严格的指纹会损害复用率。

## 12. 与其他已读文献的关系

当前批次只完成 CATWILD 一篇论文，因此不能虚构与其他已读论文的实证对比，也不存在已确认的批次内重复项。与 CARBS（候选检索中发现但因 ACM Cloudflare 阻断未完成 PDF 门槛）相比，CATWILD 更偏生产级端到端部署、成本预测和配置生命周期；CARBS 更偏通用编译 flag 搜索算法。CARBS 不应计入“已阅读论文”或正式去重成功项。

CATWILD 最适合作为传统 compiler autotuning 的系统/基础设施参考，尤其适合支撑搜索空间治理、低成本 evaluator、版本化配置和安全部署；它不是 LLM pass 选择、IR 改写或代码生成的 baseline。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 在 Google TPU 生产机群中进行持续、可部署的 XLA 编译器自动调优 |
| 核心问题 | 大搜索空间、真实多芯片评估昂贵、配置易过时且需要安全复现 |
| 输入 | 生产计算图/算子、profiling、编译环境、候选 flags/tile sizes |
| 输出 | 版本化 graph/op tuned configurations，由 XLA 透明应用 |
| 核心方法 | Fleet Profiling + XTAT 扩展 autotuner + 单芯片预测器 + Validator/fallback |
| 使用的模型 | ML communication cost model；具体模型规模未明确说明 |
| 使用的编译器工具 | XLA、XProf、低层 IR 改写、内部编译/执行 worker |
| 是否使用强化学习 | 否；本文没有使用强化学习奖励函数 |
| 是否使用形式化验证 | 否；使用数值检查、性能回归和编译失败回退，不是形式化证明 |
| 数据集规模 | 生产机群；约 70% daily TPU training chip-time 进入每日调优覆盖，公开数据集划分未说明 |
| 主要指标 | 覆盖率、speedup、预测误差、配置命中、chip savings、profiling overhead |
| 最重要实验结果 | profiling 约 90% accelerator 覆盖；预测平均误差 2.0%–4.8%；图级调优贡献约 80% 总芯片节省；profiling overhead <0.1% |
| 核心创新 | 把自动调优做成持续生产 FDO 闭环，并用单芯片执行降低多芯片搜索成本 |
| 主要局限 | 强依赖内部 TPU/XLA 基础设施；预测器、数值检查和匿名平台限制外部复现与迁移 |
| 与 RISC-V 研究的相关性 | 中：可借鉴搜索空间治理、预测器验证和配置生命周期，但 TPU 专用机制不能直接迁移 |
| 最适合作为 | 传统自动调优系统参考、搜索空间/代价模型支撑文献、生产部署设计参考 |

> 这篇论文最值得学习的是把搜索、成本评估、版本化交付和持续验证看成一个整体；最主要的局限是公开证据不足以复现 Google TPU 内部系统，也不能把预测 speedup 等同于所有候选都经过真实多芯片测量；如果用于后续研究，最合理的使用方式是借鉴其反馈闭环和生命周期设计，再用公开的 RISC-V/LLVM 工具与真实硬件建立可复现实验，而不是简单把 TPU 名称替换成 RISC-V。
