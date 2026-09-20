# 历史归档：v2，不得用于新实验执行

原文件：ALGORITHM_SPEC.md。2026-09-21 被 v3 替代；保留原文用于追溯，不代表当前约束。

# SalvageIR 算法规格

## 1. 符号与状态

| 符号 | 含义 |
|---|---|
| `S` | 规范化后的源 LLVM IR 函数 |
| `T0` | LLM 生成、可解析但被 Alive2 明确反驳的完整候选 |
| `A` | 源到目标的候选编辑原子集合 |
| `C={c1...cn}` | 对编辑原子求 LLVM 依赖闭包后得到的 CEC 集合 |
| `G=(C,D)` | 组件依赖图；`D` 表示组合合法性约束 |
| `x in {0,1}^n` | 组件选择向量；1 表示重放目标组件，0 表示保持/回滚到源 |
| `Build(S,T0,G,x)` | 用选择向量构造完整候选 IR；可能返回结构非法 |
| `V(S,R)` | Alive2 的整函数 refinement 结果 |
| `K_tau(R)` | 目标后端 `tau` 上锁定配置的真实性能成本 |
| `W` | 已收集的 Alive2 反例与相关切片集合 |

### 1.1 研究问题：RCES

把核心任务定义为 **Refinement-Constrained Edit Salvage（RCES）**。给定 `(S,T0,G,K_tau,B)`，在由结构可重放组件诱导的状态格上寻找：

```text
R* = argmin K_primary(Build(S,T0,G,x))
     subject to x dependency-closed
                LLVMVerify(Build(...)) = PASS
                V(S, Build(...)) = VERIFIED
                K_primary(Build(...)) < K_primary(S)
```

由于该组合问题可能指数增长，SalvageIR 只在小组件数时穷举；正式算法是有预算的近似搜索。本文不在完成严格归约证明前声称 RCES 为 NP-hard，也不把依赖闭包称为语义独立性证明。

RCES 与普通“找一个触发失败的最小输入”不同：其可行解必须相对原源函数通过 refinement，目标是保留最大真实后端收益，而不是最小化文本或编辑数量。该差异必须通过强基线而不是定义本身证明有研究价值。

## 2. 规范化

规范化必须可逆或至少保留语义相关信息：

1. 用 `opt -passes=verify` 检查输入；
2. 移除 debug location、注释和与语义无关的名称噪声；
3. 稳定编号匿名基本块和 SSA 值；
4. 对交换律操作数按语义指纹排序；
5. 保留 datalayout、triple、函数属性、调用约定和语义 flag；
6. 保存原始、规范化和清理后 IR 的内容哈希。

禁止为了便于对齐而删除 `nsw/nuw/exact/inbounds`、`freeze`、参数属性或内存语义，因为这些经常就是错误来源。

## 3. 结构对齐

### 3.1 基本块匹配

优先级依次为：

1. entry、return/exit 锚点；
2. 前驱/后继度数与边标签；
3. dominator/post-dominator 深度；
4. 块内操作码和类型多重集；
5. 已匹配邻居的一致性。

若匹配置信度低于冻结阈值，则把相关 CFG 子图作为不可再分的 coarse component，而不是强行逐指令匹配。

### 3.2 指令匹配

指令语义指纹包括：

```text
opcode + result type + operand origin signatures
+ constants + semantic flags + memory/call attributes
```

对换序、局部公共子表达式消除和临时值重命名允许多轮固定点匹配。匹配算法及阈值只在开发集调整，保留集冻结。

### 3.3 编辑原子

- `InsertInst`
- `DeleteInst`
- `ReplaceOpcode`
- `ReplaceOperand`
- `ReplaceConstant`
- `ChangeSemanticFlag`
- `ChangeAttribute`
- `Add/Delete/RetargetEdge`
- `ChangePhiIncoming`

文本重命名和非语义 metadata 不形成编辑原子。

## 4. 组件闭包

从一个编辑原子出发，重复加入违反以下条件的关联编辑，直到固定点：

```text
SSA：每个使用都有唯一且支配该使用的定义
CFG：terminator、边和 PHI incoming 集合同步
Control：控制谓词变化与受控区域同步
Memory：可能冲突的 load/store 与 MemorySSA 关系同步
UB：产生或消费 poison/undef 的 flag、freeze、branch/address 同步
ABI：函数签名、calling convention、attribute 保持契约
```

若两个闭包互相包含则合并。得到的组件图必须是 DAG；检测到环时合并强连通分量。

