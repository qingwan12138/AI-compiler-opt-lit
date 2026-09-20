# SalvageIR v3 预实验协议

> salvageir_v3 · 2026-09-21。设计版本，不是已完成的预注册、运行结果或启动授权。
> 本文是执行细节的唯一权威；数值默认值见 [参数文件](configs/pilot_v3.json)。实施时逐字段核对并冻结 lock。

## 0. 与 v2 的关系

v2 已归档。其 post-Oz 唯一输入、三模型120个失败硬门、112个fixture、双跑相同witness、98%对齐率等要求不再支配新实验。
旧数据不删除、不重标成 v3；保存 old_protocol_hash、实际输入/命令与 LEGACY 标签。
“已有32条，其中2条有效”是待核验的报告，不是已确认的 REFUTED 漏斗。

## 1. 阶段与授权

| 阶段 | 输入与任务 | 输出/边界 |
|---|---|---|
| P0a | 现有日志审计、24个定向fixture、最小枚举与成本链 | smoke和自然两例诊断；默认交接执行范围 |
| P0b | 固定64函数、1模型、2输入设置，各生成1次 | 最多128次生成；可行性发现，非正式统计确认 |
| P1 | 冻结表示后开发完整搜索、强基线和消融 | 方法开发，不对已看数据做确认性推断 |
| P2 | 独立项目/模型的固定样本确认；系统及RV64GC | 另行锁定正式 manifest 与算力预算后执行 |

P0a 不需要下载全量 artifact，不强制三模型，不运行 GPU 生成。
没有原始失败对时可以完成合成 smoke，但必须写 NATURAL_DATA_MISSING。
P0b 需要明确 GPU/资源授权；本文不授予远端访问、付费调用、安装或长时作业权限。

## 2. 工具与来源锁

优先审计 AutoDL 已工作的匹配 LLVM/Alive2 组合；不自动追逐最新版。
toolchain.lock 保存各二进制/源码 commit、LLVM/Clang/opt/llc/readobj、Alive2/Z3、
构建选项、指针表示、OS/CPU、容器摘要(若有)、命令参数和 timeout。
不把“major相同”当作充分兼容；跑 smoke 后才接受。升级创建新锁，不覆盖旧结果。

datasets.lock 保存数据 revision、文件哈希、许可证、上游 project/commit/路径及提取命令。
不知道原始 IR 来源、优化史或 datalayout 时记 UNKNOWN_PROVENANCE，不用于 PRE_SSA 主结果。
同一输入的参数/返回属性、triple/datalayout 都必须保留；禁止用“改 triple”兼容目标。

## 3. 主输入 PRE_SSA

### 3.1 准备顺序

1. 保存不可修改的 raw IR；要求有未经过通用 LLVM 优化流水线的出处。
2. 核验 parser/verifier、目标和函数上下文。
3. optnone 处理：仅移除可证明由本次前端 O0 模式产生的 optnone；源代码显式属性不移除。来源不明则拒绝主池。
4. 保留 noinline 及其他语义/ABI属性；不得盲目删 noinline。
5. 对两条输入路径共同设置 optsize/minsize，记录为优化目标提示；已有冲突属性拒绝，不静默覆盖。
6. 仅执行 mem2reg，将可提升栈槽转成 SSA；不调用 instcombine、simplifycfg、DCE、SROA 或 default<O*>。
7. 仅去调试信息并稳定命名，形成 S；保留 lifetime 与全部语义 flag。
8. 保存 raw、属性准备结果、mem2reg结果、S 及命令/hash。

mem2reg 是实际 IR 变换，不是纯格式清理。未提升的内存残留按范围筛选，不能偷偷加 pass。
本设置准确名称是 PRE_SSA，不声称原汁原味 O0。
支持自己的源码提取时，前端参数必须锁定并核查未执行优化；本协议不假设数据卡等于实际执行日志。

### 3.2 生成前验收

在匹配的上下文中要求 raw→S、S→S、S→P(S) 分别由 Alive2 VERIFIED。
raw→S 的比较允许非语义名字对应，不允许隐藏编译语义变化。
无法证明 raw→S 不代表变换错误，记 NORMALIZATION_UNVERIFIED 并排除主池；可另列探索集。
自等检查不是规范化正确性证明，也不能证明输入没有 UB。
原始→S 是 refinement，而非擅自声称双向等价；LLVM/frontend 本身仍有可信边界。

### 3.3 核心语言范围

