# 历史归档：v2，不得用于新实验执行

原文件：CODEX_P0A_EXECUTION_HANDOFF.md。2026-09-21 被 v3 替代；保留原文用于追溯，不代表当前约束。

# SalvageIR P0a 预实验实施计划与 Codex 执行工单

> **交给 Codex 的直接指令：**从本文开始执行，不要重新设计 SalvageIR，不要把范围扩大到 P0b/P0c/P1/P2，也不要以缺少完整实验为由只写计划不落代码。先完成只读环境审计，再按任务顺序以测试驱动方式实现 P0a；遇到需要下载大型 artifact、模型权重、安装工具链、接受许可证或启动长时 GPU 作业时，列出精确命令、大小、许可证与预计耗时并请求一次授权。其余本地、可逆、范围内工作持续推进，直到产生 P0a 裁决报告或命中本文的硬停止条件。

> **For agentic workers:** REQUIRED SKILLS: use the ARIS project-local `experiment-bridge` as the main workflow, with `AUTO_DEPLOY=false` until external resources and long-running jobs are explicitly approved; use `run-experiment` only after sanity passes, and `experiment-audit` before accepting results. Do not use `experiment-queue` for the CPU/toolchain smoke stages; use it only when an approved SSH backend exists and Track B generation is materialized as at least 10 independent GPU jobs.

**Goal:** 在不训练模型、不使用 RL、不开启 RISC-V 实验的前提下，搭建并实际运行 SalvageIR 的 P0a smoke，验证 parser/verifier、IR 对齐、结构可重放组件、Alive2 反例适配器和 x86-64 代码尺寸测量链是否可靠到足以进入 P0b。

**Architecture:** 文献仓库中的 SalvageIR 文档是不可修改的研究契约；ARIS 只负责实施、运行、审计和产物管理。实验实现放在 ARIS 根目录下的独立工作区，所有输入、工具、模型和输出均以内容哈希连接。正确性只由锁定工具链上的整函数 `Alive2(S,R)` 裁决；P0a 不实现最终 SalvageIR 搜索，也不比较 B5/B6/B7。

**Tech Stack:** Python 3.11、pytest、LLVM/Clang/LLC/llvm-readobj、Alive2 `alive-tv`、Z3、Git、Hugging Face/Transformers（仅经授权后的 Track B 推理）、ARIS experiment workflow。

## Material Passport

- Origin Skill: academic-research-suite / experiment-agent + ARIS experiment-bridge
- Origin Mode: plan-to-run handoff
- Origin Date: 2026-09-20
- Verification Status: UNVERIFIED — 本文件是执行契约，不是实验结果
- Version Label: `salvageir_p0a_handoff_v1`
- Source of Truth: `SalvageIR_验证失败候选部分回收框架` 的冻结研究文档

## 1. 启动方式

将本文路径完整交给 Codex，并使用下面的启动语句：

```text
请在 C:\Users\2025111355\Desktop\Auto-claude-code-research-in-sleep 中使用项目本地
experiment-bridge，严格执行
C:\Users\2025111355\Desktop\文献\AI编译器与RISC-V优化相关文献\研究框架\SalvageIR_验证失败候选部分回收框架\CODEX_P0A_EXECUTION_HANDOFF.md。
只做 P0a；先进行只读 preflight，再实施和运行。不要覆盖 ARIS 根目录现有的 UnlockIR 文件。
除大型下载、安装、许可证/凭据和长时 GPU 作业外，不要中途停在计划阶段。
```

Codex 的第一条工作更新必须给出：

1. 已读取的契约文件及其 SHA-256；
2. ARIS 与实验工作区的 Git 状态；
3. LLVM、Alive2、Z3、Python、GPU 的可用性；
4. 当前可直接执行的任务与需要授权的外部资源；
5. 明确声明“当前只执行 P0a，不运行 RISC-V”。

## 2. 两个仓库的职责与隔离

### 2.1 研究规范仓库：只读事实源

规范仓库：

```text
C:\Users\2025111355\Desktop\文献\AI编译器与RISC-V优化相关文献
```

必须按以下顺序完整阅读：

1. `研究框架/SalvageIR_验证失败候选部分回收框架/RESEARCH_CONTRACT.md`
2. `研究框架/SalvageIR_验证失败候选部分回收框架/PILOT_PROTOCOL.md`
3. `研究框架/SalvageIR_验证失败候选部分回收框架/GATE_CARD.md`
4. `研究框架/SalvageIR_验证失败候选部分回收框架/ALGORITHM_SPEC.md`
5. `研究框架/SalvageIR_验证失败候选部分回收框架/DATA_AND_FAIRNESS_PROTOCOL.md`
6. `研究框架/SalvageIR_验证失败候选部分回收框架/EXPERIMENT_PLAN.md`

