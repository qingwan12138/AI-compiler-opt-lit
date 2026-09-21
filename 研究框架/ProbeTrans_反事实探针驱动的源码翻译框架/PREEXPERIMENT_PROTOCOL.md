# ProbeTrans 完整预实验协议（P0）

**版本**：IR-v2
**目标周期**：10–14 天
**资源**：AutoDL，1×RTX 4090，Linux；CPU 型号和独占程度未知
**目的**：验证机制是否值得进入正式实验，而不是追求论文级最好结果。

## 0. P0 必须回答的五个问题

1. 能否稳定构造“进入 LoopVectorize 前”的 LLVM IR，并准确追踪同一目标循环？
2. 有限反事实探针能否在合理查询预算内稳定翻转部分 missed decisions？
3. 证书中有多少条件能够在不偷偷强化语义的情况下实现？
4. 在相同模型和生成预算下，证书是否比 remark-only 更有用？
5. LLM 是否比读取相同证书的确定性模板多解决非机械结构改写？

P0 不负责证明广泛泛化、跨版本稳健性或 RVV 性能。

## 1. 建议服务器目录

只复制一个自包含目录到服务器：

```text
ProbeTrans/
  README.md
  pyproject.toml
  configs/
    p0.yaml
    toolchain.lock.yaml
    models.yaml
  third_party/
    IR-OptSet/           # 固定 commit 的 fork/submodule
    llvm-test-suite/     # 固定 commit
    autovec-benchmark/   # 固定 commit
    PolyBenchC/          # P0 可暂不下载
    alive2/              # 固定 commit/预编译版本
  probetrans/
  scripts/
  tests/
  data/
    raw/
    registry/
    snapshots/
  runs/
  artifacts/
```

任何脚本不得依赖 Windows 路径。API key 仅从环境变量读取，不能写入仓库。

## 2. 环境冻结

### 2.1 主工具链

- Ubuntu 22.04/24.04 容器或 Conda 环境；
- LLVM/Clang 19.1.x，优先与 IR-OptSet 论文工具链对齐；
- CMake、Ninja、Python 3.10+；
- Alive2 使用与 LLVM 主版本匹配的构建；
- `perf` 若服务器允许；否则使用 benchmark 自带 wall-clock runner。

不在 P0 同时比较多个 LLVM 版本。LLVM 21/22 只在正式实验做版本稳健性。

### 2.2 环境探测产物

运行 `scripts/doctor.sh` 后生成：

```json
{
  "os": "...",
  "cpu_model": "...",
  "cpu_cores": 0,
  "gpu": "...",
  "clang": "...",
  "opt": "...",
  "llvm_mca": "...",
  "alive2_commit": "...",
  "ir_optset_commit": "...",
  "benchmark_commits": {},
  "model_id": "...",
  "model_revision": "...",
  "quantization": "..."
}
```

### 2.3 M0 通过条件

- `clang --version`、`opt --version`、`llvm-as`、`llvm-dis` 主版本一致；
- IR-OptSet 自带最小测试可运行；
- 10 个正确/错误 IR fixture 中 verifier 判定 100% 符合预期；
- Alive2 能证明至少 3 个等价 fixture，并反驳至少 3 个 seeded wrong fixture；
- LLM endpoint 用固定 seed/temperature=0 连续两次请求格式稳定。

任何一项失败，停止 benchmark 与模型实验。

## 3. 冻结 IR 表示点

### 3.1 两阶段策略

**P0 canonical pipeline**：从前端 raw IR 运行一条显式、版本固定的 canonicalization pipeline，形成适合 loop analysis 的 IR，再单独运行 LoopVectorize。该设置用于快速验证机制。

**正式 O3-aligned pipeline**：若 P0 通过，再通过 PassBuilder instrumentation 捕获 `default<O3>` 中目标 LoopVectorize instance 前的 IR，并保存可重放的 suffix。

P0 不能把 canonical pipeline 称为“完整 O3 中间态”。

### 3.2 M1 实际冻结流程

1. 用 IR-OptSet frontend 从 benchmark source 产生 `raw.ll`；
2. 运行候选 canonical prefix；
3. 对 12 个 sanity loops 检查 LoopInfo、LCSSA、dominance 和 verifier；
4. 输出 pass trace；
5. 将最终 pipeline 字符串和工具版本写入 `toolchain.lock.yaml`；
6. 后续不得按样本修改。

### 3.3 Loop identity

每个 loop ID 至少包含：

```text
benchmark / function / header-block structural hash / debug source span / parent-loop depth
```

