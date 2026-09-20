# SalvageIR 预实验与门控协议

> 版本：2026-09-20 精化版。
> 性质：运行前预注册协议，不是实验结果。
> 主任务：判断“被明确证伪的 LLM LLVM-IR 优化中，是否存在值得搜索的正确且有收益严格子集”。

## Material Passport

- Origin Skill: academic-research-suite / experiment-agent
- Origin Mode: plan
- Origin Date: 2026-09-20
- Verification Status: UNVERIFIED
- Version Label: salvageir_pilot_protocol_v1

## 1. 预实验只回答什么

P0 不证明 SalvageIR 搜索优于基线。它只回答：

1. 至少三个非强化学习模型家族是否都产生可解析且被 Alive2 明确反驳的候选；
2. 这些候选是否能被稳定对齐并分解成依赖闭合组件；
3. 在可穷举的小组件候选中，是否存在 `verified + profitable + strict` 子集；
4. 组件规模和验证成本是否允许后续进行预算化搜索。

P0 不得使用 SalvageIR 的最终排序效果作为继续条件。否则会把“研究现象是否存在”和“当前实现是否找得到”混在一起。

## 2. 冻结的主目标

正式受控候选的主目标冻结为：

```text
x86-64 generic 后端上，目标函数 ELF 符号的机器码字节数
```

源函数先经过同一 LLVM `default<Oz>` 中端流水线形成 `S`。LLM 被要求在保持签名和可观察行为的前提下继续减小 code size。源程序、候选和恢复结果使用同一个后端配置编译。

不使用以下量作为主收益：

- LLVM IR 指令数；
- 文本行数或 Token 数；
- 未校准的 LLM 评分；
- 仅由 `llvm-mca` 给出的吞吐预测；
- 整个对象文件大小。

目标函数大小由 `llvm-readobj --symbols` 的 `ST_Size` 读取，并用 section 表统计单函数对象中全部 `SHF_ALLOC` 且非 BSS 的字节。若符号消失、被合并、大小为零或不能唯一定位，该记录进入 `COST_UNMEASURABLE`，不能算成功。

冻结命令族为：

```text
opt -passes='default<Oz>' source.bc -o source.oz.bc
llc -O=3 -mtriple=x86_64-unknown-linux-gnu -mcpu=generic \
    -filetype=obj candidate.bc -o candidate.o
llvm-readobj --symbols --sections candidate.o
```

实际可执行文件中补齐路径、commit 和兼容参数，但不得改变优化级别、triple、CPU 或计量字段。源 `S` 与结果 `R` 都从同一“单目标函数 + 必要声明”模块生成对象，避免无关函数和节对齐影响。

定义：

```text
absolute_gain = size(S) - size(R)
relative_gain = absolute_gain / size(S)

profitable(R) :=
  absolute_gain >= max(2 bytes, ceil(0.01 * size(S)))
  and alloc_non_bss(R) <= alloc_non_bss(S)
```

第二个条件防止把机器码搬到 jump table、常量池或只读数据后伪装成代码缩小。一字节收益单列为敏感性分析，不进入主要成功率。机器码大小是确定性指标；同一输入重复三次必须逐字节一致，否则环境门失败。

LLM-VeriOpt 公开候选原本面向 latency。它们只承担“失败候选是否可分解、是否能语义回收”的外部重放，不与上述 code-size 主结果合并。

## 3. 数据集分层

### 3.1 D-A：公开失败候选全量重放

来源为 LLM-VeriOpt CGO 2026 artifact。仓库公开说明包含模型配置、全部 IR 输出、Alive2 日志和详细指标：

- <https://github.com/carrotProgrammer/llmveriopt-AE>
- <https://zenodo.org/records/17672452>

对 artifact 做全量清点，不抽样选择“看起来可回收”的候选。

非 RL 允许列表：

- `sft_codellama_7b`；
- `sft_llama3_3b`；
- `sft_llama3_8b`；
- `sft_qwen_3b`；
- `sft_qwen_7b`；
- `sft_qwen_32b`；
- `model_zero` 仅在模型卡和配置确认未使用 RL 后纳入。