这些文件是研究契约。P0a 执行者不得为迁就实现结果而修改阈值、样本选择、成功定义或候选分类。发现矛盾时，先按 `GATE_CARD.md` 和版本更新的 `PILOT_PROTOCOL.md` 执行，同时写入 `refine-logs/P0A_PROTOCOL_DEVIATIONS.md`；不得静默选择对结果有利的解释。

### 2.2 ARIS：实施与运行框架

ARIS 根目录：

```text
C:\Users\2025111355\Desktop\Auto-claude-code-research-in-sleep
```

先完整阅读：

```text
AGENTS.md
.agents/skills/experiment-bridge/SKILL.md
.agents/skills/run-experiment/SKILL.md
.agents/skills/experiment-audit/SKILL.md
.agents/skills/experiment-queue/SKILL.md
```

ARIS 根目录当前可能已有属于其他研究的 `refine-logs/EXPERIMENT_PLAN.md`、`FINAL_PROPOSAL.md` 或未跟踪文件。它们不是 SalvageIR 输入，禁止覆盖、移动、删除或纳入 SalvageIR 提交。

### 2.3 独立实验工作区

在 ARIS 根目录下建立唯一工作区：

```text
C:\Users\2025111355\Desktop\Auto-claude-code-research-in-sleep\workspaces\salvageir-p0a
```

所有新增代码、运行记录和数据清单都写入该目录。不要在 ARIS 根目录的 `refine-logs/` 中写 SalvageIR 结果。若目录已经存在，先检查内容与 Git 状态；保留用户文件，在原工作区继续，不得另建名称近似的重复目录。

## 3. 不可漂移的研究边界

### 3.1 P0a 要回答的问题

P0a 只验证：

- 冻结工具链能否稳定分类模型输出；
- IR 对齐器能否恢复已知编辑原子并形成依赖闭合组件；
- Alive2 明确反例能否稳定规范化为三态记录；
- context-preserving harness 能否给出可复现且经 `clang -Oz` 校准的函数机器码字节；
- 三个冻结模型在同一 10 个源函数上的候选生成和完整漏斗能否被可靠记录。

P0a 不回答“SalvageIR 是否优于基线”，也不允许声称存在普遍可回收现象。

### 3.2 明确禁止

- 不做 SFT、DPO、PPO、GRPO、MCTS 或任何任务特定训练/RL；
- 不实现或运行完整 SalvageIR 搜索、B5、B6、B7、MaxSAT/ILP 或最终论文消融；
- 不运行 D-B 全量、D-BR、D-C、D-N、D-H 端到端系统实验；
- 不运行 RV64GC/RVV，不把 RISC-V 作为 P0a 故障修补路径；
- 不把 parser error、timeout、unsupported、崩溃或 fuzz 未发现错误计作正确；
- 不把返回源 IR、仅有 IR 指令数下降、仅优于错误候选计作有益回收；
- 不调用第二个 LLM 修复候选；
- 不用手工挑选“看起来容易”的样本，不因观察结果更换模型或函数；
- 不为了过门降低 `95%`、`98%`、`100%`、`kappa=0.80` 等阈值。

### 3.3 固定成功口径

任何后续试验性恢复结果只有同时满足下式才能标成 `SALVAGED_PROFITABLE`；P0a 自身不需要产出该结果：

```text
LLVMVerify(R) = PASS
Alive2(S, R) = VERIFIED
size(S) - size(R) >= max(2 bytes, ceil(0.01 * size(S)))
alloc_non_bss(R) <= alloc_non_bss(S)
```

## 4. P0a 固定工作量

### 4.1 D-U：全部执行

构造并执行 112 对不进入效果统计的单元样本：

- 48 个人工 IR 对：8 类 × 每类 6 例；
- 64 个 provenance 变异对：来自 16 个 verified 自然变换，每个 4 个冻结 mutation。

八类必须齐全：

1. SSA def-use；
2. CFG、PHI 与控制依赖；
3. `nsw/nuw/exact/inbounds`；
4. poison、undef 与 freeze；
5. load/store 与别名；
6. 函数属性、调用与 ABI；
7. 浮点与 fast-math flags；
8. 向量与目标相关 intrinsic。

每类同时包含可分、不可分、需要联合回滚三种结构。超出核心支持范围的例子用于验证拒绝路径，不能被删掉来提高覆盖率。

### 4.2 D-A：两个互不偷换的样本

