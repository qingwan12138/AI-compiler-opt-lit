# ProbeTrans P0 实现规范

**规范版本**：P0-spec-v1
**冻结日期**：2026-09-22
**适用范围**：M0–M6；本文是 schema、搜索预算、loop lineage、gate 状态和错误码的唯一规范来源。
**当前事实状态**：仅完成设计；没有实现或实验结果。

## 1. 系统边界

- 输入：保留原始 target triple、data layout 和 CPU features 的 canonical pre-LoopVectorize LLVM IR。
- 输出：一个同名、同签名的 replacement function definition。
- 表示：**target-aware but ISA-intrinsic-free LLVM IR**；禁止手写 x86/RVV intrinsic。
- 允许改动：目标函数体内的 CFG、指令、局部 metadata，以及符合本规范的 runtime guard、fast path 和 fallback。
- 禁止改动：函数签名、callee/global/declaration、vector-function mapping、module flags、module metadata、新 helper function，以及需要跨函数证明的事实。
- 超出函数边界的诊断证书标为 `MODULE_CHANGE_REQUIRED`，不得进入当前 Translator 的安全可实现率分子或 LLM 输入。

## 2. Probe instance registry

P0 只启用 `PROFITABILITY`、`ALIAS_DEPENDENCE`、`TRIP_COUNT` 三族。`ALIGNMENT` 默认延后；只有能以函数内显式地址检查实现、且不加强原有 load/store alignment 时才可在后续注册。`CONTROL_CALL` 移出 P0。

每一个探针是冻结 registry 中的有限实例，不允许按案例手工发明。JSONL schema：

```json
{
  "probe_id": "alias.arg0_arg1.noalias_scope.v1",
  "family": "ALIAS_DEPENDENCE",
  "target_kind": "argument_pair",
  "target_identity": ["arg:0", "arg:1"],
  "exact_ir_edit": "registry/edits/alias_scope_v1.yaml",
  "parameter_value": {"scope": "pairwise", "direction": "symmetric"},
  "applicability_predicate": "both pointer args reach target loop memory ops",
  "expected_compiler_observation": "target loop changes MISSED to VECTORIZED",
  "semantic_requirement": "non-overlap for all bytes touched by vector fast path",
  "realization_class": "GUARD_REALIZABLE",
  "diagnostic_edit_hash": "sha256:...",
  "query_cost": 1,
  "conflicts": [],
  "dependencies": [],
  "enabled_in_p0": true
}
```

必填字段和约束：

| 字段 | 约束 |
|---|---|
| `probe_id` | registry 内唯一且版本化 |
| `family` | P0 三族之一 |
| `target_kind/target_identity` | 可由 pre-LV IR 确定性解析，不使用模糊源码文本 |
| `exact_ir_edit` | 指向机器可执行的声明式 edit；hash 后不可变 |
| `parameter_value` | 显式记录 VF、阈值、argument pair 等所有自由参数 |
| `applicability_predicate` | 查询前执行；不满足时不创建查询 |
| `expected_compiler_observation` | 固定为同一目标 lineage 的 decision/effect |
| `semantic_requirement` | 诊断 edit 假设了什么，不得留空 |
| `realization_class` | 下列五态之一 |
| `diagnostic_edit_hash` | edit 内容的 SHA-256 |
| `query_cost` | P0 固定为 1；cache hit 为 0 |
| `conflicts/dependencies` | 用 probe_id 明确组合合法性 |

`realization_class`：

- `DECISION_ONLY`：只区分 profitability/legality，不代表缺失语义事实，也不直接实现。
- `FUNCTION_REALIZABLE`：只改目标函数即可保持语义地实现。
- `GUARD_REALIZABLE`：需要函数内 runtime guard + 原循环 fallback。
- `MODULE_CHANGE_REQUIRED`：需要 declaration/callee/global/module 或跨函数事实。
- `UNREALIZABLE`：当前系统不能安全实现。

Profitability 探针（例如强制 vectorize metadata/VF）一律先标 `DECISION_ONLY`。它只能说明默认决策受成本/启发式影响，不能自动解释为缺少某个程序语义条件，也不能直接复制到最终候选。

## 3. Compiler query 与 16 次预算

### 3.1 一次查询

