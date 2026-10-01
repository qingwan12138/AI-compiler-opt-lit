# APIRAT 文献阅读总结

论文题目：**APIRAT: Integrating Multi-source API Knowledge for Enhanced Code Translation with LLMs**<br>
作者：Chaofan Wang, Guanjie Qiu, Xiaodong Gu, Beijun Shen<br>
年份：2025；正式发表：2025 IEEE 49th Annual Computers, Software, and Applications Conference (COMPSAC)，pp. 1400–1405；预印本 arXiv:2504.14852v1<br>
DOI：10.1109/COMPSAC65507.2025.00176；IEEE 记录：https://ieeexplore.ieee.org/document/11126711 ；arXiv：https://arxiv.org/abs/2504.14852<br>
本地 PDF：`../02_LLM_as_Translator/03_Translation_CrossLanguage_CrossISA/C229-APIRAT-COMPSAC2025/paper.pdf`（10 页，arXiv v1；IEEE 官方元数据与 arXiv 标题、作者一致，正式 PDF 端点受访问限制）<br>
拟分类：**TRANSLATOR / T3_Translation_CrossLanguage_CrossISA**。LLM 直接生成目标语言源代码，API 检索和测试工具只提供上下文与验证。

## 1. 研究背景

跨语言代码迁移可复用已有软件并降低维护成本，但不同语言的 API、类型和调用约定差异会导致 LLM 生成代码不可编译或语义不等价。作者对 CodeNet 和 AVATAR 的初步分析发现，超过 60% 的翻译错误来自 API 误翻译。

## 2. 论文要解决的问题

论文要解决单个 API 误用、API 序列遗漏或冗余、参数和返回值类型错误，以及相似但语义不同的目标 API 选择问题。目标是让 LLM 在 Python↔Java 翻译中利用外部 API 知识生成能通过测试的目标程序。

## 3. 核心方法概述

APIRAT（API Retrieval Augmented Translation）先让 LLM 直接翻译源程序并同步生成目标语言测试。初译失败后，系统从三个来源补充知识：目标语言 API 序列检索、目标 API 序列反向翻译、单 API 映射池。增强后的 prompt 重新要求 LLM 输出完整目标代码，并用测试判断是否成功。

## 4. 实验框架与训练流程

离线阶段从 GitHub 热门 Java/Python 项目抽取 API 序列，构建每种语言 200,000 条记录的向量库；从官方 Python/Java 文档整理并人工复核 586 条 Java→Python 和 179 条 Python→Java API 映射。在线阶段解析输入的 API 序列，使用 text-embedding-3-large 检索 top-k 序列和 top-n 映射，把结果与源代码及历史对话放入增强 prompt。翻译失败时才触发知识增强流程。

## 5. 奖励函数、损失函数或关键公式

论文没有训练新的生成模型损失或 RL 奖励。翻译目标是从源程序 `x` 生成语义等价目标程序 `y`，并使用测试集计算 Computational Accuracy（CA）。API 序列检索使用余弦相似度；API 映射池和序列均以向量检索方式获取上下文。通过测试的程序被视为增强有效。

## 6. 实验设置

CodeNet 包含 Java→Python 和 Python→Java 各 200 个样本；AVATAR 包含 Java→Python 249 个、Python→Java 250 个样本。基线为直接翻译、两步翻译、EXP 和 SpecTra。主实验使用 GPT-3.5-turbo；跨模型实验替换为 StarCoder 与 GPT-4o-mini。检索编码使用 text-embedding-3-large，运行环境为 Ubuntu 23.10、两张 RTX 4090、CUDA 12.0。

## 7. 实验结果与结论

