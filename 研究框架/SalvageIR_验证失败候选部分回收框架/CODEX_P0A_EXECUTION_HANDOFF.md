# SalvageIR v3：AutoDL 独立执行交接

> salvageir_v3 · 2026-09-21。只复制SalvageIR文件夹即可；不需要Windows路径或ARIS。
> 这是实施规格，不表示下述CLI或测试已经存在。先检查用户已有实现并适配，禁止覆盖重建。

## 1. 直接开始的提示词

> 读取README.md、本文、RESEARCH_CONTRACT.md、PILOT_PROTOCOL.md和ALGORITHM_SPEC.md，按v3执行P0a。先审计环境及已有32条结果，再实现/运行最小测试、编辑重放和两例自然候选审计。保留旧代码与日志，不做训练/RL，不运行RISC-V，不自动启动P0b或付费/大型下载。不要停在只写计划；缺件时完成其余本地可逆任务后明确列出缺件及未完成项。

## 2. 先做只读preflight

在用户当前SalvageIR目录工作，所有路径相对该目录解析。检查已有AGENTS.md和Git状态。
记录Python、pytest、clang、opt、llc、llvm-readobj、alive-tv、Z3、RAM/磁盘；GPU只读查询，P0a不需要GPU。
Alive2若不支持--version，保存binary SHA及源码commit/build记录，不因缺该参数误判不存在。
列出已有src/tests/configs/runs/data/logs，不递归读取权重、密钥或无关文件。
识别原来32条所在目录及协议；找不到时只向用户索要该目录/文件，不要求重新复制整个ARIS。

按PILOT_PROTOCOL验收锁定工具链。缺LLVM/Alive2时：
- 可先实现schema、日志解析、预算和离线fixture测试；
- 真集成测试标SKIPPED并列依赖；
- 安装、大型下载、模型访问或长时任务需要具体资源授权，不能假装跑过。

若当前目录存在ARIS技能，可在读完其指令后用它记录/审计；不存在不阻塞。
不修改其他项目的refine-logs，不依赖本机C盘路径。

## 3. 产物目录

尽量复用现有实现；新内容放本目录experiment/，旧实现通过adapter接入，不迁移或删除用户文件。

~~~text
experiment/
  src/salvageir/           # 新建或适配现有模块
  tests/                  # 纯单元 + 真实工具集成，分别统计
  locks/
  manifests/
  runs/<protocol-hash>/<run-id>/
  reports/P0A_DECISION.json
  reports/P0A_DECISION.md
~~~

首次创建协议锁包含所有当前规范SHA、configs/pilot_v3.json SHA及实现版本。
models.lock在P0a可NOT_USED；legacy模型信息来自原日志，缺失就UNKNOWN。
大型输入引用用户现有只读位置；没有授权不复制模型权重、不下载全量artifact。
git提交/推送不是P0a运行的前置条件，不自动push。

## 4. 六项实施任务与验收

每项使用测试先行；先跑具体失败断言，再最小实现，再保存真实通过记录。
以下是接口契约而非声称已有命令。若用户已有等价CLI，建立映射表，勿另造重复实现。

### T0 锁与来源

实现read-only inventory和hash记录。测试：
- 同一内容ID稳定，协议/模型/输入设置变化导致不同ID；
- 未知值为null+reason，不能默认0；
- 日志不包含secret环境字段；
- 旧run的输出不能被新run覆盖。

交付：environment.json、legacy_inventory.jsonl、protocol/toolchain锁、缺件列表。

### T1 唯一状态机与命令运行器

按DATA_AND_FAIRNESS_PROTOCOL的candidate_class顺序实现。
使用argv数组，统一捕获exit/stdout/stderr/timeout/hash/RSS。
测试：parse失败不是REFUTED；timeout/crash不是VERIFIED；截断保留原文；
scope不支持与verifier不支持分开；所有预定槽位只有一个终态；没有记录不能判PASS。

交付：全部已有记录的新旧终态对照；不自动把用户报告的2条当成失败对。

### T2 composer与24个fixture