排除列表：

- `model_correctness`；
- `model_latency`；
- 名称、模型卡或训练记录显示使用 GRPO/PPO/其他 RL 的任何输出。

同一家族的不同参数规模不是独立模型家族。主分层仍为 CodeLlama、Llama 3 和 Qwen 2.5；规模只作为家族内亚组。

所有历史标签必须在两条工具链上重放：

1. artifact 自带工具链，用于复现原标签；
2. 本研究冻结工具链，用于形成统一主标签。

两者冲突时不挑对 SalvageIR 有利的标签；主分析使用本研究工具链，冲突率单列报告。

### 3.2 D-B：IR-OptSet 受控发现集

IR-OptSet 是 NeurIPS 2025 Datasets and Benchmarks Track 数据集，公开卡片记录约 170K LLVM IR、1,704 个仓库、八个应用域，并提供 `repo_name`、`structural_hash` 和许可证字段：

- <https://proceedings.neurips.cc/paper_files/paper/2025/hash/a4ab7aefc004bed00e577164c57eafd7-Abstract-Datasets_and_Benchmarks_Track.html>
- <https://huggingface.co/datasets/YangziResearch/IR-OptSet>

冻结 Hugging Face revision、Parquet SHA-256 和数据卡许可证。只纳入许可证允许研究再分发或记录派生元数据的样本。

发现集固定为 160 个源函数：

```text
8 domains × 4 post-Oz instruction bins × 5 functions = 160
```

八个域为：HPC、Machine Learning、Multimedia、Embedded Systems、System Software、Security、Reusable Libraries、Algorithms。

四个复杂度箱按删除 debug/lifetime 后的非 PHI LLVM 指令数定义：

| 箱 | 指令数 |
|---|---:|
| B1 | 10–39 |
| B2 | 40–79 |
| B3 | 80–159 |
| B4 | 160–320 |

每个“域 × 复杂度箱”选择五个不同仓库，且全体发现集中同一仓库最多贡献两个函数。选择过程完全确定：

1. 用冻结 LLVM 把 `original_ir` 转成 opaque-pointer 规范形式；
2. 运行 `default<Oz>` 得到源函数 `S`；
3. 执行纳入/排除过滤；
4. 计算 `SHA256(dataset_revision | repo_name | repo_file_path | function_name | structural_hash)`；
5. 按哈希升序选择前五个不同仓库，并跳过已经达到全局两个函数上限的仓库。

若某个格不足五个，按 B2、B3、B1、B4 的固定优先级从同域其他复杂度箱补齐，并保留空格和补齐日志。不得手工挑函数。

### 3.3 D-BR：顺序扩样保留池

在 D-B 之后使用同一规则生成第二个、互不重叠的 160 函数保留池。只有候选供应门不足时才启用；是否启用只由 `REFUTED_ELIGIBLE` 数量决定，不能查看 oracle 可回收结果。

### 3.4 D-C：LLVM Opt Benchmark 未触碰确认集

确认集来自已迁移到 `llvm-opt-benchmark-nightly` 的真实项目 IR 数据：

- <https://github.com/dtcxzyw/llvm-opt-benchmark-nightly>

冻结 Git commit、Hugging Face bucket snapshot 和上游项目 commit。D-C 在 P0 的组件规则、`n_oracle`、收益阈值和门控结论全部冻结前不得运行。

D-C 固定为 240 个函数：

```text
12 projects × 20 functions = 240
```

项目层分为四个 C、四个 C++、四个 Rust 项目；项目与 D-B、D-BR 和 artifact 源项目去重。每个项目从四个复杂度箱各抽五个函数。每个函数固定生成 greedy、seed 17 和 seed 29 三个输出，不使用结果驱动的顺序扩样。

D-C 的作用是确认门控结论，而不是继续调对齐器、组件粒度或阈值。任何调参都会使 D-C 失效，必须重新选择未触碰项目。

### 3.5 D-H：端到端与跨后端保留集

来自 LLVM 官方 test-suite 的 24 个可执行 benchmark：