组件不是被假定正确的优化规则。它只是搜索中能够合法重放的最小工程单位。为避免误解，论文正文统一称其为 **结构可重放组件**；“闭合”只表示满足构建所需的 SSA/CFG/属性联动，不表示组件可独立保持语义。

## 5. 反例切片

### 5.1 反例分类

对 Alive2 refutation 保留以下类别：

- return value mismatch；
- target more undefined；
- memory mismatch；
- poison/undef mismatch；
- attribute/precondition mismatch；
- 其他可解析的明确 refutation。

parser error、unsupported 和 timeout 不进入反例切片主流程。

### 5.2 反例适配器契约

每个验证状态产生一个 `CounterexampleRecord`：

```text
candidate_id, state_id, alive2_version, source_hash, target_hash,
failure_kind, witness_inputs, source_observation, target_observation,
ub_poison_facts, memory_events, raw_log_hash, normalized_record_hash
```

适配器只能输出三种状态：

| 状态 | 定义 | 搜索用途 |
|---|---|---|
| `CE_LOCALIZABLE` | witness 可在同一状态稳定重放，且观察差异能映射到 IR 值/内存事件 | 动态+静态切片软排序 |
| `CE_STATIC_ONLY` | 明确 refuted，但 witness 无法稳定物化或映射 | 静态 backward slice 软排序 |
| `CE_UNLOCALIZABLE` | 只能保留明确 refutation，无法形成可信 slice | 不提供切片分数，仍保留完整样本 |

同一 `(S,R)` 独立运行两次后，`failure_kind`、规范化 witness 和观察差异必须一致，才允许标为 `CE_LOCALIZABLE`。不一致记录降级；任何降级都不得从端到端分母删除。

### 5.3 切片构造

在能可靠重放反例时，从可观察差异逆向遍历：

- SSA def-use；
- control dependence；
- MemorySSA dependence；
- 触发 UB 的消费点及其输入；
- 与该路径相关的 CEC。

若无法可靠物化具体输入，退化为静态 backward slice，并把置信度记为低。切片集合 `slice(w)` 只是可疑集合。

多反例形成软命中分数：

```text
score_coverage(state, RollbackSet) =
  sum_w weight(w) * I[active(state) intersects slice(w)
                       and RollbackSet intersects slice(w)]
```

反例是 **state-conditioned**：由状态 `x` 得到的 slice 只能为 `x` 的后继排序，不能被当作所有状态都成立的全局因果事实。不得把“不命中”作为 soundness 剪枝。硬剪枝只来自 IR 合法性、依赖约束与已执行的验证结果。

反例信息增益必须用四方对照检验：真实动态反例、仅静态切片、在同失败类别内打乱归属的反例、完全无反例。若真实反例不优于打乱反例，不能声称验证 witness 提供了定位信息。

## 6. 搜索算法

### 6.1 第一阶段：建立安全前沿

```text
procedure SALVAGE(S, T0, Budget B):
    G = BuildEditComponentGraph(S, T0)
    Q = priority queue containing state FullTarget(G)
    Seen = empty set
    W = {InitialAlive2Counterexample(S, T0)}
    SafeFrontier = ParetoSet({S}, max_size=k_safe)

    while Q not empty and within B:
        state = Q.pop_best(frozen_priority)
        if state in Seen: continue
        Seen.add(state)

        R = Build(S, T0, G, state)
        if R is structurally invalid:
            enqueue dependency-closed supersets of rollback(state)
            continue

        verdict = Alive2(S, R)
        if verdict == REFUTED:
            W.add(extract_state_conditioned_counterexample(
                    verdict, R, state, G))
            enqueue rollback successors guided by W
        else if verdict == VERIFIED:
            SafeFrontier.add_non_dominated(
                R, retained_gain_proxy, rollback_size, stable_id)
            enqueue rollback successors and legal re-add successors
        else:
            record UNKNOWN

    return PROFIT_RECOVERY(SafeFrontier, remaining_budget)
```

不再在首个 verified 状态停止。`k_safe` 在开发数据前冻结，主配置为 8；前沿以真实已测成本优先，其次为保留组件数和稳定 ID。若预算耗尽前未获得 strict verified 状态，返回 `NO_SALVAGE`。

主排序不使用可调权重，冻结为字典序 tuple：

```text
frozen_priority = lexicographic_max(
  newly_covered_real_ce_count,
  newly_covered_static_slice_count,
  backend_gain_upper_bound,
  -structural_risk,
  -rollback_atoms,
  -stable_id)
```