在获得 LLM-VeriOpt artifact 后，先锁定 artifact revision、档案 SHA-256、许可证和原工具链。

**E0 标签复现样本：100 条。**在允许的非 RL 配置中按家族分层，使用以下不可变键排序：

```text
SHA256(artifact_revision | config_id | source_id | generation_index | raw_output_sha256)
```

按 CodeLlama 34、Llama 3 33、Qwen 2.5 33 取样；某家族不足配额时不跨家族补齐，写 `INSUFFICIENT_ARTIFACT_STRATUM`，E0 不通过。100 条用于 parser/verifier 标签一致率和 Alive2 终态复现率，不能由 30 条失败候选替代。

**结构烟测样本：30 条。**只从允许的、用冻结主工具链重放后仍为 `REFUTED_ELIGIBLE` 的记录中选择，每家族取上述哈希最小的 10 条；同一 `source_id` 最多 2 条。某家族不足 10 条时不跨家族补齐，保留真实不足结论。

公开 artifact 原目标是 latency。其结果只用于复现、对齐和语义回收管线烟测，不进入 code-size 盈利率。

### 4.3 Track B：同一组 10 个源函数

从冻结规则产生的 D-B 候选池中，在任何模型调用前按 `PILOT_PROTOCOL.md` 的源函数哈希顺序取前 10 个合格函数，形成 `D-B-smoke-10`。三个模型必须使用完全相同的 10 个函数：

```text
meta-llama/CodeLlama-7b-Instruct-hf
meta-llama/Llama-3.2-3B-Instruct
Qwen/Qwen2.5-Coder-3B-Instruct
```

P0a 每个模型、每个源函数只生成 1 个 greedy 输出，总计 30 个原始输出：

```text
do_sample=false
temperature omitted
top_p omitted
max_new_tokens=4096
max_raw_output_chars=32768
prompt_version=salvageir_codesize_v1
```

推理前解析并锁定每个模型的不可变 revision、权重文件哈希、许可证、chat template、Transformers/PyTorch/CUDA 版本、dtype 和量化方式。首选顺序加载、`float16`、不量化；若单张 GPU 不能容纳，不得静默切换量化或远程服务，必须先提交协议偏差和新成本说明。

本研究自身不做训练；模型卡披露的上游对齐历史必须原样记录，不得误写为“模型从未使用 RL”。

### 4.4 成本校准：至少 30 个函数

从冻结 D-H 的 24 个程序中按程序轮转选择至少 30 个可比函数：每个程序内按函数稳定哈希排序，每轮每程序最多增加 1 个，同一程序最多 2 个，直到达到 30。不得按校准是否一致挑函数。

对每个函数执行：

1. `clang -Oz` 参考路径；
2. `default<Oz>` + `size-harness` + `llc -O=2` IR 路径；
3. 抽取 harness 与原模块路径；
4. 相同输入连续三次对象构建。

只有目标函数 `ST_Size` 唯一且非零、上下文闭合的函数进入可比样本；排除原因全部保留。

## 5. P0a 唯一门控

P0a 的输出是基础设施裁决，不是 F/I/S 论文裁决。

| 检查 | 通过阈值 | 失败动作 |
|---|---:|---|
| artifact parser/verifier 标签一致率 | `>=99%`，样本 100 | `STOP_ENV_LABELS` |
| artifact Alive2 终态复现率 | `>=95%`，样本 100 | 历史标签禁用；主工具链全量重分 |
| 对象确定性 | 每个校准输入三次逐字节一致 | `STOP_ENV_NONDETERMINISTIC` |
| IR 成本路径 vs `clang -Oz` | 可比机器码完全一致率 `>=95%` | `STOP_ENV_COST` |
| 抽取 harness vs 原模块 | 可比目标符号字节一致率 `>=95%` | `STOP_ENV_CONTEXT` |
| provenance 编辑原子 | precision/recall 均 `>=98%` | `STOP_ENV_ALIGNMENT` |
| 预声明结构依赖边 | recall `=100%` | `STOP_ENV_CLOSURE` |
| 自然样本对齐审计 | Cohen's kappa `>=0.80` | `STOP_ENV_ALIGNMENT` |
| 反例规范化双跑一致率 | `>=95%` | `STOP_ENV_COUNTEREXAMPLE` |

`CE_LOCALIZABLE >=60%` 是后续 G1 的现象覆盖门。P0a 必须报告该比例，但 30 条烟测不足以宣告 G1 通过。

P0a 最终状态只能取：

