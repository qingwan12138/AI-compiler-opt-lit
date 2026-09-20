# SalvageIR 数据与公平性协议

> salvageir_v3 · 2026-09-21。数值与输入定义以 PILOT_PROTOCOL 为准。

## 1. 数据来源与模型角色

- LEGACY：现有32条及其他既有运行；先保留原标签、输入阶段与目标，再用冻结工具链重分类。
- CONTROLLED：有未优化出处的源码/IR，按v3生成；唯一主盈利性来源。
- ARTIFACT：LLM-VeriOpt等公开输出，只用于独立外部重放；目标为latency时不混入code-size主统计。
- SYNTHETIC：定向fixture/注入错误，只用于机制测试或单列的来源对照。

本研究不做任务特定训练/RL。LLM-VeriOpt的Model-Zero、Model-Correctness、Model-Latency属RL训练系列，不因zero名称纳入“非RL”输出池；仅核验训练来源的base/SFT输出可用于非任务RL外部重放。
通用模型上游对齐史完整披露，不声称基础模型从未经历RL。
不要求三旧模型才能开始P0；P0沿用实际已部署模型。正式确认至少包含未用于调参的不同模型家族和一个较强可访问模型，具体名单在看确认结果前锁定，不能按失败数量选“最方便”的模型。
checkpoint不可变revision、文件hash、许可证、dtype/量化、chat template、引擎版本均须记录。
某模型没有核心失败时条件恢复率是NA；端到端仍有定义，不把它删掉或换成更弱模型。

## 2. 固定候选，不比较模型谁更强

source_id绑定project/snapshot/函数/原始hash；source_group绑定镜像与近重复家族。
candidate_id=SHA256(protocol|source_id|setting|model_revision|prompt_hash|decode|slot|raw_output_hash)。
模型输出被冻结后，所有回收方法从同一个candidate_id开始。
相同IR输出可以复用计算，但重复生成槽位仍留在端到端分母；候选去重率另报。
共同的初始验证与baseline成本只计算一次、对各方法同样可见；搜索额外验证全部计费。
LLM修复基线的额外请求是比较成本，不是SalvageIR核心偷偷新增的能力。

## 3. 唯一分类状态机

按以下顺序分类，每个生成槽位恰有一个candidate_class。中间status与终态分开存。

| 顺序/状态 | 条件 |
|---|---|
| SOURCE_UNAVAILABLE | 该设置的源前置证明/测量失败，未调用模型 |
| GENERATION_ERROR | 请求/进程失败，无法获得有效原始响应 |
| GENERATION_TRUNCATED | 达到输出上限或缺失完整结束，保留原文 |
| FORMAT_INVALID | 无法提取唯一完整目标函数 |
| IR_INVALID | LLVM parser/verifier拒绝 |
| CONTRACT_VIOLATION | signature/ABI/属性/目标上下文等禁止字段改变 |
| VERIFIER_ERROR | 崩溃/无法解析的终态/矛盾裁决 |
| UNKNOWN_TIMEOUT | 明确达到验证超时 |
| UNKNOWN_UNSUPPORTED | 明确不支持 |
| UNKNOWN_OTHER | 其余无决定性语义裁决 |
| VERIFIED_COPY | VERIFIED且规范化内容等于源 |
| VERIFIED_COST_UNAVAILABLE | VERIFIED但成本或X→P(T0)验收不可用 |
| VERIFIED_NO_GAIN | 初始正确候选未满足主盈利门 |
| VERIFIED_PROFITABLE | 初始正确且满足主盈利门，包括post-Oz直接证明 |
| REFUTED_UNSUPPORTED_SCOPE | 明确REFUTED，T0超出冻结语言范围 |
| REFUTED_ELIGIBLE | 明确REFUTED，属于核心语言范围 |

无法验证的超范围候选仍按UNKNOWN分类，同时保存scope_status；不凭静态特征制造REFUTED。
相同内容可缓存已知源自检证明，但不得仅凭字符串不同判定错误。
初始正确候选的最终Oz输出若REFUTED，标VERIFIED_COST_UNAVAILABLE并发出OPT_PIPELINE_REFUTED故障告警；不转成SalvageIR失败输入。
REFUTED_ELIGIBLE中的ALIGNMENT_FAILED、REPRESENTATION_UNSUPPORTED、SINGLE_COMPONENT、CE_UNLOCALIZABLE等是后处理状态，不改变分母。
规范化代码接口禁止用随意正则编辑LLVM语法；解析器策略/version及原始响应齐全。