- 8 个 `SingleSource/Benchmarks`，覆盖 Polybench、Misc 和 Shootout；
- 8 个 `MultiSource/Benchmarks`，覆盖 MiBench、BitBench、Trimaran 和 Olden；
- 8 个较大程序，优先来自 CTMark；若冻结 commit 中跨 x86-64/RV64GC 的合格 CTMark 少于八个，按预先固定的路径哈希顺序从 `MultiSource/Applications` 补齐。

LLVM test-suite 官方说明其提供参考输出、运行时间和代码大小测量，并支持交叉编译及 `TEST_SUITE_RUN_UNDER`：<https://llvm.org/docs/TestSuiteGuide.html>。

具体程序不是人工挑选：在冻结 commit 上先按“x86-64 与 RV64GC 均可构建、许可证可用、有参考输出”过滤，再在每一层按路径哈希升序选择。D-H 不参与阈值、Prompt 或组件规则调整。

### 3.6 D-U：不进入统计的语义单元集

构造 48 个人工 IR 对：八类各六例。

1. SSA def-use；
2. CFG、PHI 和控制依赖；
3. `nsw/nuw/exact/inbounds`；
4. poison、undef 和 freeze；
5. load/store 与别名；
6. 函数属性、调用和 ABI；
7. 浮点与快速数学标志；
8. 向量及目标相关 intrinsic。

每类包含可分、不可分、需要联合回滚三种情况。D-U 只验证实现和拒绝路径，绝不计入恢复率。

## 4. 源函数纳入与排除

### 4.1 受控生成纳入条件

- 单个定义函数和所需类型/常量/声明能形成可解析模块；
- `default<Oz>` 后有 10–320 条非 PHI、非 debug/lifetime 指令；
- 基本块数 1–32；
- Prompt 总 Token 不超过所选三模型最小上下文窗口的 75%；
- x86-64 generic 后端能产生唯一、非零函数符号；
- 源函数通过 LLVM verifier；
- `Alive2(S,S)` 在冻结 timeout 内返回 verified。

### 4.2 预实验核心范围排除

以下样本仍进入数据漏斗，但标为 `REFUTED_UNSUPPORTED_SCOPE`，不进入核心 oracle 分母：

- 异常处理、setjmp/longjmp、coroutine；
- inline asm、`callbr`、`indirectbr`、`musttail`；
- volatile 或 atomic 内存操作；
- convergent 操作；
- 可扩展向量和未冻结语义的目标 intrinsic；
- 跨过程变换或函数签名变化；
- Alive2 unsupported 或 timeout。

核心 P0 先限定整数、位运算、比较、cast、select 和普通 CFG。内存、调用、浮点和向量作为覆盖率层报告，不得混入核心通过门后声称已经支持。

## 5. 模型与生成协议

受控 P0 使用 artifact 中三个最小的非 RL SFT 配置：

| 家族 | 冻结配置 | 当前公开配置所示基础模型 |
|---|---|---|
| CodeLlama | `sft_codellama_7b` | `meta-llama/CodeLlama-7b-Instruct-hf` |
| Llama 3 | `sft_llama3_3b` | 以 artifact 配置和权重哈希为准 |
| Qwen 2.5 | `sft_qwen_3b` | `Qwen/Qwen2.5-3B-Instruct` |

模型 ID、revision、adapter SHA-256、chat template、推理引擎和量化方式写入 `models.lock.json`。任一模型无法合法获取或 adapter 来源不能确认时，环境门失败；不能在看到结果后悄悄换模型。

统一 Prompt 只提供：完整源函数、数据布局/目标 triple、保持语义要求和 code-size 目标。不给 Alive2 反例、测试结果、`-Oz` 目标答案或其他模型输出。

顺序生成计划：

| 阶段 | 源函数 | 每个源函数/模型输出 | 解码 |
|---|---:|---:|---|
| G1 | D-B 160 | 1 | greedy，`do_sample=false` |
| G2 | D-B 160 | 再加 2 | `temperature=0.2, top_p=0.95`，种子 17、29 |
| G3 | D-BR 160 | 先 1，必要时再加 2 | 与 G1/G2 相同 |