S：8–256条非debug指令，包含PHI；1–16基本块；无循环；标量 i1/i8/i16/i32/i64 参数和返回。
白名单：整数算术/除余/位运算/移位、icmp、整数cast、select、phi、freeze、br、ret；
只允许锁定版本支持的相应整数 flags。常量包含正常整数/undef/poison，保持其语义。
不含指针/aggregate/向量/浮点、内存、global引用、调用、intrinsic、异常、inline asm、atomic/volatile。
标量整数ABI属性只接受白名单并在S/T0完全相同；模型不得增加 noundef 等前提。
T0 不要求满足源长度限制，避免按生成长度挑结果，但受生成输出上限和核心语言限制。
CFG 重构候选若仍符合语言范围，进入核心 REFUTED 分母；builder 不支持记覆盖失败，不删样本。

### 3.4 POST_OZ 与历史输入

POST_OZ 源 Z=P(S)，使用与 PRE_SSA 相同模型、任务说明、解码和源身份。
该设置的直接验证基准是 Z，收益基准为 P(Z)，不是偷偷少跑一遍 Oz。
同时记录 C(P(S)) 与 C(P(Z))：Oz 不假定幂等。
比较两种设置的端到端效果时统一外部基准 C(P(S))，POST_OZ 的纯 LLVM 再跑收益单列为无LLM对照；
设置内回收增量分别相对各自 P(input)，不能把重复Oz收益归给模型/回收。
POST_OZ 若本身无法在冻结协议下证明/测量，记该设置源不可用，配对端到端分母仍保留。
已存在的 post-Oz32条只能 LEGACY 审计，不能把优化后的 IR “逆转回 O0”。

## 4. 成本与接纳

令 X 为当前设置的源，P 为相同 default<Oz>，R 为回收结果：
- 基线 B=P(X)；
- 最终候选 Q=P(R)；
- 直接证明 X→R 与 X→Q；
- size 为 llc 输出 ELF 中唯一目标函数 ST_Size；
- gain=size(B)−size(Q)；
- profitable 当 gain≥max(2,ceil(0.01×size(B)))，且 alloc_non_bss(Q)≤alloc_non_bss(B)。

主平台 x86_64-unknown-linux-gnu，CPU generic，llc -O=2。
固定 target features、relocation/code model，记录显式参数和默认值；不允许继承不一致 host CPU 属性。
不得在模型输出后才单侧添加 size 属性；R 必须保持 X 的锁定属性。
命令族如下，实际路径和受版本支持的参数需写进 lock：

~~~text
opt -passes=mem2reg prepared.bc -o source.ssa.bc
opt -passes='default<Oz>' source.ssa.bc -o baseline.oz.bc
opt -passes='default<Oz>' recovered.bc -o recovered.oz.bc
llc -O=2 -mtriple=x86_64-unknown-linux-gnu -mcpu=generic -filetype=obj baseline.oz.bc -o baseline.o
llc -O=2 -mtriple=x86_64-unknown-linux-gnu -mcpu=generic -filetype=obj recovered.oz.bc -o recovered.o
llvm-readobj --symbols --sections baseline.o
llvm-readobj --symbols --sections recovered.o
~~~

P0 使用单个外部可见定义函数的固定模块壳，类型/必要声明/属性不变；不靠提取重写链接语义。
源已 internal、comdat/alias复杂、符号不可保留或非闭合上下文则主池排除并记录。
alloc_non_bss 统计全部 SHF_ALLOC 且非 SHT_NOBITS 的节字节，含常量池及展开表等，规则两边相同。
不把对象文件总文件长度当成本；缺失/零长/重名/合并符号记 COST_UNMEASURABLE，不能成功。

环境fixture与每个最终成功对各编译三次：比较目标函数字节和所有计量节的大小/内容，
不要求含时间戳/debug等非计量元数据的整个对象哈希一致；对象完整哈希仍留档。
重定位表一并保存；符号或重定位归属不可解释时不接纳，不能靠全零重定位占位字节做等价判断。
IR路径准确称“固定 LLVM Oz 中端＋锁定后端”，不宣称与 clang -Oz 完全等同。
有源码时 clang -Oz 做诊断对照；不再强制95%字节相同，不因不相同而事后挑除。
函数级收益不等于链接后二进制收益，正式系统实验另外验证。

## 5. P0a：小而完整的基础设施与现有数据审计

### 5.1 24个必需fixture

6类，每类4例；全部有来源端点、预期构造性质和确定的语义裁决或拒绝原因：

