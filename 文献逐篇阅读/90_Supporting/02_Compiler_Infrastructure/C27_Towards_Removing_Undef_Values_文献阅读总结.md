# Towards Removing Undef Values from LLVM IR 文献阅读总结

论文题目：**Towards Removing Undef Values from LLVM IR**

作者：Pedro Lobo、John McIver、George Mitenkov、Juneyoung Lee、Kirshanthan Sundararajah、Nuno P. Lopes

发表时间：2026年

发表平台：PLDI 2026，Proceedings of the ACM on Programming Languages，第10卷，Article 172；Distinguished Paper Award
元数据核验来源：[作者主页正式发表记录](https://web.ist.utl.pt/nuno.lopes/pubs.php?id=byte-type-pldi26)；[DOI](https://doi.org/10.1145/3808250)

论文链接或编号：DOI 10.1145/3808250
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

关键词：LLVM IR、undef、poison、原始内存、byte type、freezing load、Alive2、translation validation

> 本文档用于文献阅读、组会汇报和后续研究分析。论文事实与阅读后的研究思考严格分开。

---

## 1. 研究背景

LLVM IR 是编译器前端、分析、优化和后端之间的核心表示。它必须协调不同源语言和目标架构的语义，因此使用未定义行为（Undefined Behavior，UB）表达某些语言或硬件差异。LLVM 的 deferred UB 包括 `poison`、`undef` 和通过 `freeze` 产生的固定非确定值。

`undef` 的问题在于每次观察都可能得到不同值，导致直观的代数重写不一定成立，例如 `y + y` 不一定等于 `2*y`。这增加了优化正确性推理的复杂度，并长期导致 Alive2 报告的错误优化。论文指出，LLVM 10 到 LLVM 21 期间许多 `undef` 使用已经被移除，但未初始化内存仍是最后的主要障碍。

## 2. 论文要解决的问题

### 2.1 如何表达未初始化原始内存

现有整数、指针等 IR 类型不能安全表达任意原始内存数据，尤其是复制包含指针 provenance（来源信息）的字节时，直接用整数 load/store 可能丢失 provenance。

### 2.2 如何在移除 undef 后保持 bitfield 和 memcpy 等代码正确

`undef` 非粘性曾被 bitfield lowering 利用；改为粘性的 `poison` 后，逐位处理未初始化内存需要新的语义。论文需要同时支持原始内存复制、类型重解释和 bitfield 访问。

### 2.3 如何让 LLVM 与验证工具平稳迁移

新语义必须兼容旧 IR，并在 LLVM、Clang 和 Alive2 中实现，且不能对生成代码、编译时间和验证时间产生不可接受的影响。

> 本文主要研究：如何用新的原始字节类型和 freezing load 表达未初始化内存，使 LLVM 能把其从 `undef` 迁移为 `poison`，并保持优化正确性与工程可用性。

## 3. 核心方法概述

论文提出两个 LLVM IR 扩展：任意位宽的 raw byte type `b_w`，以及在加载后逐位冻结原始数据的 freezing load。byte type 能保存原始内存的逐位信息；bytecast 可以在字节、整数和指针之间转换；freezing load 防止单个 poison bit 污染整个目标值，同时保留非 poison bits。

```text
未初始化内存/混合类型内存
        ↓
以 b_w 原始字节表示并保留逐位 poison/provenance
        ↓
bytecast 或 freezing load 转为目标类型
        ↓
LLVM/Clang lowering、优化与 Alive2 语义支持
        ↓
LLVM 单元测试、Translation Validation 和 20 个真实程序评测
```

论文还修改 Clang 的 char、bitfield、memcpy/memmove lowering，加入 bytecast 优化、SLP vectorizer 支持和 cost model 调整，并扩展 Alive2 的内部 byte 表示以支持 IR 寄存器中的字节。

## 4. 实验框架与训练流程

本文不涉及模型训练、SFT 或强化学习，主要采用 IR 语义设计、编译器实现、验证工具扩展和基准评测。

### 4.1 LLVM/Clang 实现

实现 raw byte type、bytecast、byte 级 freeze 和 freezing load；修改 Clang 将 char 变量编译为 byte type，bitfield 访问使用 freezing load，memcpy/memmove 在适合的位置以字节 load/store 表达，并加入 bytecast 的 constant folding、store forwarding、SLP vectorization 和 cost model 支持。

### 4.2 Alive2 实现

Alive2 原本已有用于内存的内部字节模型。论文将其暴露给 LLVM IR 寄存器，并调整与指针字节、type punning 和 partial-order reduction 相关的优化，避免旧假设在新 byte type 下失效。

### 4.3 迁移与验证

旧 IR 中的 undef 创建点使用 freeze poison 或零常量替换；旧 bitcode 通过 auto-upgrade 机制兼容。论文用 LLVM 测试套件、Alive2 translation validation 和真实基准的编译结果检查迁移影响。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数或机器学习损失函数。关键语义包括：

```text
byte type b_w = w 个按位表示的原始内存 bit
freeze(bit) = 非 poison bit 保持不变；poison bit 变为固定的非确定 bit
```

精确 bytecast 只有当各 bit 的底层类型与目标类型匹配时才产生目标值，否则为 poison；type-punning bytecast 在没有 poison bit 时强制转换，可能得到没有 provenance 的指针。freezing load 等价于先加载 byte、逐位 freeze，再转换为目标类型。这里的目标是让未初始化内存从 `undef` 变为可安全传播的 `poison`，不是优化奖励。

## 6. 实验设置

### 6.1 数据集来源

实验使用 20 个 Phoronix Test Suite 基准，覆盖安全、编译器、压缩、图像、音频、HPC、数据库、并行处理和 SMT 等领域，总计约 6.0 million LoC，最大程序超过 2 million LoC。超过一半基准使用 bitfield；`build-llvm`、`ngspice` 和 `z3` 特别大量使用 bitfield。论文还使用 LLVM 单元测试以及 Draco、eSpeak、FLAC、TJBench 四个基准做 translation validation。

### 6.2 模型与工具

实验服务器为 AMD EPYC 9455 48-Core、128 GB RAM、Ubuntu 24.04.3 LTS；基准使用单线程、taskset 和 nice 降低环境噪声。基线为 LLVM 21。工具包括 LLVM、Clang、Alive2 和 Phoronix Test Suite。论文中未明确说明所有编译器构建参数的完整清单。

### 6.3 对比方法

对比对象是 baseline LLVM 21，以及四类原型变体：启用 byte type；再加入 bytecast 优化；再把非精确 bytecast 折叠到 load；最后把未初始化内存改为 poison 并用 freezing load 降低 bitfield。

### 6.4 评价指标

| 指标 | 含义 |
|---|---|
| Run-time performance | 基准程序运行时间变化，正值表示更快 |
| Binary size | 生成二进制大小变化，正值表示更小 |
| Compile time | 编译耗时变化，正值表示更快 |
| Peak compilation memory | 编译峰值内存变化，正值表示更低 |
| LLVM IR instruction count | IR 指令数变化，通常越少越好 |
| Alive2/translation validation | 优化和迁移是否通过语义验证 |

## 7. 实验结果与结论

### 7.1 主要结果

仅引入 byte type 时，运行时间最差回退 6.4%，平均回退 1.6%；加入 byte type 优化后，最差回退降为 4.4%，平均回退 0.8%；允许 load type punning 后平均回退进一步降到 0.2%。将未初始化内存改为 poison 对运行时间平均为净零影响，John the Ripper 有 1.4% 回退。

### 7.2 代码尺寸与编译成本

启用优化时二进制大小平均增加 0.2%，不启用优化时为 0.7%，加入 load type punning 后为 0.1%。byte type 优化将平均编译耗时变化从 0.5% 降到 0.15%；type punning 将 IR 指令数开销降到约 0.2%，bytecast 占比从平均 2.5% 降到 1.6%，再降到 0.1%。

### 7.3 正确性结果

Alive2 之前在 LLVM 单元测试中标记为不 sound 的 11 个测试（论文称占总数 8%）通过 byte type 和相关 lowering 得到修复，且没有报告新的测试失败。四个基准进行了 translation validation；Draco 中修复了 330 个 Alive2 报告，但出现两类新报告，其中一类是已知的 store-undef 问题，另一类来自 Alive2 对逃逸栈对象的流不敏感追踪，后者是工具本身的既有局限。

### 7.4 消融与失败案例

simdjson 仍有明显回退，原因是 LazyValueInfo 目前只支持整数类型，无法充分合并某些基本块。论文把扩展该分析留给未来工作。部分代码尺寸离群值来自寄存器分配对 IR 微小变化的敏感性，以及尚未更新的 cost model。

## 8. 主要创新点

### 8.1 创新点一：raw byte type

论文为 LLVM IR 寄存器引入可表示任意位宽原始内存数据的 `b_w`，并逐位保留整数 bit、指针 provenance 和 poison。它解决了把 memcpy 等原始复制错误地降低为整数操作的问题。

### 8.2 创新点二：freezing load

freezing load 在把原始数据转换成目标类型前逐位冻结 poison，避免一个 poison bit 使整个值变成完全非确定，同时支持 bitfield lowering。

### 8.3 创新点三：贯穿 LLVM、Clang 与 Alive2 的实现

论文不仅给出语义，还实现了前端 lowering、优化和验证工具支持，并用真实大型程序评估工程代价。这使“移除 undef”成为可迁移的编译器基础设施方案，而不是孤立的语义模型。

## 9. 局限性

### 9.1 论文明确承认的局限

LazyValueInfo 尚未支持 byte type，导致 simdjson 回退；一些 bitfield 相关优化没有实现，因为在所用基准上看起来不重要。Alive2 对逃逸栈对象的流不敏感追踪会产生既有的 false positives。完整移除 undef 仍需逐步识别和更新依赖 undef 的优化。

### 9.2 阅读后发现的潜在局限

实验主要在单一 AMD EPYC 平台和 LLVM 21 基线进行，不能直接推出 RISC-V/RVV 后端上的性能结果。translation validation 只覆盖四个基准，且有界/工具模型的覆盖不能等同于所有程序和所有输入的数学证明。社区后续已经放弃独立 bytecast、改用 bitcast 形式，说明论文设计仍在演化，使用时需要区分论文原型与当前 LLVM 主线设计。

## 10. 阅读后的研究方向反思

最值得借鉴的是把 IR 语义、优化规则、前端 lowering 和验证工具作为一个整体设计；对 LLM 编译器而言，若 IR 的 poison、provenance 和原始内存语义不清晰，生成或重写再强也可能无法可靠验证。不能把 byte type 直接移植到 RISC-V 就当作创新，真正可研究的是如何让 RVV lowering、跨 ISA 迁移或 LLM 重写显式携带这些语义约束。该论文最适合作为形式验证、LLVM IR 基础设施和正确性基线。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RVV lowering 的 provenance-aware 验证

#### 研究问题

如何在 LLVM 到 RVV 的向量化和 load/store 合并中保持 byte-level provenance 与 poison 语义。

#### 与原论文的区别

原论文主要解决通用 LLVM IR 表示；新方向关注向量化、尾部处理和 RVV memory operation 的后端语义。

#### 可能的创新点

扩展 Alive2 或等价检查以表达 RVV 向量长度和逐位 poison，并自动检测不安全的 load merging。

#### 实验框架

```text
LLVM IR → RVV lowering → provenance/poison 验证 → RVV 模拟器或真机性能评测
```

#### 可行性

需要 LLVM/RVV、Alive2 或 SMT、Spike/QEMU 或 RVV 真机，以及向量化基准。

#### 主要风险

向量长度无关语义、掩码和内存别名会放大验证状态空间；形式验证通过不保证真实微架构性能。

### 11.2 LLM 辅助的语义约束优化规则生成

#### 研究问题

如何让 LLM 生成 LLVM 优化规则时显式遵守 poison、freeze、pointer provenance 等约束。

#### 与原论文的区别

原论文由人工设计并实现 IR 语义；新方向使用 LLM 探索规则，但每条规则必须经过 Alive2 translation validation。

#### 可能的创新点

把语义类型和反例反馈编码进提示或结构化动作空间，区分有限测试通过与形式等价证明。

#### 实验框架

```text
LLVM IR rewrite 候选 → 语义约束过滤 → Alive2 证明/反例 → 规则保留或修复 → LLVM 回归测试
```

#### 可行性

可复用 Alive2、LLVM InstCombine 和现有 LLM 编译优化数据集。

#### 主要风险

证明失败原因可能难以自然语言化；模型可能过拟合已有 LLVM 规则或生成不可维护的重写。

## 12. 与其他已读文献的关系

本批次中 C28 通过 CBMC 对生成的 TACO 程序做有界等价验证，C27 则从更底层的 LLVM IR 语义和 Alive2 支持出发。C27 可作为 C28 这类可验证代码生成系统的 IR 语义基础，但两者的验证对象分别是 C/TACO 程序和 LLVM IR 变换。C27 适合作为正确性基础设施与形式验证基线，C28 适合作为多硬件/DSL 综合和搜索基线。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 从 LLVM IR 中移除 undef 的最后主要用途 |
| 核心问题 | 未初始化内存、bitfield、memcpy 与 poison/指针 provenance 的语义冲突 |
| 输入 | LLVM IR、Clang 源程序和内存操作 |
| 输出 | byte type、freezing load 及对应 LLVM/Clang/Alive2 实现 |
| 核心方法 | 原始字节逐位语义 + freezing load + 工具链迁移 |
| 使用的模型 | 不使用模型 |
| 使用的编译器工具 | LLVM 21、Clang、Alive2、Phoronix Test Suite |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 是，Alive2 与 translation validation |
| 数据集规模 | 20 个基准，约 6.0 million LoC；另有 LLVM 测试套件 |
| 主要指标 | 运行时间、二进制大小、编译时间、峰值内存、IR 指令数、验证结果 |
| 最重要实验结果 | 优化后平均运行时间回退 0.8%，允许 type punning 后 0.2%；切换 poison 平均净零；修复 11 个既有 Alive2 不 sound 测试 |
| 核心创新 | 用 byte type 和 freezing load 表达原始内存并支撑移除 undef |
| 主要局限 | LazyValueInfo、部分 Alive2 精度和社区设计仍需演化 |
| 与 RISC-V 研究的相关性 | 中高：直接涉及 LLVM IR/后端正确性，但未评估 RVV |
| 最适合作为 | LLVM IR 语义、形式验证和编译器基础设施基线 |

这篇论文最值得学习的是把语义设计、前端 lowering、优化器和验证器协同推进；最主要的局限是部分分析和工具仍不完整且原型设计在演化；如果用于后续研究，合理方式是把它作为 LLVM/RVV 正确性底座，而不是把有限的 x86 实验直接当作 RISC-V 性能结论。
