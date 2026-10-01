# QiMeng-NeuComBack 文献阅读总结

论文题目：**QiMeng-NeuComBack: Self-Evolving Translation from IR to Assembly Code**<br>
作者：Hainan Fang, Yuanbo Wen, Jun Bi, Yihan Wang, Tonghui He, Yanlin Tang, Di Huang, Jiaming Guo, Rui Zhang, Qi Guo, Yunji Chen<br>
年份：2025；发表状态：NeurIPS 2025（论文首页标注 39th Conference on Neural Information Processing Systems），arXiv:2511.01183v1<br>
权威记录：https://arxiv.org/abs/2511.01183 ；DOI：https://doi.org/10.48550/arXiv.2511.01183<br>
本地 PDF：`../02_LLM_as_Translator/03_Translation_CrossLanguage_CrossISA/C226-QiMeng-NeuComBack-NeurIPS2025/paper.pdf`（25 页，PDF 首页与正文可读）
拟分类：**TRANSLATOR / T3_Translation_CrossLanguage_CrossISA**。理由是 LLM 的最终输出是目标 ISA 汇编程序；NeuComBack 数据集和 prompt 优化属于辅助机制。

## 1. 研究背景

传统编译器经过多年人工维护才能把 IR 稳定降低到特定 ISA。论文研究 Neural Compilation：让 LLM 直接把 LLVM IR 翻译为 x86_64 或 AArch64 汇编，以降低新 ISA 编译器开发成本并探索传统优化器可能遗漏的优化。

## 2. 论文要解决的问题

论文关注两个问题：其一，LLM 直接生成汇编的功能正确率低；其二，即使可运行，生成结果也常常落后于 clang-O3。现有工作还缺少专门的 IR-to-ASM 基准，使不同模型和方法难以比较。

## 3. 核心方法概述

作者提出 NeuComBack 基准和 Automatic Prompt Learning（APL）。系统先由 LLM 从 LLVM IR 生成汇编，编译并执行测试；失败时进行自调试。离线阶段从完整的生成—编译—修复轨迹中提取错误模式和有效修复规则，演化统一 prompt；在线阶段用演化后的 prompt 生成、验证并迭代优化汇编。

## 4. 实验框架与训练流程

NeuComBack-L1 从清洗后的 ExeBench 选取 200 个较长 IR 程序，评估基础编译正确性；L2 使用 TSVC 的 151 个循环优化程序，评估正确性和超过 clang-O3 的性能。离线 prompt 学习按 mini-batch 和 3 个 epoch 更新，在线每次生成或优化后最多执行若干轮编译/测试自调试。目标架构包括 x86_64 和 aarch64。

## 5. 奖励函数、损失函数或关键公式

论文没有训练 LLM 的新参数损失函数，核心是黑盒生成与验证。任务形式为 `A_target = f_NC(P_source, M; θ)`，其中输入可为 LLVM IR，输出为目标架构汇编。功能正确性要求 `⟦A_target⟧_M ≡ ⟦P_source⟧`；性能比较使用生成汇编与 clang-O3 的运行时间。主要指标为 ACC 和 ACC+Perf。

## 6. 实验设置

基线模型包括 GPT-4o、o3-mini、o1、DeepSeek-V3 和 DeepSeek-R1。L2 每个程序重复运行 11 次，去除 3 次预热和 3 次冷却，取中间 5 次运行时间的中位数。APL 的 x86_64 实验使用 L1 的 120/40/40 和 L2 的 101/25/25 训练/验证/测试划分；aarch64 使用 L2 的 101/25/25 划分。

## 7. 实验结果与结论

