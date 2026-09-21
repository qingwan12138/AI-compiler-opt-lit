# ProbeTrans IR-v2 Refinement Report

**日期**：2026-09-22
**最终裁决**：CONDITIONAL_GO
**最终评分**：8.8/10

## Score Evolution

| Round | 版本 | Score | Blocking issue |
|---|---|---:|---|
| 0 | source-level ProbeTrans | 6.0 | 抽象层错误 |
| 1 | IR-level concept | 7.1 | 无代码基座、输入 IR 不明确 |
| 2 | IR-OptSet-based | 7.9 | benchmark 与 probe safety 不清 |
| 3 | certificate+SOC | 8.4 | LLM 必要性和归因仍弱 |
| 4 | benchmark-frozen | 8.7 | 预实验执行细节不足 |
| 5 | execution-ready P0 | **8.8** | 剩余差距只能由实验证据补齐 |

## Most Important Corrections

1. 将输出从可移植源码改回 target-independent LLVM IR replacement function。
2. 选择 IR-OptSet（NeurIPS 2025）为唯一开源代码基座。
3. 输入从含糊的 `O0/Oz` 改为 frozen pre-LoopVectorize canonical IR。
4. 将 benchmark 分为诊断、主性能、外部、静态压力和 RVV 五种不同职责。
5. 将 template baseline 提升为会杀死 LLM 论文叙事的强基线。
6. 为 AutoDL 共享 CPU 加入 `GO_MECHANISM_ONLY`，避免用噪声性能作错误结论。

## Remaining Weaknesses

- LLVM `default<O3>` 中精确 capture/resume 需要 PassBuilder 工程；
- alias/alignment 探针容易产生不可安全实现的证书；
- Alive2 对循环和 memory IR 的 timeout/unsupported 可能较高；
- 24-case P0 只能作机制 gate，不能提供论文级统计效力；
- 目前无跨模型独立审稿。

## Stop Reason

已经达到“可以动手证伪”的方案成熟度。继续加 RAG、多 Agent、SFT 或更多 benchmark 不会提高核心可信度。下一步只能执行 `PREEXPERIMENT_PROTOCOL.md`。