一次 query 是：对一个规范化 IR + 一个有序 probe set，在全新 `opt` 进程中运行冻结 pipeline，并在同一进程产物上同时收集 optimization record 与 post-LV structural effect。二者不是两次查询。

- baseline query：计入 16 次预算，并占 discovery 的第 1 次。
- cache hit：只有 toolchain fingerprint、input hash、ordered probe-set hash、pipeline hash、target feature hash 全部相同时才复用；计 `query_cost=0`，但记录 `cache_hit=true`。
- timeout/crash：计 1 次，状态分别为 `QUERY_TIMEOUT`/`QUERY_CRASH`；不得静默重试。
- 不适用探针：在查询前判定，不计费。
- clean replay：必须禁用查询缓存，固定占 2 次。

### 3.2 确定性预算分配

- discovery/search：最多 11 次，包含 baseline。
- deletion minimization：最多 3 次。
- clean-process replay：固定 2 次。
- 总计：最多 16 次。

Discovery 候选顺序在 registry freeze 时确定：先按族 `PROFITABILITY → ALIAS_DEPENDENCE → TRIP_COUNT`，族内按 `probe_id` 字典序；随后只枚举 registry 允许且依赖闭合的二元组合，再枚举三元组合。最大证书大小为 3；不搜索大小大于 3 的集合。

```text
search(case, ordered_instances):
  budget = {discover: 11, delete: 3, replay: 2}
  q0 = query(case, {})                         # discover 1/11
  if q0 is not stable MISSED: return BASELINE_NOT_ELIGIBLE

  applicable = filter_applicable(ordered_instances, case)
  if applicable is empty: return NO_APPLICABLE_PROBE

  winner = NONE
  for S in deterministic_sets(applicable, max_size=3):
    if discover_used == 11: break
    r = query(case, S)
    if r == TARGET_VECTORIZED:
      winner = S
      break
  if winner is NONE: return NO_CERT_WITHIN_BUDGET

  unchecked = []
  S = winner
  for p in deterministic_reverse_order(S):
    if delete_used == 3: unchecked.append(p); continue
    r = query(case, S - {p})
    if r == TARGET_VECTORIZED: S = S - {p}

  if unchecked is not empty:
    return NON_MINIMAL_CERT_WITHIN_BUDGET(S, unchecked)

  r1 = fresh_query_no_cache(case, S)
  r2 = fresh_query_no_cache(case, S)
  if r1 != TARGET_VECTORIZED or r2 != TARGET_VECTORIZED:
    return UNSTABLE_CERTIFICATE
  return CERTIFICATE_FOUND(S, minimality="inclusion-minimal")
```

由于最大集合大小为 3，完整删除检查最多 3 次。只有每个保留元素都完成删除检查，才能写 `inclusion-minimal=true`。任何 `NON_MINIMAL_CERT_WITHIN_BUDGET` 不进入 certificate-conditioned 系统，但保留作覆盖/失败统计。

终态：`NO_APPLICABLE_PROBE`、`NO_CERT_WITHIN_BUDGET`、`NON_MINIMAL_CERT_WITHIN_BUDGET`、`UNSTABLE_CERTIFICATE`、`CERTIFICATE_FOUND`；基础设施另有 `BASELINE_NOT_ELIGIBLE`、`QUERY_TIMEOUT`、`QUERY_CRASH`。

### 3.3 Certificate schema

```yaml
schema_version: p0-cert-v1
case_id: tsvc_s000_xxx
toolchain_fingerprint: sha256:...
pipeline_hash: sha256:...
registry_hash: sha256:...
original_loop_id: loop:...
baseline_query_id: q000
probe_ids: [alias.arg0_arg1.noalias_scope.v1]
search_status: CERTIFICATE_FOUND
minimality: inclusion-minimal
deletion_checks:
  - removed: alias.arg0_arg1.noalias_scope.v1
    observation: TARGET_MISSED
replay_query_ids: [q014, q015]
queries:
  discovery: 7
  deletion: 1
  replay: 2
  cache_hits: 0
  charged_total: 10
realization_class: GUARD_REALIZABLE
diagnostic_only: true
```

## 4. D0 calibration 与独立标签

D0 保存所有同时满足下列条件的 hidden/exposed pair：hidden 稳定 missed、exposed 稳定 vectorized、oracle equivalent、三次 clean build 可复现、sanitizer 未发现已知问题。**不得因为 registry 无法覆盖而删除。**

