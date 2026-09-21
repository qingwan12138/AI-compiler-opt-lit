# Experiment Plan — LLVM IR ProbeTrans

**Problem**：可靠修复 LLVM LoopVectorize 漏优化，而不是让 LLM 自由猜测和重写整个程序。
**Method Thesis**：编译干预得到的最小证书是比 remarks 更有效的 IR 翻译条件；SOC/VAG 确保成功安全且可归因。
**Date**：2026-09-22

详细 P0 操作见 `PREEXPERIMENT_PROTOCOL.md`；本文定义论文级实验逻辑。

## Claim Map

| Claim | Minimum Convincing Evidence | Blocks |
|---|---|---|
| C1 Certificate information gain | 同模型同预算下，ProbeTrans 在 held-out TSVC/外部集的 CVUR/APIR 高于 Direct 与 Remark-only；诊断集 set-F1 更高 | B1, B2, B4 |
| C2 LLM 与可信验收均非装饰 | Template 未追平；SOC 拦截实际语义泄漏；VAG 排除实际错误归因 | B3 |
| Anti-claim | 收益不是更多查询、完整函数重写、模型差异、TSVC 泄漏或静态代理造成 | B2, B3, B4, B5 |

## Benchmark Roles

| Dataset | Split | Role | Main metrics |
|---|---|---|---|
| autovec-benchmark re-certified pairs | group-wise dev/test | 诊断标签 | set-F1, exact match, queries, realizability |
| TSVC eligible missed loops | dev families / held-out families | 主端到端 | CVUR, APIR, fallback geomean |
| PolyBench/C + 3–5 apps | frozen external | 泛化 | CVUR, APIR, failure shift |
| IR-OptSet 200–500 loop functions | fixed-hash static sample | 工具压力 | parse/verify/certificate/codegen coverage |
| frozen successes on RV64GCV | no retuning | 跨 ISA 外部验证 | correctness, vector effect, native speedup if available |

## Baseline Families

### B-Family 1：LLM information baselines

- Direct function IR translation；
- Remark-only translation；
- ProbeTrans certificate-conditioned translation。

### B-Family 2：Non-LLM necessity baseline

- Certificate→Template；
- 可选：固定 LLVM canonical repair templates。

### B-Family 3：Recent IR optimizer baseline

- IntOpt-style structured intent prompt；
- IR-OptSet/LLM Compiler 可复现模型（若获取与依赖允许）。

不同模型分别报告，不能混入同一汇总成功率。

## Experiment Blocks

### B0：Infrastructure validity

- **Claim**：捕获的是稳定且可重放的 pre-LV IR，gate 本身可信。
- **Data**：12 sanity loops、6 semantic fixtures、3 seeded scalar-optimization controls。
- **Metrics**：loop-ID stability、remark/IR agreement、verifier/Alive2 sensitivity、timing noise。
- **Success**：identity/decision 100% 重放；seeded error 全被预期 gate 捕获；稳定 CPU CV/MAD 达标。
- **Failure**：基础设施失败，不运行模型。
- **Placement**：Appendix methodology，MUST-RUN。

### B1：Certificate diagnosis

- **Claim**：主动干预比 remarks 更准确地定位决策翻转条件。
- **Data**：held-out re-certified autovec pairs。
- **Systems**：raw remark、precise-remark LLM、greedy single-probe、minimal certificate search。
- **Metrics**：set-F1、exact match、flip precision、queries、certificate size、realizable rate。
- **Success**：达到 P0 D0 gate；正式实验使用 paired bootstrap CI，且 ProbeTrans 对最强 remark baseline 的下界为正。
- **Failure**：若只靠 remark 已等价，主创新不存在。
- **Placement**：Main Table 1，MUST-RUN。

### B2：End-to-end IR translation

- **Claim**：certificate 提高正确的目标向量化解锁与真实收益。
- **Data**：held-out TSVC eligible missed loops。
- **Systems**：LLVM baseline、Direct、Remark-only、Template、ProbeTrans。
- **Metrics**：APIR 主指标；CVUR、gate funnel、fallback-inclusive geomean、negative transfer、tokens/queries/wall time。
- **Success**：ProbeTrans 对 Direct/Remark 的 paired APIR/CVUR 差异为正；完整实验 95% CI 不跨 0；不存在 safety regression。
- **Failure**：仅 parse/verify 提升而 effect/APIR 不升，不支持主张。
- **Placement**：Main Table 2 + funnel figure，MUST-RUN。

### B3：Novelty and necessity isolation

- **Claim**：最小 certificate、LLM、SOC 与 VAG 各自改变关键结论。
- **Data**：B2 全部 cases/candidates。
- **Ablations**：non-minimal certificate、w/o SOC、w/o VAG、certificate→template、whole-function free generation。
- **Metrics**：CVUR/APIR、semantic leakage、false attribution、token cost、accepted-set change。
- **Success**：Template 未追平；SOC/VAG 至少各揭示一类真实错误或由 seeded controls 证明灵敏度；minimal certificate 不劣于全探针且成本更低。
- **Failure**：删除某机制无任何变化，则从最终系统删除。
- **Placement**：Main Table 3，MUST-RUN。

