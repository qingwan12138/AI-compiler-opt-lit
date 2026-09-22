# ProbeTrans P0 预实验协议

**协议版本**：P0-v3
**目的**：实现前注册的工程 kill-gate；不产生论文级结论
**当前状态**：未实现、未运行
**唯一 schema/算法来源**：[P0_IMPLEMENTATION_SPEC.md](P0_IMPLEMENTATION_SPEC.md)

## 1. 服务器目录与可移植性

用户只需把 ProbeTrans 文件夹复制到 AutoDL。所有路径必须相对项目根，第三方仓库固定 commit，secrets 只读环境变量。

```text
ProbeTrans/
  README.md
  P0_IMPLEMENTATION_SPEC.md
  configs/{p0.yaml,toolchain.lock.yaml,models.yaml}
  third_party/{IR-OptSet,llvm-test-suite,autovec-benchmark,alive2}
  probetrans/{capture,probes,certificates,translator,audit,gates,benchmarks}
  scripts/  tests/  data/{raw,registry,snapshots}/
  artifacts/p0/  runs/
```

环境锁必须记录 OS、CPU/GPU、LLVM/Clang/Alive2 commit、IR-OptSet 与 benchmark commit、Python、target triple、data layout、CPU features、pipeline hash。P0 固定一个 LLVM 19.1.x 工具链；若实际基座要求不同版本，先记录不兼容并冻结统一版本，不得混用。

## 2. M0：环境与 gate fixtures

**输入**：冻结依赖候选、人工构造的 valid/invalid/equivalent/non-equivalent/unsupported IR fixtures、seeded lineage/effect controls。
**输出**：`environment.lock.json`、verifier/Alive2/detector JSONL、`M0_DECISION.json`。
**失败状态**：工具版本不一致、verifier 误判、Alive2 四态无法区分、detector 阳/阴性不敏感。

通过条件：工具可调用；verifier fixtures 全部符合预期；Alive2 能区分 proved/disproved/timeout-or-unsupported；lineage/effect seeded controls 全部符合预期。LLM endpoint 不属于 M0，不在此时部署。

## 3. M1：canonical capture、lineage、effect detector

从 source 产生 raw IR，运行统一 canonical prefix，保存 pre-LV snapshot；不得声称这是 `default<O3>` 的精确中间态。对 12 个预注册 sanity loops 做三次 clean build。

**输出**：每次 capture、pass trace、pipeline hash、original loop ID、一对多 lineage、remarks、post-LV structure、round-trip oracle。
**通过条件**：baseline decision 三次一致；remark 与 target-lineage structural detector 一致；重新序列化/原样 splice 不改变 oracle；无 verifier error；lineage 无歧义。
**失败状态**：`BASELINE_NOT_ELIGIBLE`、`AMBIGUOUS_LOOP_LINEAGE`、`PIPELINE_DRIFT`、effect mismatch。

Loop metadata、源码 span、header 和 depth 只是提示；匹配规则与 G4 见实现规范。

## 4. M2a：probe registry 与 bounded search

只实现 profitability、alias/dependence、trip-count 的冻结 instance；不得逐案例新增 probe。每个 instance 必须满足 schema、hash、applicability、semantic requirement、realization class 和冲突/依赖检查。

搜索单元测试必须覆盖：baseline 计费、cache hit 不计费、timeout/crash 计费、不适用 probe 不计费、11+3+2 最坏路径、大小>3不搜索、删除预算不足返回 non-minimal、两次 replay 禁用缓存，以及五个 certificate 终态。

**通过条件**：任何执行路径 charged queries ≤16；只有完整 deletion checks 可输出 `inclusion-minimal`；日志可重建每次查询。
**输出**：`probes.jsonl`、registry hash、search tests、status tests。

## 5. M2b：6–12 loop vertical slice

在冻结 registry 前选定 6–12 个不同 blocker/结构的 loop；选择规则和 case hash 先写入 manifest。运行完整 capture → applicability → search → replay → realizability → lineage/effect，但不接 LLM。

必须全部满足：

1. baseline decision 三次 clean run 一致；
2. remark 与 IR effect detector 一致；
3. 每例 charged query 从未超过 16；
4. 所有 `CERTIFICATE_FOUND` clean replay 100%；
5. 每例有可审计终态，包括 zero-success；
6. 无逐案例人工修改 probe；
7. loop lineage 无歧义。

任何一项失败，`M2B_DECISION=FAIL`，停止在诊断层。**M2b PASS 前不得部署/调用 LLM，也不得运行 24-case E0。**

## 6. M3：完整 calibration 与 D0

### 6.1 Pair 纳入

保存所有 hidden 稳定 missed、exposed 稳定 vectorized、oracle equivalent、三次可复现且 sanitizer 未显示已知问题的 autovec pair。registry 不覆盖时标 `OUT_OF_REGISTRY`，不删除。按 original kernel group 切分 dev/test；所有阈值只在 dev 冻结。

### 6.2 两项任务

- Decision flip：全部有效 pair；报告 registry coverage、搜索终态、clean replay、queries。
- Blocker classification：只用独立标签；标签来自原始 transformation、冻结规则或对 probe 结果盲化的双人标注。

输出 coverage、conditional accuracy、overall accuracy；OUT_OF_REGISTRY 留在 overall 分母。不得用搜索结果作标签。

### 6.3 D0 gate

以下均是预注册工程阈值，不是实际结果：test 有效 pair ≥24；replay precision=100%；P90 charged queries≤16；coverage≥30%；overall blocker accuracy 对最强 remark baseline 的差值为正。独立标签不足 24 时输出 `D0_LABEL_INSUFFICIENT`，分类 F1 不作为 Go 依据。