只有某模型家族尚未达到候选供应目标时，才为该家族进入下一生成阶段。停止由失败候选数量触发，不读取 SalvageIR、B5 或 oracle 的成功结果。

同一源—模型最多三个输出。原始输出全部保存。完全相同的目标 IR 在生成漏斗中保留，但在 oracle 计算中按目标哈希只执行一次，统计时仍以源函数为聚类单位。

## 6. 候选状态机

按固定顺序分类，第一项匹配后停止：

| 状态 | 判定 | 核心恢复分母 |
|---|---|---|
| `FORMAT_INVALID` | 无法提取一个完整函数 | 否 |
| `IR_INVALID` | parser 或 LLVM verifier 失败 | 否 |
| `SIGNATURE_CHANGED` | 函数签名/ABI 改变 | 否 |
| `VERIFIER_UNKNOWN` | Alive2 timeout/unsupported/crash | 否 |
| `VERIFIED_COPY` | verified 且规范化后等于 S | 否 |
| `VERIFIED_NO_GAIN` | verified 但无主收益 | 否 |
| `VERIFIED_GAIN` | verified 且有主收益 | 否；另作模型质量报告 |
| `REFUTED_UNSUPPORTED_SCOPE` | Alive2 明确反驳，但超出核心范围 | 否；报告覆盖率 |
| `REFUTED_ELIGIBLE` | 可解析、范围支持、Alive2 给出明确反例 | 是 |

`REFUTED_ELIGIBLE` 必须保存反例、命令、stdout/stderr、工具哈希和运行时。测试失败或 fuzzing 找到输入不能替代 Alive2 明确反驳。

## 7. 对齐、组件和 oracle

### 7.1 对齐冻结

只在 D-U 和 D-B 的前 40 个哈希样本上调试对齐器。随后从 D-B 剩余候选中按模型和复杂度分层抽取 60 个进行人工审计。

审计单位为编辑原子，记录：

- 源/目标指令匹配是否正确；
- 插入、删除、替换和移动分类是否正确；
- 依赖边是否遗漏；
- coarse component 是否过度合并。

其中 20 个样本由两名审计者独立标注，Cohen's kappa 不低于 0.80；其余 40 个由一名审计者标注，所有分歧在不知道 oracle 结果的情况下裁决。审计通过条件：

- 编辑原子匹配 precision 不低于 95%；
- 语义关键依赖边 recall 不低于 98%；
- 60 个样本中不得出现会让不闭合组合进入 Alive2 的系统性错误。

达不到则只允许在开发样本上修正一次；修正后重新抽取未审计样本。不能在 D-C 上修正。

### 7.2 Oracle 上限

`n_oracle` 冻结为 12 个候选编辑组件。最多有 `2^12=4096` 个原始子集；实际只枚举满足依赖闭包的唯一状态。

每个状态依次执行：

1. 组件闭包检查；
2. IR compose；
3. LLVM verifier；
4. `Alive2(S,R)`；
5. 只有 verified 时才生成对象并读取函数符号大小。

源程序状态和完整失败目标不算 strict subset。完全回退到 S 即使 verified 也不算恢复。

`n > 12` 的候选只做组件分布和预算搜索可行性描述，不进入 oracle 是否存在的主要分母。不得通过结果导向的组件合并把它们塞进 oracle。

## 8. 工具和资源预算

工具链选择规则：冻结日选择 Alive2 明确支持的最新 LLVM release branch；记录 LLVM、Clang、Alive2、Z3 的 commit、构建参数、容器摘要和宿主 CPU。artifact 工具链只用于复现，不作为主工具链。

默认单状态限制：

| 资源 | 上限 |
|---|---:|
| LLVM verifier | 5 秒 |
| Alive2 | 30 秒 |
| 对象生成与读取 | 10 秒 |
| RSS | 8 GiB |

Oracle 单候选限制：

| 资源 | 上限 |
|---|---:|
| 依赖闭合状态 | 4096 |
| 墙钟 | 6 小时 |
| Alive2 timeout/unsupported 比例 | 20% |

超过单候选限制标为 `ORACLE_INCOMPLETE`，不能作为 `NO_VERIFIED_SUBSET`。P0 主比例只使用 oracle complete 候选，同时给出把 incomplete 全算失败和全算成功的上下界。

