# VERT 文献阅读总结

论文题目：**VERT: Polyglot Verified Equivalent Rust Transpilation with Large Language Models**
作者：Aidan Z.H. Yang，Yoshiki Takashima，Brandon Paulsen，Josiah Dodds，Daniel Kroening
发表时间：2024；arXiv v2，2404.18852v2（2024-05-25）
发表平台：ASE 2025 Research Papers, pp.1453–1463
论文链接或编号：DOI 10.1109/ASE63991.2025.00123；arXiv:2404.18852
元数据核验来源：[ASE 正式论文页](https://conf.researchr.org/details/ase-2025/ase-2025-papers/57/VERT-Polyglot-Verified-Equivalent-Rust-Transpilation-with-Large-Language-Models)；[IEEE DOI](https://doi.org/10.1109/ASE63991.2025.00123)；[arXiv](https://arxiv.org/abs/2404.18852)
代码/数据/工件：未找到可确认的作者代码仓库或工件
PDF：本 staging 目录 `VERT/paper.pdf`，26 页，`%PDF-` 签名有效，正文可抽取。

关键词：Rust 转译、LLM、WebAssembly、语义等价、Kani、Verus、少样本修复

## 1. 研究背景

论文关注把现有语言程序迁移到安全、可读的 Rust。规则式 compile-then-lift 方法能够依托编译器语义，但输出往往是难读的低层 Rust；LLM 输出更接近人工 Rust，却没有等价性保证。论文因此把“可读性”和“可验证正确性”作为同时目标。正文指出，Rust 的所有权、借用和内存安全规则也使直接生成 Rust 变得困难。

## 2. 论文要解决的问题

### 2.1 可读转译与正确性保证的冲突

如何利用 LLM 生成可维护 Rust，同时避免仅凭编译通过或少量测试就宣称语义等价。

### 2.2 多语言和复杂指针程序的可扩展转译

如何借助 WebAssembly 编译器覆盖多个源语言，并在含指针、循环和多函数程序上进行验证。

> 本文主要研究：如何用“可信 oracle Rust + LLM 可读候选 + 逐级等价检查”生成可读且经过验证的 Rust 转译。

## 3. 核心方法概述

VERT 使用源语言编译器生成 WebAssembly，再用 rWasm 得到语义可信但难读的 oracle Rust；并行地让 LLM 生成可读 Rust。候选先经 rustc 和规则式语法修复，再和 oracle 放入等价检查 harness；失败反馈作为下一轮少样本提示，直到成功或达到尝试上限。

```text
源程序
  ↓
源语言编译器 → WebAssembly → rWasm → oracle Rust
  ↓                                  ↘
LLM 生成可读 Rust → rustc/错误修复 → PBT → Kani bounded → Kani full
                                      ↓失败反馈/反例
                                  重新提示 LLM
                                      ↓
                              通过验证的 Rust 输出
```

LLM 直接输出候选 Rust，故按 taxonomy v2 属于 `TRANSLATOR/T3`，不是 Selector。Kani/Verus 执行验证，LLM 不承担证明责任。

## 4. 实验框架与训练流程

本文不采用在线强化学习，主要是推理时生成、编译、验证和错误引导重试。

### 4.1 Oracle 路径

源程序编译到 Wasm，再由 rWasm 生成 Rust。该程序作为等价性参考，不作为最终可读输出。

### 4.2 LLM 路径

LLM 生成 Rust；rustc 错误会被整理进下一轮提示。instruction-tuned Claude-2 使用失败反例进行少样本提示，最多重试 20 次。

### 4.3 验证路径

先做 Bolero property-based testing（PBT），再做 Kani 有界模型检查，最后尝试带 unwind checks 的 full verification。Kani 超时或失败时，部分案例改用 Verus 手写循环不变量进行验证。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数或训练损失作为系统控制信号。核心判定是两个 Rust 程序在 harness 中对相同输入产生相同输出；具体验证工具为 PBT、Kani 和 Verus。

验证层级：

| 层级 | 含义 | 论文限制 |
|---|---|---|
| PBT | 随机生成输入寻找反例 | 120 秒内未找到反例，不等于穷尽 |
| Kani bounded | 固定 unwind bound 内检查 | 不能覆盖超出 bound 的执行 |
| Kani full | 开启 unwind checks 的完整尝试 | 循环和复杂程序常超时 |
| Verus | 用规格和循环不变量证明 | 自动化较低，需要人工规格 |

## 6. 实验设置

### 6.1 数据集来源

主集合来自 TransCoder-IR 基准，过滤后为 569 个 C++、520 个 C、341 个 Go 程序；论文前文摘要曾概括为 1,394 个程序。另收集 14 个含指针的 C 程序，用于检验安全 Rust 能力。作者按模型训练截止时间筛选并讨论数据泄漏风险。

### 6.2 模型与工具

模型包括 TransCoder-IR、CodeLlama-2 13B、微调 StarCoder 15.5B、Anthropic Claude-2（论文约记 130B）。工具包括源语言 Wasm 编译器、rWasm、rustc、Bolero、Kani、Verus、Clippy。实验环境为 Ubuntu 22、32 个 Intel Xeon 2.30 GHz CPU、240GB RAM、4 张 Tesla V100 GPU。

### 6.3 对比方法

主要对比是 TransCoder-IR 单次生成；另比较 CodeLlama-2、微调 StarCoder 和 Claude-2 在 single-shot、few-shot repair、few-shot repair + counterexamples 三种模式下的结果；14 个指针程序还与 CROWN、rWasm 的代码规模和 Clippy 警告比较。

### 6.4 评价指标

编译通过数、PBT 通过数、Kani bounded/full verification 通过数；每个验证指标均有 120 秒限制。辅助指标包括代码行数、Clippy 警告和单组件运行时间。

## 7. 实验结果与结论

### 7.1 主要结果

表 1 中，Claude-2 + VERT 的 counterexample 模式在 569 个 C++ 程序中编译 539、PBT 295、bounded 233、full 19；520 个 C 程序中编译 339、PBT 209、bounded 193、full 15；341 个 Go 程序中编译 317、PBT 195、bounded 159、full 9。这里的“通过”是程序数，不是百分比，也不是所有程序都完成 full verification。

### 7.2 与基线比较

TransCoder-IR 在上述三种源语言上均没有 full verification 通过项。论文报告 Claude-2 + VERT 相比基线在 bounded verification 上分别提高 40%、37%、47%（C++、C、Go），并给出 Wilcoxon 检验 p 值约 1×10^-5 到 4×10^-5。

### 7.3 可读性与安全 Rust

14 个指针程序中，7 个通过 PBT，2 个通过 bounded verification；程序更大、指针更多或多函数时更容易失败。VERT 输出的 Clippy 平均无警告；CROWN 输出约 10.9 条/函数、rWasm 约 372 条/函数。CROWN 输出约为 VERT 的 5 倍行数，rWasm 约为 10 倍。

### 7.4 消融和成本

Claude-2 的 few-shot + counterexamples 相比 single-shot 的 PBT 通过数在 C++、C、Go 上分别增加约 43%、46%、43%。平均一次成功编译约需 15 次尝试；表 2 给出的平均验证时间为 PBT 25 秒、bounded 52 秒、full 67 秒，验证占主要成本。

### 7.5 其他验证器

对 Kani 超时的 5 个案例，作者用 Verus 手工规格成功验证 3 个；其中一个解析函数规格为 92 行，说明可扩展验证依赖较高人工成本。

## 8. 主要创新点

### 8.1 双路径 oracle + LLM 可读候选

创新不在单独使用 LLM 或 rWasm，而在于用低层可信 Rust 作为等价参照，使 LLM 可以输出更自然的 Rust，同时把正确性检查外置给验证器。

### 8.2 逐级验证和反例驱动少样本修复

PBT、bounded、full verification 形成由低到高的检查层级；失败输入或 Kani 反例加入后续提示。实验表明反例驱动对 bounded/full verification 的增益最大。

### 8.3 多验证器可插拔

Kani 适合自动化但对循环不友好，Verus 能借助不变量处理部分循环；这种验证器分工是工程上可迁移的设计。

## 9. 局限性

### 9.1 论文明确承认或实验显示的局限

- Kani full verification 对循环和大程序容易超时。
- 含更多指针、多函数、较长代码时，安全 Rust 生成明显变难。
- Verus 成功案例依赖人工写规格，不再是完全自动流程。
- “PBT 通过”只表示 120 秒内未找到反例；bounded 也只覆盖设定 unwind 范围。

### 9.2 阅读后的潜在局限

oracle 的等价性和 Wasm/rWasm 工具链仍是信任边界；不同源语言的未定义行为、I/O、系统调用和浮点语义不一定能由统一 harness 表达。论文实验主要是函数/程序基准，不能直接推断大型真实仓库迁移效果。

## 10. 阅读后的研究方向反思

最值得借鉴的是把 LLM 生成与可检查 oracle 分离，以及让验证反例成为下一轮上下文。它适合作为 TRANSLATOR 正确性基线/验证模块，而不是编译优化 Selector。简单替换为 RISC-V 后端不足以形成创新；需要研究 Wasm/RISC-V/LLVM IR 之间的等价 oracle、ISA 特有未定义行为或跨架构验证差异。

## 11. 可进一步尝试的研究方向

### 11.1 RISC-V 目标的 oracle 分层验证

#### 研究问题

如何区分源语言语义、LLVM IR 语义和 RISC-V 汇编/执行语义，避免把后端差异误判为转译错误。

#### 与原论文的区别

增加 RISC-V 多层 oracle 和 ISA 语义检查，不只是换目标语言。

#### 可能的创新点

基于 Alive2/ISA 模型/运行时测试的分层反例归因。

#### 实验框架

```text
C/C++ → LLVM IR → RISC-V codegen → oracle/执行
  ↘ LLM Rust/IR 候选 → 分层等价检查 → 反例归因
```

#### 可行性与主要风险

可复用 LLVM、QEMU/Spike 和现有验证器；风险是浮点、系统调用和未定义行为难以统一建模。

### 11.2 反例类型驱动的验证器调度

研究根据 rustc、PBT、Kani 的失败类型选择下一验证器或提示模板；区别于论文固定层级，风险是调度策略本身引入不可控偏差。

## 12. 与其他已读文献的关系

本批次中 VERT 是通用多语言转 Rust + oracle 验证；Syzygy 面向更大 C 仓库，采用动态规格和内部代码-测试翻译；LANTERN 和 RTT APR 则把翻译作为 APR 的中间表示/修复通道。VERT 最适合作为“形式/半形式等价验证”基线，Syzygy 适合作为“动态分析 + 测试反馈”基线。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 可读、可验证的多语言到 Rust 转译 |
| 核心问题 | LLM 可读性与等价保证冲突 |
| 输入/输出 | 源程序 / 可读 Rust |
| 核心方法 | Wasm-rWasm oracle + LLM + 分层验证 + 反例修复 |
| 使用的模型 | TransCoder-IR、CodeLlama-2、StarCoder、Claude-2 |
| 编译器工具 | Wasm 编译器、rWasm、rustc、Bolero、Kani、Verus |
| 强化学习 | 否 |
| 形式化验证 | Kani、Verus；另有 PBT |
| 数据集规模 | 569 C++、520 C、341 Go，另 14 个指针 C 程序 |
| 主要指标 | 编译、PBT、bounded/full verification |
| 最重要结果 | Claude-2 + VERT 在 C++/C/Go 分别有 233/193/159 个 bounded 通过程序 |
| 核心创新 | oracle 与可读 LLM 候选的验证闭环 |
| 主要局限 | 循环、指针、多函数、验证成本和覆盖边界 |
| 与 RISC-V 相关性 | 中：验证架构可迁移，但论文未研究 RISC-V |
| 最适合作为 | TRANSLATOR/T3 的验证基线和工具模块 |

这篇论文最值得学习的是把“生成好代码”和“证明/检查正确”拆成协作组件；最主要的局限是验证成本和复杂程序覆盖；用于后续研究时应复用其验证闭环，而不是简单把目标平台替换成 RISC-V。
