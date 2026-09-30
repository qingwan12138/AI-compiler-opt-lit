# LLM-CodeGen-RAG 文献阅读总结

论文题目：**LLM-based and Retrieval-Augmented Control Code Generation**
作者：Heiko Koziolek、Sten Grüner、Rhaban Hark、Virendra Ashiwal、Sofia Linsbauer、Nafise Eskandani
发表时间：2024；LLM4Code’24，2024-04-16，Lisbon
发表平台：Proceedings of the 1st International Workshop on Large Language Models for Code（与 ICSE 2024 同期）
论文链接或编号：DOI `10.1145/3643795.3648384`；[官方 PDF](https://llm4code.github.io/2024/assets/pdf/papers/39.pdf)
关键词：retrieval-augmented generation、IEC 61131-3、Structured Text、PLC、OpenPLC、GPT-4

## 1. 研究背景

工业自动化控制逻辑通常由工程师在 IEC 61131-3 Structured Text（ST）中编写，并依赖包含大量预构建 function block 的专有库。通用 LLM 不知道这些库中的接口和参数，可能生成复杂、错误或难以复用的控制代码。

## 2. 论文要解决的问题

论文研究如何把 function block 规格文档检索到 LLM prompt 中，使模型生成能够实例化既有控制模块的 ST 程序，并能导入 PLC IDE 编译和仿真。

## 3. 核心方法概述

方法将 function block 文档切分、嵌入并保存到向量库；控制叙事被整理成生成 prompt，再通过相似度检索加入相关模块规格，最后由 LLM 生成 ST 控制逻辑。

```text
Function block规格 → 加载/切分/嵌入 → Vector Store
控制叙事 → 生成控制逻辑prompt → 相似度检索相关模块
        ↓
拼接增强prompt → GPT-4生成IEC 61131-3 ST
        ↓
导入OpenPLC → 编译、调试器仿真、工程师检查
```

## 4. 实验框架与训练流程

本文不涉及模型训练，采用 GPT-4-32k（版本 0613），temperature=0。原型用 LangChain 管理检索链，FAISS-CPU 1.7.4 保存向量。作者没有实现自动 Control Narrative Extractor，而是从 50 多份客户控制叙事中人工制定若干 prompt。

## 5. 奖励函数、损失函数或关键公式

本文没有强化学习奖励函数或训练损失。检索阶段使用文本 embedding 和相似度搜索；论文未给出需要优化的统一数学目标。

## 6. 实验设置

### 6.1 数据集来源

function block 规格采用 OSCAT BASIC 开源库，包含 400 多个 block，规格 PDF 为 496 页、约 4.3 MB。作者还收集 50 多份客户 Control Narratives，并人工制定测试 prompt；论文没有给出训练/验证/测试集划分。

### 6.2 模型与工具

模型为 GPT-4-32k 0613；embedding 使用 OpenAI text-embedding-ada-002，1536 维；向量库为 FAISS-CPU 1.7.4；应用框架为 LangChain；PLC IDE、编译器和调试器为 OpenPLC-Editor。生成代码在 OpenPLC 中编译为 C-code 并仿真。

### 6.3 对比方法

论文进行三个代表性 spot tests，没有与不同 LLM、无检索 LLM 或人工实现做系统定量对比；这类比较被列为未来工作。

### 6.4 评价指标

主要检查生成代码是否选择了正确 function block、能否编译、仿真输出是否符合设定控制逻辑。没有统一准确率、延迟或性能基准指标。

## 7. 实验结果与结论

### 7.1 主要结果

三个测试覆盖采样平均值、正弦波/阶梯函数和 PID 控制器。GPT-4 能从检索结果中选出相关 block；代码经过少量人工修正后均成功导入 OpenPLC 并完成编译和仿真。

### 7.2 与传统方法的比较

论文不报告与规则式控制代码生成或人工实现的定量比较。方法主要提供了复用既有 block、降低从零编写 glue logic 工作量的工程路径。

### 7.3 与其他 LLM 方法的比较

没有不同 LLM 的对照实验。作者明确把不同模型的生成质量和生成时间留作未来工作。

### 7.4 消融实验

没有独立消融实验。论文通过检索到的 block 与无检索时的接口缺失问题说明 RAG 的作用，但未提供统计消融表。

### 7.5 案例分析

采样平均值测试中，模型因 OSCAT 文档没有明确列出变量名而误用 `OUT_MAX` 和 `OUT`，作者手动修正后仿真正确。PID 测试中，模型正确选择 CTRL_PID 与 TON，并按给定参数实现自动模式延迟。

## 8. 主要创新点

### 8.1 面向工业控制 function block 的检索增强生成

论文把 function block 规格文档作为结构化外部知识，检索后约束 LLM 生成 ST，使输出能调用既有模块。价值在于连接自然语言控制叙事与 PLC 工具链，而非提出新的 embedding 或编译算法。

## 9. 局限性

论文明确承认只有三个 spot tests，测试不充分；生成代码仍需人工修正，且 OSCAT 文档中的接口信息不完整。OpenPLC 比商业 PLC IDE 简单，OSCAT 也不是商业库；Control Narrative Extractor 尚未实现，通用性仍待验证。

## 10. 阅读后的研究方向反思

可借鉴“检索接口规格—生成源代码—编译/仿真反馈”的闭环。若迁移到 LLVM IR 或 RISC-V，应检索 target intrinsic、ABI 和 ISA 约束，并用编译器诊断与差分执行反馈修复输出，而不能只替换提示词中的语言名称。

## 11. 可进一步尝试的研究方向

可以构建面向 RISC-V RVV 的检索增强 kernel 生成器：检索 intrinsic/ABI 规格，生成 RVV C 或汇编，使用 LLVM/GCC 编译、仿真器和板卡测试验证。与本文区别在于增加 ISA 语义等价和跨编译器约束，形成可量化的低层代码研究问题。

## 12. 与其他已读文献的关系

本论文与 CUDA-LLM、GPU Kernel Scientist、Astra 都让模型输出可编译的低层程序并使用工具反馈，但本论文的对象是 IEC 61131-3 ST 控制源代码，重点是文档检索和 function block 复用，后三者重点是 GPU kernel 性能优化。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | 检索增强的 PLC 控制代码生成 |
| 核心问题 | LLM 不掌握 function block 库接口 |
| 输入 | 控制叙事、function block 规格 |
| 输出 | IEC 61131-3 Structured Text |
| 核心方法 | embedding、FAISS 检索、GPT-4 生成 |
| 使用的模型 | GPT-4-32k 0613 |
| 使用的编译器工具 | OpenPLC-Editor 编译器和调试器 |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否；采用编译和仿真检查 |
| 数据集规模 | 400+ blocks、50+ narratives、3 个 spot tests |
| 主要指标 | block 选择、编译成功、仿真行为 |
| 最重要实验结果 | 三个测试经人工修正后编译并仿真正确 |
| 核心创新 | 面向控制库规格的 RAG 代码生成 |
| 主要局限 | 样本少、需人工修正、无系统 baseline |
| 与 RISC-V 研究的相关性 | 中低；可借鉴规格检索和工具链反馈 |
| 最适合作为 | 受约束源代码生成与检索 baseline |

这篇论文最值得学习的是把库接口文档纳入生成上下文并接入真实 PLC 工具链；主要局限是验证规模很小，后续使用时应把它视为可复现的 RAG 原型，而不是已经证明鲁棒的工业代码生成器。

## 元数据与分类核验

- Paper_ID：C223；Primary_Category：TRANSLATOR；Secondary_Category：T3_Translation_CrossLanguage_CrossISA。
- DOI：10.1145/3643795.3648384；arXiv：无。
- Code_Status：PUBLIC_REPO；Code_URL：https://github.com/hkoziolek/LLM-CodeGen-RAG；Code_Checked_At：2026-10-01。