remark、pre-LV IR、post-LV IR 和 executable timing 必须通过该 ID 关联。只靠源码行号不够。

### 3.4 M1 通过条件

- 12 个 sanity loops 的 identity 在三次 clean build 中 100% 稳定；
- remark 与 IR structure 对“是否向量化”的判断 100% 一致；
- 原始/重新序列化 IR 的 harness 输出一致；
- pipeline 不产生 verifier error。

## 4. Benchmark 构造

### 4.1 Calibration pairs

从 `autovec-benchmark` 生成 hidden/exposed variants。一个 pair 仅在以下条件全部成立时有效：

1. 两个版本均可构建和运行；
2. 测试 oracle 相同；
3. hidden 的目标 loop 连续三次 clean build 均 missed；
4. exposed 的同一 loop 连续三次均 vectorized；
5. 差异可映射到一个或一组已注册 probe family；
6. sanitizer 未发现明显 UB。

按 original kernel group 做 30/70 calibration-dev/test split。所有阈值和 prompt 只能在 dev 上调整。

**数量 gate**：test 中至少 24 个有效 pairs；不足则停止以该资产支撑诊断主张。

### 4.2 TSVC pilot registry

对完整 TSVC 自动运行：

- correctness harness；
- pre-LV snapshot；
- LoopVectorize effect check；
- 可用时做目标 loop profile。

纳入条件：稳定 missed、harness 通过、无已知 UB、目标 loop 热度≥20%。若 profiling 不可得，P0 可用 benchmark 结构和计时对比暂定热点，但必须标为弱证据。

从 held-out kernel families 中按 TSVC category 分层选择 24 个；若合格数不足 24，使用全部且记录不足，不补入开发样本。

### 4.3 Registry schema

`data/registry/p0_cases.jsonl` 每行至少包含：

```json
{
  "case_id": "tsvc_s000_xxx",
  "suite": "TSVC",
  "group_id": "original-kernel-id",
  "function": "...",
  "loop_id": "...",
  "source_hash": "...",
  "pre_lv_ir_hash": "...",
  "baseline_effect": "missed",
  "remark_ids": [],
  "harness": "...",
  "inclusion_reason": "...",
  "exclusion_reason": null,
  "split": "p0-test"
}
```

必须同时保存 `excluded_cases.jsonl`，防止选择性报告。

## 5. Probe Registry v0

### 5.1 允许的五族探针

| ID | Family | P0 操作 | 目的 | 最终候选是否可直接复制 |
|---|---|---|---|---|
| P0 | profitability | 强制 vectorize enable/固定候选 VF | 区分成本与合法性 | 否 |
| P1 | alias | 临时 argument `noalias` 或 scoped noalias metadata | 测试 alias blocker | 否 |
| P2 | alignment | 临时加强 argument/load/store alignment | 测试 alignment blocker | 否 |
| P3 | trip-count | 临时 assume 下界/倍数或固定 trip fact | 测试 SCEV/remainder blocker | 否 |
| P4 | control-call | 临时暴露 invariant/memory effects 或可向量化调用事实 | 测试控制/调用 blocker | 否 |

### 5.2 禁止进入 P0 的探针

- fast-math/reassociation；
- 删除可能有副作用的 call；
- 改变 signed overflow 定义；
- target intrinsic；
- 直接替换为手写 vector IR；
- 任何会改变 benchmark oracle 的干预。

### 5.3 查询与最小化

每 case 上限 16 次 `opt`：

1. P0–P4 单探针；
2. 只对 remark/analysis 相关族形成二元组合；
3. 必要时最多测试两个三元组合；
4. 成功后执行 deletion minimization；
5. 新进程重放两次。

超过预算返回 `NO_CERT_WITHIN_BUDGET`，不能继续人工试探。

### 5.4 证书 schema

```yaml
case_id: ...
toolchain_fingerprint: ...
loop_id: ...
baseline_effect: missed
probe_set:
  - family: alias
    target: arg0,arg1
    diagnostic_edit_hash: ...
effect_after_probe: vectorized
minimality: inclusion-minimal
replay: [vectorized, vectorized]
queries: 9
realization_status: unknown
```

## 6. 诊断实验 D0

### 系统

1. Raw remark rule parser；
2. Precise-remark LLM classifier；
3. 单探针 greedy；
4. 完整 minimal certificate search。

### 指标

- blocker-family exact match；
- set precision/recall/F1；
- flip precision：声称成功的证书在 clean replay 中实际成功的比例；
- median/P90 compiler queries；
- certificate size；
- realizable/guardable/unrealizable 三态比例。

