# ProbeTrans：反事实探针驱动的 LLVM IR 翻译框架

> 当前裁决：`READY_FOR_M0`。研究设计已通过实现前终审；实现尚未开始，实验尚未运行。`conditional_go` 只授权进入 P0 的 M0，不代表方法有效。

## 五分钟理解

ProbeTrans 是一个 **pre-LoopVectorize LLVM IR → LLVM IR Translator**。它先用有限、诊断专用的反事实 probe instance 测试哪些改变能让 LLVM 的目标循环从 missed 变为 vectorized，再把预算内、可重放、包含最小的 intervention certificate 提供给 LLM。LLM 只能返回一个同名同签名 replacement function；候选随后经过 Semantic-Obligation Contract（SOC）、语义验证、目标循环 effect 检测和 Vectorization Attribution Gate（VAG）。

本项目不是 C/C++ 重写系统，也不输出显式 SIMD intrinsic。顶层目录中的“源码翻译”是早期历史命名，为避免破坏外部路径暂不重命名；正文与实际研究对象统一为“LLVM IR 翻译框架”。

```text
C/C++ benchmark → frozen canonical pre-LV IR
→ bounded counterfactual probe search（≤16 queries）
→ intervention certificate
→ function-level LLVM IR translation
→ verifier / SOC / semantic validation / target-loop effect
→ x86 performance + VAG；冻结 IR 再做 RV64GCV 外部验证
```

## 当前完成到哪里

- 已完成：研究问题、能力边界、P0 schema、确定性查询算法、D0 标签协议、baseline 公平性、loop lineage、验证和 VAG 协议。
- 未开始：`probetrans/` 实现、benchmark registry、模型部署和任何实验。
- 当前没有实验结果；文档数字均为预算、计划样本量或预注册阈值。

## 第一项工程任务

从 [P0 实现规范](P0_IMPLEMENTATION_SPEC.md) 的 M0 开始：建立环境锁文件和 gate fixtures；随后完成 M1 的 canonical pre-LV capture、loop lineage 与 effect detector。不得先部署 LLM。

M2b vertical slice 通过前，禁止接 LLM、禁止运行 24-case E0。M2b 要求 6–12 个预注册 loop 的三次 baseline decision 一致、remark/effect 一致、每例查询不超过 16、found certificate 100% clean replay、终态可审计、无逐例手改 probe、lineage 无歧义。

## 核心边界

- 表示：**target-aware but ISA-intrinsic-free LLVM IR**；保存 triple、data layout 和 CPU features。
- P0 probe：profitability、alias/dependence、trip-count。alignment 延后或严格限域；control/call 不进入 P0。
- 输出：一个 replacement function；不得改 signature、callee、declaration、global 或 module metadata，不得新增 helper。
- Profitability probe 是 `DECISION_ONLY`，不能被解释为缺少语义事实。
- Alive2 未解决但差分测试通过只记 `DIFFERENTIAL_ONLY_SUPPORTED`，不是形式证明。
- VAG 主协议只开关目标 LoopVectorize；SLP 不变。

## 基座与 benchmark

- 代码基座：[IR-OptSet](https://github.com/yilingqinghan/IR-OptSet)（MIT）；复用 IR extraction/preprocessing、LLVM wrapper、verify、Alive2 adapter、静态分析、模型接口和日志，ProbeTrans 自行实现向量化专用模块。
- D0 校准：现代 LLVM 重认证的 `autovec-benchmark`，保留 OUT_OF_REGISTRY pair。
- 主评测：按 kernel family 隔离的 held-out TSVC eligible missed loops。
- 外部：PolyBench/C 与少量公开应用热点。
- 静态压力：IR-OptSet 子集，不冒充 runtime benchmark。
- RV64GCV：只接收冻结 IR；不重新 prompt，不承诺同一决策或普遍加速。

## 文档入口

1. [研究契约](RESEARCH_CONTRACT.md)
2. [P0 实现规范](P0_IMPLEMENTATION_SPEC.md)——schema 与算法的唯一规范
3. [完整预实验协议](PREEXPERIMENT_PROTOCOL.md)
4. [最终方法提案](FINAL_PROPOSAL.md)
5. [正式实验计划](EXPERIMENT_PLAN.md)
6. [实验追踪表](EXPERIMENT_TRACKER.md)
7. [实现前终审变更记录](REFINEMENT_CHANGELOG.md)
8. [证据矩阵](EVIDENCE_MATRIX.md)

## 停止条件

出现任一情况即停止或降级：M0/M1 gate 不可信；M2b 不能在 16 次内稳定 replay；D0 的证书对最强 remark/query-matched baseline 无信息增量；多数证书为 module-required/unrealizable；Template 追平 LLM；strict semantic 口径没有收益；VAG 显示表面加速主要不是目标 LoopVectorize 所致。