| 类 | 四例必须覆盖 |
|---|---|
| 编辑来源/端点 | 纯重命名、插入删除、替换操作数、all-source/all-target重建 |
| 结构依赖 | 新定义被使用、删除仍被使用、支配错误、类型不匹配 |
| 语义独立于结构 | 独立撤销错误flag、联合才正确、freeze、poison消费 |
| CFG | 固定CFG条件变化、PHI取值、前驱不匹配拒绝、CFG重构覆盖失败 |
| 范围/状态 | 残留内存、调用/ABI变化、可控timeout、unsupported/崩溃区分 |
| 成本/预算 | 有符号与零符号、常量池守门、Oz后收益归零、最后证明预算耗尽 |

每类4例可以参数化覆盖多个断言；合成成功不进入自然恢复率。
预期标签须有真实LLVM/Alive2结果支撑；不能因为实现生成标签相同就声称语义测试通过。
timeout/crash可用受控进程fixture测试适配器，不伪称真实Alive2发现的程序性质。
整套阻断性测试必须全通过，失败修代码不删难例。没有工具时标 SKIPPED，不当 PASS。

### 5.2 现有32条审计

先只读列清实际记录数、模型、源哈希、prompt/目标、输入阶段、完整生成与初始验证日志。
逐条重新分类，保留旧标签与新标签；“有效2条”需确认是否真是REFUTED而非verified或解析成功。
对全部明确REFUTED对执行：
- 人工读差异、失败种类、可能正确编辑与需要联合的编辑；
- 自动图、硬边理由、组件数和coarse比例；
- 小空间枚举/固定预算witness search；
- 正确但无收益、Oz抹平、对齐失败、工具unknown等分别列出。
超出v3范围的旧候选只做探索审计，不宣称核心范围已扩大。
手工可回收结果仅诊断表示；自动结果另列，禁止混用。
没有raw/T0/log时保存缺件清单并继续无数据依赖任务，不推测结果。

## 6. P0b：固定规模发现集

主来源 IR-OptSet 的 original_ir，必须核验实际出处和许可证；optimized字段不得冒充原始输入。
如快照不提供可靠project或raw字段，停止该来源主池，不临时换成LLVM nightly已优化IR。
nightly已优化语料只适合POST_OZ/外部压力集，不能承诺作为PRE_SSA确认集。
有完整源码的LLVM test-suite可在另行冻结的新批次作为替代来源，不与既有批次混算。

选样在生成前完成：
1. 先按项目/镜像关系/近重复族分组；开发中过的旧32条及其项目排除出scout。
2. 预筛只看源、工具支持、baseline测量和上下文限制，不看模型输出。
3. 每项目内对规范化源去重；四个指令箱8–31、32–63、64–127、128–256。
4. 项目按 SHA256(seed|revision|project_group) 升序；要求每组至少4个合格函数。
5. 选前16组，每组先按四箱各取hash最小1个；空箱按全部剩余函数hash补足到4个，记录缺箱。
6. 前4项目共16函数为development；后12项目共48函数为scout。两者项目/近重复族不交叉。
7. 全部64函数对两种设置均尝试一次；来源不足则 SOURCE_POOL_INSUFFICIENT，报告实际数，不事后放宽。

此为项目均衡、受语言限制的benchmark，不代表互联网代码发生率。
P0b仅一个模型：优先已经部署、许可证明确的冻结checkpoint，不因失败多/少挑模型。
在首条新生成前写明真实 model ID/revision/权重hash/template/dtype/quantization/context预算。
没有可用模型时阻塞生成，不默认为某个模型已在AutoDL上。
greedy一次，max_new_tokens=4096，输出字符上限32768；截断单列GENERATION_TRUNCATED。
输入+输出预算不得超过实际上下文；PRE_SSA和POST_OZ用同一条任务文本（只有IR/设置元数据不同）。
任务明确“改善固定Oz流水线后的机器码尺寸”，不给Oz参考答案、反例或其他模型输出。
同一源对两设置调用顺序用source hash最低位交替，避免模型装载/时间顺序系统性偏置。
生成总上限128；任何请求失败也保留该槽位，不自动多采样补出想要的失败数。
仅明确没有产生输出的基础设施故障允许另行记录恢复执行；不得丢弃已产生的首次输出。

development允许调试；锁定实现后才处理scout的输出与恢复。
已查看的scout转为development，不能再叫确认集。P0b整体只做可行性，不是正式检验。

## 7. Oracle 与预算