### D0 Go 条件

- ≥24 test pairs；
- flip precision =100%；
- median queries ≤12、P90 ≤16；
- certificate method 的 set-F1 相对 precise-remark 至少提高 0.10，或 exact match 至少提高 15 个百分点；
- 至少 30% test pairs 的证书被静态规则判为 potentially realizable。

这些是预实验工程决策阈值，不是论文显著性结论。

## 7. LLM 配置与公平性

### 7.1 P0 主模型

- 选择一个能在 RTX 4090 稳定服务的 14B 级 coder/instruct 模型或等价量化模型；
- 通过 OpenAI-compatible endpoint 调用；
- exact model/revision/quantization 在第一次正式请求前冻结；
- `temperature=0`；若服务不支持确定性，使用三个固定 seeds 并逐 seed 配对；
- 最大输入 16k tokens，最大输出 4096；
- 每 case 每系统最多 3 个候选；
- 仅允许一次 parser/verifier-error repair；
- 不反馈 correctness 反例或性能，避免系统变成开放式搜索。

### 7.2 四个系统

| System | 模型看到的信息 | Probe queries | 候选预算 |
|---|---|---:|---:|
| Direct | IR function + “unlock auto-vectorization” | 0 | 3 |
| Remark | Direct + raw/precise LLVM remarks | 0 | 3 |
| Template | certificate，确定性规则，无 LLM | 与 ProbeTrans 共享缓存 | 每模板1个 |
| ProbeTrans | IR + remarks + certificate + SOC | 与 Template 共享缓存 | 3 |

Template 与 ProbeTrans 必须读取同一个 certificate artifact，不能分别搜索。

### 7.3 防止模型差异造成不公平

- 所有方法在同一模型内做 paired comparison；
- 不把不同模型的成功数相加；
- P0 只用一个模型决定机制可行性；
- 第二模型只在 P0 Go 后做稳健性；
- prompt 长度差异和 token 使用量单独报告。

## 8. Candidate Gates

按顺序执行；失败即停止该候选，不能跳关。

### G0：格式与 splice

- 输出仅包含一个同名同签名函数；
- deterministic splicer 成功；
- module declarations/attributes 引用完整。

### G1：解析与 verifier

- `llvm-as` 成功；
- `opt -passes=verify` 成功；
- SSA dominance、PHI predecessor 和类型合法。

### G2：Semantic-strengthening audit

比较原/候选 function：新 attributes、metadata、instruction flags、assumes 与 call memory effects。未经 contract 允许的强化直接拒绝。

### G3：语义验证

- 首先 Alive2，结果分 `proved / disproved / timeout / unsupported`；
- `disproved` 永久拒绝；
- `timeout/unsupported` 进入 differential tests，不得写成 formally verified；
- differential tests 包含 benchmark oracle、边界输入、随机输入和 sanitizer。

P0 建议每 case 至少 1,000 个有效随机/边界输入；随机 seed 固定并公开。

### G4：目标 effect

同时要求：

- 原始 target loop 为 missed；
- 候选 target loop 出现 vectorization-success remark；
- post-LV IR 中出现对应 vector loop body；
- 不能仅是其他 loop 被向量化。

### G5：性能

先运行 CPU 噪声试验：相同 binary 重复 50 次。若 median absolute deviation/median >3%，尝试 pinning、扩大 workload、批内随机化；仍>5%则 P0 不做 APIR 决策，只报告 CVUR，并明确性能阶段被环境阻塞。

稳定时，每 binary：5 次预热、至少 30 次有效测量；原/候选与 on/off 采用随机交错顺序；保存所有原始样本。

### G6：VAG

构建四个 binary：`O_on, O_off, E_on, E_off`。关闭配置必须同时禁用 loop 和 SLP vectorization，或通过 pass pipeline 只移除目标 LoopVectorize；具体定义在 M1 冻结。

接受性能成功需：

- `E_on` 相比 `O_on` 超过 noise-derived threshold；
- paired/bootstrap 95% CI 为正；
- attribution interaction `A` 的 95% CI 为正。

## 9. 端到端实验 E0

### 样本

24 个 held-out TSVC eligible loops。若少于 24，使用全部，不从开发集补齐。

### 主指标

- `CVUR = correct vectorization unlocks / all eligible cases`；
- CPU 稳定时：`APIR = attributed profitable IR rewrites / all eligible cases`。

### 辅助漏斗

`eligible → certificate found → potentially realizable → parse → verify → semantic → vectorized → profitable → attributed`

### E0 Go 条件