- `P0A_PASS_READY_FOR_P0B`：以上环境门全部通过，30 个 Track B 输出和 30 个 D-A 结构样本均形成完整漏斗/审计记录；
- `P0A_PARTIAL_EXTERNAL_RESOURCE`：本地实现与 D-U 通过，但经授权的 artifact、模型或 D-H 资源尚未齐全；
- `P0A_STOP_ENV`：至少一个硬环境门失败；
- `P0A_BLOCKED_LICENSE_OR_ACCESS`：许可证、模型访问或凭据阻止了预注册资源获取。

不得把 `PARTIAL` 写成 `PASS`。

## 6. 目标目录和文件契约

在独立工作区创建以下结构。不要把大型权重、数据档案或构建产物提交到 Git。

```text
workspaces/salvageir-p0a/
├── AGENTS.md
├── README.md
├── pyproject.toml
├── .gitignore
├── configs/
│   ├── p0a.yaml
│   ├── prompt_salvageir_codesize_v1.txt
│   └── logging.yaml
├── locks/
│   ├── protocol.lock.json
│   ├── toolchain.lock.json
│   ├── datasets.lock.json
│   └── models.lock.json
├── manifests/
│   ├── du_cases.jsonl
│   ├── da_replay_100.jsonl
│   ├── da_structure_30.jsonl
│   ├── db_smoke_10.jsonl
│   └── dh_calibration_30.jsonl
├── src/salvageir_p0a/
│   ├── __init__.py
│   ├── cli.py
│   ├── schema.py
│   ├── hashing.py
│   ├── command_runner.py
│   ├── ir_normalizer.py
│   ├── candidate_funnel.py
│   ├── alive2_adapter.py
│   ├── ir_aligner.py
│   ├── component_graph.py
│   ├── composer.py
│   ├── context_harness.py
│   ├── cost_measure.py
│   ├── model_generate.py
│   ├── audit_metrics.py
│   └── p0a_report.py
├── tests/
│   ├── unit/
│   ├── fixtures/du/
│   ├── fixtures/tool_logs/
│   └── integration/
├── scripts/
│   ├── preflight.ps1
│   └── reproduce_p0a.ps1
├── refine-logs/
│   ├── EXPERIMENT_LOG.md
│   ├── EXPERIMENT_TRACKER.md
│   ├── EXPERIMENT_CODE_REVIEW.md
│   ├── P0A_PROTOCOL_DEVIATIONS.md
│   └── P0A_DECISION.md
└── runs/
    └── p0a-YYYYMMDDThhmmssZ/
        ├── run_manifest.json
        ├── commands.jsonl
        ├── stdout/
        ├── stderr/
        ├── raw/
        ├── normalized/
        ├── metrics/
        └── P0A_DECISION.json
```

`runs/` 只保存小型日志、哈希和结构化结果；大数据和模型缓存使用机器现有缓存或用户批准的位置，并在 lock 文件中记录绝对路径与哈希。

## 7. 结构化数据接口

使用 Python dataclass 或 Pydantic 实现以下逻辑记录；磁盘格式为 JSONL/JSON，字段名不得随意改写。

### 7.1 `SourceRecord`

```text
source_id, dataset_id, dataset_revision, project, project_commit,
module, function, language, license, split, domain, instruction_bin,
raw_sha256, normalized_sha256, llvm_version, datalayout, target_triple,
eligible, exclusion_reason
```

### 7.2 `CandidateRecord`

```text
candidate_id, source_id, track, model_id, model_revision,
prompt_version, decode_config, generation_index, raw_output_sha256,
extracted_ir_sha256, parse_status, verify_status, alive_status,
alive_log_sha256, candidate_class, scope_status, component_count,
split, frozen
```

候选漏斗枚举固定为：

```text
FORMAT_INVALID
IR_INVALID
REFUTED_ELIGIBLE
REFUTED_UNSUPPORTED_SCOPE
UNKNOWN_TIMEOUT
UNKNOWN_UNSUPPORTED
VERIFIED_COPY
VERIFIED_NO_GAIN
VERIFIED_PROFITABLE
```

### 7.3 `CounterexampleRecord`

```text
candidate_id, state_id, alive2_version, source_hash, target_hash,
failure_kind, witness_inputs, source_observation, target_observation,
ub_poison_facts, memory_events, raw_log_hash, normalized_record_hash,
localization_state, downgrade_reason
```

`localization_state` 只能是：

```text
CE_LOCALIZABLE
CE_STATIC_ONLY
CE_UNLOCALIZABLE
```

同一 `(S,T0)` 独立运行两次；只有 failure kind、规范化 witness 和观察差异一致时才能标为 `CE_LOCALIZABLE`。

