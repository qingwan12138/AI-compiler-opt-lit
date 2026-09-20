# SalvageIR 唯一裁决卡

> 作用：这是执行时的单页裁决入口。详细定义以 [`PILOT_PROTOCOL.md`](PILOT_PROTOCOL.md) 为准；若其他概述文件与本卡冲突，以本卡和预注册版本较新的协议为准。
> 原则：F、I、S 三层分别裁决，不用一个综合分数掩盖某层失败。

## 0. 固定分析单位

| 项目 | 固定值 |
|---|---|
| 主要候选 | Track B 冻结 instruct checkpoint 产生的唯一 `REFUTED_ELIGIBLE`；本研究不做任务特定训练/RL |
| 主要正确性 | 最终直接 `Alive2(S,R)=VERIFIED` |
| 主要收益 | 校准的 x86-64 `ST_Size` 至少减少 `max(2 bytes,1%)`，alloc non-BSS 不增加 |
| 主要聚类 | 项目→源函数层级；模型先分层、再等权宏平均 |
| 主预算点 | 128 次 Alive2、256 状态、15 分钟、8 GiB |
| 预算剖面 | 16/32/64/128/256 次 Alive2 |
| 主搜索排序 | 无权重字典序：real CE→static slice→gain upper bound→risk→rollback atoms→stable ID |
| oracle | 结构可重放组件数 `n<=12` 的全部闭合状态 |
| 大规模下界 | `n>12` 的 128-call fixed witness search；未找到不算不存在 |

## E：环境有效性门

全部满足才可进入自然候选实验：

- artifact parser/verifier 标签复现率 `>=99%`，Alive2 终态复现率 `>=95%`；
- 机器码三次构建逐字节一致；
- IR 成本路径与 `clang -Oz` 参考路径在至少 30 个可比函数上机器码一致率 `>=95%`；
- 抽取 harness 与原模块编译的目标函数字节一致率 `>=95%`；
- provenance 集编辑原子 precision/recall `>=98%`，预声明结构依赖边 recall `=100%`；
- 自然对齐审计 Cohen's kappa `>=0.80`；
- 反例规范化双次重放一致率 `>=95%`。

任一项失败：**STOP-ENV**，先修基础设施，不得进入效果统计。

## F：现象门

### F0 候选供应

- 三个模型家族各 `>=30` 个唯一核心失败，总计 `>=120`；
- 至少 60 个源函数、20 个项目；单项目 `<=15%`，单源函数 `<=3%`。

失败：删除“跨模型通用”，只能重写为足量家族的范围受限研究。

### F1 可分解与覆盖

- `>=60` 个 oracle-complete，每家族 `>=15`；
- 来自 `>=30` 个源函数、`>=20` 个项目；
- oracle-complete 覆盖核心失败候选 `>=30%`；
- 至少 50% 核心候选具有 2–20 个组件；oracle incomplete `<=20%`；
- `CE_LOCALIZABLE` 覆盖核心候选 `>=60%`。

反例覆盖单项失败但其余通过：仍可研究结构回滚，但删除动态反例定位主张。

### F2 现象存在

主要端点：`OracleEligibleUsefulRate`。D-B/D-BR 必须满足：

- 模型宏平均点估计 `>=10%`；
- 项目→源函数层级 bootstrap 95% CI 下界 `>5%`；
- 至少两个模型家族点估计为正；
- 单项目、模板或模型家族不贡献超过 50% 的成功。

D-C 独立复核：点估计必须 `>=5%`，且至少两个家族观察到 oracle 或 large-n witness 正例。D-B 通过而 D-C `<5%`：**STOP-PHENOMENON**。

D-B 区间跨 5% 时只允许启用一次 D-BR；上界 `<=5%` 立即停止。

### F3 搜索可行性

- 75% 核心候选组件数 `<=20`；
- Alive2 单状态 median `<=5s`、P90 `<=30s`；
- timeout/unsupported `<=10%`；
- oracle-positive 中“只能保留一个组件”的比例 `<=80%`。

失败：现象论文仍可能成立，但不宣称可扩展搜索系统。

## I：机制识别门

### I1 对强结构基线

对 B5 无反例 best-first、B6 hierarchical-ddmin、B7 MaxSAT/ILP diagnosis 分别定义配对差 `Delta_j=M-Bj`。主预算点必须对三个比较同时满足：

- 每个 `Delta_j >= 5pp`；
- 三个比较经 Holm 调整后的配对层级 bootstrap 95% simultaneous CI 下界均 `>0`；
- 每个比较至少两个模型家族差值 `>0`；
- 首个有益结果的验证调用数 median 不劣于三个基线中的最佳者；
- 共同成功案例的收益相对 oracle 不超过 `max(2 bytes,1%)` 非劣界。

### I2 反例信息增益

在相同状态图、初始队列和预算下比较 real CE 与同失败类别内 shuffled CE。定义：

```text
CEInfoGain = (AUC_real - AUC_shuffled) / max(AUC_shuffled, epsilon)
```

其中 AUC 是 16/32/64/128/256 验证预算下 `UsefulSalvageRate` 的归一化曲线面积。必须满足：

- `CEInfoGain >=10%`；
- 项目→源函数配对 bootstrap 95% CI 下界 `>0`；
- 至少两个模型家族方向为正；
- real CE 同时不劣于 pure static slice。

I1 或 I2 任一失败：不得声称“counterexample-guided algorithm”，降级为结构可重放 rollback 工具。只在一个预算点获胜不算通过。

## S：系统真实性与跨后端门

### S1 x86 端到端

在 D-H 全部 24 个预注册程序上，未恢复程序安全回退并计零收益：

- 链接后二进制 allocatable code+rodata 宏平均 delta `<0`；
- 以程序为簇的 bootstrap 95% CI 上界 `<0`；
- 参考输出全部通过；任何错误程序按方法失败并回退。

失败：只能声称函数级 salvage，不声称端到端系统收益。

### S2 RV64GC 外部验证

只使用从同一源代码独立生成的 `S_rv/T0_rv`：

- RV64GC 核心候选对象生成率 `>=95%`；
- 每个输出独立 `Alive2(S_rv,R_rv)=VERIFIED`；
- 回退计零后的链接二进制 code+rodata 宏平均 delta `<0`；
- 以程序为簇的 bootstrap 95% CI 上界 `<0`；
- 至少两个模型家族方向一致。

冻结编辑决策的跨目标锚点映射覆盖 `>=70%` 才单列“decision transfer”；否则只报告冻结算法重跑，不把映射成功案例外推。

失败：只能写“存在 RISC-V 可迁移案例”或“RISC-V 上无总体提升”，不得写框架在 RISC-V 上有指标提升。

## 最终投稿定位

| 通过情况 | 允许定位 |
|---|---|
| E+F | 失败候选可回收性测量/数据论文 |
| E+F+I | Translator 后处理方法论文；系统收益仍限函数级 |
| E+F+I+S1 | 有端到端证据的 Translator 方法论文 |
| E+F+I+S1+S2 | 可声称在独立 RV64GC 流水线上也有总体代码尺寸提升 |
| F 失败 | 停止 SalvageIR 主线 |
| I 失败 | 降级结构 rollback，不保留反例算法主张 |
