# ProbeTrans Research Contract — LLVM IR Edition

**日期**：2026-09-22
**类别**：TRANSLATOR
**输入/输出**：pre-LoopVectorize LLVM IR function → semantically equivalent LLVM IR function
**开源基座**：IR-OptSet, NeurIPS 2025 Datasets and Benchmarks, MIT License

## 1. Immutable Problem Anchor

- **底线问题**：在不修改 LLVM 后端、不使用 RL、不依赖 ISA intrinsic 的条件下，修复一部分 LLVM `LoopVectorize` 漏优化，使 LLM 产生正确、可验证、可归因的 LLVM IR 变换。
- **必须解决的瓶颈**：普通 optimization remark 只能描述失败表象，不能可靠指出“改变哪些 IR 条件足以翻转当前编译器决策”；自由 IR 生成又容易引入语法错误、SSA 错误或更隐蔽的语义强化。
- **非目标**：通用 `-O3` 替代器、多 Pass 协调、Pass ordering、源码重写、显式 SIMD、RL、以 RISC-V 为主场景。
- **资源约束**：单张 RTX 4090；P0 优先零训练；服务器运行；公开数据和开源工具；x86 主实验，RVV 外部验证。
- **成功条件**：在预注册 held-out benchmark 上，同模型同预算下，ICIT 相比 direct/remark-only IR 翻译提高“正确且由 LoopVectorize 归因的有益翻译率”，并明显超过读取相同证书的确定性模板。

## 2. Frozen Representation Point

研究对象不是 `-O0` 原始 IR，也不是完整 `default<Oz>`/`default<O3>` 后的 IR，而是：

> **完成固定的 target-independent canonicalization prefix、尚未运行 LoopVectorize 的 IR snapshot。**

P0 使用 IR-OptSet 所兼容的 LLVM 19.1.x。具体 prefix 在 M0 通过 pass trace 冻结；最低必须包含形成稳定循环表示所需的 `mem2reg/SROA`、`simplifycfg`、`loop-simplify`、`LCSSA`、`loop-rotate` 与 `indvars` 等等价阶段。不得根据结果为单个 benchmark 改 pipeline。

选择该位置的原因：

1. `-O0` IR 含大量前端噪声，问题会退化为重做常规优化；
2. `Oz/O3` 完整结束后，目标决策和可恢复的前提已经被多轮 pass 改写；
3. pre-LoopVectorize snapshot 让“输入条件—向量化决策—后端效果”边界明确。

## 3. Base-Code Contract

### 直接继承 IR-OptSet

- `IRDS/core/llvm/`：Clang/`opt` wrapper 与版本管理；
- `IRDS/core/preprocessing/`：IR 清洗、命名规范化；
- `IRDS/tools/opt_verify.py`：解析和 verifier；
- `IRDS/tools/alive2.py`：Alive2 接口；
- `IRDS/tools/mca_cycles.py`：静态分析，只作辅助；
- `IRDS/llm/`：prompt/model adapter；
- 并行、日志、token 统计和 dataset schema。

### ProbeTrans 新增目录

```text
probetrans/
  capture/       # pre-LV snapshot 与 loop identity
  probes/        # 诊断专用 IR interventions
  certificates/  # 搜索、最小化、重放
  translator/    # 函数级 IR 生成与 splice
  audit/         # semantic-strengthening 检查
  gates/         # verify, Alive2, differential, effect, performance, attribution
  benchmarks/    # TSVC/autovec/PolyBench/IR-OptSet adapters
  configs/       # 冻结工具链、模型、预算、阈值
```

## 4. Main Mechanism: ICIT

### 4.1 输入

- 一个可独立 splice 的 LLVM IR function；
- 目标 loop identity；
- 原始 `LoopVectorize` missed/analysis record；
- frozen target triple、data layout、CPU features 与 pass pipeline；
- 可执行 correctness harness（主性能 benchmark 必需）。

### 4.2 诊断探针

所有探针产物标记 `DIAGNOSTIC_ONLY`，不能直接成为模型输出：

| 探针族 | 临时干预示例 | 所识别障碍 |
|---|---|---|
| Profitability | `llvm.loop.vectorize.enable/width` | 仅成本模型拒绝还是 legality 拒绝 |
| Alias/dependence | 临时 `noalias`/alias scope metadata | 未证明的内存相关 |
| Alignment/memory | 临时加强 alignment/连续访问条件 | 对齐或访问形态阻碍 |
| Trip/count | 临时 `llvm.assume` 下界、倍数或已知 trip count | 小 trip、remainder 或 SCEV 不充分 |
| Control/call | 临时暴露不变量、内存效应或可 vectorize call | 控制流或调用副作用障碍 |

FP reassociation、overflow weakening 和异常语义变化默认不进入 P0。若以后研究，必须作为单独语义许可层。

### 4.3 证书搜索