- `registry_coverage=IN_REGISTRY`：冻结 registry 至少有一个适用实例。
- `registry_coverage=OUT_OF_REGISTRY`：没有适用实例；仍保留在 D0 总分母。

D0 分成两个任务：

1. **Decision-flip task**：无需 blocker ground truth；报告 coverage、certificate found/replay、query cost 和终态。
2. **Blocker-classification task**：只对具有独立标签的 pair 计算分类指标。标签优先来自 benchmark 原始 hidden→exposed transformation；不能唯一映射时，由两位不知道 probe 搜索结果的标注者独立标注并仲裁；可机器推导的标签使用在搜索前冻结并 hash 的规则。`probe_set` 永远不能反充 ground truth。

必须分别报告：

- `coverage = IN_REGISTRY / all_valid_pairs`；
- `conditional_accuracy = correct / independently_labeled_IN_REGISTRY`；
- `overall_accuracy = correct / all_independently_labeled_pairs`，其中 OUT_OF_REGISTRY 计错；
- 未取得独立标签的样本数量，不混入 F1。

D0 进入下一阶段的工程 gate：有效 test pairs ≥24；clean replay precision 100%；P90 charged queries ≤16；registry coverage ≥30%；overall blocker accuracy 相对最强 remark baseline 的预注册差值为正。若独立标签不足 24，不以 F1 作 Go 条件，只使用 decision-flip/replay gate并标记 `D0_LABEL_INSUFFICIENT`。

## 5. Loop lineage

### 5.1 Schema

```json
{
  "original_loop_id": "sha256:...",
  "stage": "pre_lv|candidate_pre_lv|post_lv",
  "candidates": [
    {"loop_id":"...","role":"FAST_PATH|FALLBACK|VECTOR_BODY|EPILOGUE|OTHER",
     "score":0.0,"evidence":["memory-op fingerprint","SCEV relation"]}
  ],
  "mapping_cardinality": "1:1|1:N",
  "status": "RESOLVED|AMBIGUOUS_LOOP_LINEAGE"
}
```

原始 loop ID 由 function signature hash、canonical induction/SCEV 摘要、循环内 memory-access multiset、exit predicate 摘要、debug span（若有）和 CFG neighborhood hash 组成。`llvm.loop` metadata、block name、源码行、header hash 和 depth 只作追踪提示，不能作为唯一证据；LLVM 修改 loop metadata 时可能产生新 LoopID。

### 5.2 一对多匹配优先级

1. 显式插入的内部 lineage tag（仅用于跟踪，不能证明语义或向量化）；
2. memory-access multiset 与 induction/SCEV 映射；
3. exit/live-out 对应关系；
4. CFG neighborhood 与 debug span 作为 tie-breaker；
5. LLVM 产生的 `llvm.loop.vectorize.body` / `llvm.loop.vectorize.epilogue` 作为 post-LV 角色提示。

header、depth 或 block count 攻变不直接判失配。若最高分候选并列、关键 memory effects 无法对应，或 fast/fallback 角色无法区分，则返回 `AMBIGUOUS_LOOP_LINEAGE`，该案例不得计入 CVUR/APIR 分子。

G4 只在目标 `FAST_PATH` 的后继 lineage 出现成功 remark、vector-typed body/effect 和 `VECTOR_BODY` 角色时通过；其他循环向量化、fallback 或 epilogue 向量化均不算目标成功。

## 6. Function translator 与 runtime versioning

模型只输出 replacement function。P0 runtime versioning 规则：

- guard 位于目标循环的 preheader 之前，并支配 fast/fallback 分支；
- fast path 可以 clone/restructure 目标 loop；fallback 必须保留原循环的语义等价副本；
- 两条路径在函数内汇合，PHI/live-out 必须完整；
- 不允许新增 helper function或修改 signature/callee/module；
- guard 只检查函数内可计算的整数 trip count、pointer range non-overlap 等条件；检查本身不得越界解引用、产生 overflow/poison 或假设 object size；
- P0 排除异常边、invoke/EH、convergent call、deopt/guard intrinsic、不可建模 side effect、musttail、irreducible CFG，以及无法证明 termination/progress 条件的 `mustprogress` 情形；
- certificate 含上述情形或需要 module edit 时返回 `MODULE_CHANGE_REQUIRED` 或 `UNREALIZABLE`，不调用 LLM。