### 7.4 `AlignmentAuditRecord`

```text
pair_id, expected_atoms, recovered_atoms, true_positive_atoms,
false_positive_atoms, false_negative_atoms, expected_dependency_edges,
recovered_dependency_edges, atom_precision, atom_recall,
dependency_recall, annotator_a, annotator_b, adjudication, notes
```

### 7.5 `CostRecord`

```text
source_id, function, path_kind, object_sha256, target_symbol,
st_size, alloc_non_bss, code_bytes, rodata_bytes, build_index,
command_id, comparable, exclusion_reason
```

### 7.6 `CommandRecord`

```text
command_id, argv, cwd, environment_allowlist, started_at, ended_at,
wall_ms, exit_code, timed_out, stdout_path, stderr_path,
input_hashes, output_hashes, tool_versions
```

不要把 token、用户名、完整环境变量或凭据写入日志。命令必须以 argv 数组保存，不能只留一段不可解析的 shell 文本。

## 8. 实施任务与提交检查点

严格按顺序执行。每项先写失败测试，再写最小实现，再运行测试并保存输出。一次提交只包含 SalvageIR P0a 自己的文件。

### Task 0：冻结规范与只读 preflight

**创建：**

- `locks/protocol.lock.json`
- `locks/toolchain.lock.json`
- `refine-logs/EXPERIMENT_LOG.md`
- `refine-logs/EXPERIMENT_TRACKER.md`

**动作：**

1. 记录六个研究契约文件的绝对路径、Git commit、SHA-256 和读取时间；
2. 记录 ARIS commit、OS、CPU、RAM、可用磁盘、GPU；
3. 只读探测 `python`、`git`、`clang`、`opt`、`llc`、`llvm-readobj`、`llvm-dis`、`alive-tv`、`z3`、`nvidia-smi`；
4. 检查 LLVM 各组件 major version 是否一致；
5. 检查现有缓存中是否已有三个模型、LLM-VeriOpt、IR-OptSet 和 LLVM test-suite；
6. 不安装、不下载、不修改系统 PATH。

**必须保存的命令族：**

```powershell
python --version
git --version
clang --version
opt --version
llc --version
llvm-readobj --version
llvm-dis --version
alive-tv --version
z3 --version
nvidia-smi
```

某个命令不存在时，以 `MISSING` 记录，不要用另一个未锁定版本悄悄替代。把缺失项分为“可继续实施、暂不能集成测试”和“阻止 P0a 运行”。

**检查点提交：**

```text
由Codex提交：实验：冻结SalvageIR P0a规范与环境清单
```

### Task 1：建立可测试骨架和数据 schema

**测试先行：**

- 所有枚举拒绝未知值；
- JSONL round trip 不丢字段；
- 同一输入生成相同 ID；
- `candidate_id` 包含 source/model/revision/prompt/decode/index/raw output；
- 日志环境变量只保留显式 allowlist；
- Windows 路径和 UTF-8 中文路径可 round trip。

**命令：**

```powershell
python -m pytest tests/unit/test_schema.py tests/unit/test_hashing.py -q
```

**检查点提交：**

```text
由Codex提交：实验：搭建SalvageIR P0a骨架与数据契约
```

### Task 2：可复现命令运行器与候选漏斗

**测试先行：**

- 正常退出、非零退出、timeout、工具崩溃分别记录；
- stdout/stderr 原样保存并计算哈希；
- parser error 不会被映射为 Alive2 refuted；
- timeout/unsupported 不会被映射为 verified；
- 规范化后无语义变化的输出进入 `VERIFIED_COPY`；
- 目标函数不可唯一定位时进入 `COST_UNMEASURABLE` 辅助状态，不构造收益。

所有外部命令必须经过 `command_runner.py`，不得在各模块里散落 subprocess 调用。

**命令：**

```powershell
python -m pytest tests/unit/test_command_runner.py tests/unit/test_candidate_funnel.py -q
```

**检查点提交：**

```text
由Codex提交：实验：实现可审计命令运行与候选漏斗
```

### Task 3：Alive2 适配器与双跑稳定性

**测试先行：**

- 用保存的 fixture 覆盖 return mismatch、target more undefined、memory mismatch、poison/undef、attribute/precondition mismatch；
- 同一原始日志得到同一 normalized hash；
- 双跑 witness 不一致自动从 `CE_LOCALIZABLE` 降级；
- 无法映射 witness 但明确 refuted 时为 `CE_STATIC_ONLY` 或 `CE_UNLOCALIZABLE`；
- unknown、timeout、unsupported 保留原终态。

