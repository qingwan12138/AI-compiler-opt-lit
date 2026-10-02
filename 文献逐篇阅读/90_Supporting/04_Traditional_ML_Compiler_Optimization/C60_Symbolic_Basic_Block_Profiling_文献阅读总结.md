# Symbolic Basic Block Profiling 文献阅读总结

论文题目：**Symbolic Basic Block Profiling for Machine Learning Kernels**

作者：Jingyu Qiu, Rongcui Dong, Sreepathi Pai

发表时间：2026

发表平台：OOPSLA 2026；论文正文版本为 arXiv:2608.20605，正式 DOI 10.1145/3839483
元数据核验来源：[arXiv:2608.20605](https://arxiv.org/abs/2608.20605)；[ACM OOPSLA 论文记录](https://doi.org/10.1145/3839483)

论文链接或编号：[arXiv:2608.20605](https://arxiv.org/abs/2608.20605)
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

关键词：symbolic profiling、basic block、LLVM、TVM、ML kernel、auto-tuning

> 本文档用于文献阅读。论文事实与阅读后的分析分开描述。

## 1. 研究背景

基本块剖析通常通过 LLVM PGO 插入计数器并运行程序获得执行次数。大型机器学习 kernel 的输入尺寸可能很大，动态插桩既有运行开销又需要实际执行。论文观察到，许多 ML kernel 具有规则循环和可分析的控制流，因此可以把基本块执行次数表示成输入形状的符号公式。

## 2. 论文要解决的问题

### 2.1 静态获得精确计数

在一类结构化 ML kernel 上，不运行插桩程序而推导每个基本块的执行次数。

### 2.2 将剖析用于代价模型

验证符号 profile 是否能作为 LLVM IR 指令特征，帮助 TVM 自动调优减少经验评估依赖。

> 本文主要研究：如何从 LLVM IR 静态提取适用于 ML kernel 的符号基本块计数，并比较其与动态 PGO 的精确性和成本。

## 3. 核心方法概述

系统把 LLVM IR 控制流图转换为带循环、分支和路径计数函数的图表示。利用 LoopInfo、Scalar Evolution、phi 节点和受限的仿射条件，递归构造每个基本块的符号表达式，最后输出 SMT-LIB 或编译后的求值程序。

```text
TVM kernel → LLVM IR/CFG
        ↓
循环边界、phi、分支条件分析
        ↓
构造 trueRatio 与 loopCount 符号函数
        ↓
每个基本块的符号执行次数
        ↓
代入输入形状求值 / 构建 XGBoost 代价模型
```

本文没有使用 LLM。GPT、SFT、强化学习和奖励函数均不涉及；自动调优实验使用 XGBoost 代价模型，不是语言模型。

## 4. 实验框架与训练流程

本文不涉及模型训练，主要采用静态分析和推理时求值。

1. 从 ONNX Zoo 的 50 个模型中抽取 78 类 ML operator；在 TVM 0.21.0 中优化、lower 到 LLVM IR。
2. 对 LLVM IR 使用符号分析，生成每个基本块的执行计数公式。
3. 用和 PGO 相同的输入尺寸（64 到 8192）代入公式，并与动态 instrumentation 结果比较。
4. 以 LLVM PGO 的 Knuth-Stevenson 计数器放置作为动态 baseline，比较分析和求值时间。
5. 在 TVM MetaSchedule 的 conv2d 任务上，以符号 profile 形成 LLVM IR instruction map，并训练/使用 XGBoost 代价模型。

## 5. 奖励函数、损失函数或关键公式

本文不涉及强化学习奖励函数。核心公式包括循环内分支比例和范围长度：

```text
trueRatio(AC) = (Σ len(I) - Σ len(overlap)) / len(R0)
len(range(s,e,D)) = (e-s+1) - |D ∩ [s,e]|
```

`R0` 是原始迭代范围，`I` 是满足各合取条件的范围，`D` 是被排除的值。公式用于计算分支为真的迭代比例，再递归组合循环次数与控制流。论文没有把它定义为训练损失。

## 6. 实验设置

### 6.1 数据集来源

50 个 ONNX Zoo 模型产生 78 类 operator/kernel，主要采用 TVM 内置 operator；LSTM、RoiAlign 等缺少内置实现的 operator 按 ONNX 1.22 定义手工构造。评测输入尺寸为方形矩阵宽度 64、128、256、512、1024、2048、4096、8192，每个实验运行 3 次取平均。

### 6.2 模型与工具

工具实现为 LLVM 分析 pass，使用 LLVM PGO、LLVM Scalar Evolution、Z3/SMT-LIB、TVM 0.21.0、TVM MetaSchedule 和 XGBoost。机器为 Intel i7-12700H、32 GB RAM。论文未明确给出完整 LLVM 版本。

### 6.3 对比方法

主要 baseline 是 LLVM PGO 动态插桩。自动调优部分比较随机模型、TVM 默认 XGBoost 模型和使用 LLVM IR instruction map 的 xgb-symbolic-only 模型。

### 6.4 评价指标

| 指标 | 含义 |
|---|---|
| exact count match | 符号计数与 PGO 具体计数一致 |
| analysis time | 编译/静态分析时间 |
| execution/evaluation time | 给定输入后产生 profile 的时间 |
| speedup | 相对 PGO 动态执行时间的加速，不是 kernel 运行加速 |
| best runtime | 自动调优找到的 kernel 实测运行时间 |

## 7. 实验结果与结论

### 7.1 适用性与精确性

78 个 kernel 中，73 个能为所有基本块提取正确符号表达式。pad、prelu 部分支持；topk 的数据相关排序、cumsum 的 runtime wrapper 和 ONNX graph-level loop 超出当前范围。作者将符号表达式代入 64–8192 的输入并与 PGO 结果匹配。

### 7.2 时间结果

小于 20 个基本块的 kernel 上，符号分析有时比 PGO 编译更快；大型 kernel 的分析时间更高，当前实现对基本块和计数函数近似二次敏感。求值时间对输入尺寸近似常数，而 PGO 动态执行时间随输入变大；大输入的 matmul/conv 上，compiled symbolic 版本超过 10^9 倍 speedup，论文结论给出的最大值为 23,480,043,367 倍。该数字是 profile 获取时间比，不是生成 kernel 的运行加速。

### 7.3 自动调优

在 512×512、64 channel 的 conv2d 上进行 300 trials、30 个随机种子。符号模型优于随机模型，但总体上 TVM 默认 XGBoost 达到最好的 kernel runtime；符号模型方差更大，LLVM IR lowering 和重复符号分析造成明显额外开销。它证明 profile 可提供预测信号，但没有取代默认代价模型。

## 8. 主要创新点

### 8.1 面向 ML kernel 的符号 profile

论文把基本块计数从一次运行的具体值提升为随输入形状变化的符号表达式，覆盖了包含 phi、早退和仿射分支的 LLVM IR kernel。

### 8.2 LLVM 分析 pass 实现

该方法不是只在源级示例上推导，而是从 LLVM IR 复用 LoopInfo/SCEV 等分析，并提供 SMT-LIB 与可编译求值路径。

### 8.3 将静态 profile 接入自动调优

论文展示了从符号 profile 生成 LLVM IR 指令地图特征并进入 XGBoost/TVM MetaSchedule 的路径，同时诚实报告其当前额外 lowering 成本和较大方差。

## 9. 局限性

### 论文明确承认的局限

方法限于规则 ML kernel；data-dependent 控制流、非仿射条件、运行时 wrapper 和复杂图级 loop 不能完整处理。自动调优中每个候选都重复 lower 到 LLVM IR，阻碍了 profile 复用；作者将跨 tile 版本的轻量增量分析留给未来工作。

### 阅读后发现的潜在局限

对 allocator 等过程返回值，实验假设调用成功并赋常数 1，可能不代表异常路径。PGO 作为 ground truth 的比较验证了计数一致性，但不构成对任意 LLVM 程序的形式化正确性证明。自动调优案例仅为一个 conv2d 配置，不能推断所有硬件和算子都受益。

## 10. 阅读后的研究方向反思

最值得借鉴的是把 profile 设计成可复用的输入参数函数，而不是固定输入的计数。它属于传统编译器基础设施/代价建模，可作为 TVM、MLIR 或 RISC-V kernel 调优模块。仅把 LLVM 目标换成 RISC-V 不足以形成创新；需要研究 RVV 向量长度、tile 参数与符号控制流的联合代价。论文的核心贡献是静态 profile，不应误写成 LLM 或强化学习方法。

## 11. 可进一步尝试的研究方向

### 11.1 RVV 向量长度参数化 profile

#### 研究问题
构造同时依赖输入形状和 `VLEN`/向量化参数的基本块、指令和访存计数。

#### 与原论文的区别
原论文主要预测控制流次数；新方向还建模 RVV lowering 和尾处理成本。

#### 可能的创新点
跨 VLEN 的统一符号代价模型与真实板卡校准。

#### 实验框架
```text
MLIR/TVM kernel → LLVM RVV IR → 符号 profile → RVV 模拟器/硬件校准
```

#### 可行性与主要风险
可复用 LLVM 分析与 TVM kernel；风险是微架构吞吐、缓存和动态 VL 使静态计数不足。

### 11.2 增量 tile profile

#### 研究问题
不同 tile size 的 kernel 是否能共享一个带 tile 参数的公式集合。

#### 与原论文的区别
避免每个自动调优候选重新 lower 和全量分析。

#### 可能的创新点
基于 IR 结构差异的公式复用、失效检测和增量更新。

#### 实验框架
```text
候选 tile 参数 → IR 差分 → 复用/更新符号公式 → XGBoost/MetaSchedule
```

#### 可行性与主要风险
与论文明确提出的未来方向一致；风险是变换改变 CFG 后公式复用不安全。

## 12. 与其他已读文献的关系

与 C50 的 profile propagation 都关注优化管线中的 profile，但 C50 研究 CFG 变换后的 profile 重建，本文研究 ML kernel 的静态基本块计数。与 C48 的稀疏 ML 自动调度可组合：本文提供低成本静态特征，C48 仍需处理稀疏格式和循环排序。与 C15/RIFS 的传统代价模型同属 SUPPORTING/B4，但任务分别是 profile 生成与运行时不变量函数特化。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | ML kernel 的静态符号基本块剖析 |
| 核心问题 | 降低 PGO 动态计数的运行成本 |
| 输入/输出 | LLVM IR / 每个基本块的符号次数 |
| 核心方法 | CFG、循环、phi、仿射条件符号分析 |
| 使用的模型 | 无 LLM；XGBoost 仅用于代价模型案例 |
| 编译器工具 | LLVM、PGO、SCEV、TVM、Z3 |
| 强化学习/形式验证 | 否；不是形式化证明 |
| 数据集规模 | 50 个 ONNX 模型、78 类 kernel |
| 主要指标 | 计数一致性、分析/求值时间、调优 runtime |
| 最重要结果 | 73/78 kernel 完整支持，求值对大输入显著快于 PGO |
| 核心创新 | 输入参数化的静态 profile 表达式 |
| 主要局限 | 仅适用于结构化 ML kernel，lowering 成本高 |
| 与 RISC-V 相关性 | 中高：可扩展到 RVV 代价模型，但本文未做 RISC-V 实验 |
| 最适合作为 | 静态代价特征、ML 编译器基础设施模块 |

这篇论文最值得学习的是可跨输入尺寸复用的 profile 表达式；最主要的局限是适用域和自动调优接入成本。如果用于后续研究，合理方式是结合 RVV 参数化和增量分析，而不是简单替换目标后端。
