# Experiment Tracker — ProbeTrans IR-v2

| Run ID | Milestone | Purpose | Variant | Data | Metric/Gate | Priority | Status |
|---|---|---|---|---|---|---|---|
| R001 | M0 | 环境探测 | doctor | fixtures | lock file complete | MUST | TODO |
| R002 | M0 | IR-OptSet smoke | upstream base | upstream tests | pass/fail | MUST | TODO |
| R003 | M0 | verifier 灵敏度 | valid/invalid fixtures | 10 IRs | 100% expected | MUST | TODO |
| R004 | M0 | Alive2 灵敏度 | eq/neq fixtures | 6 pairs | expected tri-state | MUST | TODO |
| R005 | M0 | LLM endpoint | deterministic request | 2 repeats | schema stability | MUST | TODO |
| R006 | M0 | CPU noise | same binary | sanity | MAD/median | MUST | TODO |
| R010 | M1 | pass trace | candidate prefix | 12 loops | trace artifact | MUST | TODO |
| R011 | M1 | loop identity | clean builds×3 | 12 loops | 100% stable | MUST | TODO |
| R012 | M1 | effect detector | remark+IR | 12 loops | 100% agreement | MUST | TODO |
| R013 | M1 | splice roundtrip | unchanged function | 12 loops | oracle pass | MUST | TODO |
| R014 | M1 | freeze toolchain | final prefix | all | lock committed | MUST | BLOCKED |
| R020 | M2 | build calibration | autovec hidden/exposed | all | valid pair count | MUST | BLOCKED |
| R021 | M2 | group split | calibration | valid pairs | no group leakage | MUST | BLOCKED |
| R022 | M2 | build TSVC registry | automatic selector | full TSVC | eligible/excluded | MUST | BLOCKED |
| R023 | M2 | freeze pilot | stratified held-out | ≤24 | registry hash | MUST | BLOCKED |
| R024 | M2 | UB/sanitizer audit | calibration+pilot | selected | clean/flagged | MUST | BLOCKED |
| R025 | M2 | profile hotness | baseline | pilot | loop share | MUST | BLOCKED |
| R026 | M2 | freeze thresholds | dev only | dev | config hash | MUST | BLOCKED |
| R030 | M3 | remark baseline | parser | calibration-test | exact/F1 | MUST | BLOCKED |
| R031 | M3 | precise remark | same LLM | calibration-test | exact/F1/tokens | MUST | BLOCKED |
| R032 | M3 | greedy probes | single probe | calibration-test | F1/queries | MUST | BLOCKED |
| R033 | M3 | certificate search | ICIT | calibration-test | F1/flip/queries | MUST | BLOCKED |
| R034 | M3 | clean replay | certificates | successes | flip precision | MUST | BLOCKED |
| R035 | M3 | realizability audit | SOC rules | certificates | three-state rate | MUST | BLOCKED |
| R036 | M3 | D0 decision | aggregate | calibration-test | GO/NO-GO | MUST | BLOCKED |
| R040 | M4 | Direct baseline | IR translator | TSVC pilot | funnel/CVUR | MUST | BLOCKED |
| R041 | M4 | Remark baseline | remark translator | TSVC pilot | funnel/CVUR | MUST | BLOCKED |
| R042 | M4 | Template baseline | shared certificate | TSVC pilot | funnel/CVUR | MUST | BLOCKED |
| R043 | M4 | ProbeTrans | ICIT+SOC | TSVC pilot | funnel/CVUR | MUST | BLOCKED |
| R044 | M4 | semantic gates | all candidates | TSVC pilot | tri-state outcomes | MUST | BLOCKED |
| R045 | M4 | effect gate | exact loop | survivors | unlock rate | MUST | BLOCKED |
| R046 | M4 | performance/VAG | four cells | survivors | APIR/interaction | MUST | BLOCKED |
| R047 | M4 | E0 decision | aggregate | TSVC pilot | decision | MUST | BLOCKED |
| R050 | M5 | non-minimal ablation | all probes | pilot | CVUR/cost | MUST | BLOCKED |
| R051 | M5 | w/o SOC audit | free realization | pilot | unsafe leakage | MUST | BLOCKED |
| R052 | M5 | w/o VAG | naive perf accept | pilot | false attribution | MUST | BLOCKED |
| R053 | M5 | template necessity | template vs LLM | pilot | paired difference | MUST | BLOCKED |
| R054 | M5 | failure taxonomy | all systems | pilot | counts/examples | MUST | BLOCKED |
| R055 | M5 | final P0 decision | all | P0 | one verdict | MUST | BLOCKED |
| R060 | M6 | full held-out TSVC | core systems | held-out | APIR/CVUR/CI | MUST | BLOCKED |
| R070 | M6 | external generalization | core systems | PolyBench/apps | APIR/CVUR | MUST | BLOCKED |
| R075 | M6 | second model | paired systems | main+external | paired delta | MUST | BLOCKED |
| R080 | M7 | static scale | ProbeTrans | IR-OptSet | coverage/cost | NICE | BLOCKED |
| R090 | M7 | RVV codegen | frozen IR | RV64GCV | compile/effect | MUST-LATE | BLOCKED |
| R091 | M7 | RVV correctness | frozen IR | QEMU/board | oracle | MUST-LATE | BLOCKED |
| R092 | M7 | RVV performance | frozen IR | native board | speedup/CI | NICE | BLOCKED |

## 状态转换

- 前置 milestone 未通过时，后续保持 `BLOCKED`。
- 每个 `DONE` 行必须在 notes/artifact 字段或外部 run manifest 中记录实际路径。
- `FAILED`、timeout 和 zero-success 不删除。
- 触发 kill criterion 后，将后续状态改为 `CANCELLED-KILL`。