禁止依赖易变的人类日志文本作为唯一解析入口；同时保存工具原始输出和版本化 parser 结果。

**命令：**

```powershell
python -m pytest tests/unit/test_alive2_adapter.py tests/integration/test_alive2_double_run.py -q
```

**检查点提交：**

```text
由Codex提交：实验：实现Alive2反例三态适配器
```

### Task 4：D-U、IR 对齐、组件闭包和 composer

**测试先行：**

- `Build(all-source)` 与 `S` 规范同构；
- `Build(all-target)` 与 `T0` 规范同构；
- 每个 build 结果先通过 LLVM verifier 才能进入 Alive2；
- SSA、CFG/PHI、control、memory、UB、ABI 依赖闭包均有定向测试；
- 图中强连通分量被合并，最终组件图为 DAG；
- text rename 和非语义 metadata 不形成 edit atom；
- unsupported 范围进入可解释拒绝状态，不从分母消失。

生成 48+64 的 manifest 和 provenance ground truth。mutation script、seed、基底自然变换哈希必须入锁文件。人工 48 对须由第二次独立审计核对预期原子和依赖；若没有第二位人工标注者，可使用隔离的 fresh-agent review，但在报告中标为机器辅助审计，不能伪装成人际一致性。

**命令：**

```powershell
python -m pytest tests/unit/test_ir_aligner.py tests/unit/test_component_graph.py tests/unit/test_composer.py -q
python -m salvageir_p0a build-du --config configs/p0a.yaml
python -m salvageir_p0a audit-du --manifest manifests/du_cases.jsonl
```

**硬阈值：**编辑原子 precision/recall 均不低于 98%，预声明结构依赖边 recall 为 100%。未达标时停止自然样本效果运行，修对齐器；不得删难例。

**检查点提交：**

```text
由Codex提交：实验：实现D-U对齐与结构组件闭包
```

### Task 5：context-preserving size harness 与校准

`size-harness` 只能：

- 对 `S` 与候选同样添加 `optsize,minsize`；
- 保留目标函数可达声明、globals、aliases、comdat、datalayout 和属性；
- 形成可独立编译但语境闭合的模块。

不得改写函数体，不得删语义属性，不得按结果为某一路径添加额外优化。

冻结命令语义：

```text
opt -passes='default<Oz>' source.bc -o source.oz.bc
size-harness --function=<f> --add-attrs=optsize,minsize source.oz.bc -o source.cost.bc
llc -O=2 -mtriple=x86_64-unknown-linux-gnu -mcpu=generic -filetype=obj candidate.bc -o candidate.o
llvm-readobj --symbols --sections candidate.o
```

在实际实现中允许补充锁定版本所需的兼容参数，但 triple、CPU、优化级别和计量字段不得变化。

**测试先行：**

- 多函数、global、alias、comdat、常量池和函数属性 fixture；
- 符号缺失、合并、零大小、重名时 fail closed；
- `ST_Size` 与 alloc non-BSS 分开记录；
- 三次构建对象哈希逐字节一致；
- 校准比较只在同一目标函数和闭合语境上进行。

**命令：**

```powershell
python -m pytest tests/unit/test_context_harness.py tests/unit/test_cost_measure.py -q
python -m salvageir_p0a select-dh-calibration --config configs/p0a.yaml
python -m salvageir_p0a calibrate-cost --manifest manifests/dh_calibration_30.jsonl --repeat 3
```

未达到两个 95% 一致率，或对象不确定，写出 `P0A_STOP_ENV` 并停止 Track B 效果解释；可以继续修复基础设施，但不能跳门。

**检查点提交：**

```text
由Codex提交：实验：实现代码尺寸测量与双路径校准
```

### Task 6：D-A artifact 获取、100 条复现与 30 条结构烟测

先生成 `locks/datasets.lock.json` 的拟议条目，列出：

- GitHub/Zenodo 来源；
- 预计下载大小；
- 许可证；
- 目标缓存目录；
- 预期 archive checksum 或下载后核验方式；
- 仅允许的非 RL 配置和明确排除的 RL 配置。

1. 若本地没有完整 artifact，向用户请求一次明确下载授权；
2. 获批后下载到缓存，不提交档案；
3. 全量清点记录，再按第 4.2 节生成两个冻结 manifest；
4. 用 artifact 原工具链复现历史标签；
5. 用本研究主工具链重新分类；
6. 冲突全部报告，不选择有利标签；
7. 对 30 条结构烟测运行规范化、对齐、组件构造和 CE 双跑。

**命令接口：**

