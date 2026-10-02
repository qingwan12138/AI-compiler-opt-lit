# GRANII 文献阅读总结

论文题目：**GRANII: Selection and Ordering of Primitives in GRAph Neural Networks using Input Inspection**

作者：Damitha Lenadora、Vimarsh Sathia、Gerasimos Gerogiannis、Serif Yesil、Josep Torrellas、Charith Mendis

发表时间：2026

发表平台：CGO 2026 主会（IEEE/ACM International Symposium on Code Generation and Optimization）
元数据核验来源：[CGO 2026 官方论文页](https://2026.cgo.org/details/cgo-2026-papers/5/GRANII-Selection-and-Ordering-of-Primitives-in-GRAph-Neural-Networks-using-Input-Ins)；[作者 PDF](https://charithmendis.com/assets/pdf/26-cgo-granii.pdf)

论文链接或编号：[CGO 2026 官方论文页](https://2026.cgo.org/details/cgo-2026-papers/5/GRANII-Selection-and-Ordering-of-Primitives-in-GRAph-Neural-Networks-using-Input-Ins)；[作者公开 PDF](https://charithmendis.com/assets/pdf/26-cgo-granii.pdf)；DOI：论文中未明确说明。
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

关键词：图神经网络（GNN）、primitive 选择、primitive 排序、矩阵重结合、输入敏感优化、代价模型、XGBoost、稀疏-稠密计算

> 本文档用于文献阅读、组会汇报和后续研究分析。事实依据为本地 14 页 PDF 正文；论文事实与阅读后的研究思考分开描述。

---

## 1. 研究背景

图神经网络（Graph Neural Network，GNN）通常同时包含稀疏矩阵操作和稠密矩阵操作。典型 GNN 层先聚合邻居节点状态，再用稠密矩阵运算更新节点表示；图聚合常由广义稀疏矩阵-稠密矩阵乘法（g-SpMM）或采样稠密-稠密矩阵乘法（g-SDDMM）表达，更新常由 GEMM 表达（第 2 节）。

现有框架和编译器通常把 GNN 表示为若干阶段，并为不同输入采用固定的 primitive 组合及顺序。这样做实现简单，但忽略了输入图的稀疏性、非零元素分布、节点/边规模、嵌入维度、硬件和底层系统等因素。论文第 1 节以 GCN 为例说明，同一数学计算通过不同矩阵重结合可以产生不同的 SDDMM、SpMM、GEMM 组合；输入图和模型配置变化后，最佳组合也可能变化。

论文还指出，已有的循环排序或相邻 kernel 选择方法通常在较小局部范围内做决定，不能同时观察完整 GNN 的稀疏-稠密交互与领域特定的重结合机会。因而，研究目标不是训练一个语言模型，而是让 GNN 编译器在已知输入后自动选择更合适的 primitive 组合与顺序，并控制运行时决策开销。

## 2. 论文要解决的问题

### 2.1 如何暴露等价但不同的 primitive 组合

消息传递式 GNN API 会把计算降低为直线式代码，容易丢失矩阵结合律、矩阵属性和可重结合结构。GRANII 需要一种保留这些信息的矩阵中间表示（matrix-based IR），以便系统枚举合法的矩阵重结合。

### 2.2 如何根据输入选择性能更好的组合

单独观察嵌入维度、图稀疏度或硬件都不足以稳定预测最佳组合。论文因此用输入图特征和输入/输出嵌入维度训练轻量级机器学习代价模型，预测各 primitive 的执行时间，再累加为 association tree 的估计成本。

### 2.3 如何把运行时决策开销控制在收益以内

如果运行时逐一实际执行所有候选组合，输入敏感优化本身可能过慢。GRANII 将流程分为离线编译和在线选择：离线枚举并用输入无关规则删除明显不优候选；在线只检查剩余候选，并优先使用嵌入维度规则，必要时才调用代价模型。

> 本文主要研究：如何在 GNN 的稀疏-稠密矩阵计算中，利用矩阵重结合、离线剪枝和输入感知代价模型，在低运行时开销下自动选择 primitive 组合与顺序。

## 3. 核心方法概述

GRANII 是一个面向 GNN 的编译器与运行时系统。它先把 WiseGraph 或 DGL 的 Python GNN 代码解析到矩阵 IR；再把合法重结合转换成 association forest，每棵树代表一种 primitive 组合；然后进行输入无关剪枝、生成条件代码，并在运行时用输入图特征、嵌入维度和轻量级 XGBoost 代价模型选择成本最小的候选。

```text
GNN 框架 Python API（WiseGraph / DGL）
        ↓ AST 规则解析与矩阵属性收集
矩阵 IR（矩阵叶节点 + 操作树 + 稀疏/稠密属性）
        ↓ 枚举合法矩阵重结合
Association forest（每棵树对应一种 primitive 组合）
        ↓ 输入无关规则剪枝与公共子表达式复用
Promoted association trees
        ↓ 生成条件代码与底层 kernel 调用
运行时提取图特征 + 读取嵌入维度
        ↓ 维度条件或 XGBoost primitive cost models
选择估计成本最低的组合
        ↓
执行 GNN 前向计算；训练时反向计算仍由底层系统负责
```

矩阵 IR 的叶节点保存矩阵尺寸、dense/sparse 等属性以及 data/weighted 等子属性；非线性操作如 ReLU、SoftMax 被作为阻止重结合的 barrier，以保持论文所关注的语义等价重结合。对于可结合的乘法，系统可将 row broadcast 进一步改写为矩阵乘法形式，从而暴露更多候选。

论文最终系统输出的是现有 GNN 框架可执行的 primitive 组合和条件选择逻辑，不是源代码级大型语言模型生成结果。因此按 Taxonomy v2 的角色规则属于 SUPPORTING，而不是 SELECTOR/TRANSLATOR/GENERATOR 中的 LM 角色。

## 4. 实验框架与训练流程

本文不涉及大语言模型训练、SFT、PPO、GRPO 或其他强化学习训练；也没有把模型训练成编译器语言模型。论文包含的是代价模型训练和编译器运行时执行流程。

### 4.1 离线编译阶段

1. 使用 Python AST 规则解析器把 GNN 框架代码降到矩阵操作，并收集矩阵尺寸、稀疏性和权重等属性。
2. 生成矩阵 IR；对可结合操作建立 association tree。
3. 用深度优先递归枚举所有合法 association trees，并把对应操作标注为 SDDMM、SpMM、GEMM、row broadcast 等 primitive。
4. 扫描 association forest，复用公共计算，并根据 primitive 子集关系、输入矩阵更大且 primitive 集合相同等规则删除明显不优或重复候选。
5. 对保留候选生成条件代码；可直接由嵌入维度判断的候选不使用更重的代价模型，其他候选嵌入 cost-model 条件。

### 4.2 代价模型训练阶段

论文为每类稠密或稀疏 primitive 以及每种目标硬件训练 XGBoost 回归模型。训练数据由 CPU、NVIDIA A100 和 NVIDIA H100 机器上的 primitive profiling 得到；输入图来自 SuiteSparse，使用不同图、采样结果以及 32–2048 的输入/输出嵌入维度，论文报告每个 primitive 约收集 700–8000 个数据点。验证图不包含最终评价图。

### 4.3 在线运行时阶段

运行时先提取输入图稀疏度等手工特征，并与 GNN 输入/输出嵌入维度拼接。系统先应用仅依赖嵌入维度的条件；对于剩余候选，分别预测 primitive 成本并求和，选择估计总成本最低的 association tree。生成的 kernel 调用只在选定候选中执行，输入图特征提取和选择开销在一次运行时决策后摊销到多次 GNN 迭代。

### 4.4 训练与推理执行

论文评估 inference 和 training，但 GRANII 只优化 forward pass 的 primitive 选择；training 的 backward pass 不由 GRANII 进行同类 operator selection。因此 training speedup 通常低于 inference speedup。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习，因此不存在强化学习奖励函数。论文的核心优化目标是比较候选 association tree 的预测执行成本。

### 5.1 GCN 的两类重结合

论文第 3.1 节给出两种 GCN 组合。动态归一化组合使用 row broadcast、聚合和更新；预计算组合先用 SDDMM 生成归一化邻接矩阵，再进行聚合和 GEMM。前者在较密图上可能更有利，后者在较稀图上可能更有利。

### 5.2 GAT 的复用与重计算

GAT 的两类组合分别复用注意力计算中已经得到的更新嵌入，或重新计算一次 GEMM 后再聚合。若输入嵌入维度小于输出嵌入维度，额外 GEMM 可能换来更便宜的聚合；最佳选择取决于配置和实际输入。

### 5.3 Association tree 的预测成本

```text
Cost(tree) = Σ Cost(primitive_i | graph features, input embedding, output embedding)
```

每个 `Cost(primitive_i | ...)` 由对应 primitive 和目标硬件的 XGBoost 回归模型预测。论文没有把该式作为强化学习损失，也没有报告统一的训练损失公式；成本之和只是用于运行时组合比较。若多棵树成本相同，系统任选一棵等价候选。

## 6. 实验设置

### 6.1 数据集来源

代价模型训练图来自 SuiteSparse Matrix Collection。论文选择非零值数量约为 1 百万到 1 亿的无向图，并通过采样扩展输入变化；训练时改变输入和输出嵌入维度 32–2048。验证子集不含最终评价图。

最终评价使用 6 个图，来源分别为 DGL、SuiteSparse（SS）和 Open Graph Benchmark（OGB）：Reddit、com-Amazon、mycielskian17、belgium osm、coAuthorsCiteseer、ogbn-products。评价图均为无向、无权图，非零分布具有差异，且与代价模型训练图不重叠。论文没有报告传统意义上的训练/验证/测试样本行数；只报告每个 primitive 的 profiling 点数范围。

### 6.2 模型与工具

- GNN 模型：GCN、GAT、GIN、TAGCN、SGC。
- 系统与接口：WiseGraph、DGL 2.4（论文注明为 2024 发布版本）、PyTorch Python API。
- 代价模型：按 primitive 和目标硬件分别训练的 XGBoost 回归模型。
- 机器：Intel Xeon Gold 6348 CPU（1 TB RAM、无 GPU）；NVIDIA A100 + Intel Xeon Platinum 8358；NVIDIA H100 + AMD EPYC 9454。
- 编译实现：Python；前端用 Python AST 解析，后端把 association tree 映射到 GNN 框架支持的 kernel 调用。

论文没有使用 LLVM、MLIR、Alive2 或形式化验证器；也没有使用 LLM。代价模型是机器学习回归器，而不是语言模型。

### 6.3 对比方法

主要对比对象是 WiseGraph 的默认稀疏-稠密 primitive 组合和 DGL 的 PyTorch 后端实现。部分 baseline 已基于模型配置进行 operator reorder；没有现成实现的模型，作者构造了配置感知的较强 baseline。第 VI-G 节进一步对比五种启发式 oracle：只看模型配置（Config.）、只看硬件（HW）、只看输入图（Graph）、只看底层系统（Sys.）以及理想最优配置（Optimal）。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
|---|---|---|
| Speedup | baseline 执行时间 / GRANII 执行时间；论文表 III 以 100 次迭代的几何平均报告 | 越大越好 |
| Geomean speedup | 跨图、配置、硬件和模型的几何平均加速比 | 越大越好 |
| Runtime overhead | 输入特征提取与组合选择的额外时间 | 越小越好 |
| Cost-model decision accuracy | GRANII 选择与 Optimal 或启发式 oracle 的一致性/加速表现 | 越高越好 |
| Execution time | H100 等平台上 forward 或 end-to-end 的毫秒数 | 越小越好 |

## 7. 实验结果与结论

### 7.1 主要结果

在 WiseGraph、DGL、H100、A100 和 CPU 的多组配置上，表 III 报告 GRANII 相对默认组合的总体几何平均 speedup：inference 为 1.56×，training 为 1.40×。其中 training 的收益较低，因为 GRANII 只选择 forward pass，backward pass 未被同样优化。整体结果覆盖五种 GNN 模型与多种输入图、嵌入维度。

在表 III 的分模型结果中，WiseGraph/A100/GCN 的 inference 几何平均 speedup 为 10.39×，但 WiseGraph/A100/GAT 为 1×；DGL/H100/GAT 为 1.74×，DGL/A100/GAT 为 1.80×。这些数字是跨评价图和配置的几何平均，不是单个程序的保证值。

### 7.2 与传统方法的比较

GRANII 的优势来自输入图和配置联合决策。第 VI-C 节显示，默认系统有时偏向密图或稀图的一类实现；GRANII 能通过归一化重结合或 GEMM 位置调整避开不适合当前输入的实现。论文特别观察到 DGL 的 GIN 和 SGC 中，自动改变 update GEMM 的位置可得到明显收益。

表 VI 给出 GRANII 与不同启发式 oracle 的 speedup。对 GCN，Optimal 为 1.98×、GRANII 为 1.88×、Config. 为 1.88×、HW 为 1.64×、Graph 为 0.94×、Sys. 为 1.71×；对 GAT，Optimal 为 1.46×、GRANII 为 1.45×、Config. 为 1.22×、HW 为 1.34×、Graph 为 1.37×、Sys. 为 1.39×。这支持了论文关于“同时考虑多个因素”的结论。

### 7.3 与其他 GNN 系统的比较

端到端表 IV 在 H100 上测试 Reddit 和 ogbn-products，使用 GCN/GAT、单隐藏层以及 32、256、1024 隐藏维度。示例包括：Reddit GCN、输入/输出维度 32 时，WiseGraph 为 48.3 ms，GRANII 为 9.4 ms，即 5.14×；ogbn-products GAT、维度 1024 时，WiseGraph 发生非法内存访问，而 GRANII 为 215.4 ms；DGL 在相同设置下为 547.3 ms。表 IV 的结果是特定端到端配置，不应与表 III 的几何平均混用。

### 7.4 消融、敏感性与多层实验

- 采样：在 mycielskian17 上对 GCN/GAT 使用 10 个随机邻居样本，采样大小包括 1000、100、10，并额外检查 5000 到 5 的五个采样大小。相同采样大小的运行时间波动较小；除“两个组合收益非常接近”的情况外，GRANII 很少做出错误选择。
- 多层：在 Reddit、隐藏维度 32 的实验中，2、3、4、8 层相对 WiseGraph 的 speedup 分别为 5.14×、5.19×、5.22×、5.26×。论文说明可逐层选择并复用决策，但评估模型的图稀疏度在层间基本不变。
- 剪枝与模型数量：GCN、GAT、GIN 通过重结合得到的组合数分别为 12、2、8；输入无关剪枝后分别保留 2、0、4 对候选（论文将其称为 composition pairs）。

### 7.5 开销、失败案例与结论

输入图特征提取和 composition selection 的额外开销在 GPU 上最多 7 ms，在 CPU 上为 0.42 s；论文将其折算为最多约 4.4 个 GPU 单次 GNN iteration、约 1.1 个 CPU iteration，且每次运行时只发生一次。由于代价模型预测并非完全准确，当两个组合实际成本非常接近时，GRANII 可能选错并出现 slowdown；第 VI-C 节明确展示了该失败原因。

总体上，论文证明了将矩阵重结合保留下来、把输入图与模型配置同时纳入代价模型，能够在多个 GNN 系统和硬件上改进默认 primitive 组合，但结果依赖训练硬件、输入分布和代价模型精度。

## 8. 主要创新点

### 8.1 输入敏感的稀疏-稠密 primitive 重结合

现有 GNN 框架通常固定 primitive 组合。GRANII 把 GNN 的矩阵结合结构显式保留，并为同一语义计算生成多个稀疏-稠密组合；论文的案例研究覆盖 GCN、GIN、SGC、TAGCN 和 GAT。表 III、VI 的跨配置结果支持该设计的有效性。

### 8.2 保留结合信息的矩阵 IR 与 association forest

矩阵 IR 不只保存操作拓扑，还保存尺寸、dense/sparse、data/weighted 等属性，并把可结合操作置于同一层级。随后以递归算法枚举 association trees，使编译器能从 API 代码自动生成候选，而不要求人工列出所有组合。

### 8.3 离线剪枝与在线轻量 cost model 的协同

GRANII 没有把所有候选都留到运行时测量，而是先用输入无关规则删除明显不优候选，再用嵌入维度条件和按硬件训练的 XGBoost primitive cost models 做最后选择。该设计同时针对搜索空间和决策开销，属于论文的核心系统机制。

### 8.4 跨系统、硬件与采样场景的实证验证

论文不只在单一 GNN 框架或单一 GPU 上报告结果，而是覆盖 WiseGraph、DGL、CPU、A100、H100、五种 GNN 模型以及采样和多层场景。该验证证明的是当前实验范围内的可迁移性，不等同于对所有硬件或所有 GNN 的保证。

## 9. 局限性

### 9.1 论文明确承认或直接呈现的局限

- GRANII 只对 forward pass 做 primitive 选择，training 的 backward pass 没有获得同等优化。
- 当前多层说明依赖评价模型中图稀疏度跨层基本不变；论文也指出存在跨层图稀疏度变化的 GNN 类别。
- 代价模型可能在两个候选成本相近时做出错误决策，造成 slowdown。
- 代价模型按目标硬件训练；论文没有证明训练于 CPU/A100/H100 的模型能直接迁移到未见硬件。
- 为了可扩展性，作者采用手工输入特征，明确没有采用自动特征提取的稀疏卷积网络。
- 论文的语义等价依赖规则和非线性 barrier；没有使用 Alive2、SMT 或其他独立形式化验证工具。

### 9.2 阅读后的潜在局限

- 训练图与测试图不重叠降低了直接泄漏风险，但训练图来源仍集中于 SuiteSparse，真实生产图分布偏移可能使 XGBoost 预测变差。
- 代价采用 primitive 成本求和，可能无法完全表达 kernel 融合、缓存、同步、内存分配和跨 primitive 交互；论文正文没有给出对该加和近似的普适误差界。
- association tree 的组合数量可能随表达式规模增长，当前规则和剪枝效果不能直接外推到更深、更复杂或含更多非结合操作的 GNN。
- 结果主要是 CPU 与 NVIDIA GPU 平台；对 RISC-V、AMD GPU、专用 NPU 等平台的适用性在当前 PDF 中未验证。

## 10. 阅读后的研究方向反思

GRANII 对“大语言模型与编译器优化、LLVM IR、RISC-V、多架构优化”的直接启发有限，因为论文没有 LM，也没有 LLVM IR 或 RISC-V 实验。它更适合作为传统编译器自动调优、输入敏感代价模型和 GNN 编译框架的 baseline/tool module。

值得借鉴的是“最终执行动作由编译器产生、代价模型只负责选择”的边界：如果后续使用 LLM，LLM 可以作为候选组合提议器或搜索策略模块，但不能把论文的 XGBoost cost model 和矩阵 IR 机制误称为 LLM 方法。论文的核心贡献已是输入敏感的重结合枚举、剪枝与代价选择，简单替换成 RISC-V 后端不足以构成同等创新；需要证明 RISC-V 特有的向量长度、缓存层级、稀疏访存或 RVV 指令选择会改变 primitive 组合空间和 cost model。

对现有语料主线而言，GRANII 与 pass/phase ordering 文献在“选择执行顺序”层面相似，但它排序的是 GNN 中跨算子矩阵 primitive 及重结合，不是 LLVM pass 序列。它与张量编译器自动调度文献的共同点是输入/硬件感知成本预测，差别是 GRANII 的候选由代数重结合和 primitive 规则生成，而不是通用 schedule 搜索空间。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RVV 的输入敏感稀疏-稠密 primitive 选择

#### 研究问题

在 RISC-V Vector（RVV）上，图稀疏模式、VL/VLEN、向量加载代价和嵌入维度如何共同影响 SpMM、SDDMM、GEMM 及重结合顺序？

#### 与原论文的区别

不是把 H100/CPU 换成 RISC-V，而是把 RVV 的向量长度和稀疏访存机制纳入候选生成与 cost model，并验证现有重结合是否仍然完整。

#### 可能的创新点

加入 RVV-specific primitive、VL-aware 特征和跨缓存层级成本；比较静态启发式与输入感知模型在真实 RVV 芯片上的误差。

#### 实验框架

```text
GNN API → RVV-aware matrix IR → 合法重结合/primitive 枚举
       → RVV kernel profiling → 硬件感知 cost model
       → 输入图 + VL/VLEN 条件选择 → RVV 执行
```

#### 可行性

需要一个可用 RVV 编译器/模拟器、SpMM/SDDMM/GEMM kernel、SuiteSparse 和 GNN benchmark。

#### 主要风险

真实硬件噪声、稀疏 kernel 向量化效率和模拟器与硬件不一致都可能掩盖 cost model 的收益。

### 11.2 代价模型不确定性感知的 primitive 选择

#### 研究问题

当两个候选成本接近且模型预测误差较大时，如何避免 GRANII 的错误选择和 slowdown？

#### 与原论文的区别

原论文使用单点 XGBoost 回归并按预测成本最小选择；新方向增加预测区间、校准和少量在线测量。

#### 可能的创新点

用置信区间或 conformal prediction 识别“难区分”候选，只对高不确定样本触发一次实际测量；研究测量预算与端到端收益的关系。

#### 实验框架

```text
输入特征 → 多候选成本预测 + 不确定性
        → 低不确定性直接选择 / 高不确定性少量测量
        → 更新局部模型 → 执行最佳候选
```

#### 可行性

可复用 GRANII 的五种 GNN、六类图和 profiling 数据，并加入硬件迁移测试。

#### 主要风险

在线测量可能抵消收益；不确定性校准在图分布外样本上可能失效。

### 11.3 LLM 辅助但受代价模型约束的重结合搜索

#### 研究问题

LLM 能否从 GNN 代码和矩阵 IR 中提出新的合法重结合或剪枝规则，同时由编译器规则和 cost model 保证可执行性与性能评估？

#### 与原论文的区别

原论文完全采用规则枚举和 XGBoost 选择；新方向把 LLM 放在候选生成/规则归纳位置，而不是让 LLM 直接生成未经验证的 kernel。

#### 可能的创新点

设计 IR 约束解码、语义等价检查和成本反馈闭环，研究 LLM 生成候选相对于穷举 association forest 的覆盖率与搜索成本。

#### 实验框架

```text
GNN → 矩阵 IR → LLM 提议合法重结合
    → 规则/类型/等价约束过滤 → cost model 排序
    → 少量实际 profiling → 选择可执行候选
```

#### 可行性

需要结构化 IR 序列化、约束检查器、现有 GNN kernels、代价模型和可控 LLM 推理接口。

#### 主要风险

LLM 提议可能重复已有候选或违反代数约束；推理成本和可复现性可能超过传统枚举收益。

## 12. 与其他已读文献的关系

当前批次只完整阅读 GRANII 一篇，因此不存在同批论文之间的实验横向比较。就研究问题定位而言，GRANII 与本仓库已收录的传统 compiler phase ordering、tensor autotuning 和 cost-model 文献属于相邻方向，但它的具体动作空间是 GNN 的矩阵 primitive 组合和代数重结合，不是 LLVM pass 序列；本笔记没有重新阅读那些论文，因此不把它们的具体结果写入本文事实结论。

在方法角色上，GRANII 最适合作为输入敏感 primitive selection 的 baseline 和可复用编译器组件；它可与 LLVM/RVV 后端、稀疏张量调度或 LLM 候选生成模块组合，但组合后的效果不能归因于 GRANII 原论文已经实现的能力。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | GNN 中输入敏感的稀疏-稠密 primitive 组合与顺序选择 |
| 核心问题 | 固定 primitive 组合无法适应图、配置、硬件和系统变化 |
| 输入 | WiseGraph/DGL Python GNN 代码、输入图、嵌入维度 |
| 输出 | 矩阵 IR、association trees、条件化 primitive kernel 执行路径 |
| 核心方法 | 矩阵重结合枚举 + 输入无关剪枝 + XGBoost primitive cost model |
| 使用的模型 | XGBoost 回归模型；无 LLM |
| 使用的编译器工具 | GRANII、Python AST、WiseGraph、DGL 2.4、PyTorch |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；用规则和非线性 barrier 约束语义等价重结合 |
| 数据集规模 | 训练每个 primitive 约 700–8000 个 profiling 点；评价 6 个图、5 个 GNN 模型 |
| 主要指标 | 几何平均 speedup、端到端执行时间、运行时开销、代价选择表现 |
| 最重要实验结果 | 跨设置 inference 1.56×、training 1.40× 几何平均 speedup；H100 Reddit GCN 某配置 5.14× |
| 核心创新 | 把 GNN 的矩阵重结合和输入图特征纳入 primitive 选择，并把离线剪枝与在线轻量代价模型结合 |
| 主要局限 | 仅优化 forward；依赖按硬件训练的代价模型；近成本候选可能选错；未覆盖 RISC-V/LLVM/多种新硬件 |
| 与 RISC-V 研究的相关性 | 中：可借鉴输入敏感 cost model 和组合选择，但论文没有 RVV 实验，直接换平台不足以构成创新 |
| 最适合作为 | 传统编译器自动调优 baseline、输入敏感 cost-model 方法参考、GNN 编译器工具模块 |

> 这篇论文最值得学习的是把“候选生成”和“候选选择”拆开：先用矩阵 IR 与规则完整暴露等价组合，再用输入感知代价模型做低开销选择；最主要的局限是 cost model 对硬件和输入分布敏感，且只优化 forward。用于后续研究时，合理方式是把它作为输入敏感 primitive selection baseline 或 cost-model 模块，而不是简单把硬件替换为 RISC-V 或把 XGBoost 误称为 LLM 方法。
