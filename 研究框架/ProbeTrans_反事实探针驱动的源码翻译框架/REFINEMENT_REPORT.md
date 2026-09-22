# ProbeTrans 实现前终审报告

**日期**：2026-09-22
**总体裁决**：`READY_FOR_M0`
**状态边界**：proposal/design 已完成到可实现规范；implementation 未开始；experiments 未运行。

## 本轮修复结果

| 阻断项 | 终审处理 | 验证位置 |
|---|---|---|
| 项目名/状态误导 | 正文统一为 LLVM IR 翻译；历史目录名仅作路径兼容；状态三分 | README、REFINE_STATE |
| Probe 不是有限动作 | instance schema + realization class；P0 三族 | P0 spec §2 |
| 16 次预算超限 | baseline-inclusive 11+3+2 确定性算法 | P0 spec §3 |
| D0 循环标签 | 全 pair 保留；OUT_OF_REGISTRY；独立 ground truth | P0 spec §4 |
| Baseline 不公平 | generation matched + query-matched + total cost | P0 spec §10 |
| Translator 越界 | replacement function；module/cross-function 退出 | P0 spec §1/6 |
| Loop identity 脆弱 | 一对多 lineage 与 AMBIGUOUS 状态 | P0 spec §5 |
| 语义证据混报 | source→target refinement；四类结果；strict 主表 | P0 spec §7 |
| VAG 定义混杂 | 主 off 只移除目标 LoopVectorize | P0 spec §9 |
| Target 主张过宽 | target-aware、无手写 ISA intrinsic | P0 spec §12 |
| P0 顺序不合理 | 新增 M2a/M2b，M2b 前禁止 LLM | P0 spec §11 |

## Readiness 判定依据

另一位工程师无需猜测以下关键协议：probe/certificate/lineage/gate schema；query 计费；search 终态；realization class；D0 标签来源；baseline 成本；replacement-function 边界；Alive2 方向；VAG off 定义；run ID/artifact。

## 尚未解决

设计无法替代真实证据。Registry coverage、安全可实现率、Alive2 solved rate、LLM 对 Template 的增量、真实性能和外部迁移均未知。若 M2b 失败，应停止并修正诊断层，不能通过增加 LLM、RAG、RL、Agent 或搜索预算绕过。

## 下一 gate

只执行 M0：环境锁、verifier/Alive2/lineage/effect fixtures 和 `M0_DECISION.json`。没有任何结果时，tracker 继续保持 TODO/BLOCKED。

本报告不使用主观分数；是否前进完全由 readiness checklist 与 milestone artifacts 决定。
