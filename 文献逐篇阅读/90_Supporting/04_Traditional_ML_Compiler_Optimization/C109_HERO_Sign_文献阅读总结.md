# HERO-Sign 文献阅读总结

论文题目：**HERO-Sign: Hierarchical Tuning and Efficient Compiler-Time GPU Optimizations for SPHINCS+ Signature Generation**

作者：Yaoyun Zhou，Qian Wang

发表时间：2026（arXiv v1 于 2025-12-30 发布；论文 PDF 标注为 HPCA 2026 accepted paper）

发表平台：HPCA 2026（arXiv 正文标注 accepted；未核实正式 DOI）

论文链接或编号：arXiv:2512.23969
元数据核验来源：[arXiv 论文记录](https://arxiv.org/abs/2512.23969)
代码/数据/工件：未找到可确认的作者代码仓库或工件

关键词：SPHINCS+、GPU 编译期优化、CUDA、PTX、自动调优、Tree Fusion、CUDA Graph、后量子密码

> 本文档只依据作者公开 PDF 正文整理。论文事实、阅读后的分析和可进一步尝试的方向分开描述。该论文属于面向异构 GPU 的传统编译/运行时协同优化，不使用大语言模型。

---

## 1. 研究背景

SPHINCS+ 是无状态哈希签名方案，安全性依赖哈希函数，能够抵抗量子计算威胁，但签名生成包含大量 SHA-2 哈希和 Merkle 树计算，因此吞吐率较低。论文引言指出，SPHINCS+-128f 的签名较大；在 Arm Cortex-A72 上，SPHINCS+-256f 的签名生成约比 FALCON-512 慢 18 倍、比 Dilithium-2 慢 259 倍（第 1 节）。

已有优化主要分为 FPGA 定制加速器、CPU SIMD/密码指令集优化和 GPU 并行化。已有 GPU 工作已经利用 Hypertree 的部分并行性，但仍存在三类问题：

1. FORS 包含大量分支，单个 CUDA block 的线程和共享内存约束造成资源利用不足。
2. FORS、TREE 和 WOTS+ 三类 kernel 的计算强度、寄存器压力和编译器优化效果不同，统一使用 native 或 PTX 路径并不总是合理。
3. 多 CUDA stream 虽能重叠任务，但 kernel 之间仍有空闲时间，并且过多 stream 会带来调度和启动开销。

论文因此研究一种分层的 GPU 优化框架：算法级并行化、共享内存与寄存器布局、编译期指令路径选择，以及批处理任务图共同参与优化。

## 2. 论文要解决的问题

### 2.1 FORS 并行资源利用不足

FORS 由许多独立的 Merkle 子树组成。已有实现主要在单个子树或有限范围内并行，难以同时利用多个子树的独立性；共享内存容量和每个 block 的最大线程数又限制了直接展开全部叶节点（第 II-B、III-B 节）。

### 2.2 不同 kernel 的编译优化效果不一致

SHA-2 的 PTX 指令级改写可能减少寄存器压力，但 PTX 也可能限制编译器的进一步优化。因此，对于 FORS_Sign、TREE_Sign、WOTS+_Sign，最佳选择可能不同，不能假设一个全局路径适用于所有 kernel（第 III-C 节）。

### 2.3 批量签名的 kernel 间空闲和启动延迟

FORS_Sign 与 TREE_Sign 可以较早并发执行，而 WOTS+_Sign 依赖二者生成的根节点。仅靠 host 发起多个 stream 不能充分表达这种依赖关系，也不能完全消除 kernel-level idle time（第 II-B、III-F 节）。

> 本文主要研究：如何在不同 NVIDIA GPU 资源约束下，通过自动 Tree Tuning、编译期 PTX/native 选择和 CUDA Graph 任务图，提升 SPHINCS+ 多参数集签名生成吞吐率。

## 3. 核心方法概述

HERO-Sign 将 SPHINCS+ 签名生成拆分为 TREE_Sign、FORS_Sign 和 WOTS+_Sign 三个 CUDA kernel，并对三者采用不同并行和编译策略。核心设计包括：

- 对 Hypertree/MSS 做 Multiple Merkle Trees Parallelization（MMTP）。
- 对 FORS 做 Tree Fusion；离线 Tree Tuning 根据 GPU 的线程和共享内存限制生成候选配置，再用 profiling 选择配置。
- 对 SPHINCS+-256f 使用 Relax-FORS，以寄存器中的 Relax Buffer 减少底层叶节点的共享内存需求。
- 使用 PTX `prmt`、`mad` 等指令改写 SHA-2 的部分操作，并按 kernel 在编译期选择 PTX 或 native 路径。
- 使用 constant/global/shared memory 的混合布局，降低共享内存压力。
- 插入 padding bank，减少 16、24、32 字节线程访问在 reduction 中的 shared-memory bank conflict。
- 用 CUDA Graph 表达 FORS、TREE、WOTS+ 的依赖，并按批大小组织多个消息。

整体数据流：

```text
SPHINCS+ 消息
    ↓
消息摘要、FORS indices、Hypertree leaf_idx
    ↓
FORS_Sign 与 TREE_Sign 并行生成根节点/认证路径
    ↓
WOTS+_Sign 使用上游根节点完成签名
    ↓
编译期已选定的 PTX/native kernel
    ↓
CUDA Graph 调度多个消息批次
    ↓
签名吞吐率、启动延迟和 Nsight profiling 指标
```

论文中的“自动”主要是面向 GPU 配置和编译路径的搜索/选择，不是 LLM 智能体或机器学习模型。

## 4. 实验框架与训练流程

本文不涉及模型训练、预训练、SFT、PPO、GRPO 或强化学习，主要采用静态 CUDA 系统实现、离线配置搜索、编译和运行时 profiling。

### 4.1 离线 Tree Tuning

算法 1 输入 FORS 参数 `k`、`log2(t)`、`n` 以及目标 GPU 的可用共享内存。它枚举每棵 FORS 子树分配的线程数、融合的 Set 数量和共享内存占用，过滤超出线程/共享内存上限的配置，再优先选择同步点较少且线程/内存利用率较高的候选（第 III-B 节）。最终候选还要通过实测 profiling 选择最佳配置。

### 4.2 编译期分支选择

HERO-Sign 为 SHA-2 同时提供 PTX 和 native 实现。CUDA 模板参数 `UseOptimizedPath` 通过 `if constexpr` 在编译时决定路径，每个 kernel 的最终二进制只保留一个固定路径；论文表 V 给出了不同参数集下三个 kernel 的 PTX/native 选择。

### 4.3 运行时执行

实验先完成 kernel 配置和编译，再按消息批次捕获 CUDA Graph。FORS_Sign、TREE_Sign 作为无相互依赖的节点提前并发；WOTS+_Sign 在需要的根节点产生后执行。实验使用 Nsight Systems 和 Nsight Compute 分析启动延迟、occupancy、计算吞吐和内存吞吐。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有神经网络损失函数。关键目标是配置搜索和 GPU 资源/性能约束。

### 5.1 GPU occupancy 公式

论文式（1）给出占用率近似：

```text
Occupancy = (1 / Wmax) × floor(Rtotal / (Rthread × Tblock)) × (Tblock / 32)
```

其中 `Rtotal` 是每个 SM 的总寄存器数，`Rthread` 是每线程寄存器数，`Tblock` 是 block 线程数，`Wmax` 是每个 SM 支持的最大 warp 数。寄存器数增大通常会降低可驻留 warp 数。

### 5.2 Tree Tuning 的配置目标

Tree Tuning 不是单一标量奖励，而是按以下优先级筛选候选：满足线程和共享内存约束、避免同时将线程和共享内存都推到极限、减少同步点、提高线程与共享内存利用率。算法 1 的候选排序大致为：先最小化同步点，再最大化 `UT` 和 `US`。

### 5.3 共享内存 padding 公式

对于 16/32 字节访问，论文式（2）要求 128 字节事务区域与每线程访问的 bank 数、线程间隔满足：

```text
128 = Bn × 4 × Th
```

对于 24 字节访问，论文式（3）扩展为：

```text
128 × R = Bn × 4 × Th
```

论文明确说明 24 字节情形涉及跨 128 字节边界的访问，并基于硬件合并行为提出扩展策略；这不是形式化保证。

## 6. 实验设置

### 6.1 数据集来源

本文没有训练/验证/测试数据集。输入是 SPHINCS+ 消息和固定的 128f、192f、256f 参数集；不同消息长度实验使用 1K、2K、3K、4K 输入。论文说明实际签名流程先对输入消息哈希，再由摘要导出索引，因此树结构和签名操作数量基本固定。

### 6.2 模型与工具

| 项目 | 论文设置 |
| --- | --- |
| 编程/编译 | CUDA C++、NVCC；具体 NVCC 版本论文中未明确说明 |
| 指令路径 | native CUDA 路径、PTX inline assembly |
| GPU | GTX 1070、GTX 2080 Ti、V100、A100、RTX 4090、H100 |
| 主要详细平台 | RTX 4090；block size 主要使用 1024 |
| profiling | NVIDIA Nsight Systems、Nsight Compute |
| 图调度 | CUDA Graph、非阻塞 CUDA stream |
| CPU 对照 | Intel Xeon 配置见论文表 VII；具体实验代码/完整环境版本未完全说明 |

论文给出了 GPU 架构、SM 版本、基础频率和 CPU 型号，但没有给出所有软件版本、完整编译命令或可复现实验脚本。

### 6.3 对比方法

- TCAS-SPHINCSp：论文选定的公开 SOTA GPU 实现，也是主要 baseline。
- HERO-Sign without CUDA Graph 与 HERO-Sign with CUDA Graph：用于隔离任务图的影响。
- FPGA 实现、SPHINCSLET ASIC 实现和 AVX2 单线程/16 线程实现：用于跨平台参考，不是完全相同硬件或哈希实现条件下的严格同构 baseline。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
| --- | --- | --- |
| KOPS | 每秒完成的千次签名操作 | 越大越好 |
| Kernel launch latency | kernel 启动延迟，单位微秒 | 越小越好 |
| Warp occupancy | 活跃 warp 与硬件最大 warp 的比例 | 通常越高越好，但需结合资源压力 |
| Compute throughput | Nsight 统计的计算吞吐 | 需结合 workload 解读 |
| Memory throughput | Nsight 统计的内存吞吐 | 不一定越高越好，减少无效 off-chip 流量也可能降低该值 |
| PPS | 每次签名的功耗，单位 Watt/签名 | 越小越好 |

## 7. 实验结果与结论

### 7.1 主要结果

在 RTX 4090 上，相对于主要 GPU baseline，HERO-Sign 在 SPHINCS+-128f、192f、256f 上报告的总体吞吐提升范围分别为 1.28×–3.13×、1.28×–2.92×、1.24×–2.60×（摘要和图 13）。在图 12 的 block size=1024 配置下，加入 CUDA Graph 后的最终吞吐为 119.47、65.43、33.88 KOPS。

表 VIII 的 kernel 级结果如下：

| 参数集/Kernel | Baseline KOPS | HERO-Sign KOPS | 加速比 |
| --- | ---: | ---: | ---: |
| 128f / FORS_Sign | 442.9 | 946.3 | 2.14× |
| 128f / TREE_Sign | 125.2 | 157.7 | 1.26× |
| 128f / WOTS+_Sign | 2493.1 | 4915.7 | 1.97× |
| 192f / FORS_Sign | 128.9 | 222.0 | 1.72× |
| 192f / TREE_Sign | 88.2 | 93.6 | 1.06× |
| 192f / WOTS+_Sign | 1457.6 | 2464.9 | 1.69× |
| 256f / FORS_Sign | 66.6 | 116.4 | 1.75× |
| 256f / TREE_Sign | 36.4 | 44.9 | 1.23× |
| 256f / WOTS+_Sign | 776.8 | 1570.9 | 2.02× |

### 7.2 与传统实现的比较

表 IX 报告，RTX 4090 上 HERO-Sign 的 128f/192f/256f 吞吐分别为 119.47/65.43/33.88 KOPS。相对于表 X 的 AVX2 16 线程实现，论文计算出 144.29×、116.84×、95.17× 的提升；但这是 GPU 与 CPU 平台比较，不能解释为同一硬件上的编译器加速。

与 FPGA/ASIC 参考实现相比，论文还报告更高吞吐和更低 PPS。不过这些实现使用的硬件、哈希算法和优化目标并不完全一致，结论应视为跨平台工程参考。

### 7.3 与其他 GPU 方法的比较

主要对照是 TCAS-SPHINCSp。论文在 RTX 4090、A100、H100 和 GTX 2080 Ti 等架构上扩展 Tree Tuning；不同架构的收益不同。H100 的 256f 加速比最高达到 1.88×，但 RTX 4090 的绝对吞吐仍更高，论文将其部分归因于 RTX 4090 更高的基础频率。

### 7.4 消融实验

FORS_Sign 的逐步实验显示：

- 128f：MMTP 后吞吐从 442.9 提升到 702.7 KOPS；完整组合达到 946.3 KOPS，累计 2.14×。
- 192f：从 128.9 提升到 222.0 KOPS，累计 1.72×。
- 256f：Relax-FORS 相关阶段使吞吐明显提高，最终从 66.6 达到 116.4 KOPS，累计 1.75×。

表 VI 显示 padding 后 FORS/TREE 的 load/store bank conflict 在列出的配置中降至接近 0。表 V 显示 PTX 选择是 kernel 和参数集相关的：128f/192f 主要在 FORS_Sign 使用 PTX，256f 三类 kernel 均使用 PTX。

### 7.5 CUDA Graph、批大小与编译开销

CUDA Graph 将 kernel launch latency 最多降低 221.3×；论文明确排除了 Graph instantiation time。block size 从 2 到 1024 的敏感性实验表明，小 block size 区间也能获得较高 speedup，而接近资源上限时收益下降。表 XI 给出的平均编译时间为：128f baseline 18.68 秒、HERO-Sign 14.61 秒；192f 为 23.25/21.72 秒；256f 为 24.19/19.18 秒。论文将缩小编译器优化空间带来的收益归因于编译时间下降，但没有提供更完整的编译器内部因果分析。

## 8. 主要创新点

### 8.1 创新点一：面向 FORS 的自动 Tree Fusion

既有方法难以同时展开大量 FORS 子树。HERO-Sign 用 Set 和 Fusion 组织多个子树，并用目标 GPU 的线程/共享内存限制自动产生候选，再用 profiling 选择配置。价值在于把算法结构中的独立性转化为硬件资源可行的 block 级并行；实验中 FORS_Sign 的提升支持该设计有效。

### 8.2 创新点二：按 kernel 进行编译期 PTX/native 选择

论文不是简单地“使用 PTX”，而是观察到 PTX 可能减少寄存器压力、也可能限制 native 编译器优化，因此为每类 kernel 静态实例化单一路径。这比运行时分支减少了多路径同时存在造成的寄存器和指令缓存压力。表 V 和表 VIII 支持了该选择具有参数集依赖性。

### 8.3 创新点三：适配 16/24/32 字节访问的 bank padding

论文将 shared-memory reduction 的 padding 从常见对齐情况扩展到 SPHINCS+ 三个安全级别的访问宽度，尤其讨论 24 字节访问跨 128 字节事务边界的情形。其结果来自 GPU profiling 和硬件行为假设，不是形式化硬件证明。

### 8.4 创新点四：面向签名依赖图的 CUDA Graph 批处理

论文利用 FORS/TREE 与 WOTS+ 的依赖关系构造 DAG，把多个消息批次封装为 CUDA Graph，减少 host 侧 launch 和 kernel 间空闲。图 12 的启动延迟结果证明该工程机制对整体吞吐有独立贡献。

## 9. 局限性

### 9.1 论文明确或正文可直接确认的边界

- 实验聚焦 SPHINCS+ 的 `-f` 参数集，即 128f、192f、256f；论文没有报告 `-s` 参数集的完整结果。
- 实现基于 CUDA、PTX 和 NVIDIA GPU；跨到 AMD GPU、Intel GPU、CPU 或 RISC-V GPU 后端需要重新实现和验证。
- 主 baseline 是 TCAS-SPHINCSp；论文没有与完整现代 CUDA 编译器自动调优系统做广泛比较。
- PTX 分支选择依赖目标 GPU 的实际 profiling，不能仅靠编译器规则普遍预测。
- 24 字节 bank padding 的推导依赖作者对硬件事务合并的假设，正文没有给出硬件级形式化证明。
- 论文没有提供 LLM、强化学习、形式化验证或语义等价检查结果。

### 9.2 阅读后发现的潜在局限

- Tree Tuning 的候选搜索空间和最终 profiling 成本没有系统量化；不同 GPU/驱动/NVCC 组合的离线成本可能较高。
- 论文将 throughput、occupancy 和 memory throughput 结合分析，但没有给出完整能耗测量方法、置信区间或多次运行统计。
- GPU 与 FPGA/ASIC/AVX2 的跨平台比较具有参考价值，但硬件、频率、哈希实现和优化目标差异使其不等同于严格公平 benchmark。
- 代码级优化以 CUDA kernel 为中心，未讨论如何把这些资源反馈上升为可迁移的 LLVM/MLIR 中间表示或后端代价模型。

## 10. 阅读后的研究方向反思

本文对“大语言模型与编译器优化、LLVM IR、RISC-V、多架构优化”的直接相关性为中等偏低：它不是 LLM 论文，也不使用 LLVM/MLIR 或 RISC-V；但它清楚展示了真实硬件资源反馈如何约束代码生成和 kernel 调度。

值得借鉴的是“结构独立性 → 资源约束配置 → 编译期路径选择 → 真实硬件反馈”的闭环。不能直接照搬的是 NVIDIA-specific PTX 指令、CUDA Graph API 和 shared-memory bank 规则；把 CUDA 换成 RISC-V 并不能自动构成新贡献。

从 Taxonomy v2 角色角度看，本文更适合作为 SUPPORTING / B5 Hardware/ISA Background 或 B2 Compiler Infrastructure 的硬件协同优化参考；若作为新论文候选，正式分类、Paper_ID 和路径必须由维护流程另行决定，本 staging 笔记不做正式登记。

## 11. 可进一步尝试的研究方向

### 11.1 硬件反馈驱动的跨后端配置迁移

#### 研究问题

Tree Fusion、寄存器预算、共享内存/片上 SRAM 预算等配置能否从 NVIDIA GPU 迁移到 AMD GPU、RISC-V 向量核或 NPU 后端？

#### 与原论文的区别

不只是重新实现 CUDA kernel，而是学习/抽取跨后端可复用的资源约束表示。

#### 可能的创新点

设计统一的“并行树任务—内存层次—同步点”中间表示，并用硬件实测反馈校准配置代价模型。

#### 实验框架

```text
树任务图与资源特征
    ↓
后端无关配置候选
    ↓
CUDA/ROCm/RVV/NPU lowering
    ↓
真实硬件 profiling
    ↓
跨后端迁移误差与吞吐评估
```

#### 可行性

需要至少两种 GPU 后端和一个 RISC-V/RVV 或 NPU 环境；现有论文的参数集和 kernel 分解可作为起点。

#### 主要风险

不同后端的同步和内存一致性语义不同，可能无法使用同一套 bank-conflict 模型；迁移收益也可能低于重新搜索。

### 11.2 将 Tree Tuning 表达为 MLIR/LLVM 后端调优问题

#### 研究问题

能否把树融合、block 配置和编译期 PTX/native 选择表示为 MLIR Transform 或 LLVM 后端 pass 的可搜索配置？

#### 与原论文的区别

原论文在 CUDA 实现层搜索配置；新方向把配置决策和 lowering 过程显式连接到可审计的 IR 变换。

#### 可能的创新点

定义跨层代价特征，包括寄存器压力、共享内存、同步深度、occupancy 和实测吞吐，并研究静态估计与硬件反馈的结合。

#### 实验框架

```text
MLIR kernel/任务图
    ↓
Transform dialect 配置候选
    ↓
LLVM/CUDA/RVV lowering
    ↓
编译器报告 + 真实硬件测量
    ↓
配置排序与误差分析
```

#### 可行性

可先从 GPU-independent 的树 reduction 和任务图开始，再接入具体后端；不需要 LLM 才能验证核心问题。

#### 主要风险

IR 中的静态资源估计可能无法预测真实 launch/occupancy 行为，且后端版本变化会破坏稳定性。

### 11.3 验证约束下的编译期指令选择

#### 研究问题

如何保证 PTX/native 或 RVV 指令路径在优化后仍保持哈希函数和签名算法的语义与安全要求？

#### 与原论文的区别

原论文主要通过性能 profiling 选择路径，没有形式化翻译验证；新方向加入 bit-accurate 语义检查、差分测试或 SMT 辅助验证。

#### 可能的创新点

把“性能候选”和“正确性证据”绑定，禁止只凭 KOPS 选择可能改变位级行为的指令序列。

#### 实验框架

```text
候选 IR/PTX/RVV 变换
    ↓
位级语义/差分验证
    ↓
编译与真实硬件测量
    ↓
只在通过验证的候选中选择最快路径
```

#### 可行性

可从 SHA-2 round 这类局部、位级明确的函数开始，再扩展到完整 kernel。

#### 主要风险

GPU 内存模型、内联汇编和未定义行为可能超出验证器覆盖范围；有限测试不能被表述为形式化证明。

## 12. 与其他已读文献的关系

本 slot-1 本轮只完整阅读 HERO-Sign 一篇论文，因此不存在同批次第二篇可作事实级横向比较的文献。与正式 corpus 中的 LLVM/MLIR、RISC-V/RVV 和传统编译器优化论文只能作主题层定位，不能在本笔记中补写未经本轮正文复核的方法或结果。

在研究角色上，HERO-Sign 更接近硬件感知的 kernel/compiler co-design 和后端优化参考，不是 pass 顺序选择器、LLM 源码翻译器、形式化验证器或 fuzzing 工具。它可作为后端资源反馈、GPU 任务图和跨架构性能评测的 baseline/reference；若与 LLM 编译器工作组合，LLM 更适合作为候选配置提议者，编译器和真实硬件 profiling 仍应负责生成与筛选证据。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | 加速 SPHINCS+ 多参数集的 GPU 签名生成 |
| 核心问题 | FORS 资源利用不足、kernel 间优化差异、批处理启动/空闲开销 |
| 输入 | SPHINCS+ 消息、128f/192f/256f 参数集、GPU 资源属性 |
| 输出 | 优化后的 CUDA kernel、PTX/native 编译路径和 CUDA Graph |
| 核心方法 | MMTP、Tree Fusion、Relax-FORS、编译期分支、bank padding、CUDA Graph |
| 使用的模型 | 无机器学习模型；无 LLM |
| 使用的编译器工具 | CUDA/NVCC、PTX inline assembly、Nsight Systems/Compute |
| 是否使用强化学习 | 否；Tree Tuning 是离线搜索与 profiling，不是 RL |
| 是否使用形式化验证 | 否；bank-conflict 结论来自分析和 profiling |
| 数据集规模 | 无数据集；消息长度敏感性为 1K/2K/3K/4K |
| 主要指标 | KOPS、kernel launch latency、occupancy、compute/memory throughput、PPS |
| 最重要实验结果 | RTX 4090 上最终 128f/192f/256f 吞吐为 119.47/65.43/33.88 KOPS；启动延迟最多降低 221.3× |
| 核心创新 | 把树结构并行性、编译期指令选择与 GPU 资源反馈统一到分层优化流程 |
| 主要局限 | NVIDIA/CUDA 依赖；缺少 LLVM/MLIR、RISC-V、形式化验证和完整复现实验细节 |
| 与 RISC-V 研究的相关性 | 中低：可借鉴资源反馈和任务图思想，但 PTX/CUDA 设计不能直接迁移 |
| 最适合作为 | 异构后端优化参考、硬件反馈 baseline、GPU kernel/compiler 协同案例 |

这篇论文最值得学习的是：它把算法中的独立性、编译器的静态路径选择和真实 GPU 资源反馈连成了可测量的优化闭环；最主要的局限是强依赖 NVIDIA CUDA/PTX，且没有语义验证或跨 ISA 抽象。如果用于后续研究，合理用法是把它作为硬件感知配置与后端协同的参考基线，再研究可迁移的 IR/代价模型/验证机制，而不是简单把 GPU 换成 RISC-V 就声称形成新方法。