## 7. M4：splicer、SOC、Template

构建 TSVC eligible registry 与 excluded registry，然后实现同名同签名 function splicer、SOC 静态审计、允许的 runtime versioning 和 Certificate→Template。

P0 runtime guard 只允许函数内可安全计算的 trip/pointer-range 条件；fast path 与保留原语义的 fallback 在函数内汇合。EH/invoke、convergent、deopt、不可建模副作用、irreducible CFG、跨函数或 module 改动直接排除。

**输入**：冻结 TSVC cases、found certificates、semantic fixtures。
**输出**：splicer round-trip、SOC seeded violations、Template candidates、四态语义结果。
**通过条件**：原样 splice oracle 不变；所有 seeded semantic strengthening 被捕获；module 未变；Template 全部保留失败记录。

## 8. M5：LLM systems 与公平性

仅在 M2b、M3、M4 通过后冻结一个可在 4090 运行的 model ID/revision/quantization。P0 不训练。每个 LLM 系统匹配 model、input/output 上限、K=3、repair≤1 和 seed 策略。

系统：Direct、Remark-only、Iterative Remark/Analysis（query-matched）、Template、ProbeTrans。Natural-cost 与 query-matched 分开报告。Template/ProbeTrans 共享 certificate artifact，同时列 amortized/non-amortized 成本。

逐 case 记录 model calls、generated tokens、compiler queries、cache hits、verifier/Alive2 calls、wall time 和失败。Compiler query 不等价时，不声称等成本优越，只检验额外 certificate 信息的效用。

## 9. Candidate gates

| Gate | 输入 | PASS | 明确失败/未决状态 | Artifact |
|---|---|---|---|---|
| G0 splice | replacement function + module | 同名同签名且仅函数体替换 | FORMAT/SIGNATURE/MODULE_CHANGE | spliced IR + diff |
| G1 verify | spliced IR | assemble + verifier pass | PARSE/VERIFY_FAIL | logs |
| G2 SOC | original/candidate | 无未授权强化 | SEMANTIC_STRENGTHENING | audit JSON |
| G3 semantics | original source/candidate target | FORMALLY_PROVED；补充口径可 differential-only | DISPROVED/TIMEOUT/UNSUPPORTED/INTERNAL_ERROR/DIFF_FAIL | Alive2 + tests |
| G4 effect | lineage + post-LV + remarks | 目标 fast-path vector body | AMBIGUOUS/TARGET_EFFECT_NOT_FOUND | lineage/effect JSON |
| G5 performance | strict survivors | noise threshold 与 CI 通过 | NOISY/NO_SPEEDUP | raw timings |
| G6 VAG | four cells | interaction 方向/CI 通过 | PIPELINE_DRIFT/NOT_ATTRIBUTED | manifests + stats |

G3 的 differential matrix 包括 overlap、misalignment、0/small/odd/max trip、overflow、poison/undef、null/object-size/dereferenceability、guard true/false 和 fast/fallback。浮点默认 bitwise/IEEE；只有原 IR 明确许可时才改变比较规则。固定随机数只是压力测试，不是证明。

Strict CVUR/APIR 只接受 `FORMALLY_PROVED`；`DIFFERENTIAL_ONLY_SUPPORTED` 单列并做敏感性分析。

## 10. M6：性能与 VAG

先冻结 CPU noise threshold。若环境不稳定，M6 可输出 `PERFORMANCE_ENV_BLOCKED`，不得把 CVUR 当 APIR。

主 VAG 四格只改变目标 LoopVectorize：`O_on/O_off/E_on/E_off`；SLP、其他 pass、PGO、features、codegen 完全相同。全局关闭 SLP 仅作敏感性分析。VAG 提供机制归因证据，不写成完整因果证明。

建议计时计划（均为计划值）：5 次 warm-up、至少 30 个随机交错有效样本，报告 median、MAD、bootstrap CI 和全部 raw timings。

## 11. P0 判定

`P0_DECISION.md` 只能选择一个：`GO`、`GO_MECHANISM_ONLY`、`NO_GO_DIAGNOSIS`、`NO_GO_TRANSLATOR`、`NO_GO_SAFETY`、`NO_GO_EFFECT`、`PERFORMANCE_ENV_BLOCKED`。

P0 的具体成功阈值必须在 M2b 前作为 config hash 冻结；当前文档里的 24 cases、差值和数量均为计划/gate，不是结果。零成功、超时和崩溃保留在分母。

## 12. P0 后工作

只有 P0 Go 类裁决后，才运行全量 held-out TSVC、PolyBench/C/应用、第二模型、LLVM 版本稳健性、IR-OptSet 静态压力和冻结 IR 的 RV64GCV 外部验证。QEMU 只作 correctness，不作性能。

## 13. 必备 artifacts

```text
artifacts/p0/
  environment.lock.json  toolchain.lock.yaml
  registry/{probes.jsonl,cases.jsonl,excluded_cases.jsonl}
  m0/ m1/ m2a/ m2b/ m3/ m4/ m5/ m6/
  certificates/ prompts/ responses/ candidates/
  gate_results.jsonl costs.csv timing_raw/
  failure_taxonomy.md P0_DECISION.md
```

所有 gate result 使用实现规范中的 schema，带 input hashes、状态、reason code、artifact path、toolchain fingerprint 和 duration。缺 artifact 的 run 不能标 PASS。