```powershell
python -m salvageir_p0a inventory-da --config configs/p0a.yaml
python -m salvageir_p0a select-da-replay --count 100 --output manifests/da_replay_100.jsonl
python -m salvageir_p0a replay-da --manifest manifests/da_replay_100.jsonl --toolchain both
python -m salvageir_p0a select-da-structure --per-family 10 --output manifests/da_structure_30.jsonl
python -m salvageir_p0a smoke-components --manifest manifests/da_structure_30.jsonl --ce-repeat 2
```

如原工具链不可复建，明确标记 `ARTIFACT_ORIGINAL_TOOLCHAIN_UNAVAILABLE`；主工具链重放仍可进行，但 E0 历史复现项不能伪装完整。

**检查点提交：**

```text
由Codex提交：实验：完成LLM-VeriOpt复现与结构烟测
```

### Task 7：Track B 10×3 受控生成

先完成源函数选择和模型锁文件，后生成输出。模型访问或权重下载需要授权时，把三个模型分别列出许可证、解析 revision、下载大小估计、缓存位置和预计显存。

统一 Prompt 只提供完整源函数、datalayout/target triple、保持语义要求和 code-size 目标；禁止提供 Alive2 反例、测试结果、`-Oz` 答案或其他模型输出。

生成前执行每个模型 1 个源函数的 sanity；sanity 必须验证：

- 推理进程正常退出；
- 原始响应完整保存；
- complete IR 抽取不截断；
- 生成元数据和哈希齐全；
- 显存与预计运行时间可接受；
- 同一源函数不会串入前一模型上下文。

通过后再生成剩余 27 个输出。按模型串行加载即可，不为并行而引入量化差异。

**命令接口：**

```powershell
python -m salvageir_p0a select-db-smoke --count 10 --output manifests/db_smoke_10.jsonl
python -m salvageir_p0a resolve-models --config configs/p0a.yaml
python -m salvageir_p0a generate-track-b --manifest manifests/db_smoke_10.jsonl --sanity-only
python -m salvageir_p0a generate-track-b --manifest manifests/db_smoke_10.jsonl --resume
python -m salvageir_p0a classify-track-b --manifest manifests/db_smoke_10.jsonl
```

若已配置并批准 SSH 后端，且总作业数以独立任务形式超过 10，可使用 ARIS `experiment-queue`；本地单 GPU 不套用 SSH 队列。manifest 的 `expected_output` 必须指向每条生成的结构化结果，且 `stuck` 不是成功。任何重试保留第一次输出和失败原因，不能覆盖。

**检查点提交：**

```text
由Codex提交：实验：完成Track B三模型P0a候选生成
```

### Task 8：整体复现、审计与 P0a 裁决

运行完整测试与 P0a 汇总：

```powershell
python -m pytest -q
python -m salvageir_p0a preflight --config configs/p0a.yaml
python -m salvageir_p0a audit-p0a --config configs/p0a.yaml
python -m salvageir_p0a report-p0a --config configs/p0a.yaml
```

然后使用 ARIS `experiment-audit` 对以下内容做独立审计：

- manifest 是否在运行前冻结；
- 30 条 D-A 与 30 条 Track B 是否没有结果驱动替换；
- 三模型是否同源函数、同 Prompt 信息、同生成次数；
- 所有 unknown 和 unsupported 是否保留；
- 原始日志、解析标签和哈希是否可追溯；
- 95%/98%/100%/kappa 阈值是否按分母正确计算；
- P0a 是否被错误外推成 F/I/S 结论。

审计发现 blocking issue 时先修复并重新运行受影响测试一次；保存审计前后差异。不得删除失败 run。

**最终检查点提交：**

```text
由Codex提交：实验：生成SalvageIR P0a预实验裁决
```

## 9. 测试与质量要求

### 9.1 最低测试集合

- schema/hash/runner 的纯单元测试；
- LLVM parser/verifier 集成测试；
- Alive2 每类终态和双跑测试；
- 112 个 D-U 参数化测试；
- context harness/符号解析/section 统计测试；
- 三次构建确定性测试；
- 候选漏斗互斥且完备测试；
- manifest 冻结后不可无痕修改测试；
- end-to-end 迷你 fixture：`source -> candidate -> classify -> align -> components -> CE -> report`。

### 9.2 失败处理

一次运行失败后，先读原始 stderr/日志再修改。不得原命令无变化盲目重跑。相同故障经两次有依据修补仍失败时，可以重写仅由 Codex 新建的失败模块；不得删除规范、manifest、输入、日志或已有结果。

自动重试只允许：