## 4. 四个必须同时报告的量

设N_gen是所有预定生成槽位（含SOURCE_UNAVAILABLE/错误），N_ref是所有明确REFUTED，
N_core是REFUTED_ELIGIBLE，N_use是自动满足契约的有益回收数。

- 候选漏斗：每个candidate_class的数/N_gen。
- 全失败回收率：N_use/N_ref；包括范围外失败带来的覆盖代价。
- 核心条件恢复率：N_use/N_core；表示失败、unknown、超预算都记未成功。
- 端到端：接受初始正确有益候选或回收结果的源函数数/全部预定源函数。

每个模型与输入设置单列。P0一次输出时槽位与源一一对应；未来多输出必须预先固定聚合策略。
“增加的端到端成功”相对同模型相同首次输出的accept-or-fallback管线计算，避免把初始正确输出归功于SalvageIR。
成本比按失败回退1，字节收益按失败0；不只对成功例求平均。
模型宏平均只在条件率有定义的模型中可计算，必须标明家族数与NA；不作全模型通用结论。
正式首要推断使用EXPERIMENT_PLAN的全源端到端增量，避免不同模型失败稀少造成隐性筛选。

## 5. 输入设置与来源比较

PRE_SSA和POST_OZ是不同候选分布，不能比较两个条件恢复率后宣称哪个输入造成更高回收。
配对源函数的全流程收益、全部失败供应、编辑覆盖及成本应一起比较。
POST_OZ重复Oz带来的纯编译器变化按PILOT_PROTOCOL单列。
ARTIFACT原任务latency，主目标不一致；只能报告外部可验证性与独立注明的探索性code-size结果。
合成负对照不是“更公平的主数据”，也不能替代自然现象。
方法在非LLM变异上同样有效不否定LLM应用，只限制“LLM特有结构”主张。

## 6. 冻结和泄漏防护

按上游项目、镜像及近重复连通组划分，不按独立输出随机划分。
一源的不同模型/输入设置/采样都在同一split。
已用于prompt、对齐、rank或门控调试的资料是development。
正式确认前冻结代码/参数/基线/数据manifest；修改后重新登记，不沿用原确认标签。
不把完整P0b称未触碰确认集；其中scout只能验证当前工程假设，后续用来设计P1便成为开发证据。
工具版本漂移时所有配对方法一致重跑受影响样本，不选有利的历史标签。
不先看oracle-positive再挑正式样本；oracle-positive只能作为明确标注的条件诊断子集。

## 7. 最低机器可读记录

| 文件/记录 | 必填字段 |
|---|---|
| SourceRecord | source_id,source_group,project,revision,function,license,split,raw_hash,prepared_hash,ssa_hash,triple,datalayout,normalization_proof,source_eligible,reason |
| GenerationSlot | slot_id,source_id,setting,model_lock_hash,prompt_hash,decode,attempt,raw_hash,candidate_id,candidate_class,first_attempt_preserved |
| CandidateRecord | candidate_id,input_hash,target_hash,initial_verdict,scope_status,representation_status,ce_state,initial_log_hash,protocol_hash |
| ComponentRecord | representation_version,component_id,atoms,source_target_provenance,hard_constraints,hard_reason,soft_risk_edges,coarse_reason |
| StateRecord | run_id,candidate_id,mask,parent,proposal_rank,ir_hash,build_status,verdict_pre,verdict_post,cost_status,strict,profitable,elapsed,calls,cost_calls |
| CostRecord | input_hash,oz_hash,object_hash,symbol,st_size,alloc_non_bss,section_sizes,relocations,commands,repeat_evidence |
| RunDecision | protocol_hash,locks,implementation_hash,counts,missing,oracle_status,budget_spent,stop_reason,decision,next_allowed_stage |
| CommandRecord | argv,cwd,start,end,exit,timeout,input/output_hash,stdout_path,stderr_path,allowlisted_environment |

所有proof关联确切输入/输出hash，不能仅按函数名缓存。
JSON中未知数值用null加reason，不用0；NA分母明确记录。
不记录令牌、密钥或整份环境变量；仅白名单。日志和运行结果追加保存，不覆盖。