P1 搜索预算预冻结为：

```text
B_verify = 128 direct Alive2 calls
B_states = 256 unique composed states
B_wall   = 15 minutes on one pinned CPU core
B_rss    = 8 GiB
```

任一预算先到即停止。所有确定性基线共享这些预算；LLM 修复基线另报告 Token 和请求数，并同时受相同墙钟限制。

## 9. P0 门控

### E0：环境与复现门

必须全部满足：

- 数据、模型、工具和容器均有内容哈希；
- 从 artifact 按哈希抽取 100 个记录，parser/verifier 标签一致率不低于 99%；
- artifact Alive2 终态复现率不低于 95%；
- 同一函数对象三次构建的目标符号字节完全一致；
- 任一 provenance 或许可证不明的模型/数据被排除并记录。

若 Alive2 复现率不足 95%，D-A 只能作为原始候选来源，不能复用历史标签；所有候选必须用主工具链重新归类。

### G0：失败候选供应门

G0 只使用目标一致的 D-B/D-BR 受控候选；D-A artifact 不能替受控数据凑数。完成预声明的顺序扩样后，必须满足：

- 三个模型家族各至少 30 个唯一 `REFUTED_ELIGIBLE`；
- 总计至少 120 个唯一 `REFUTED_ELIGIBLE`；
- 来自至少 60 个源函数和 12 个项目；
- 单一项目不超过候选的 15%；
- 单一源函数不超过候选的 3%。

若达到最大 2,880 次受控生成仍未满足，停止“跨模型通用”主张。可以收窄为有足量失败候选的模型范围，但必须改写论文问题。

### G1：可分解与 oracle 覆盖门

G1 和 G2 同样只以 D-B/D-BR 的 code-size 受控候选作主要分母；D-A 结果平行报告。必须满足：

- 对齐人工审计通过第 7.1 节标准；
- 至少 60 个 oracle-complete 候选；
- 每个模型家族至少 15 个 oracle-complete 候选；
- oracle-complete 候选来自至少 30 个源函数和 12 个项目；
- oracle-complete 候选至少占核心 `REFUTED_ELIGIBLE` 的 30%；
- 至少 50% 的核心候选有 2–20 个组件，而不是全部原子化或极度碎片化；
- `ORACLE_INCOMPLETE` 不超过进入 oracle 候选的 20%。

若失败，先判断是组件粒度问题还是候选天然不可分。只能在开发集修改一次组件规则；确认集不得修改。

### G2：现象存在门

主要端点为：

```text
OracleEligibleUsefulRate =
  存在 profitable verified strict subset 的 oracle-complete 候选
  / 全部 oracle-complete REFUTED_ELIGIBLE 候选
```

统计单位为源函数，按项目做 10,000 次 cluster bootstrap，随机种子固定为 `20260920`；模型结果先分别计算再做等权宏平均。实用最小效应 `p_min` 预注册为 5%。

由于 `n <= 12` 可能偏向简单候选，必须同时对全部核心 `REFUTED_ELIGIBLE` 报告部分识别界：

```text
lower_all = oracle_positive / all_core_eligible
upper_all = (oracle_positive + non_oracle + oracle_incomplete) / all_core_eligible
```

P0 只证明 oracle-eligible 范围内存在现象；在预算搜索覆盖更复杂候选前，不得把该比例外推到全部失败候选。

判定：

| 结果 | 动作 |
|---|---|
| 95% CI 下界 `> 5%`，点估计 `>= 10%`，且至少两个家族为正 | PASS，进入 P1 |
| 95% CI 上界 `<= 5%` | STOP，停止 SalvageIR 主线 |
| 区间跨越 5% | INCONCLUSIVE，只允许按预声明保留池扩样一次 |

此外，单一项目、模板或一个模型家族不得贡献超过 50% 的成功案例；否则只能形成范围受限结论。

### G3：搜索可行性门

必须满足：