1. ProbeTrans 至少取得 5/24 个正确向量化解锁；
2. 相比 Direct 和 Remark 各至少多 3 个正确解锁；
3. 相比 Template 至少多 2 个，且至少两个成功涉及非机械 CFG/loop restructuring；
4. accepted candidates 中 semantic-strengthening violation 为 0；
5. 若 CPU 稳定，至少 3/24 达到 APIR；
6. VAG 至少能重新分类一个“表面加速但非向量化归因”候选，或者通过预先植入的标量优化阳性对照证明 gate 有辨识力；
7. 全部 LLM 调用、候选和失败均有日志。

若只差性能条件且 CPU 噪声不合格，状态为 `GO_MECHANISM_ONLY`，换稳定 CPU 后补测；不能把它判为完整 Go。

## 10. 必做消融 A0

在 E0 24 cases 上做：

- `w/o certificate`：即 Remark system；
- `non-minimal certificate`：给全部成功探针；
- `w/o SOC`：允许模型自由实现，但仍执行 audit，统计会被拒绝的泄漏；
- `w/o VAG`：比较会被错误计入的性能成功；
- `certificate→template`：检验 LLM 必要性。

不做 RAG、多 Agent、微调或多 LLVM 版本消融。

## 11. 统计与报告

- 二元 paired outcome：给出每系统成功数、成对差异和 exact McNemar/paired bootstrap CI；P0 小样本重点报告 effect size，不以 `p<0.05` 作为唯一 Go 条件。
- 性能：报告 median、MAD、bootstrap 95% CI；geomean 只对 accepted 和 fallback-inclusive 两种口径分别报告。
- 多候选：case-level success 只计一次；同时报告 candidate-level failure taxonomy。
- 超时：probe 60s、verifier 30s、Alive2 300s、单候选总验证 15min；实际阈值在 sanity 后冻结。
- 缺失/崩溃/超时均计失败，不删除分母。

## 12. 运行顺序与预计成本

| 日程 | 工作 | GPU | 退出条件 |
|---|---|---:|---|
| Day 1 | 环境、IR-OptSet、LLVM、Alive2 smoke | 0h | M0 未通过则停止 |
| Day 2 | canonical pipeline 与 loop ID | 0h | M1 未通过则停止 |
| Day 3–4 | autovec 重认证与 registry | 0h | test pairs <24 则停止 |
| Day 5 | probes + certificate search | 0h | D0 gate 未通过则停止 |
| Day 6 | SOC、function splice、template baseline | 0h | fixture tests 未通过则停止 |
| Day 7–9 | Direct/Remark/Template/ProbeTrans 生成 | 4–10h | 保留所有候选 |
| Day 9–11 | verifier/Alive2/differential/effect | 0h | 生成漏斗 |
| Day 12 | CPU noise 与性能/VAG | 0h | 噪声高则 mechanism-only |
| Day 13 | 消融与失败分类 | 0–3h | 完成 kill decision |
| Day 14 | P0 report 与下一阶段裁决 | 0h | GO/GO_MECHANISM_ONLY/NO-GO |

## 13. 必须生成的产物

```text
artifacts/p0/
  environment.lock.json
  toolchain.lock.yaml
  benchmark_registry.jsonl
  excluded_cases.jsonl
  calibration_pairs.jsonl
  certificates/*.yaml
  prompts/*.json
  responses/*.json
  candidates/*.ll
  gate_results.jsonl
  timing_raw/*.csv
  summary.csv
  failure_taxonomy.md
  P0_DECISION.md
```

`P0_DECISION.md` 必须明确写一个且只能写一个：

- `GO`：诊断、翻译和性能归因 gate 均通过；
- `GO_MECHANISM_ONLY`：诊断/翻译通过，但共享 CPU 无法可靠测性能；
- `NO_GO_DIAGNOSIS`：证书无信息增量；
- `NO_GO_TRANSLATOR`：证书有效但 LLM 不超过 remark/template；
- `NO_GO_SAFETY`：主要收益依赖不可接受语义强化；
- `NO_GO_EFFECT`：生成正确但不能解锁目标 vectorizer。

## 14. P0 后才允许的工作

只有 GO 或 GO_MECHANISM_ONLY 后才进行：

- 全量 held-out TSVC；
- PolyBench/C 和真实应用；
- 第二模型；
- LLVM 版本稳健性；
- 200–500 个 IR-OptSet 静态压力样本；
- RV64GCV 外部验证；
- 可能的 SFT。

在 P0 之前做这些会稀释失败信号并浪费算力。
