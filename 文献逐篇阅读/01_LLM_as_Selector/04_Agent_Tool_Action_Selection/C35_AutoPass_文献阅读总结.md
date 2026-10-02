# AutoPass 文献阅读总结

论文题目：**AutoPass: Evidence-Guided LLM Agents for Compiler Performance Tuning**
作者：Zepeng Li, Jie Ren, Zhanyong Tang, Jie Zheng, Zheng Wang
发表时间：2026-06-18；发表平台：arXiv 预印本（v1；PDF 首页会议栏仍为占位文本）
论文链接或编号：[arXiv:2606.20373](https://arxiv.org/abs/2606.20373)；论文 PDF 首页的会议信息仍为占位，暂记 arXiv 预印本；代码仓库未确认。
元数据核验来源：[论文/来源](https://arxiv.org/abs/2606.20373)；[论文/来源](https://arxiv.org/abs/2606.20373v1)
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）
关键词：LLVM、Pass 调优、多智能体、编译器证据、运行时反馈、免训练

> 阅读模型：gpt-5.6-luna；本轮日期：2026-09-05；实际 PDF：arXiv 2606.20373v1（12 页）。发表会议正式信息在 PDF 中未明确说明。

## 1. 研究背景

LLVM/GCC 的 `-O3` 是固定的专家 pass pipeline，而不同程序、硬件和输入往往需要不同顺序及参数。现代编译器已有上百个 transformation/analysis pass，phase ordering 和参数组合形成巨大、稀疏且程序依赖的搜索空间。传统 PGO、AutoFDO、CSSPGO 主要在固定 pipeline 内调 heuristics，依赖代表性 profile；OpenTuner 等黑盒搜索需要大量 compile-run。学习策略又需要离线数据和跨架构迁移。

AutoPass 的出发点是性能目标不能只靠静态源码推理：微架构、cache、分支和向量化产生噪声，LLM 需要看到 compiler remarks、LLVM IR 和真实 runtime。它因此把 compiler 从黑盒变成可查询的证据源，并采用 inference-only、多智能体、三轮内反馈，不做离线训练。

## 2. 论文要解决的问题

### 2.1 小预算性能调优

在目标侧只有三轮 compile-run 的情况下，能否超过 PGO 和三轮 OpenTuner（RQ1）。

### 2.2 反馈作用

迭代反馈相对一次生成提升多少性能和稳定性（RQ2）。

### 2.3 架构适应与可诊断性

系统能否在 x86-64 与 ARM64 采用不同 pass 行为（RQ3），Score Agent 能否找到比 PGO 更有价值的函数（RQ4），多智能体哪些组件重要（RQ5），compiler/runtime evidence 是否使决策可解释（RQ6）。

> 本文主要研究：一个不训练、以 LLVM 内部 remarks/IR 和真实 runtime 为证据的多智能体系统，能否在有限编译执行预算内生成、验证并迭代优化 pass pipeline。

## 3. 核心方法概述

```text
源代码/调用图/LLVM IR + -O3 remarks + 硬件信息
        ↓
Score Agent：静态特征筛选高潜函数
        ↓
Analysis Agent：语义 hint、-Rpass/-Rpass-missed/-Rpass-analysis 归一化
        ↓
Reasoning Agent：从 -O3 选择/重排/参数化合法 pass
        ↓
修复语法、映射非法 pass、成员/参数/LLVM 验证
        ↓
Executor/Performance Agent：编译、执行三次、收集 runtime、计数器与新 remarks
        ↓
Evaluation/Judge：比较当前最优，诊断退化，反馈下一轮；退化则 rollback
```

Score Agent 使用 basic-block、loop、call、conditional-branch 数等特征建立函数优先级；只把高潜且上下文可容纳的 raw IR 交给后续。Reasoning Agent 只能从初始 `-O3` pass 集合内编辑，验证候选 schema、pass membership、参数范围和编译成功。`P*` 为迄今最快合法 pipeline，候选均值更快才接受；最终只有超过 `-O3` 才输出，否则回退。

## 4. 实验框架与训练流程

本文不涉及预训练、SFT 或强化学习训练。系统使用 DeepSeek-V3.2 作为主要 reasoning backend，以 CrewAI 组织四类 agent；另比较 ChatGPT-4o、Qwen3、Gemini 3 Flash。每轮候选编译并运行三次，Evaluation Agent 使用均值；报告 benchmark binary 时 baseline 和优化版各运行五次。

系统的反馈不是奖励函数，而是文字诊断：例如实验 trace 中 Round 1 时间由 1.5355 s 增至 1.5474 s（+0.77%），L1 misses 约 0.96M 增至 2.25M（+133.1%），Evaluation Agent 将其归因于 unroll-count=8、threshold=600 和过宽 SLP vectorization，下一轮调保守，第三轮达到 1.4941 s、相对 `-O3` 1.028×（第10页）。

## 5. 奖励函数、损失函数或关键公式

本文没有强化学习奖励函数。性能指标为：

```text
Speedup = T_O3 / T_opt
```

其中 `T_O3` 和 `T_opt` 是各 binary 五次运行均值；套件结果用 benchmark-level geometric mean。迭代接受条件是 `t(P_candidate) < t(P*)`，其中候选均值为三次执行均值；小于 1.0 的性能退化在启用 rollback 时丢弃并回到 `-O3`。这是一种确定性选择/回滚规则，不是 RL reward，也没有模型参数更新。

## 6. 实验设置

### 6.1 数据集来源

包含 cBench 31 个通用程序、PolyBench 30 个循环密集 kernel、CoreMark 1 个嵌入式基准、MiniFE 1 个稀疏线性代数 proxy、LULESH 1 个冲击流体 proxy，共 64 个 distinct workloads（表4，第5页）。cBench 主分析、其余套件用于跨领域测试。论文没有将它们划分为训练/验证集，因为系统免训练。

### 6.2 模型与工具

两平台：Intel Core i9-11900K x86-64 server（64GB）和 Raspberry Pi 5 Cortex-A76 ARM64（8GB）；Ubuntu 20.04 LTS，server 关闭 Turbo/动态频率。LLVM/Clang 17.0.6 New Pass Manager，考虑 74 个 LLVM optimization passes，序列最多 107 个 pass。编译器 remarks 使用 `-Rpass`、`-Rpass-missed`、`-Rpass-analysis`；硬件计数器用于诊断。LLM 主 backend 为 DeepSeek-V3.2。

### 6.3 对比方法

Instrumentation PGO、CSSPGO（仅 x86）、AutoFDO、OpenTuner（三次搜索与 AutoPass 同预算）；另报告 OpenTuner 500 次作为高预算参考。所有比较启用 rollback 时，speedup<1 自动丢弃。未纳入 ACPO 是因权重不可复现，Autophase/CompilerGym 主要目标并非本文 runtime。

### 6.4 评价指标

speedup 越大越好，套件用 geometric mean；Wins 为 speedup≥1，Losses 为 <1；pass coverage 以 compiler remarks 中 pass 生效次数相对 `-O3` 的增/减/不变为代理；Edit Similarity（ES）衡量 pass 序列拓扑相似度。runtime 是真实目标平台执行时间，不是静态 latency；静态 blocks/loops 是函数优先级特征而非性能结果。

## 7. 实验结果与结论

### 7.1 总体结果

表 5（第6页）启用 rollback：x86 cBench/PolyBench/CoreMark/MiniFE/LULESH 的 AutoPass R3 分别为 1.059×、1.009×、1.137×、1.008×、1.102×；ARM64 分别为 1.111×、1.149×、1.091×、1.068×、1.046×。论文摘要给出的套件几何均值为 x86 1.043×、ARM64 1.117×，相对 LLVM-O3。R3 在 10 个平台-套件设置中 9 个最佳。

### 7.2 无 rollback 与迭代

表 6 cBench 无 rollback：x86 AutoPass R3 1.040×、25 wins、6 losses、最大 1.366×、最小 0.784×；ARM64 1.109×、27 wins、4 losses、最大 2.040×、最小 0.961×。三轮优于一轮：x86 R1 1.010×/13 losses，R3 1.040×/6 losses；图3显示从一轮 1.010×到二轮 1.033×，三轮 1.040×，之后接近饱和，因此采用三轮预算。OpenTuner 500 轮 x86 1.057×、ARM64 1.126×，但预算远高于 AutoPass。

### 7.3 架构行为

AutoPass 增加 loop-unroll coverage 的 benchmark 比例为 x86 90.3%、ARM64 93.5%；LICM 增加约 55–61%。SLP vectorization 在 x86 增加 32.3%、减少 35.5%，ARM64 增加 41.9%、减少 25.8%，表现出架构敏感性。表7 ES：AutoPass vs `-O3` x86 0.943±0.050、ARM64 0.930±0.042；x86 与 ARM pipeline 互相 ES 0.917±0.046，不是同一策略简单复制。

### 7.4 后端消融与案例

表10 cBench 无 rollback 中，DeepSeek R3 x86 1.040±0.114、ARM64 1.109±0.206；ChatGPT-4o R3 1.029/1.088，Qwen3 1.040/1.080，Gemini 3 Flash 1.040/1.091（表中 ± 为 speedup 跨 benchmark 标准差）。这说明反馈可补偿单轮模型差异，但不是每一轮都无回归。case trace 将 cache miss、IPC、pass 参数和修复方向关联，属于可解释诊断，不是因果证明。

## 8. 主要创新点

### 8.1 compiler evidence 与 runtime evidence 联合闭环

区别于黑盒 autotuning，模型直接读取 IR snapshots、optimization remarks 和硬件 runtime，能解释为何改变 pass。

### 8.2 受约束的 pipeline 编辑与确定性回滚

候选只来自合法初始 pass 集，脚本修复括号和非法 token，验证后才运行，性能回退保留当前最优；这把 LLM 的开放生成限制在 LLVM 合法空间。

### 8.3 免训练多智能体跨架构调优

Score/Analysis/Reasoning/Evaluation 分工，三轮反馈，避免离线 RL 的训练成本。实验显示 x86/ARM pipeline 拓扑和 vectorization 行为不同。

## 9. 局限性

**论文明确承认/可见局限：** runtime 测量有噪声；三轮预算有限；比较只覆盖 LLVM 17.0.6、两种硬件和所列基准；OpenTuner 500 轮结果说明高预算搜索可更高；模型依赖目标侧 compile-run 与硬件计数器；LLM context 使 Score Agent 必须丢弃低优先函数。论文 PDF 首页出版信息和若干会议字段仍是占位文本。

**阅读后潜在局限：** pass coverage 次数是 remarks 代理，不等于实际周期收益；ES 只比较序列拓扑，参数和后端调度仍可能完全不同；每轮三次和最终五次的统计口径不同，不能将其当作统一置信区间；没有形式化正确性证明，LLVM 编译成功只证明 pipeline 合法；RISC-V/RVV 未实验，不能由 ARM64 结果直接继承。

## 10. 阅读后的研究方向反思

AutoPass 最适合作为 LLVM pass-level runtime tuning baseline/agent 模块。值得借鉴的是让 remarks、IR、硬件计数器成为 LLM 的可审计证据，并把 rollback 作为性能安全边界；核心贡献是免训练多 agent 闭环，简单替换 DeepSeek 或目标 ISA 不是新问题。对 RISC-V，可研究 RVV VLEN、寄存器压力、代码尺寸和真实板卡测量如何进入证据与回滚，而不是把 x86/ARM pass 结论直接迁移。

## 11. 可进一步尝试的研究方向

### 11.1 RISC-V/RVV 证据条件 pass 调优
#### 研究问题
如何把 VLEN、LMUL、寄存器压力、RVV lowering remarks 与 runtime 共同用于 pass 选择？
#### 与原论文区别
新增 RVV 特有静态/后端证据和跨 VLEN 泛化，而非换一台机器。
#### 实验框架
```text
LLVM IR + RVV remarks + VLEN/硬件计数器 → constrained pass edits → 编译/板卡测量 → rollback/反馈
```
#### 风险
板卡计时噪声、工具链后端成熟度和 VLEN 配置差异会削弱结论。

### 11.2 证据可靠性校准
研究 remarks 与真实 speedup 的一致性，给每类证据置信度并在候选选择中校准；需区分静态指标、runtime 和诊断相关性。

### 11.3 低开销跨程序迁移
用少量目标测量加历史摘要迁移到新 benchmark，比较零样本、检索和小规模 SFT；风险是 benchmark 过拟合和数据泄漏。

## 12. 与其他已读文献的关系

C33 SeGaBench 的对象是缺失语义与源码工件，C35 的对象是编译器可见 pass pipeline，前者可作为 AutoPass 上游语义候选来源但不共享实验数字。C34 T-LLM Compiler 修改 C 循环并用 CBMC/LLM 验证，C35 不做源码语义变换，依赖 LLVM 内部证据和真实 runtime；C34 可作为 source-rewrite baseline，C35 作为 pass-level baseline。三者分别覆盖 semantic artifact、source rewrite、pipeline tuning，组合时必须重新定义接口与正确性。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLVM runtime performance tuning |
| 核心问题 | 小预算下如何选 pass、参数并跨架构适应 |
| 输入/输出 | 源码、IR、remarks、硬件信息 / 合法优化 pass pipeline |
| 核心方法 | Score/Analysis/Reasoning/Evaluation 多 agent + 运行时反馈 |
| 使用模型 | DeepSeek-V3.2 主模型；比较 GPT-4o、Qwen3、Gemini 3 Flash |
| 工具 | LLVM/Clang 17.0.6、remarks、硬件计数器、CrewAI |
| 强化学习/形式化验证 | 无 RL 训练；无形式化正确性证明，含确定性合法性验证与 rollback |
| 数据集规模 | cBench31、PolyBench30、CoreMark1、MiniFE1、LULESH1，共64 workloads |
| 主要指标 | speedup、geomean、wins/losses、pass coverage、Edit Similarity |
| 最重要结果 | 在论文所列五类 benchmark 套件、相对各自 LLVM-O3 的几何均值上，x86-64 为 1.043×、ARM64 为 1.117×；R3 在 10 个平台-套件设置中 9 个最佳 |
| 核心创新 | compiler/runtime evidence 联合闭环、受约束编辑、免训练多 agent |
| 主要局限 | 两架构、LLVM17、有限预算、测量噪声、无形式化证明 |
| 与 RISC-V 相关性 | 中：闭环可迁移，但正文无 RISC-V/RVV 评测 |
| 最适合作为 | pass 调优 baseline、编译器证据反馈模块 |

这篇论文最值得学习的是把 LLM 的推理落在 LLVM remarks、IR 和目标机 runtime 上，并用合法性检查与回滚控制风险；主要局限是实验架构和工具链有限、证据代理不等于因果证明；后续应围绕 RVV 特有证据和真实硬件泛化展开，而不是简单照搬 pass 序列。