在 L2 x86_64 基线中，DeepSeek-R1 的 ACC 为 45.70%，ACC+Perf 为 21.85%。APL 在 L1 测试集把 ACC 从 50.00%（20/40）提高到 80.00%（32/40）；在 L2 x86_64 初始生成中从 44.00% 提高到 64.00%，完整两轮优化后的 ACC+Perf 从 28.00% 提高到 56.00%。aarch64 的 ACC 从 36.00% 提高到 72.00%，ACC+Perf 从 8.00% 提高到 28.00%。x86_64 中演化 prompt 生成的 16 个正确程序有 14 个最终超过 clang-O3。

## 8. 主要创新点

1. 提出专门面向 LLVM IR-to-ASM 的 NeuComBack-L1/L2 基准。
2. 从完整 self-debugging 轨迹提取规则并自动演化 prompt，而不是只依赖静态 few-shot 示例。
3. 在 x86_64 与 aarch64 上同时验证功能正确性和超过 clang-O3 的性能。
4. 展示了 prompt 在不同数据分布之间的迁移能力，并统计了自调试轮数下降。

## 9. 局限性

正确率仍远低于可直接替代传统编译器的水平；样本来自 ExeBench 和 TSVC，真实大型软件、复杂递归、并发和异常控制流覆盖不足。实验主要针对 x86_64/aarch64，未覆盖 RISC-V。APL 依赖大量编译、执行和 LLM 调用，成本及不同模型间的可复现性仍需评估。论文公开版主要是 arXiv v1，正式 NeurIPS 版本与预印本的差异需继续核验。

## 10. 阅读后的研究方向反思

该工作说明，编译器反馈不仅能修复一次生成错误，也能沉淀为跨样本的 prompt 规则。对 RISC-V/RVV 迁移而言，可把目标 ISA 约束、ABI、寄存器分配和向量长度语义作为结构化错误类别，避免把所有错误都交给自然语言 prompt 处理。其 ACC+Perf 指标也适合衡量“可运行且真正有益”的翻译。

## 11. 可进一步尝试的研究方向

可构造 LLVM IR→RV64GC/RVV 的 NeuComBack 子集，加入 QEMU 或 Spike 执行验证和 LLVM 后端交叉编译；将自调试轨迹归纳为 ISA 约束规则库；把 prompt 演化与检索到的 ABI/指令语义文档结合；报告编译成本、token 成本和跨编译器迁移能力；使用独立隐藏测试集防止 prompt 规则过拟合。

## 12. 与其他已读文献的关系

与已有 `Towards LLM-Based Optimization Compilers: Peephole Optimization` 相比，本论文处理完整 LLVM IR→目标汇编流程和多轮优化，而不是单个 peephole 变换。与 `LLM Compiler` 系列相比，本论文重点是可执行的 prompt 学习和 IR-to-ASM 验证流程。与 `LEGO-Compiler` 相比，两者都让 LLM 产生低级代码并使用外部验证，但 LEGO 通过程序分块和可组合翻译扩展规模，本论文通过 self-debug 轨迹演化 prompt。与仓库已有 `QiMeng-xpiler` 的张量程序转译不同，本论文输出 LLVM IR 对应的 CPU 汇编，属于独立工作。

## 13. 一页式总结

**问题**：LLM 直接生成汇编的正确性和性能不足，缺少专用 IR-to-ASM 基准。<br>
**方法**：NeuComBack-L1/L2 + 从自调试轨迹提取经验的 Automatic Prompt Learning。<br>
**输入/输出**：LLVM IR + 目标架构 → x86_64/AArch64 汇编。<br>
**证据**：x86_64 L2 ACC 44%→64%，ACC+Perf 28%→56%；aarch64 ACC 36%→72%。<br>
**角色**：TRANSLATOR/T3；LLM 最终直接输出目标 ISA 汇编。<br>
**代码元数据**：`OPEN_SOURCE`，https://github.com/QiMeng-IPRC/QiMeng-NeuComBack，核验日期 2026-10-01；仓库公开，许可证需中央验收时再次确认。<br>
**查重**：arXiv:2511.01183、标题和 DOI 未命中正式 taxonomy；NeurIPS 版与 arXiv 版应作为同一版本族处理。
