# SalvageIR 算法规格

> salvageir_v3 · 2026-09-21。算法是可实施的候选设计，未由自然样本证明有效。

## 1. 形式化对象

输入 (S,T0) 已满足研究契约，T0 为 REFUTED。
A 是有来源的编辑原子，C 是仅按必需共变关系合并的组件，x 是组件选择掩码。
Build(S,T0,x) 构造 R；0 表示源版本，1 表示目标编辑。H(x) 是结构可构造性约束。
主目标：在预算内最小化 C_backend(P(R))，满足严格子集、两次直接 refinement 和收益门。
P 是固定 Oz，不是搜索 pass 的变量。不声称 NP-hard、完备定位或全局近最优。

每个状态必须绑定 source_hash、target_hash、representation_version、mask、built_ir_hash。
不同掩码产生相同 IR 时缓存计算，但保留所有掩码来源；相同最终 Oz IR 也可共享成本与对应证明。
缓存键还包括工具/配置/超时/目标哈希；跨方法允许共享计算工件，但模拟预算须收取相同逻辑查询与冷缓存成本。真实 wall-clock 比较各方法独立冷缓存。

## 2. 规范化与对齐

输入准备在 PILOT_PROTOCOL 中定义；这里不能再调用 instcombine、DCE、simplifycfg 等改写差异。
只去 debug、稳定名字；不排序可能影响观察/元信息的任意文本，不删除语义属性。
匹配先用参数、返回、固定 CFG 块锚点；再用 opcode/type/operand origin/constant 匹配指令。
flag 与可变操作数是独立字段，不能把 flag 差异当成两条完全无关的指令。

P0 builder 首版：
- 单块或唯一结构匹配的固定无环 CFG；
- 每块中匹配指令顺序单调，无调度歧义；PHI 所属前驱相同；
- 同一槽位的源/目标版本及原始顺序全部保留；
- 非唯一对齐片段合为 coarse replacement；若无法满足两端重建，返回 REPRESENTATION_UNSUPPORTED。
- 不支持 CFG 重构时记录覆盖失败，不偷偷调用 LLM、LLVM 优化 pass 或合成器弥补。
固定 CFG 允许改变分支条件表达式，不把“控制含义变化”误当作 CFG 拓扑变化。

原子：InsertInst、DeleteInst、ReplaceOpcode、ReplaceOperand、ReplaceConstant、ChangeSemanticFlag。
接口、函数属性、datalayout、triple 和外部声明不可由模型改变。扩展版 CFG/PHI/属性原子另行注册。

## 3. 结构约束与风险关联必须分开

| 关系 | 处理方式 |
|---|---|
| 新操作数引用目标端新增定义 | 必须启用该定义；结构硬蕴含 |
| 删除定义但还保留使用 | 必须重定向/删除相应使用，或构建失败 |
| 指令类型/操作码/操作数槽位耦合 | 必需字段组合约束；互相必需才合并 |
| PHI 与前驱集合、支配、终结指令 | 结构硬检查；P0 不补造新 PHI |
| nsw/nuw/exact、freeze 与消费者 | 语义风险关联；不因存在 def-use 自动合并 |
| 分支条件与受控区域 | 语义风险关联；不是整片区域强制共变 |
| 内存别名/MemorySSA | P0 不支持；扩展时分别证明构造约束与语义风险 |
| 签名/ABI/语义属性变化 | P0 拒绝契约变化，不当可恢复编辑 |

硬蕴含可用有向图；互相蕴含的强连通分量可合并。
多选一等关系保留为约束表达式，不能强行转换成“所有相关编辑必须一起改”。
每条硬约束保存 reason、source/target实体及最小失败示例；风险边不得硬剪枝。
组件称“结构可重放组件”，不是“语义独立优化组件”。

必须满足：
- Build(全0) 与 S 规范同构，Build(全1) 与 T0 规范同构；
- 构建不引入两端都不存在的可执行计算；
- 每个结果再次运行 LLVM verifier，静态图不是 verifier 的替代；
- 不把某次整函数验证通过传播为其他上下文中的组件正确性。

## 4. 重放与邻居

composer 从 S 克隆；应用选中编辑，依据两端已记录映射重定向操作数。
仅允许 alpha-renaming、重建对象引用和采用原有端点的块/指令顺序。
无法恢复引用、支配或类型时返回 BUILD_INVALID；不得按当前结果临时“加一条 glue 指令”。

基本邻居为逐组件翻转：
- 回滚组件时，级联回滚依赖该组件的已启用组件；
- 加回组件时，级联启用其必需前提；
- 非蕴含约束由 H(x) 检查，不满足则该提案 BUILD_INVALID；
- 两类邻居都去重、稳定排序，不能只扩展 VERIFIED 状态。
只支持上述闭包能够表达的表示进入预算搜索；不支持的通用约束记为表示覆盖失败。
oracle 则枚举全部掩码并检验 H，不依赖启发式邻居可达性。

全0和全1作为公共起点，已有源验证/初始反例对所有方法相同。
全0可生成单组件加回邻居，避免只沿着错误目标回滚而错失独立可行区域。
对非法状态可继续生成未访问邻居；不把某个状态非法推成所有超集/子集非法。
额外去重、限制队列/提前停止如未在 lock 中声明，不得静默增加。