### B4：External generalization

- **Claim**：不是 TSVC 模式记忆。
- **Data**：frozen PolyBench/C hot missed loops + 3–5 open applications。
- **Systems**：Remark-only、Template、ProbeTrans；prompt 和阈值不变。
- **Metrics**：CVUR、APIR、fallback geomean、probe/certificate distribution shift。
- **Success**：外部 eligible loops ≥15 时，ProbeTrans paired delta 对最强基线为正；否则只作案例研究。
- **Failure**：只在 TSVC 成功则降级为 benchmark-specific。
- **Placement**：Main Table 4，MUST-RUN before submission。

### B5：Scale, version, and ISA robustness

- **Claim**：工具链可扩展，输出保持 target-independent，但不主张处处加速。
- **Data**：IR-OptSet static sample；LLVM 次版本；冻结成功 IR 的 RV64GCV 编译。
- **Metrics**：pipeline success、certificate coverage、compile/codegen、RVV effect/native speedup。
- **Success**：完整逐例报告；不要求所有平台同向。
- **Failure**：若 IR 大量只对单 LLVM 版本有效，收窄适用边界。
- **Placement**：Appendix，NICE-TO-HAVE；RVV 是用户要求的必要外部验证。

## Model Protocol

- P0：单个可在 4090 部署的冻结 coder model，零训练；
- 正式实验：主模型 + 一个不同能力档模型；逐模型 paired；
- `K=3`，固定 token，最多一次 verifier repair；
- 性能、隐藏测试结果不反馈给模型；
- 完整 prompt/response 公布；
- 若后期 SFT，必须增加“相同 backbone 零训练”对照，且 SFT 不能替代 certificate 消融。

## Correctness Protocol

结果分别报告：

- verifier pass；
- Alive2 proved；
- Alive2 disproved；
- Alive2 timeout/unsupported + differential pass；
- differential fail。

Alive2 unknown 不得并入 formally verified。随机测试 seed、输入域、浮点比较规则和 sanitizer 选项在测试前冻结。

## Performance and Attribution

- stable CPU 上随机交错测量 `O_on/O_off/E_on/E_off`；
- 5 warm-ups、≥30 samples，保存原始数据；
- noise-derived minimum effect threshold；
- median/MAD、bootstrap CI；
- APIR 以所有 eligible cases 为分母；
- 失败/超时在 fallback geomean 中记 1.0×；
- QEMU 不用于性能。

## Run Order

| Milestone | Goal | Runs | Decision Gate | Cost |
|---|---|---|---|---|
| M0 | 环境与 gate sanity | R001–R006 | B0 通过 | 0 GPUh, 1天 |
| M1 | pre-LV pipeline/loop ID | R010–R014 | 三次重放稳定 | 0 GPUh, 1天 |
| M2 | benchmark registry | R020–R026 | calibration-test≥24，TSVC pilot≤24 | 0 GPUh, 2天 |
| M3 | probe/certificate | R030–R036 | D0 Go | 0 GPUh, 1–2天 |
| M4 | P0 translator | R040–R047 | E0 Go/Mechanism-only | 4–10 GPUh, 3–4天 |
| M5 | P0 ablation/decision | R050–R055 | 明确单一裁决 | 0–3 GPUh, 2天 |
| M6 | full TSVC/external/second model | R060–R079 | 论文 claims | 30–80 GPUh |
| M7 | IR-OptSet/RVV/version | R080–R095 | robustness boundary | 依硬件 |

## Compute Budget

- P0 GPU：约 4–13 小时，取决于本地模型吞吐；
- P0 CPU：约 2,000–5,000 次短编译/验证，Alive2 是主要长尾；
- Full：约 30–80 GPUh；先依据 P0 实测重新估算，不提前承诺；
- 存储：保存 IR、responses、logs、timings，预留 50–100GB；
- 最大风险不是 GPU，而是证书安全可实现率和 CPU 测量稳定性。

## Paper-Readiness Gates

进入论文完整实验必须同时满足：

1. D0 证明 certificate 对 remark 有信息增量；
2. E0 的 ProbeTrans 正确解锁数超过 Direct/Remark/Template；
3. SOC/VAG 实际改变接受集合或由强阳性对照验证；
4. 稳定 CPU 上有至少若干 APIR successes；
5. 外部集趋势不反转；
6. 相同 IR 至少能在 RV64GCV 编译并出现目标 effect 的正例。

## Intentionally Cut

- RL、RAG、多 Agent、MCTS；
- 多 Pass 和 pass ordering；
- 源码 patch；
- explicit intrinsic；
- 在线性能反馈搜索；
- 在 P0 同时比较多个模型/LLVM 版本。
