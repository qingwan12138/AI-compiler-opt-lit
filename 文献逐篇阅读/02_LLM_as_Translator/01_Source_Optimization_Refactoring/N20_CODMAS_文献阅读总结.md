# CODMAS 文献阅读总结
代码/数据/工件：尚未找到可确认的作者代码仓库或工件（本轮复核，状态保留“待核验”）

论文题目：**CODMAS: A Dialectic Multi-Agent Collaborative Framework for Structured RTL Optimization**

作者：Che-Ming Chang、Prashanth Vijayaraghavan、Ashutosh Jadhav、Charles Mackin、Vandana Mukherjee、Hsinyu Tsai、Ehsan Degan

发表时间：2026

发表平台：EACL 2026 Industry Track
元数据核验来源：[ACL Anthology EACL Industry Track](https://aclanthology.org/2026.eacl-industry.57/)

关键词：RTL 优化、多智能体、PPA、Verilog、Yosys

> 本文档基于 PDF 全文整理。

## 1. 研究背景

LLM 修改 RTL 时既要保持时序功能，又要改善功耗、性能、面积（PPA）。普通提示容易遗漏流水延迟、握手和时钟门控前提，且既有数据集往往只有功能题，缺少“未优化—优化—测试台”三元组。

## 2. 论文要解决的问题

如何用职责分离的辩证 Agent 显式提出变换计划、预测结果并根据综合偏差修正，再由确定性工具验证语法、功能和 PPA。

## 3. 核心方法概述

Articulator 逐步说明优化意图和假设；Hypothesis Partner 预测功能/PPA 后果并解释实际偏差；DCA 生成架构感知 Verilog；CEA 使用 Icarus Verilog、测试台和 Yosys 检查并反馈。

## 4. 实验框架与训练流程

```text
未优化 RTL → Articulator 计划 ↔ Hypothesis Partner 质询
 → DCA 修改 → Icarus/测试台/Yosys → CEA 结构化反馈
 → 更新假设并迭代 → 通过功能且改善 PPA 的 RTL
```

无需模型训练或 RL。

## 5. 奖励函数、损失函数或关键公式

无训练损失。面积报告 `A/A0`；功耗改善 `(P0-P)/P0`；时序改善按关键路径延迟降低；FR 为语法、功能、综合或目标约束失败比例。

## 6. 实验设置

### 6.1 数据集来源

作者构建 RTLOPT：120 个 Verilog 三元组，当前覆盖流水线和时钟门控，并含可综合 RTL、测试台和目标优化版本。

### 6.2 模型与工具

GPT-4o、GPT-3.5-Turbo、Llama-3、DeepSeek-V2.5、Granite-34B-Code、CodeLlama-34B；Icarus Verilog、Yosys、Liberty 标准单元库。64 核 CPU+A100，5 个随机种子。

### 6.3 对比方法

Zero-shot、普通 prompting/agentic、CoDes、ReAct、Reflexion、LLM-VeriPPA；并移除辩证 Agent、领域知识或 CEA 做消融。

### 6.4 评价指标

面积比、关键路径改善、功耗降低和失败率；配对 t 检验 `p<0.01`。

## 7. 实验结果与结论

GPT-4o+CODMAS 流水线关键路径降低 25.5%、面积比 0.960、FR 19.5%；时钟门控功耗降低 21.7%、FR 21.8%。总体流水线时序改善超过 20%，时钟门控平均功耗降低超过 19%，FR 多数低于 30%，而基线常超过 40%–50%。去掉辩证 Agent 后 GPT-4o 时序收益从 25.5% 降到 12.9%；将两个辩证角色合并后收益降至 19.2%、FR 升至 35.7%。

## 8. 主要创新点

### 8.1 计划与反事实预测分离

一个 Agent 负责可追踪计划，另一个负责预测和解释偏差。

### 8.2 确定性 CEA 闭环

用仿真、综合和 PPA 工具约束生成，而不是由 LLM 自评。

### 8.3 RTLOPT 指标特定数据集

同时提供功能可验证性和明确 PPA 目标。

## 9. 局限性

论文只覆盖流水线和时钟门控，120 个规模较小；测试台可能漏掉稀有功能错误，Yosys 估计不等于完整工业物理实现。作者建议安全关键部署增加形式验证、审计和人工复核。

## 10. 阅读后的研究方向反思

CODMAS 说明多 Agent 只有在职责带来可测消融时才算贡献。它与软件编译优化距离较远，可借鉴“计划—预测—工具验收”结构，但不宜直接作为毕业论文主线，除非转向软硬协同编译。

## 11. 可进一步尝试的研究方向

### 11.1 编译器优化的辩证因果审计

#### 研究问题

让一个 Agent 预测某 pass/Hint 会改变哪些 IR/硬件事件，另一个用实测偏差定位错误假设，是否能减少无效搜索。

#### 与原论文的区别

从 RTL PPA 转为软件编译过程和真机性能因果链。

#### 可能的创新点

预测—实测残差驱动的优化策略修正。

#### 实验框架

```text
策略计划 → 预测 IR/计数器 → 编译实测 → 偏差归因 → 下一候选
```

#### 可行性与风险

perf/remark 可提供证据；因果归因仍可能不唯一。

## 12. 与其他已读文献的关系

与 N10 GraphGlue 都是职责分离多 Agent+确定性工具；与 LLM-VeriPPA 直接对比并显著降低失败。与 N25 CoV 的共同点是多层验证，但 CODMAS 仍以仿真/测试为主。

## 13. 一页式总结

| 项目 | 内容 |
|---|---|
| 论文研究任务 | LLM 自动优化 RTL PPA |
| 核心问题 | 变换需兼顾功能和 PPA |
| 输入/输出 | 未优化 Verilog / 流水或门控 RTL |
| 核心方法 | 辩证双 Agent + DCA + CEA |
| 使用的模型 | GPT、Llama、DeepSeek、Granite |
| 使用的编译器工具 | Icarus Verilog、Yosys |
| 是否使用强化学习 | 否 |
| 是否使用形式化验证 | 否，使用仿真/综合 |
| 数据集规模 | RTLOPT 120 三元组 |
| 主要指标 | Area、Timing、Power、FR |
| 最重要实验结果 | GPT-4o 时序 +25.5%、功耗 +21.7% |
| 核心创新 | 计划与预测辩证分工及确定性验收 |
| 主要局限 | 仅两类 RTL 优化、测试台不完备 |
| 与 RISC-V 研究的相关性 | 低至中；适合软硬协同而非纯编译 |
| 最适合作为 | 多 Agent 职责设计参考 |

> 借鉴价值在可消融的角色分工；不能把增加 Agent 数量本身当成创新。