默认每进程RSS≤8GiB；verify5秒，Alive2每次30秒，Oz＋后端测量每状态30秒。
n≤10时最多1024掩码完整枚举；每候选最多2048次Alive2、600秒。
n>10且builder可用时用固定B5 witness search：128次Alive2、256状态、128次成本评估、900秒。
表示无法可靠构建时直接记REPRESENTATION_UNSUPPORTED/UNRESOLVED，不凭空启动搜索。
搜索队列最多4096个提案；溢出移除最低优先级提案并记录，不能声称完备。
任何限制先到即停。所有实际验证调用均计数，包括最终X→P(R)；初始候选/基线公共前置检查单列。
P0b搜索批次总预算14400秒（单worker），不含生成但包含构图/反例/求解/测量。
成功候选的三次编译复核也在该候选与批次墙钟内；所有编译调用单独计数，单次cost evaluation包含重复复核。
对待处理候选按setting→model→project轮转，每轮最多完成一个新状态；耗时累计到各自预算。
不先做完“看起来容易”的候选再用残余预算处理难例。预算到时其余标未完成。
n≤10枚举顺序按mask整数升序；端点复用已有验证记录，严格子集排除两端及规范等同源/目标者。

每候选存在性类别：
- POSITIVE_COMPLETE：枚举完成且至少一个有益严格子集。
- POSITIVE_PARTIAL：已找到有益子集，但还有未决/未访问状态。
- NEGATIVE_COMPLETE：全部表示内合法严格状态都有决定性非成功结论，没有有益解。
- UNRESOLVED：无已知有益解，但存在未知、未测、未访问或表示构建失败。

n=1且两端重建可靠、表示内无严格状态，可为NEGATIVE_COMPLETE；只否定这一表示空间。
UNKNOWN不能当negative；20%unknown也不算“完整oracle”。
有未知状态时可以证明存在，不能报告精确最优或精确regret。
对N个核心失败候选，L=(全部已知positive)/N，U=(N−NEGATIVE_COMPLETE)/N。
L/U是该协议与表示边界内的部分识别界，不是置信区间；任意程序改写空间的上界仍未知。
positive包含大空间witness，不能漏计已找到的正例。N=0报告NA，不除0、不记0%。

## 8. 门控与统计

P0a硬门仅限：24个必需fixture、两端重建、终态解析、来源/哈希完整性、精确成本确定性、拒绝UNKNOWN接纳。
无自然样本可标 SMOKE_PASS_NATURAL_MISSING，不能标完整P0a通过。
artifact历史标签复现、双人kappa、三模型规模不再是运行自然两例的前置条件。
自然对齐先单人/机器辅助审计并披露；没有两位独立人类标注者不报人际kappa。

P0b裁决：
- ≥2个不同项目的自然自动post-Oz有益例：支持继续P1开发；仅是工程投入门，不是显著性或普遍性结论。
- 1个正例、只有一个项目、或全部表示/验证未决：LIMITED_OR_INCONCLUSIVE。
- 0个正例且已消耗固定预算：NO_POSITIVE_WITHIN_BUDGET，停止自动扩大投入，报告表示界和unknown。
- 若所有候选均NEGATIVE_COMPLETE，只能说当前批次与表示中无有益子集。
- 若只有合成例、手工例或Oz前收益：不通过自动主现象门。
没有“2/32太少所以失败”或“0/少量就证明低于5%”的统计捷径。

P0只描述分子分母、项目/模型/设置分层、覆盖和预算，不做小样本bootstrap过度推断。
按设置配对报告全部源函数的端到端收益；条件于REFUTED的两组不是同一子总体，不能直接因果比较。
三模型确认、功效、强基线和多重比较另见 EXPERIMENT_PLAN，不能拿开发数据选择后再作显著性确认。

## 9. 最低产物与不可变项

locks/：protocol、toolchain、datasets、models（P0a无生成可为NOT_USED）。
manifests/：sources、generation_slots、legacy_inventory、dedup_groups。
runs/：commands、raw_outputs、verifier_inputs_logs、components、states、cost_records。
reports/：P0A_DECISION、P0B_DECISION（对应阶段才生成），同时保存JSON和MD。

所有决策报告含实际版本、样本数、缺件、未完成、设置分层、门控证据及下一允许阶段。
run ID必须含protocol hash；中止后resume只允许相同锁/输入/代码版本，变更另开run并记录继承。
不得覆盖原始生成、验证日志或失败run；禁止在看结果后改成功定义、支持范围或分母。