所有项只用当前状态之前可见的信息；`stable_id` 只负责完全平局。开发集不得改变字段次序。加权、TTI 学习型或 learned ranker 版本都不属于主方法，只能作为预注册补充；P1 运行删除每一字段的敏感性分析。

### 6.2 第二阶段：利润恢复

从安全非支配前沿的每个状态开始，尝试重新加入已回滚组件：

```text
procedure PROFIT_RECOVERY(SafeFrontier, RemainingBudget):
    for SafeState in round_robin(SafeFrontier):
        prioritize additions by frozen gain/risk score
        for dependency-closed additions within RemainingBudget:
            R = Build(...)
            require LLVMVerify(R)
            require Alive2(S, R) == VERIFIED
            if verified:
                update SafeFrontier
                measure exact backend cost
    return argmin exact_cost(SafeFrontier)
```

利润预测只负责次序，可使用：

- 组件导致的 IR/机器指令静态差；
- `llvm-mca` 或 LLVM TTI；
- 回滚该组件后的实际 `.text` 差；
- 组件规模。

最终选择必须重新运行真实性能成本，不能用预测值裁决。

实现层优先复用 LLVM SandboxIR 的事务 save/accept/revert 与 change tracking，避免把“可回滚 IR”包装为本工作创新。SalvageIR 的待证贡献是 **如何从外部 LLM whole-function diff 建立结构可重放组件，并用 refinement 反例在 RCES 状态格中排序**；LLVM 事务层只是基础设施。

### 6.3 小规模 oracle

当 `n <= n_oracle` 时，枚举所有依赖闭合状态，得到：

- 是否存在可回收结果；
- 最佳可回收成本；
- SalvageIR 与 oracle 的 optimality gap；
- 验证调用节省。

`n_oracle` 根据先导运行时间冻结，不在看结果后调整。

## 7. 搜索预算

每个候选同时受以下上限约束：

- `B_verify`：Alive2 调用次数；
- `B_compile`：LLVM verifier/后端编译次数；
- `B_wall`：总墙钟时间；
- `B_states`：唯一状态数；
- 可选扩展的 `B_llm_tokens`。

基线可选择共享四维预算或使用“墙钟+验证次数”主匹配并报告其他资源。不能只匹配候选数而允许某方法无限验证。

## 8. 输出状态机

| 状态 | 定义 |
|---|---|
| `SALVAGED_PROFITABLE` | 最终直接 verified，且主成本优于 `S` |
| `SALVAGED_VALID_NO_GAIN` | verified，但未优于 `S`；不算主成功 |
| `NO_VERIFIED_SUBSET` | 预算内无 strict verified 子集 |
| `ATOMIC_FAILURE` | 编辑图只有一个不可分组件或所有组件必须联动 |
| `ALIGNMENT_UNSUPPORTED` | 无法可靠构造可重放编辑图 |
| `VERIFIER_UNKNOWN` | 候选搜索被 timeout/unsupported 主导 |
| `TOOL_FAILURE` | LLVM/Alive2 崩溃或环境错误 |

## 9. 实现接口

建议模块边界：

```text
candidate_loader     -> FrozenCandidate
ir_normalizer       -> NormalizedPair
ir_aligner          -> EditAtoms + confidence
component_builder   -> ComponentGraph
counterexample      -> CounterexampleRecord + Slice
composer            -> CompleteIR | StructuralFailure
verifier            -> VERIFIED | REFUTED | UNKNOWN
cost_model          -> ProxyCost + ExactCost
search              -> Trace + SafeCandidates
reporter            -> candidate/model/project aggregates
```

所有模块通过内容哈希连接。验证器、编译器和成本工具的 stdout/stderr 原样保存；不得只保存解析后的标签。

## 10. 不变量测试

实现时至少包含：

- 规范化前后自 refinement 检查；
- `Build(all source)` 与 `S` 同构；
- `Build(all target)` 与 `T0` 同构；
- 每个构建结果通过 LLVM verifier 后才进入 Alive2；
- 依赖闭合性质测试；
- PHI/CFG、flag、poison/undef 的定向单元测试；
- 人工构造的可分错误、不可分错误和协同恢复案例；
- 小规模穷举结果与搜索结果交叉检查。
- `CounterexampleRecord` 双次重放稳定性测试；
- 真实反例、打乱反例和静态切片的隔离测试；
- `k_safe=1` 与安全前沿版本的回归测试；
- 原模块语境与抽取 harness 的机器码校准测试。