## 7. Semantic-Obligation Contract 与验证

验证方向固定为：`source = 原始函数`，`target = replacement function`；检查 target 是否 refinement source，即对 source 定义的行为，target 不得引入新 UB、poison 或可观察差异。Alive2 只验证函数内转换；涉及跨函数变换不进入 P0。

语义状态必须分开：

- `FORMALLY_PROVED`：Alive2 在冻结版本/选项下返回正确 refinement；
- `DISPROVED`：Alive2 给出反例，永久拒绝；
- `TIMEOUT`、`UNSUPPORTED`、`INTERNAL_ERROR`：均为 `UNRESOLVED`，不得合并为通过；
- `DIFFERENTIAL_ONLY_SUPPORTED`：Alive2 未解决，但预注册 differential/oracle 测试通过；不是证明。

Differential fixture 必须按适用性覆盖：pointer 完全重叠/部分重叠/相邻；misalignment offsets；trip count 0、1、小于 VF、等于 VF、奇数、接近允许上界；signed/unsigned overflow 边界；poison/undef 敏感模式；null、zero object、object-size、dereferenceability 边界；guard true/false 及边界两侧；fast/fallback 均至少命中。浮点默认 bitwise/IEEE 行为一致；只有原 IR 已有相同 fast-math permission 时才使用相应 NaN、signed-zero、ULP 策略，并逐例记录。

Sanitizer 仅用于可编译 benchmark harness，帮助发现内存/整数/未定义行为症状；它不覆盖所有输入，也不证明 LLVM refinement。固定 1,000 个随机输入只是计划的最低压力测试量，不是充分等价证据。

报告四类：`formally_proved`、`differential_only_supported`、`disproved`、`unresolved`。CVUR/APIR 主结果采用两层口径：strict 主表只接受 `FORMALLY_PROVED`；coverage 补充表允许 `DIFFERENTIAL_ONLY_SUPPORTED`，但单独标注，并做“排除 differential-only 后结论是否改变”的敏感性分析。

## 8. Gate result schema

```json
{
  "run_id": "R050",
  "case_id": "...",
  "candidate_id": "...",
  "gate": "G0|G1|G2|G3|G4|G5|G6",
  "input_hashes": {},
  "status": "PASS|FAIL|UNRESOLVED|SKIP_PREREQUISITE",
  "reason_code": "...",
  "artifact_paths": [],
  "toolchain_fingerprint": "sha256:...",
  "started_at": "...",
  "duration_ms": 0
}
```

每个 gate 都必须保存输入 hash、唯一状态、reason code 和 artifact。上游失败时只能 `SKIP_PREREQUISITE`，不得记通过。

## 9. VAG 唯一主协议（target-LV-only）

主分析只改变**目标 LoopVectorize 是否运行**；SLP 和其他 pass 保持原样。四格：

- `O_on`：原始函数 + 冻结 pipeline，目标 LoopVectorize 开启；
- `O_off`：原始函数 + 同一 pipeline，仅移除/禁用目标 LoopVectorize；
- `E_on`：改写函数 + 与 O_on 相同设置；
- `E_off`：改写函数 + 与 O_off 相同设置。

四格使用相同 target triple、CPU features、PGO/profile、链接、codegen flags 与 suffix pipeline；manifest 必须证明 pipeline hash 只在目标 LV 条件上有预期差异。全局同时关闭 LoopVectorize 和 SLP 只作敏感性分析，不是主 `off`。

令 `A = [log T(O_on)-log T(E_on)] - [log T(O_off)-log T(E_off)]`。`A>0` 表示开启目标 LV 时改写的相对收益更大。VAG 是机制归因证据，不是完整因果证明。

## 10. Baseline 与成本口径

所有 LLM 系统必须 generation-budget matched：相同模型、context/output 上限、候选数、repair 次数和 seed 策略。

Compiler-query 公平性采用两条轨：