## 5. 反例接口：有效而非相同

CounterexampleRecord 包含：
candidate_id、state_id、S/R哈希、verifier版本、failure_kind、witness、
source/target_observation、poison_undef_facts、raw_log_hash、localization_state、reason。

三态：
- CE_LOCALIZABLE：该 witness 在对应状态可可信地物化/核验，差异能映射到 IR 值或观察点。
- CE_STATIC_ONLY：明确 REFUTED，但不能可信重放；只使用静态 backward slice。
- CE_UNLOCALIZABLE：明确 REFUTED，却没有可信切片；无切片分数。

同一日志重复解析必须一致；重复求解返回不同有效 witness 不触发降级。
普通执行器不能重现 LLVM poison/undef 语义时，不把普通整数运行结果当作完整 witness 重放。
两次 verdict 相互矛盾则是工具故障，隔离该记录并调查；不是挑选 VERIFIED 结果。
witness 只影响产生它的状态的邻居排序；不得跨状态当永久故障约束。
从返回/分支/poison消费差异逆向形成可疑编辑集，允许包括需要加回的已回滚源编辑。
不声称切片里所有编辑都错误，也不声称切片外编辑正确。

## 6. 预算搜索 v3

统一队列搜索取代两套容易不一致的“安全前沿＋第二阶段”实现。
保留一个已精确验证和测量的最佳 incumbent；VERIFIED 无收益状态仍继续扩展。
每次展开同时产生回滚和加回邻居，因而允许恢复利润；并非只找最小修复。
优先级依次是：
1. 提案翻转编辑命中当前父状态可信动态切片的数量；
2. 命中当前父状态静态切片的数量；
3. 按选中编辑槽位估算的相对 S 的 IR 指令减少量（gain_estimate，可能为负）；
4. 较少的结构编辑数；
5. 稳定状态 ID。

各父状态只评价自己的提案；相同状态多条提案保留最大字典序键并保留来源。
源状态和 VERIFIED 父状态没有错误 witness，前两项为0。
gain_estimate 不是后端收益上界，不用于硬剪枝。任何代理好坏都不决定最终成功。
不得为免费给队列排序预先编译所有候选；若排序时实际构建了IR，该唯一状态须计入状态预算。
最多状态数、验证数、精确测量数及总时间共同限流；参数见配置。

~~~text
initialize queue with neighbors(all-target, initial CE) and neighbors(all-source)
incumbent = NONE
while queue not empty and budgets remain:
    x = pop deterministic priority; skip duplicate built IR if already evaluated
    R = Build(x)
    if BUILD_INVALID:
        log; enqueue unseen structural neighbors; continue
    result = direct Alive2(S,R)
    if result == VERIFIED:
        Q = P(R)
        exact cost = backend(Q)
        if profitable and strict:
            require LLVMVerify(Q) and direct Alive2(S,Q) == VERIFIED
            update incumbent using exact cost, then alloc_non_bss, then stable ID
    if result == REFUTED: extract state-conditioned CE
    if result == UNKNOWN: record reason, do not accept
    enqueue unseen legal rollback and re-add proposals using this state's information
return incumbent or NO_SALVAGE
~~~

如果最终证明/测量预算不足，该结果未成功；不得以“搜索找到”计入成功。
保存完整 anytime trace；每个预算截点只能使用此前已经完成两道证明和测量的 incumbent。
time-to-first-useful 必须计入未成功样本的删失，不只比较共同成功样本的均值。

## 7. 正确性与成本边界

LLVMVerify、Alive2、P 和 llc 都使用协议锁定版本。
证明链同时记录 S→R 与 S→P(R)，不靠局部正确性拼装代替。
P0 禁止循环，减少 bounded loop validation 的额外解释负担；未来循环支持须单列边界。
后端编译器仍是可信边界，IR 证明不等于 ELF 机器码证明。
直接 whole-function refinement 和后端真实成本决定接纳，动态切片失败只影响效率。

## 8. Oracle 与表示审计

小表示完整枚举，定义在冻结 A/C/H 上，不能因想穷举而合并到阈值内。
任意 unknown、未测成本或未完成状态阻止“没有可回收解/精确最优”的结论。
即使枚举未完成，已证实的有益状态仍是存在性证据。
状态类别和上/下界见 PILOT_PROTOCOL；oracle 不是全部可能程序变换的上界。

必须人工审计：
- flag 是否被不必要地绑到消费者；
- 是否存在联合修改才正确、分别修改都错误的案例；
- coarse component 是否吞掉人工可验证的子集；
- 需要新 glue 的案例是否被错误计为严格回收。
人工构造的方案只作表示上限诊断，不混入自动恢复率。

## 9. 基线与消融接口

B5 与主算法完全相同但把动态/静态切片键置0；其余提案、预算、接纳均相同。
StaticOnly 只移除动态键；MatchedRandom 在同一状态同一翻转方向中置换真实组件分数。
MatchedRandom 在硬依赖度数桶(0、1–2、≥3)内置换，保留非零个数及分数分布；
冻结随机种子，桶太小时记录实际可置换比例，不能宣称完成有效安慰剂。
B6/B7 的具体实现与确认比较见 EXPERIMENT_PLAN。
移除加回操作是利润恢复消融；硬结构约束与语义风险关联不得再混名为“UB闭包”。
