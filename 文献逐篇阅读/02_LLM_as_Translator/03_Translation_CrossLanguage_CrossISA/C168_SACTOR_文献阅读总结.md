# SACTOR 文献阅读总结

论文题目：**SACTOR: LLM-Driven Correct and Idiomatic C to Rust Translation with Static Analysis and FFI-Based Verification**

作者：Tianyang Zhou、Ziyi Zhang、Haowen Lin、Somesh Jha、Mihai Christodorescu、Kirill Levchenko、Varun Chandrasekaran

发表时间：2026 年 7 月

发表平台：ACL 2026 Main Conference

论文链接或编号：ACL Anthology 2026.acl-long.28；DOI 10.18653/v1/2026.acl-long.28
元数据核验来源：[ACL Anthology 正式论文页](https://aclanthology.org/2026.acl-long.28/)；[作者实现仓库](https://github.com/qsdrqs/sactor)
代码/数据/工件：作者实现：[sactor](https://github.com/qsdrqs/sactor)

关键词：C-to-Rust、源代码翻译、静态分析、FFI、编译器反馈、接口规范、idiomatic Rust

> 本文档依据本地完整 PDF（28 页，含附录）逐页阅读整理。论文事实与阅读后的研究思考分开描述。

## 1. 研究背景

本文研究 C 到 Rust 的大语言模型（Large Language Model，LLM）源代码翻译。C 便于直接操作内存和硬件，但手工内存管理带来缓冲区溢出、悬空指针和内存泄漏等风险；Rust 通过所有权、借用和生命周期在编译期提供内存安全，同时保留系统级编程能力（第 1–2 页）。

传统 C2Rust 等基于 AST 和规则的工具覆盖面较广，但生成结果大量使用 `unsafe`、显式类型转换和 C 风格接口，可读性与可维护性较差。LLM 更有能力重写成符合 Rust 习惯的代码，却可能幻觉、遗漏代码、误解指针/数组/字符串语义，且本身不给出正确性保证。既有 LLM 流程常把完整程序一次性翻译或事后修复，错误定位粗、长上下文负担大，复杂 C 特性上尤其脆弱（第 1–3 页）。

因此论文把“保持行为正确”和“形成 idiomatic Rust”拆成两个阶段，并用静态分析、编译器诊断和 C/Rust 外部函数接口（Foreign Function Interface，FFI）端到端测试共同约束生成结果。

## 2. 论文要解决的问题

### 2.1 语义保持与 Rust 风格冲突

第一阶段要尽量保持 C 的接口、指针和内存布局；第二阶段要把低级、可能含 `unsafe` 的结果改写成借用、所有权、`Vec`、`Box`、`Option` 等 Rust 抽象，同时不破坏行为（第 3–4 页）。

### 2.2 大程序的上下文与错误定位

完整项目一次性输入会超出有效上下文，且一个错误可能污染全局结果。论文将类型、全局变量和函数按依赖图拆成片段，按依赖顺序自底向上翻译；片段通过后才继续后继片段（第 3–4 页）。

### 2.3 两种接口之间的验证

非 idiomatic 结果可直接与原 C 构建通过 ABI/FFI 混合链接；idiomatic 结果改变函数签名和数据类型，不能直接链接。因此论文需要生成 C 兼容的适配 harness，把 C-facing 表示与 Rust 表示来回转换，并在已有端到端测试覆盖范围内验证（第 4 页）。

> 本文主要研究：如何利用静态分析引导 LLM，把 C 程序按依赖和函数片段分阶段翻译为可测试、较安全且更符合 Rust 习惯的代码。

## 3. 核心方法概述

SACTOR 是一个结构感知、LLM 驱动的两阶段 C-to-Rust 翻译器（第 2–4 页）。第一阶段生成接口等价的 unidiomatic Rust，允许必要的 `unsafe`；第二阶段从该结果生成行为等价的 idiomatic Rust，结合 Crown 的指针可变性、数组“胖度”和所有权提示。每个片段都先编译，再通过 FFI 和端到端测试；失败时将编译错误或运行轨迹结构化后反馈给 LLM，最多重试 6 次（第 4–5 页、附录 C）。

```text
C 源程序
  ↓ libclang/C Parser：预处理、切分类型/全局量/函数、构建依赖图
  ↓ 依赖序
LLM 阶段 1：C → 接口等价 unidiomatic Rust
  ↓ rustc 编译 + 与原 C 混合链接 + E2E 测试 + 诊断/轨迹修复
已验证的 unidiomatic Rust
  ↓ Crown 指针提示 + LLM 阶段 2 + 生成机器可读 SPEC
idiomatic Rust + SPEC
  ↓ 规则 harness 生成器；未覆盖模式用 LLM 补 TODO
  ↓ C/Rust 适配器、FFI、E2E 测试、局部修复
合并后的 Rust 项目
```

LLM 直接输出变换后的 Rust 源代码，故按 taxonomy v2 属于 `TRANSLATOR/T3_Translation_CrossLanguage_CrossISA`；编译器和测试工具提供反馈，但不是 LLM 最终角色。该工作是跨语言源代码翻译，不是 LLVM IR 或具体 ISA 翻译。

## 4. 实验框架与训练流程

### 4.1 静态分析和任务划分

SACTOR 解析 `make` 产生的编译命令，递归展开非系统头文件、保留系统头文件，并把 GCC 预处理标志用于解析条件编译。基于 libclang 的 C Parser 将输入划分为单个类型、全局变量或函数，并建立“节点依赖其直接依赖项”的有向图；环依赖当前直接判为不支持（第 3 页、附录 B）。

### 4.2 阶段一：unidiomatic Rust

类型翻译可借助 C2Rust；全局变量和函数由 LLM 翻译，提示中提供已翻译的依赖签名和类型。输出尽量保留 C 接口、C 标准库调用和指针语义，必要时允许 `unsafe`。rustc 编译通过后，Rust 片段作为共享库嵌入原 C 构建，端到端测试比较原 C 与混合程序输出；失败则用编译错误或 `trace_fn` 产生的输入/输出轨迹修复（第 3–5 页、附录 C/N.1–N.2）。

### 4.3 阶段二：idiomatic Rust

LLM 以阶段一结果为输入，结合 Crown 的指针提示，把拥有的指针映射为 `Box` 等拥有类型，把借用指针映射为引用或智能指针，并尽量消除 `unsafe`。阶段二同时输出每个函数/结构体的机器可读 SPEC，描述指针形状（scalar、cstring、slice、ref）、长度来源、可空性、所有权、返回值和比较方式（第 3–5 页、附录 L/N.3）。

### 4.4 验证、反馈和合并

规则式 harness 生成器依据 SPEC 产生 C ABI shim、Rust 适配器和结构体双向转换；不支持的 SPEC 模式局部生成 TODO，再由 LLM 补齐。编译、链接和 E2E 测试失败会触发带诊断/轨迹的修复。所有片段完成后清理重复定义，合并为统一 Rust 项目并再次运行 E2E 测试（第 4 页、附录 L）。

本文不涉及预训练、SFT、PPO、GRPO 或其他强化学习训练；主要是提示式推理、静态分析、编译/测试工具调用和迭代修复。论文使用“soft equivalence”而非完整形式化等价证明。

## 5. 奖励函数、损失函数或关键公式

本文没有使用强化学习奖励函数，也没有训练损失函数。核心运行目标是：对当前片段先满足编译，再满足 E2E 测试；阶段二还要求满足 Rust 风格与安全代码指标。附录 C 的过程可抽象为：

```text
若 compileOk = false：把编译诊断反馈给 LLM 修复
否则若 E2E testOk = false：提取轨迹/标准错误并反馈修复
否则：接受片段
```

这是接受/拒绝式验证循环，不应写成可微损失或形式化证明。SPEC 的 `by_value`、`by_slice`、`skip` 只规定 harness 自检如何比较字段；论文没有把它定义为模型训练目标（第 4 页、附录 C/L）。

## 6. 实验设置

### 6.1 数据集来源

| 数据/项目 | 规模与处理 | E2E 覆盖（行/函数） |
| --- | --- | --- |
| TransCoder-IR | 原有 698 个 C 程序；去除已有 Rust、编译/内存错误后取代码行数最多的 100 个 | 有；97.97% / 99.5% |
| Project CodeNet | 从超过 75 万个 C 程序中过滤使用 `argc/argv` 的程序；先选 200 个，保留能生成 E2E 测试的 100 个 | 自动生成；94.37% / 100% |
| CRust-Bench | 100 个真实 C 仓库组成的基准中取 50 个，排除环依赖和不支持 intrinsic | 有；76.18% / 80.98% |
| libogg | 约 2,041 行、6 个结构体、3 个全局变量、77 个导出函数 | 有；83.3% / 75.3% |

上述数字来自表 6（第 15 页）。CodeNet 的测试由 SACTOR 根据源代码生成；其余主要复用上游测试。测试覆盖决定 soft equivalence 的置信度，不能等同于全路径语义等价。

### 6.2 模型与工具

数据集实验使用 GPT-4o（`gpt-4o-2024-08-06`）、Claude 3.5 Sonnet（`claude-3-5-sonnet-20241022`）、Gemini 2.0 Flash、Llama 3.3 Instruct 70B、DeepSeek-R1 671B；真实项目使用 GPT-4o 和 GPT-5（`gpt-5-2025-08-07`）。大多数温度为 0，GPT-5 使用提供方默认值（表 1，第 5 页）。工具包括 libclang/C Parser、C2Rust、Crown、GCC、rustc、Rust-Clippy、procedural macro、FFI 和 gcov；论文未明确给出所有编译器版本和硬件规格。

### 6.3 对比方法

主要 baseline 为 C2Rust、Crown、C2SaferRust、Vert；另有 GPT-4o 单步直接翻译的 trivial baseline。C2Rust/Crown 代表规则或静态分析翻译，C2SaferRust 代表对 C2Rust 结果做 LLM 安全化，Vert 代表 LLM 加模糊测试/符号执行的特定子集翻译（第 7 页、附录 J.2）。

### 6.4 评价指标

| 指标 | 含义 | 趋势 |
| --- | --- | --- |
| Success Rate | 在阶段内生成并通过 E2E 测试的程序/函数比例，单片段最多 6 次尝试 | 越大越好 |
| Clippy lint alert | 警告与错误总数 | 越小越好 |
| Unsafe-Free | 无 `unsafe` 代码的程序比例 | 越大越好 |
| Avg. Unsafe | `unsafe` 块/函数中的 token 占总 token 比例 | 越小越好 |
| E2E coverage | 测试触达的行/函数比例 | 越大通常越强，但不是等价证明 |
| LLM token/query cost | 成功 idiomatic 翻译的平均 token 与查询次数 | 越小越省成本 |

## 7. 实验结果与结论

### 7.1 主要结果

在 TransCoder-IR 上，DeepSeek-R1 的 unidiomatic/idiomatic 成功率为 94%/93%；GPT-4o 为 84%/80%。在 CodeNet 上，DeepSeek-R1 为 86%/84%，Claude 3.5 为 86%/83%，GPT-4o 为 84%/79%（图 3，第 6 页）。

### 7.2 与传统方法比较

在 GPT-4o、两数据集、最多 6 次尝试的设置下，C2Rust 和 Crown 都能编译但生成大量 `unsafe`；C2SaferRust 在 TransCoder-IR 有 45.6% Unsafe-Free，CodeNet 为 0%；Vert 在 TransCoder-IR 为 95.7% Unsafe-Free，但只覆盖其特定任务子集。SACTOR idiomatic 阶段在两数据集均为 100% Unsafe-Free；表 2 的 SACTOR unidiomatic 阶段仍保留较高 `unsafe`，这是设计目标而非阶段二结果。

### 7.3 与 LLM 方法比较

图 4 显示 SACTOR 的 unidiomatic 输出 Clippy 问题数也低于 C2Rust、Crown、C2SaferRust，idiomatic 输出进一步减少问题。论文没有在所有指标上给出一个可直接合并的单一平均分，因此不能概括为“全面超过所有 LLM”。

### 7.4 真实项目与消融

CRust-Bench 50 个样本中，unidiomatic 平均逐样本函数成功率为 85.15%（788/966，聚合 81.57%），32/50 个样本完整通过；在这 32 个样本上，idiomatic 平均为 51.85%（249/580，聚合 42.93%），8/32 完整通过。unidiomatic 平均每函数 2.96 个 lint，idiomatic 为 0.28 个（表 3，第 7–8 页）。

libogg 的 77 个函数在 GPT-4o/GPT-5 下 unidiomatic 均为 100%；idiomatic 成功率分别为 53% 和 78%，平均尝试次数 2.00 和 1.25（表 4，第 8 页）。移除 Crown 后 GPT-4o 的 libogg idiomatic 成功函数从 41 降到 34，成功率相对下降 17%（表 10，第 19 页）。移除反馈使 Llama 3.3 在 CodeNet 的 unidiomatic/idiomatic 成功率从 62/59 提升为 83/76，GPT-4o 只从 82/77 变为 84/79（图 9，第 18–19 页）。

### 7.5 成本和失败分析

成功样本上，GPT-4o 每例平均 token 为 2,651.21（TransCoder-IR）和 2,565.36（CodeNet）；DeepSeek-R1 为 17,895.52 和 13,592.61，约为 GPT-4o 的 5–7 倍（表 8，第 17 页）。主要失败包括类型/数组/字符串映射、函数指针与可变全局量、生命周期/借用、ABI 名称漂移、harness 缓冲区长度和未支持的 variadic/intrinsic。作者结论是：两阶段结构和验证有效，但覆盖、成本、分析精度和测试依赖仍是瓶颈（第 6–9 页、附录 H–M）。

## 8. 主要创新点

### 8.1 创新点一：语义保持与 idiomatic 重写的两阶段分离

先保持 ABI/接口并允许必要 `unsafe`，再做安全、风格化改写，避免模型在一次生成中同时解决 C 语义和 Rust 所有权。libogg 与两数据集的结果支持这种分工，但两阶段设计本身不是“保证正确”，仍受测试覆盖约束。

### 8.2 创新点二：依赖序的函数级翻译与局部验证

C Parser 将类型、全局变量和函数按依赖排序，片段通过编译和 E2E 后才成为后继上下文。这为失败提供了局部归因，并减轻整项目长上下文问题；环依赖仍不支持。

### 8.3 创新点三：SPEC 驱动的 ABI 适配验证

idiomatic 翻译同时产生描述指针形状、长度、可空性、所有权和返回值的 SPEC，规则生成器据此生成 C-compatible harness；未覆盖模式只局部交给 LLM。这个接口规范把“模型输出的 Rust 类型”与“验证所需的 C ABI”连接起来，是论文区别于简单编译错误修复的关键机制。

### 8.4 创新点四：编译/运行反馈的结构化闭环

rustc 诊断、E2E 输出及 `trace_fn` 输入/输出轨迹被作为定向修复上下文。消融显示低能力模型从反馈中收益更明显；高能力模型可能已接近随机重试上限。

## 9. 局限性

### 9.1 论文明确承认的局限

论文第 9 页明确指出：模型差异显著；复杂宏、函数指针、全局状态、C variadic、内联汇编和部分 intrinsic 支持有限；harness fallback 可能脆弱；多阶段查询、编译和测试成本高；soft equivalence 依赖现有 E2E 覆盖；静态分析可能低估别名、所有权和指针形状。

### 9.2 阅读后发现的潜在局限

首先，C2Rust 仅用于类型翻译，若其生成的类型名或匿名类型不稳定，会沿依赖传播错误。其次，idiomatic 阶段的全量 E2E 测试并不提供形式化等价证明，unsafe-free 也不等于逻辑正确。再次，CodeNet 测试自动生成、CRust-Bench 只取排除后的 50 个样本，结果不能直接外推到所有 C 项目。最后，实验没有 LLVM IR、真实硬件性能或 RISC-V/跨 ISA 评估；平台迁移需要另行验证。

## 10. 阅读后的研究方向反思

SACTOR 最值得借鉴的是“静态语义提示—LLM 源码变换—编译/运行验证—局部修复”的职责分离，以及把 ABI 映射显式化为机器可读接口。对本仓库主题，它是 TRANSLATOR/T3 的直接相关工作，但相关性主要来自跨语言源码翻译，不应误称为 LLVM/RISC-V 优化论文。

其核心贡献已经是 C-to-Rust 的两阶段、依赖序和 SPEC harness 组合；仅把目标语言换成 C++、把 C 编译器换成 RISC-V 后端，创新性不足。若迁移到 RISC-V，应新增 ISA 语义差异、ABI/调用约定、向量扩展或真实硬件验证问题，才可能形成独立研究问题。最适合作为跨语言翻译 baseline、编译器反馈修复模块和接口验证设计参考，而不是直接照搬的完整方案。

## 11. 可进一步尝试的研究方向

### 11.1 面向 RISC-V ABI 的跨语言验证翻译

#### 研究问题

C/Rust 片段在 RISC-V LP64/向量 ABI 下如何保持调用约定、数据布局和指针长度语义？

#### 与原论文的区别

把 SPEC 从语言级 C ABI 扩展为含 RISC-V ABI 与 RVV 向量形状的联合规范，并在真实 RISC-V 硬件或 QEMU 上验证。

#### 可能的创新点

ABI-aware SPEC、跨编译器诊断归一化、标量/向量边界的自动 harness 生成。

#### 实验框架

```text
C/Rust 源码 → 静态分析 + ABI SPEC → LLM 翻译 → riscv64 编译/链接 → QEMU/硬件 E2E 对比
```

#### 可行性

需要 Clang/Rust、riscv64-unknown 工具链、QEMU 或开发板和可移植测试集。

#### 主要风险

ABI、未定义行为和硬件向量实现差异会使失败归因困难。

### 11.2 面向低覆盖率项目的增量验证

#### 研究问题

当原项目没有足够 E2E 测试时，如何补充能区分 C/Rust 行为的输入，而不把测试通过误当作证明？

#### 与原论文的区别

在 SACTOR soft equivalence 之上加入覆盖率引导的差分测试或符号约束，并报告覆盖边界。

#### 可能的创新点

以失败轨迹和指针形状驱动测试生成，专门覆盖 ABI 适配薄弱点。

#### 实验框架

```text
现有测试 → 覆盖率/差分缺口 → 生成输入 → C/Rust 对比 → 反馈翻译或扩充测试
```

#### 可行性

需要 gcov/LLVM coverage、模糊测试器和可重放的 C/Rust 构建。

#### 主要风险

生成测试可能偏向易达路径，仍不能给出全语义保证。

### 11.3 面向复杂指针的可验证类型迁移

#### 研究问题

如何处理别名、函数指针、可变全局状态和变长数组，使 Rust 类型选择有可审计证据？

#### 与原论文的区别

把 Crown 的提示扩展为带来源和置信度的类型候选，并在不确定时保守保留边界而不是强行生成 safe Rust。

#### 可能的创新点

别名/所有权证据链、候选类型枚举、局部不确定性报告。

#### 实验框架

```text
C 全程序分析 → 指针/别名候选 → LLM 生成类型映射 → SPEC 自检 → 编译与差分测试
```

#### 可行性

需要现有 Crown/libclang、Rust borrow checker 和含函数指针的 C 基准。

#### 主要风险

静态分析误报会增加 harness 复杂度，候选爆炸会提高 LLM 成本。

## 12. 与其他已读文献的关系

本次 staging 只完成 SACTOR 一篇论文，因此没有当前批次内可进行正文级横向比较的其他新论文。SACTOR 在正文中将 Vert、Flourine、C2SaferRust 等作为相关工作或 baseline，但本笔记没有重新阅读它们的 PDF，不能据此补写它们的完整实验结论。

与正式语料中的 C153 `LLMigrate` 存在 C-to-Rust、静态分析和编译反馈主题交集；C153 已是正式记录，本次将其作为去重排除项，不把两者合并或宣称方法等同。SACTOR 的可辨识差异是两阶段 unidiomatic→idiomatic、Crown 指针提示和 SPEC 驱动的 FFI harness。除此之外，本文不虚构与其他已读文献的组合实验结果。

## 13. 一页式总结

| 项目 | 内容 |
| --- | --- |
| 论文研究任务 | C 到 Rust 的 LLM 源代码翻译 |
| 核心问题 | 同时保持 C 行为并生成安全、idiomatic Rust |
| 输入 | C 项目、类型/全局变量/函数、编译命令和依赖 |
| 输出 | 经测试验证并合并的 Rust 项目 |
| 核心方法 | 依赖序片段翻译；unidiomatic→idiomatic 两阶段；SPEC/FFI harness |
| 使用的模型 | GPT-4o、Claude 3.5、Gemini 2.0、Llama 3.3 70B、DeepSeek-R1、GPT-5 |
| 使用的编译器工具 | libclang/C Parser、C2Rust、Crown、GCC、rustc、Clippy、FFI、gcov |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；使用测试覆盖范围内的 soft equivalence |
| 数据集规模 | TransCoder-IR 100、CodeNet 100、CRust-Bench 50、libogg 1 项目 |
| 主要指标 | E2E 成功率、Clippy、Unsafe-Free、Avg. Unsafe、覆盖率、token/query 成本 |
| 最重要实验结果 | DeepSeek-R1 在两数据集 idiomatic 成功率 93%/84%；libogg GPT-5 为 78% |
| 核心创新 | 两阶段语义/风格分离 + 依赖序局部验证 + SPEC harness |
| 主要局限 | 测试覆盖依赖、复杂 C 特性不支持、成本高、静态分析和 harness 不稳定 |
| 与 RISC-V 研究的相关性 | 中：跨语言翻译流程直接相关，但未做 RISC-V 实验 |
| 最适合作为 | TRANSLATOR/T3 baseline、反馈修复模块、ABI 验证设计参考 |

这篇论文最值得学习的是把“能保持行为”和“能写出 idiomatic Rust”拆成可验证的阶段，并用显式 SPEC 连接语言类型与 ABI harness；最主要的局限是验证仍依赖有限 E2E 测试、复杂指针和项目特性会导致级联失败。如果用于后续研究，合理方式是把它作为跨语言翻译与验证 baseline，再增加 RISC-V ABI/RVV 或更强覆盖分析，而不是只替换目标平台。
