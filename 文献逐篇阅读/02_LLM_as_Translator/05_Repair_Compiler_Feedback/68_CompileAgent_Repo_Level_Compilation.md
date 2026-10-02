# 68. CompileAgent 文献阅读总结

论文题目：**CompileAgent: Automated Real-World Repo-Level Compilation with Tool-Integrated LLM-based Agent System**
作者：Li Hu、Guoqiang Chen、Xiuwei Shang、Shaoyin Cheng、Benlong Wu、Gangyang Li、Xu Zhu、Weiming Zhang、Nenghai Yu
发表时间：2025
发表平台：Proceedings of the 63rd Annual Meeting of the Association for Computational Linguistics（ACL 2025），页 2078–2091
论文链接或编号：[ACL Anthology 2025.acl-long.103](https://aclanthology.org/2025.acl-long.103/)；arXiv [2505.04254](https://arxiv.org/abs/2505.04254)
关键词：仓库级编译, LLM Agent, 编译错误修复, 构建系统, CompileAgentBench
代码/数据/工件：作者公开代码：[AutoCompiler](https://github.com/Ch3nYe/AutoCompiler)
元数据核验来源：[ACL Anthology 正式论文页](https://aclanthology.org/2025.acl-long.103/)；[2505.04254](https://arxiv.org/abs/2505.04254)

> 本文档用于文献阅读、组会汇报和后续研究分析。

---

## 1. 研究背景

真实软件项目编译失败的自动化修复是一个极具实际价值但尚未充分解决的问题。当一个开源仓库克隆到新环境后，由于缺少依赖、构建配置错误、编译器版本差异等原因，编译往往不会一次性通过。人工排查构建问题耗时费力，而LLM直接生成代码的环境知识有限。已有工作关注单文件代码生成的功能正确性，但仓库级编译问题涉及构建系统、依赖管理和环境配置等更复杂的因素。该研究首次系统性地将LLM Agent应用于仓库级编译修复任务。

## 2. 论文要解决的问题

核心问题是如何设计一个以Agent形式工作的系统，使其能够自动搜索构建指令、诊断编译错误并解决真实仓库中的编译问题。具体需要解决：如何设计Agent的决策流程使其能够在多步精确操作的编译修复场景中可靠工作，如何构建用于评估仓库级编译修复能力的基准测试，以及Flow-based Agent和Free-form Agent在编译修复任务中的优劣比较。

## 3. 核心方法概述 (含流程图)

CompileAgent由两个核心组件构成：CompileNavigator（编译导航器）负责探索仓库结构、识别构建系统和搜索配置；ErrorSolver（错误求解器）负责诊断编译错误并生成修复方案。两组件结合五种专用工具（文件搜索、构建执行、日志分析、代码编辑和依赖管理），以基于固定流（flow-based）的Agent组织方式与软件制品交互。与Free-form Agent（自由形式，让LLM完全自主决策）不同，flow-based策略预定义了诊断-修复-验证的工作流，约束Agent的行动空间。研究同时构建了CompileAgentBench，包含多个真实仓库的编译失败场景作为评测基准。

## 4. 实验框架与训练流程

实验使用CompileAgentBench中的真实仓库编译失败场景进行评估。Agent流程包括：初始化（读取仓库结构和构建说明）、编译尝试（执行构建命令）、错误诊断（解析编译错误日志）、修复生成（LLM根据错误生成修复方案）、验证（再次编译确认修复成功）。流程预定义了固定的诊断-修复-验证工作流。比较了Flow-based、Free-form（完全自主决策）和ReAct（推理-行动交替）三种Agent组织策略。

## 5. 奖励函数/关键公式 (无则说明)

本文不涉及强化学习。Agent的优化目标为编译成功率（最终是否成功完成编译）。Flow-based策略通过预定义工作流约束Agent行动空间以提升成功率。

## 6. 实验设置 (6.1-6.4)

### 6.1 基准与数据集

CompileAgentBench：包含多个真实开源仓库在不同环境下的编译失败场景。错误类型涵盖缺失依赖、构建配置错误、编译器版本不兼容、语法错误等。

### 6.2 基线方法

直接LLM生成（不经过Agent流程的零样本修复）、Free-form Agent（LLM完全自主决策）、ReAct Agent（推理-行动交替）。Flow-based Agent为本文提出方法。

### 6.3 评价指标

编译成功率（compilation success rate）。按错误类型（缺失依赖、配置错误、语法错误等）分层的修复成功率。不同Agent策略的比较。

### 6.4 实现细节

Agent工具集包括文件搜索、构建执行、日志分析、代码编辑和依赖管理。Flow-based策略预定义诊断-修复-验证工作流。基座LLM使用通用代码模型。

## 7. 实验结果与结论

编译成功率从基线10%大幅提升至71%。Flow-based Agent策略显著优于Free-form和ReAct模式，说明在仓库级编译修复这种需要多步精确操作的场景中，预定义的固定流程比让LLM完全自主决策更可靠。错误类型分析表明缺失依赖和错误的构建配置是最常见的两类编译失败原因，而语法错误相对容易修复。固定流程和专用工具能有效约束Agent行动空间。

## 8. 主要创新点

核心创新在于将Flow-based（基于固定流程）Agent设计引入编译修复任务，证明了在需要多步精确操作的编译场景中约束Agent行动空间比让LLM完全自主决策更有效。第二个贡献是构建了CompileAgentBench基准，为仓库级编译修复研究提供了标准化的评估平台。五种专用工具的设计（文件搜索、构建执行、日志分析、代码编辑、依赖管理）覆盖了编译修复的核心操作。CompileNavigator和ErrorSolver两个组件的功能分离增强了系统的模块化。

## 9. 局限性

使用的工具仍然较为基础，对复杂的构建系统（如CMake的高级特性、Bazel的多语言构建）支持有限。系统对提示词的敏感度较高——同一问题用不同措辞描述可能导致不同的修复策略选择。存在重复尝试相同失败动作的风险。编译成功不等于测试通过或性能改善——能够编译的代码仍然可能在测试中失败或有功能缺陷。71%的成功率意味着仍有近三分之一的场景无法自动修复。

## 10. 阅读后的研究方向反思

CompileAgent可作为CABLE系统的自动构建前端参考——在进行任何编译优化之前首先需要确保目标仓库能够在测试环境下成功编译。流式Agent比起自由形式的Agent更适合需要精确、可重复行为的编译场景。但CABLE不应将CompileAgent式的仓库级编译修复作为核心贡献，而应将其定位为自动化的基础设施层。CABLE的核心创新应在优化效应的知识表示和适用边界的系统化，而非构建环境的自动化。

## 11. 可进一步尝试的研究方向

将CompileAgent的Flow-based设计扩展到编译优化场景——在编译优化中设计类似的诊断-生成-验证工作流。结合更丰富的工具集（如与LLVM Pass管理器、性能分析工具的接口）。研究Agent在处理不同编译器和构建系统时的泛化能力。构建包含更多语言和编译器的扩展版本CompileAgentBench。

## 12. 与其他已读文献的关系

本文与第61篇COCOGEN在利用编译反馈修复代码的目标上一致，但COCOGEN聚焦缺失项目上下文的补全，CompileAgent聚焦构建系统问题的修复。与第59篇Rectifier在错误修复的宽泛方向上相关，但Rectifier使用独立纠错模型而CompileAgent使用Agent工作流。与第69篇AIOS在Agent系统的设计层面相关——都涉及LLM Agent的决策流程设计，但CompileAgent的flow-based设计与AIOS的自由解释形成对比。与第42篇CompilerGPT在研究目标上有交集——都涉及LLM诊断和修复编译问题。

## 13. 一页式总结 (表格)

| 维度 | 描述 |
|------|------|
| 研究问题 | Flow-based Agent能否自动修复仓库级编译问题 |
| 核心方法 | CompileNavigator+ErrorSolver, 5种专用工具, 固定诊断-修复-验证流程 |
| 技术手段 | Flow-based Agent, CompileAgentBench基准, LLM+工具集集成 |
| 应用场景 | 开源仓库编译失败修复 |
| 关键指标 | 编译成功率从10%提升至71% |
| 主要创新 | Flow-based Agent约束行动空间优于Free-form, CompileAgentBench基准 |
| 核心局限 | 复杂构建系统支持有限, 提示敏感, 编译成功不等于测试通过 |
| 与CABLE关系 | 可作为自动构建前端参考, 定位为基础设施层而非核心创新 |