- 至少 75% 的核心候选组件数不超过 20；
- 单状态 Alive2 墙钟中位数不超过 5 秒，P90 不超过 30 秒；
- timeout/unsupported 不超过所有尝试状态的 10%；
- 在 oracle-positive 候选上，最优解不是全部只位于保留一个组件的极端状态；若超过 80% 只能保留一个组件，论文应改写为错误编辑定位，而非组合回收。

G3 不比较 SalvageIR 与 B5；比较留给 P1。

## 10. P1 机制门

P1 在冻结组件规则后运行，优先使用未触碰 D-C。所有方法从同一个 `(S,T0)` 开始并共享第 8 节预算。

主比较为 SalvageIR 对 B5（依赖闭合 best-first、无反例切片）。主端点是模型宏平均 `UsefulSalvageRate` 的配对差：

```text
Delta = UsefulSalvageRate_SalvageIR - UsefulSalvageRate_B5
```

反例引导贡献成立必须同时满足：

- `Delta >= 5` 个百分点；
- 按源函数/项目聚类的 95% bootstrap CI 下界大于 0；
- 三个模型家族中至少两个 `Delta > 0`；
- 在共同成功案例上，SalvageIR 的收益距 oracle 不显著恶化；
- 达到首个 profitable verified 结果的验证调用数中位数不高于 B5。

正式确认集还必须对 `Delta=5pp` 达到至少 80% 的估计功效；功效不足时结果标为 INCONCLUSIVE，不能用“不显著”证明无效，也不能用点估计宣称成功。

若只优于文本 ddmin 或随机回滚而不优于 B5，删除“反例引导算法”主张，降级为 LLVM-aware structured rollback。

主要消融：

- `-CounterexampleSlice`；
- `-UBClosure`；
- `-ProfitRecovery`；
- coarse component 与逐指令 component；
- 从全失败目标回滚与从源程序逐步加入。

消融不要求每项都达到显著性，但删除反例切片后若主要结果和验证成本都没有实质变化，就不能把 counterexample guidance 写成贡献。

## 11. 统计与缺失数据

- 生成候选漏斗按模型报告 Wilson 95% CI；
- 恢复率以源函数为簇、项目为外层簇 bootstrap；
- 多模型主结果采用模型等权宏平均，不按候选数量加权；
- P0 只有一个主要端点，不为描述性次要指标做多重校正；
- P1 多基线次要检验采用 Holm 校正；
- `UNKNOWN`、timeout 和 unsupported 永不改写为 verified；
- oracle incomplete 同时给 best-case/worst-case 界；
- 同一源函数的多个模型和多个采样输出不能当作独立程序；
- 顺序扩样只看供应量，禁止按显著性或恢复成功率提前停止。

报告三个不同分母：

1. `candidate-conditional`：全部 `REFUTED_ELIGIBLE`；
2. `model funnel`：全部模型原始输出；
3. `project end-to-end`：全部被冻结源函数，失败方法按回退源程序计零收益。

## 12. RISC-V 外部验证门

RISC-V 不参与 Prompt 调试、组件规则、P0/P1 阈值或主论文模型选择。方法和预算冻结后，在 D-H 与 D-C 的保留部分执行两种评价：

1. **零调参迁移：**直接把已由 x86-64 选择的结果编译到 RV64GC；
2. **后端替换：**保持搜索算法和预算不变，只把成本函数替换成 RV64GC 目标函数符号大小。

冻结基线 ISA：

```text
riscv64-unknown-linux-gnu
-march=rv64gc
-mabi=lp64d
generic CPU
```

RVV 作为 `rv64gcv` 次要分析，不与 RV64GC 合并。使用相同 `ST_Size` 口径；不把 QEMU 时间作为主指标。

可声称“在 RISC-V 下有指标提升”必须满足：

- 至少 95% 的核心候选能够生成 RV64GC 对象；
- 直接 `Alive2(S,R)` 仍为 verified；
- 以未恢复候选回退到 S 计零收益后，模型宏平均 code-size delta 小于 0；
- 项目聚类 95% CI 上界小于 0；
- 至少两个模型家族方向一致。

若只在挑出的成功案例上有提升，而端到端宏平均区间包含 0，只能写“存在 RISC-V 可迁移案例”，不能写框架整体提升。