- 明确的暂时网络中断，且下载器支持 checksum/断点续传；
- experiment-queue 明确识别的 OOM，且重试策略预先写入 manifest；
- 不改变输入、工具链和配置的进程级瞬态失败。

语义不一致、标签不一致、校准失败和非确定性不能靠重试洗掉。

## 10. 外部资源授权单

Codex 在第一次需要外部资源时应一次性列出，而不是逐个打断：

| 资源 | 用途 | 需要说明 |
|---|---|---|
| LLM-VeriOpt Git/Zenodo artifact | D-A 100+30 | URL、revision、约 1.1 GB 档案、许可证、checksum |
| IR-OptSet snapshot | 选择 D-B-smoke-10 | Hugging Face revision、Parquet hash、许可证、预计大小 |
| LLVM test-suite snapshot | D-H 30 函数成本校准 | Git commit、许可证、预计构建空间 |
| 三个模型权重 | Track B 30 输出 | 每个 revision、许可证、大小、缓存路径、显存 |
| 缺失的 LLVM/Alive2/Z3 | 工具链 | 精确版本、安装方式、磁盘影响、是否改变系统环境 |

未获授权前，继续完成不依赖这些资源的骨架、测试、D-U fixture 和离线日志解析；不要停在“等待下载”而不做其他工作。

## 11. 必须生成的最终产物

`P0A_DECISION.json` 至少包含：

```text
protocol_version
protocol_hashes
aris_commit
implementation_commit
toolchain_lock_hash
dataset_lock_hash
model_lock_hash
du_total / du_passed / du_failed
artifact_replay_total / parser_label_agreement / alive_terminal_agreement
da_structure_total / component_count_distribution / ce_state_distribution
track_b_source_count / track_b_outputs_by_model / funnel_by_model
object_determinism_rate
cost_path_exact_match_rate
context_harness_exact_match_rate
atom_precision / atom_recall / dependency_recall / alignment_kappa
ce_double_run_consistency
gate_checks
final_status
blocking_reasons
next_allowed_stage
```

`P0A_DECISION.md` 必须使用下面的顺序：

1. 一句话裁决；
2. 实际完成范围与未完成范围；
3. 工具/数据/模型版本；
4. 逐项门控表，含分子、分母、点估计和阈值；
5. 三模型完整候选漏斗；
6. D-U 和自然样本对齐审计；
7. 成本校准与不一致案例；
8. 反例适配器稳定性和降级原因；
9. 协议偏差、失败与缺失资源；
10. 唯一允许的下一步。

若 `P0A_PASS_READY_FOR_P0B`，下一步只能是“冻结 P0b manifest 后申请启动 P0b”；不得自动运行 P0b。若失败，下一步必须指向具体基础设施修复，不得改题、加 RL、加 Agent 或转向 RISC-V 规避失败。

## 12. Codex 每次汇报格式

工作更新保持简洁，但必须可核验：

```text
当前任务：Task N — 名称
已完成：具体文件、测试和样本数量
刚执行：完整命令
结果：exit code、通过/失败数、关键指标
产物：相对工作区路径
门控状态：未判定 / PASS / FAIL / BLOCKED
下一步：下一个范围内动作
需要用户授权：无，或精确资源/命令/大小/耗时
```

最终回复不得只说“预实验完成”。必须给出最终状态、门控表摘要、未完成项、Git commit 和 `P0A_DECISION.md` 路径。实验未实际运行时只能写“实现完成”或“环境审计完成”，不能写“验证通过”。

## 13. 完成定义

只有同时满足以下条件，本文工单才算真正完成：

- [ ] 独立工作区建立且未污染 ARIS 根目录现有 UnlockIR 文件；
- [ ] 协议、工具、数据、模型均有锁文件或明确的缺失状态；
- [ ] 单元/集成测试真实运行，原始输出留存；
- [ ] 112 个 D-U 样本全部进入 manifest 和审计；
- [ ] 100 条 D-A 标签复现与 30 条 D-A 结构烟测分别统计；
- [ ] 三个模型对同一 10 个源函数各生成 1 个输出，或被明确标为访问阻塞；
- [ ] 至少 30 个函数完成成本双路径和 context 校准，或被明确标为资源阻塞；
- [ ] 所有门控给出分子、分母和裁决，不以定性描述替代；
- [ ] ARIS experiment audit 完成并保存；
- [ ] `P0A_DECISION.json` 与 `P0A_DECISION.md` 一致；
- [ ] 未运行 P0b、P0c、P1、P2 或 RISC-V；
- [ ] 只提交本工单创建的 SalvageIR P0a 文件，未混入用户其他改动。