实现P0固定CFG白名单、原子来源、硬蕴含/软风险分离。
测试全部PILOT_PROTOCOL的24个fixture，特别包括：
- all0/all1端点准确重建；
- 撤销错误nsw不强迫撤销全部消费者；
- 联合修改才正确不能被错误剪枝；
- 不产生来源之外的glue；
- CFG重构覆盖失败保留在核心分母；
- 语义属性/调用/内存/向量等范围拒绝。

自然审计不能拿人工编辑脚本当自动diff的输出。
24个是测试覆盖结构，不是要求把只有端点的人工案例计入自然效果。

### T3 verifier与成本链

实现直接X→R、X→Oz(R)两个记录；确定性命令和版本化解析器。
测试：同一日志解析一致；两个不同有效witness不会仅因不同降级；
不支持poison的普通解释器不能给CE_LOCALIZABLE；
witness无法可信解释时降为STATIC_ONLY/UNLOCALIZABLE，不删候选。

精确测量双边同Oz/同后端、ST_Size与alloc_non_bss、三次计量确定性。
最终证明未完成不能成功。LLVM/Alive2命令超时、非零退出与资源耗尽均保存。
固定LLVM管线与clang -Oz的差异仅诊断，不擅自声称完全等价。

### T4 自然两例与表示相对穷举

审计全部已有REFUTED对，不按有希望程度挑两例。
若确为2例，分别提供：
source/target/log出处、原始输入阶段、错误类型、人工编辑意图、
自动组件及硬边理由、端点重建、穷举/预算搜索状态、每个已证明结果的post-Oz字节。
n≤协议阈值才穷举；大空间固定B5；oracle返回四态，不把unknown当不存在。
旧post-Oz或目标错配样本按LEGACY说明；有内存等范围外旧样本不强行改造为v3主例。
只允许自动严格子集计入自动结果；手工正例另列“表示诊断”。

如果没有原始数据：保留NATURAL_DATA_MISSING，不编造结果，也不偷偷用合成两例替换。

### T5 验收报告

运行已有测试入口或实现以下等价入口后运行：

~~~text
python -m pytest experiment/tests/unit -q
python -m pytest experiment/tests/integration -q
python -m salvageir preflight
python -m salvageir audit-legacy
python -m salvageir audit-components
python -m salvageir report-p0a
~~~

这些命令是拟议接口，需要先适配实际package安装/PYTHONPATH；没有实现不能直接宣称可运行。
测试分别记录passed/failed/skipped和真实命令。集成缺依赖则完成状态PARTIAL，不用mock结果替代。
报告JSON/MD一致，覆盖P0A_AUDITED、FAIL_INFRA、BLOCKED_TOOLCHAIN、SMOKE_PASS_NATURAL_MISSING。
P0A_AUDITED可以包含零成功，它表示审计完成，不表示H1成立。

## 5. 已有实现迁移清单

逐项确认而非仅替换opt命令：
- 旧协议保留，raw不可被source.oz覆盖；
- 增加setting与source-stage字段，明确legacy；
- PRE_SSA实际准备和raw→S证明；
- 每个结果的post-Oz成本及最终证明；
- 硬结构/软风险分开、两端重建；
- 删除“双次相同witness才能定位”的要求；
- 对齐失败仍进条件分母；
- oracle明确未决与部分正例；
- 预算包括最终证明/Oz/编译/构图；
- 当前统一状态名与旧标签映射；
- 端到端包括所有原始槽位，禁止按失败数无限生成。

## 6. 最终报告与停点

报告：实际处理几条、哪两条是否真的REFUTED、多少scope/representation失败、
是否有自动严格子集、Oz前后收益、原始日志链接、资源开销、未完成项。
不要只说“预实验完成”，不要输出无依据评分。
P0a完成后不得自动运行P0b。给出P0b预估GPU/CPU时间及费用（根据已测吞吐），请求对应预算授权。
当前机器缺数据/权限可构成具体阻塞；不因为框架还需正式实验就停在空泛计划。