## 13. 最终红黄绿决策

| 状态 | 条件 | 研究动作 |
|---|---|---|
| GREEN | E0、G0、G1、G2、G3 全通过 | 实现完整 SalvageIR 并进入 P1 |
| YELLOW-A | G2 通过但只覆盖两家族或窄 IR 子集 | 收窄论文范围，保留机制研究 |
| YELLOW-B | 现象存在但 G3 失败 | 先做组件压缩/验证缓存，不宣称可扩展 |
| YELLOW-C | SalvageIR 与 B5 近似 | 降级为结构化回滚工具 |
| RED-A | G2 的 95% CI 上界不超过 5% | 停止 SalvageIR 主线 |
| RED-B | 对齐审计或完整函数验证不可靠 | 停止实验，先修基础设施 |
| RED-C | 收益只存在于代理指标 | 删除性能回收主张 |

任何 RED 结果不得通过加入 Agent、RAG、第二个 LLM、RL 或 RISC-V 特例来改名规避。

## 14. P0 必须输出的文件

```text
locks/
  toolchain.lock.json
  models.lock.json
  datasets.lock.json
manifests/
  source_functions.csv
  raw_generations.csv
  candidate_funnel_by_model.csv
  dedup_clusters.csv
pilot/
  alignment_audit.csv
  component_distribution.csv
  oracle_salvageability.csv
  verifier_cost.csv
  gate_decision.json
  gate_decision.md
```

`gate_decision.json` 必须保存每一门的分子、分母、区间、阈值、PASS/INCONCLUSIVE/STOP 和失败原因。没有该文件不得开始 P1。

## 15. 预注册后不可更改的字段

- 数据集 revision、项目和函数选择算法；
- 模型家族、adapter、Prompt 和解码阶段；
- 主目标与 `profitable` 定义；
- `n_oracle=12`；
- timeout 和搜索预算；
- `p_min=5%`、P0 三态判定和 P1 `Delta=5pp`；
- 分母、聚类单位和 bootstrap seed `20260920`；
- RISC-V 的进入时点与评价规则。

若工具缺陷迫使更改，必须递增协议版本、说明原因，并重新选择未触碰确认集；不能覆盖原协议和结果。

## 16. 选择理由与被放弃方案

### 16.1 为什么不用单一 artifact

只使用 LLM-VeriOpt 最省成本，也能得到真实失败输出；但其候选目标是 latency，与本研究冻结的 code-size 主目标不一致，而且模型和测试集由邻近工作决定。故 D-A 只能证明现象可重放，不能承担主要盈利结论。

### 16.2 为什么不用一个混合大测试集

把 IR-OptSet、LLVM Opt Benchmark 和 test-suite 全部随机混合会造成项目泄漏，也会在看见结果后留下调整空间。分成发现集、保留池、确认集和端到端集，可以把调试、扩样、确认和跨后端评价隔离。

### 16.3 为什么不用纯合成程序

合成案例能精确覆盖 UB、PHI 和 poison，但很容易人为制造“可分错误”。因此 D-U 只验证实现，不进入任何效果分母。

### 16.4 数值门槛的含义

- `n_oracle=12` 对应最多 4096 个原始子集，是穷举与真实性之间的计算边界；
- 每家族 30 个、总计 120 个失败候选是分层描述下限，真正的不确定性仍由置信区间表示；
- `p_min=5%` 是实用而非显著性门：低于每 20 个失败候选约一个可回收案例时，系统复杂度很难支撑通用后处理主张；
- P1 的 `Delta=5pp` 要求反例引导相对强结构基线每 20 个候选至少多恢复约一个，避免把极小差异包装成算法贡献；
- 128 次验证调用只占 12 组件完整布尔空间的 3.125%，用于检验方法是否真正减少验证，而不是近似穷举；
- `max(2 bytes, 1%)` 排除一字节边缘案例，同时保留小函数和大函数可比性；allocatable non-BSS 非回归条件防止把代码搬到常量表。

这些阈值可以在任何候选结果生成之前因硬件预算重新发布协议版本，但第一条模型输出产生后不得再改。