- 预算 `B_probe=16` 次 `opt` 查询；
- 先测试单探针，再测试同族/跨族二元组合；只有前两层均失败时才允许最多三元组合；
- 每次查询要求 optimization record 与 vector IR structure 一致；
- 对成功集合逐项删除，得到探针 registry 内的 **inclusion-minimal** 集合；
- 在两次 clean process 中重放，决策不一致则证书无效；
- 不使用“全局最小”“真实唯一根因”或“因果证明”措辞。

### 4.4 LLM IR 翻译接口

模型不重写整个 module。输入为 target function、必要 declarations/data layout/attributes、missed record、certificate、Semantic-Obligation Contract 和输出 schema。

模型输出一个完整 replacement function definition，允许重新编号 SSA，禁止输出 module-level 任意文本。deterministic splicer 将其放回原 module。P0 每个样本最多 `K=3` 个候选，最多一次仅针对 parser/verifier 错误的修复；不把性能数值反馈给模型。

## 5. Semantic-Obligation Contract

最终 IR 相对输入不得无依据新增：`noalias`、`nonnull`、`dereferenceable`、更强 alignment、`nsw/nuw/exact/inbounds`、fast-math flags、更强 memory effects 或无法由原 IR 推出的 `llvm.assume`。

允许两种实现：条件能由原 IR/分析证明；或新增运行时 guard，将满足条件的 fast path 与未经修改的 fallback path 组成语义保持版本化。静态 audit 只检查明显强化；最终正确性仍由 verifier、Alive2 和 differential testing 决定。

## 6. Benchmark Contract

### 6.1 诊断校准：autovec-benchmark

- 在 LLVM 19.1.x 上重新生成 hidden/exposed variants；
- 只保留 hidden 未向量化、exposed 向量化、测试等价、决策可重放的 pair；
- 按原始 kernel 分组切分，禁止同一 kernel 的变体跨 calibration/test；
- 只用于 blocker diagnosis，不用于端到端性能主结论。

### 6.2 主测试：TSVC

- 来源为 LLVM test-suite 完整 TSVC；
- 在 LLM 调用前自动筛选并冻结所有 eligible missed loops；
- 纳入：correctness harness 有效、同一 loop 稳定 missed、目标 loop 热度 ≥20%、无已知 UB；
- 排除规则和原因全部公开；
- 主结果使用未参与 prompt/probe 调整的 held-out kernel families。

### 6.3 外部泛化与压力集

- PolyBench/C 全量扫描后得到的 hot missed loops；
- Blackscholes、Kmeans、LBM 及可复现 LLVM test-suite applications 中 3–5 个热点；
- IR-OptSet 从 HPC/Multimedia/Embedded 按固定 hash 抽取 200–500 个含循环函数，只测静态鲁棒性；
- RV64GCV 只接收 x86 阶段冻结 IR，不重新 prompt；QEMU 不用于性能结论。

## 7. Baseline Contract

在每个模型内部配对，冻结 `K`、输出 token、修复轮数和 compiler-query budget：

1. LLVM canonical IR + `LoopVectorize`；
2. Direct IR Translator：只给 function 和目标；
3. Remark-only Translator：加 raw/precise remark；
4. Certificate→Template：同一证书，确定性 IR 模板；
5. ProbeTrans：certificate + SOC + gates。

IntOpt-style intent prompting 和 IR-OptSet fine-tuned model可作为正式实验强基线，但 P0 不因复现成本阻塞核心 kill-gate。

## 8. Metrics Contract

主指标为 `APIR = Attributed Profitable IR Rewrite rate`，分母为所有 eligible inputs。分子必须同时满足 verifier、语义验证、目标循环由 missed 变为 vectorized、x86 实测超过冻结噪声阈值且 VAG 交互项 95% CI 为正。

次指标包括 Certificate Flip Rate、blocker set-F1、查询次数、证书大小、各门控通过率、Correct Vectorization Unlock Rate（CVUR）、失败记 1.0× 的 fallback-inclusive geomean、semantic-strengthening violation rate 和成本。

## 9. Kill Criteria

1. 重认证后有效 calibration pairs <24；
2. 证书不能 clean-process 重放，或 median query >16；
3. 24 个 held-out TSVC pilot 中可安全实现证书 <6；
4. ProbeTrans 的正确向量化解锁不超过 Remark-only；
5. Template 与 ProbeTrans 的 CVUR/APIR 差距≤1个案例；
6. 主要成功依赖未证明 attribute/metadata/flags；
7. VAG 显示多数表面加速在关闭 LoopVectorize 后仍存在；
8. 外部集不优于 Direct/Remark-only。

## 10. Server Portability

- 所有路径相对项目根目录；
- 用 `environment.lock.json` 记录 LLVM、Alive2、Python、模型、CPU、GPU 与 git commit；
- secrets 只通过环境变量；每个 run 生成不可变 manifest；
- Windows 文献库和 ARIS 目录不成为服务器运行依赖。