1. **Natural-cost track**：Direct/Remark 不运行 probe；ProbeTrans/Template 共享一次 certificate artifact。只主张 certificate 信息的效用，不声称等总计算成本优越。
2. **Query-matched feedback track**：Iterative Remark/Analysis baseline 最多使用与该 case ProbeTrans 相同的 charged query 数；它可以重复运行未做语义干预的冻结 analysis/remark 配置并把新获得的标准 compiler analyses/remarks反馈给模型，但不得加入 probe edit。若重复查询没有新信息，仍如实计费，结果用于隔离“更多编译器交互”而非保证信息等价。

逐系统逐 case 报告：model calls、generated tokens、charged compiler queries、cache hits、verifier calls、Alive2 calls、wall time、峰值资源。Template 和 ProbeTrans 的共享 certificate 成本同时报告：`non_amortized`（各自承担全成本）与 `amortized`（artifact 在 N 个消费者间均摊）；主正确性比较不扣除成本，成本单独成表。

## 11. Milestone 状态机与错误码

执行顺序固定：

- `M0`：环境与 gate fixtures。
- `M1`：canonical pre-LV capture、loop identity/lineage、effect detector。
- `M2a`：probe-instance registry 与 bounded search 实现。
- `M2b`：6–12 loop vertical slice。
- `M3`：完整 calibration 与 D0。
- `M4`：function splicer、SOC、Template。
- `M5`：LLM systems。
- `M6`：performance/VAG。

状态：`TODO → RUNNING → PASS|FAIL|BLOCKED|CANCELLED_KILL`。只有实际 artifact 存在且 gate 通过才能写 `PASS`。

**硬规则**：M2b 通过前，不接 LLM、不部署模型、不运行 24-case E0。M2b 必须满足：6–12 个预注册 loop 的 baseline decision 三次 clean run 一致；remark 与 effect detector 一致；任何 case charged queries ≤16；所有 found certificate clean replay 100%；每个终态可由日志审计；没有逐案例人工 probe；loop lineage 无歧义。失败即停在诊断层。

通用错误码：`NO_APPLICABLE_PROBE`、`NO_CERT_WITHIN_BUDGET`、`NON_MINIMAL_CERT_WITHIN_BUDGET`、`UNSTABLE_CERTIFICATE`、`AMBIGUOUS_LOOP_LINEAGE`、`MODULE_CHANGE_REQUIRED`、`QUERY_TIMEOUT`、`QUERY_CRASH`、`ALIVE2_TIMEOUT`、`ALIVE2_UNSUPPORTED`、`ALIVE2_INTERNAL_ERROR`、`SEMANTIC_DISPROVED`、`DIFFERENTIAL_FAILURE`、`TARGET_EFFECT_NOT_FOUND`、`PIPELINE_DRIFT`。

## 12. Target 与跨 ISA 边界

x86 是证书生成和主评测环境。系统保存 triple、data layout 和 CPU features，因此不称 target-independent。最终 IR 是“portable LLVM IR without hand-written target intrinsics”，但向量化决策仍可依赖 target cost model。

RV64GCV 只接收 x86 阶段冻结、通过 gate 的相同 IR，不重新 prompt、不调整 certificate；分别报告 compile、correctness、target loop effect 和真机性能。不承诺不同 target 得到相同 vectorization decision；跨 ISA 结果只支持 portability/robustness，不支持普遍加速。

## 13. 官方事实核查记录

核查日期：2026-09-22。

- LLVM 官方文档说明 Loop Vectorizer 与 SLP 是不同 vectorizer；loop vectorizer 可产生 runtime checks、vector body 与 scalar epilogue，因此主 VAG 只开关目标 LoopVectorize，lineage 必须支持一对多。<https://llvm.org/docs/Vectorizers.html>
- LLVM transformation metadata 文档说明修改 loop metadata 会形成新的 LoopID，且向量化后可标注 vector body/epilogue；因此 metadata 只作 lineage 提示。<https://llvm.org/docs/TransformMetadata.html>
- Alive2 官方 README 将检查定义为 original/source 到 optimized/target 的 refinement，并明确不支持 inter-procedural transformations；因此 P0 限制为函数内 replacement。<https://github.com/AliveToolkit/alive2/blob/master/README.md>
- IR-OptSet 官方仓库提供 IR extraction/preprocessing、LLVM wrapper、`opt_verify.py`、`alive2.py`、`mca_cycles.py` 与 LLM 工具；ProbeTrans 仍需自行实现向量化专用模块。<https://github.com/yilingqinghan/IR-OptSet>
