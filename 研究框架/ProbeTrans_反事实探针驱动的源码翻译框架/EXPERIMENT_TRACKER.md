# Experiment Tracker — ProbeTrans P0

> 尚无 run 被执行。状态只允许 `TODO/RUNNING/PASS/FAIL/BLOCKED/CANCELLED_KILL`；`PASS` 必须附实际 artifact。schema 与错误码见 [P0 实现规范](P0_IMPLEMENTATION_SPEC.md)。

| Run ID | Milestone | 输入 | 任务 | 输出 artifact | Gate/失败状态 | Status |
|---|---|---|---|---|---|---|
| R001 | M0 | host/toolchain | doctor + versions | `artifacts/p0/environment.lock.json` | lock 完整，否则 FAIL | TODO |
| R002 | M0 | valid/invalid IR fixtures | verifier sensitivity | `artifacts/p0/m0/verifier.jsonl` | 预期一致，否则 FAIL | BLOCKED |
| R003 | M0 | eq/neq/unsupported fixtures | Alive2 direction/status | `artifacts/p0/m0/alive2.jsonl` | 四态可区分，否则 FAIL | BLOCKED |
| R004 | M0 | seeded lineage/effect fixtures | detector sensitivity | `artifacts/p0/m0/detectors.jsonl` | 阳/阴性均正确，否则 FAIL | BLOCKED |
| R005 | M0 | M0 outputs | M0 decision | `artifacts/p0/M0_DECISION.json` | PASS/FAIL | BLOCKED |
| R010 | M1 | 12 sanity loops | canonical capture ×3 | `artifacts/p0/m1/captures.jsonl` | baseline decision 稳定 | BLOCKED |
| R011 | M1 | captures | loop identity/lineage | `artifacts/p0/m1/lineage.jsonl` | 无歧义，否则 AMBIGUOUS_LOOP_LINEAGE | BLOCKED |
| R012 | M1 | pre/post-LV + remarks | effect detector | `artifacts/p0/m1/effects.jsonl` | remark/effect 一致 | BLOCKED |
| R013 | M1 | prefix/suffix | pipeline drift check | `artifacts/p0/toolchain.lock.yaml` | hash 冻结，否则 PIPELINE_DRIFT | BLOCKED |
| R014 | M1 | M1 outputs | M1 decision | `artifacts/p0/M1_DECISION.json` | PASS/FAIL | BLOCKED |
| R020 | M2a | frozen schema | probe registry | `artifacts/p0/registry/probes.jsonl` | schema/hash/applicability valid | BLOCKED |
| R021 | M2a | registry + fixtures | bounded search unit tests | `artifacts/p0/m2a/search_tests.jsonl` | worst path ≤16 | BLOCKED |
| R022 | M2a | failures | status/error audit | `artifacts/p0/m2a/status_tests.jsonl` | 所有终态可达可审计 | BLOCKED |
| R030 | M2b | 6–12 frozen loops | vertical slice ×3 | `artifacts/p0/m2b/results.jsonl` | 全部 M2b 条件通过 | BLOCKED |
| R031 | M2b | query logs | budget/replay audit | `artifacts/p0/m2b/budget_audit.json` | ≤16；found cert replay 100% | BLOCKED |
| R032 | M2b | M2b outputs | M2b decision | `artifacts/p0/M2B_DECISION.json` | PASS 才允许任何 LLM 工作 | BLOCKED |
| R040 | M3 | all valid autovec pairs | D0 registry/split | `artifacts/p0/calibration_pairs.jsonl` | OUT_OF_REGISTRY 不删除 | BLOCKED |
| R041 | M3 | frozen label rules | independent labels | `artifacts/p0/d0_labels.jsonl` | 无 probe-result 泄漏 | BLOCKED |
| R042 | M3 | D0 test | remark baselines | `artifacts/p0/m3/remarks.jsonl` | coverage/conditional/overall | BLOCKED |
| R043 | M3 | D0 test | certificate search | `artifacts/p0/m3/certificates/` | replay/query/status | BLOCKED |
| R044 | M3 | D0 outputs | D0 decision | `artifacts/p0/D0_DECISION.json` | GO/NO_GO/LABEL_INSUFFICIENT | BLOCKED |
| R050 | M4 | frozen TSVC registry | splicer roundtrip | `artifacts/p0/m4/splicer.jsonl` | same signature/module intact | BLOCKED |
| R051 | M4 | semantic fixtures | SOC/audit/versioning | `artifacts/p0/m4/soc.jsonl` | seeded violations captured | BLOCKED |
| R052 | M4 | found certs | Template baseline | `artifacts/p0/m4/template.jsonl` | strict semantic/effect funnel | BLOCKED |
| R053 | M4 | M4 outputs | M4 decision | `artifacts/p0/M4_DECISION.json` | PASS/FAIL | BLOCKED |
| R060 | M5 | 24-case E0 registry | Direct/Remark/Iterative Remark | `artifacts/p0/m5/baselines.jsonl` | generation/query tracks logged | BLOCKED |
| R061 | M5 | same cases/certs | ProbeTrans LLM | `artifacts/p0/m5/probetrans.jsonl` | strict + differential-only funnel | BLOCKED |
| R062 | M5 | all candidates | cost/fairness report | `artifacts/p0/m5/costs.csv` | all required cost fields | BLOCKED |
| R063 | M5 | M5 outputs | translator decision | `artifacts/p0/M5_DECISION.json` | GO/NO_GO | BLOCKED |
| R070 | M6 | strict semantic survivors | x86 timing + VAG | `artifacts/p0/m6/vag.jsonl` | target-LV-only four cells | BLOCKED |
| R071 | M6 | P0 artifacts | final P0 decision | `artifacts/p0/P0_DECISION.md` | one registered verdict | BLOCKED |

M2b 通过前，R040 以后全部保持 `BLOCKED`，且不得部署或调用 LLM。外部集、第二模型、IR-OptSet 静态规模和 RV64GCV 属于 P0 之后，不占用本 tracker 的 M0–M6 run ID。
