# SalvageIR：失败 IR 优化候选的验证式部分回收

> 当前版本：salvageir_v3，2026-09-21。状态：研究设计与执行交接；尚未证明自然可回收性或算法优势。

一句话：LLM 改写 LLVM IR 后，如果整体被验证器明确反驳，尝试只保留其中一部分改动；只有完整验证通过且统一 Oz 后仍有代码尺寸收益，才接纳。

## 现在做什么

在 AutoDL 上只复制本文件夹即可，不要求本地 Windows 路径、ARIS 仓库或某个插件。
先读 [执行交接](CODEX_P0A_EXECUTION_HANDOFF.md)，再读它列出的三份规范，开始 P0a。
已有 32 条/2 条的说法仅是用户报告，原始记录尚未在此核验；先审计，不追加大规模生成。

给 Codex 的最短提示词：

> 阅读当前目录的 README.md、CODEX_P0A_EXECUTION_HANDOFF.md、RESEARCH_CONTRACT.md、PILOT_PROTOCOL.md 和 ALGORITHM_SPEC.md。按 v3 执行 P0a，优先复核已有 32 条结果及其中明确 REFUTED 的候选；实现并运行最小审计、子集枚举和统一 Oz 成本测量。保留旧代码和日志，不自动启动 P0b、大型下载或付费任务。不要只停在计划；缺少数据时先完成能做的本地测试，再报告具体缺件。

## 主线与边界

- 角色：Translator / T2，LLM 输出程序实例；内部搜索不是 LLM pass Selector。
- 核心：一次既有 LLM 输出，结构编辑重放，整函数 LLVM refinement 验证；不再次调用 LLM。
- 本研究不训练模型、不做 RL。上游通用模型的对齐史单列披露。
- 主输入：PRE_SSA，未做通用优化、仅进行明确记录的 mem2reg 与非语义规范化；不简称原始 O0。
- 主比较：Cost(Oz(R)) 对 Cost(Oz(S))；POST_OZ 为配对困难设置，旧样本为 LEGACY。
- 主目标：固定 x86-64 generic 的函数机器码字节；不是运行加速。
- RISC-V：方法冻结后的独立外部验证，不参与主方法调试。

## 文档地图与唯一权威

| 文件 | 职责 |
|---|---|
| [研究契约](RESEARCH_CONTRACT.md) | 对象、成功定义、范围与禁止偷换事项 |
| [总方案](FINAL_PROPOSAL.md) | 研究问题、机制与贡献边界 |
| [算法规格](ALGORITHM_SPEC.md) | 编辑表示、硬约束/风险关联、搜索及验证不变量 |
| [预实验协议](PILOT_PROTOCOL.md) | 输入、成本、样本、预算、oracle 与阶段执行的唯一详细权威 |
| [参数文件](configs/pilot_v3.json) | P0 默认数值；缺少版本锁不允许生成 |
| [公平性协议](DATA_AND_FAIRNESS_PROTOCOL.md) | 数据来源、模型、分母、日志契约 |
| [裁决卡](GATE_CARD.md) | 协议摘要，不得覆盖协议 |
| [执行交接](CODEX_P0A_EXECUTION_HANDOFF.md) | AutoDL 独立目录、P0a任务和验收 |
| [正式实验计划](EXPERIMENT_PLAN.md) | P1/P2 的基线、统计与外部验证；不是启动授权 |
| [证据矩阵](EVIDENCE_MATRIX.md) | 原文依据与近邻差异 |
| [风险审查](THREATS_AND_REVIEW.md) | 否定条件与降级 |
| [v3 修订审计](REVISION_V3.md) | 本轮改变、审查结果与未解决事项 |
| [旧版归档](archive/v2/README.md) | v2 历史原文，不得作为当前执行指令 |

冲突顺序：研究契约 → 预实验协议 → 参数文件 → 专题规格 → 概述/交接。
数值或公式冲突应停止受影响运行并记录偏差，不能静默选择某份文件。
协议、参数和实现全部计算哈希写入每个 run；查看结果后改变规则必须新建版本与开发批次。

## 完成意味着什么

v3 旨在达到“可以实施并检验”的设计标准，不代表实现、实验或论文已完成。
没有真实日志不宣布 P0 通过；没有强基线证据不宣布方法创新成立。
框架文档检查、LLVM/Alive2 集成测试、自然样本实验是三个不同的验收层次。