表 III 中，APIRAT 在 CodeNet 的 Python→Java/Java→Python CA 为 91.5%/74.5%，在 AVATAR 为 58.0%/75.9%，均高于直接翻译的 82.0%/63.5% 和 49.6%/67.5%。相对于 SpecTra，APIRAT 在四个方向分别提升 4.0、4.5、4.0 和 10.4 个百分点。跨模型时，GPT-4o-mini 的 CodeNet Python→Java 从 92.0% 提升到 95.5%，AVATAR Python→Java 从 63.2% 提升到 69.2%。消融实验显示序列检索、反向翻译和 API 映射组合通常达到最佳结果；text-embedding-3-large 在 APISEQDATA 上达到 Python→Java 97.85%、Java→Python 98.57% 的检索准确率。

## 8. 主要创新点

1. 对 LLM 代码翻译中的 API 错误建立十二类错误模式。
2. 将目标 API 序列检索、反向翻译和单 API 映射统一到翻译 prompt。
3. 用官方 API 文档、热门开源项目和人工复核构建可复用知识池。
4. 公开实现和评测，并在多个 LLM、语言方向和数据集上验证泛化性。

## 9. 局限性

实验只覆盖 Python 与 Java，不能直接证明对 C/C++、Rust 或 ISA 级翻译有效。APIRAT 依赖 GPT API、embedding 服务、规则测试生成器和人工构建的 API 映射池，成本与服务依赖较高。映射池规模和项目来源可能影响结果，某些 AVATAR 方向中额外 API mapping 没有带来提升。论文主要评价功能 CA，没有系统报告运行性能、资源开销或大型多文件项目迁移。

## 10. 阅读后的研究方向反思

APIRAT 的直接贡献是把 API 知识作为翻译时上下文，而不是把 LLM 当作 API 选择器。对编译器语料库而言，类似知识池可以换成 LLVM intrinsic、RISC-V/RVV intrinsic、ABI 约束和目标库调用约定，并配合交叉编译器测试。这样能把跨 ISA 翻译错误细分为语义、类型、寄存器和调用约定错误。

## 11. 可进一步尝试的研究方向

可为 CUDA↔HIP、NEON↔RVV 和 C↔Rust 构建架构 API/Intrinsic 映射池；把 API 序列检索升级为 AST/调用图联合检索；使用编译器诊断和差分执行反馈更新知识池；报告翻译速度、token、检索成本和多文件上下文长度；在项目级隐藏测试集上验证是否减少跨文件 API 漏译。

## 12. 与其他已读文献的关系

APIRAT 与仓库已有跨语言 Translator 共同使用编译/执行验证，但其中心对象是 API 序列和映射知识。与 `Rectifier` 的错误修复不同，APIRAT 在初次翻译失败后补充目标 API 知识并重新生成；与 `LLMigrate`、`SACTOR` 的 C→Rust 项目迁移相比，APIRAT 聚焦 Python↔Java 函数级 API 转换，不涉及大型项目依赖恢复；与 `UniPar` 的 CUDA/OpenMP 翻译不同，APIRAT 不处理并行语言或 ISA。IEEE 正式版和 arXiv:2504.14852 是同一版本族，只保留正式 DOI 版本。

## 13. 一页式总结

**问题**：LLM 跨语言翻译中 API 误用占比高。<br>
**方法**：目标 API 序列检索 + 序列反向翻译 + 单 API 映射，失败后增强 prompt 重翻。<br>
**输入/输出**：Python/Java 源代码 → 另一语言可执行源代码。<br>
**证据**：CodeNet CA 最高 91.5%/74.5%，AVATAR 最高 58.0%/75.9%，优于直接翻译和 SpecTra。<br>
**角色**：TRANSLATOR/T3；LLM 最终直接输出目标源代码。<br>
**代码元数据**：`OPEN_SOURCE`，https://github.com/CodeTransFusion/ApiRAT，核验日期 2026-10-01；公开仓库许可证需中央验收时再次确认。<br>
**查重**：DOI `10.1109/COMPSAC65507.2025.00176` 与 arXiv `2504.14852` 视为同一版本族，当前 taxonomy/活动索引未命中。
