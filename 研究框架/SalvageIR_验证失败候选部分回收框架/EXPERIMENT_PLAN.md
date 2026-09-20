# SalvageIR 实验计划

## 1. 总体原则

实验按“现象门—机制门—泛化门—跨后端门”推进。上一门失败就停止增加系统复杂度，不用更多模块掩盖核心假设失败。

```text
P0 数据与现象门
  ↓ 有足量明确失败，且存在 profitable verified 子集
P1 最小机制门
  ↓ 比 ddmin/随机/依赖回滚有配对优势
P2 正式多模型与项目泛化
  ↓ 留一模型、留一项目仍成立
P3 后端真实性与 RISC-V 外部验证
```

## 2. 工具链冻结

正式运行前记录：

- LLVM/Clang/opt/llc/llvm-size/llvm-mca 的 commit 或发行版本；
- Alive2 commit、Z3 版本、构建参数和 timeout；
- x86-64 主机 CPU、microcode、操作系统和 governor；
- AArch64 补充平台；
- RV64GC/RVV 编译器 triple、`-mcpu/-march/-mabi`、模拟器或真实硬件；
- 所有命令、环境变量白名单和容器镜像摘要。

Alive2 官方说明其不支持跨过程变换，并可能对相关 pass 产生伪反例，因此主数据限制为过程内变换；这不是可忽略的工程细节。[Alive2 官方仓库](https://github.com/AliveToolkit/alive2)

## 3. P0：数据与现象门

### 3.1 目标

回答两个先决问题：

1. 非 RL 模型是否产生足量可解析且明确 refuted 的候选？
2. 这些候选是否真的存在 verified 且 profitable 的严格子集？

### 3.2 步骤

P0 的测试集、顺序扩样、模型允许列表、过滤规则、`n_oracle=12`、资源预算和数值门槛全部由 [`PILOT_PROTOCOL.md`](PILOT_PROTOCOL.md) 约束。执行顺序为：

1. 全量重放 LLM-VeriOpt 的非 RL 输出，只用于外部现象复核；
2. 在 IR-OptSet 的 160 函数项目隔离发现集上生成 code-size 对齐候选；
3. 候选不足时只按预声明规则启用采样重复和 160 函数保留池；
4. 对 `REFUTED_ELIGIBLE` 进行对齐审计和组件统计；
5. 对不超过 12 个组件的候选枚举所有依赖闭合子集；
6. 每个子集执行 LLVM verifier、Alive2 和目标函数 ELF 符号大小测量；
7. 冻结全部规则后，才允许打开 LLVM Opt Benchmark 的 240 函数确认集。

LLM-VeriOpt 候选以 latency 为目标，不能与 code-size 主结果合并。受控候选统一从 `default<Oz>` 后的源 IR 出发，并在 Prompt 中明确优化 code size。

### 3.3 P0 输出

- 数据、模型、工具链三个 lock 文件；
- 源函数、原始生成、去重簇和候选漏斗 manifest；
- 对齐审计、组件分布、oracle 可回收性和验证成本表；
- 含每一门分子、分母、区间和裁决的 `gate_decision.json/.md`。

### 3.4 P0 通过条件

P0 依次执行 E0 环境门、G0 候选供应门、G1 可分解门、G2 现象门和 G3 搜索可行性门。核心数值包括：三个模型家族各至少 30 个明确失败候选、总计至少 120 个；至少 60 个 oracle-complete 候选；oracle useful rate 的模型宏平均点估计至少 10%，且项目聚类 95% CI 下界超过预注册的 5% 最小实用效应。完整红黄绿判定见 [`PILOT_PROTOCOL.md`](PILOT_PROTOCOL.md)。

区间跨越 5% 时只允许按预声明保留池扩样一次；区间上界不超过 5% 时停止 SalvageIR 主线。不得降低定义把 syntax error、unknown 或完全回退到源程序混入成功。

## 4. P1：最小机制门

### 4.1 范围

- 单函数；
- 单基本块或完全相同 CFG；
- 整数/位运算/比较/select/cast；
- 无内存、调用、浮点和向量；
- 组件数不超过 P0 冻结的上限。

### 4.2 基线

| ID | 基线 | 排除的解释 |
|---|---|---|
| B0 | 整体丢弃/回退 | 现有原子提交策略 |
| B1 | 同模型整函数重采样 | 只是再试一次即可 |
| B2 | 同模型+完整 Alive2 反馈整函数修复 | 反例反馈本身即可 |
| B3 | 文本 diff hunks + ddmin | 普通 delta debugging 即可 |
| B4 | 随机依赖闭合回滚 | 结构约束已足够 |
| B5 | 依赖闭合 best-first，无反例切片 | 反例引导是否有贡献 |
| O | 穷举 oracle | 最佳可恢复上界 |

`llvm-reduce` 用于缩减触发错误的测试用例，并通过 interestingness test 保留“仍能触发问题”的性质；它不是直接构造有收益的正确子翻译，但可包装成 B3 的补充实现。[llvm-reduce](https://llvm.org/docs/CommandGuide/llvm-reduce.html)

### 4.3 P1 主要指标

- `Useful Salvage Rate`；
- 与 oracle 的收益差；
- 达到首个 verified 候选的验证调用数；
- 达到最佳安全候选的墙钟时间；
- 非法 IR 生成比例；
- `SALVAGED_VALID_NO_GAIN` 与 `NO_VERIFIED_SUBSET` 比例。

### 4.4 P1 判定

SalvageIR 必须在相同预算下优于 B5，才能说明贡献不只是“做了 LLVM 依赖闭包”。若只优于文本 ddmin 而不优于 B5，研究应降级为工程实现，不继续宣称反例引导算法贡献。

## 5. P2：正式多模型、多项目实验

### 5.1 数据划分

```text
engineering：仅调工具，不统计
development：调对齐、组件和搜索
validation：冻结算法后的消融与预算选择
held-out project：从未用于调整的上游项目
held-out model：从未用于调整的模型家族
```

同一源函数的所有模型输出必须在同一 split；近重复函数按规范化哈希聚类后整体划分。

### 5.2 支持层级

正式实验分别报告：

- Tier A：同 CFG、纯 SSA 表达式；
- Tier B：匹配 CFG，含 PHI/分支；
- Tier C：受限内存/指针；
- Unsupported：循环、复杂调用、验证器不支持等。

总体结果不能只保留 Tier A。方法覆盖率与 Tier 内效果并列报告。

### 5.3 强基线

除 P1 基线外，加入：

- `full repair + localized hint`：向同一模型提供反例和静态切片，但仍重写完整函数；
- `component-local LLM repair`：仅重写最高风险组件，与确定性核心方法比较；
- `best-of-k resampling`：把相同模型调用和验证预算全部用于重采样；
- `LLVM -Oz/-O3`：作为源程序传统优化性能参照，不作为恢复算法基线。

### 5.4 消融

| 消融 | 要回答的问题 |
|---|---|
| `-StructuralAlignment` | 结构对齐是否优于文本 diff |
| `-DependencyClosure` | 合法状态约束是否降低失败与浪费 |
| `-CounterexampleSlice` | 反例是否减少验证调用并提高恢复率 |
| `-ProfitRecovery` | 找到正确子集后重新加入组件是否保住收益 |
| `-UBClosure` | poison/undef/flag 专用闭包是否必要 |
| `ProxyOnly` | 若只按 IR 指令数选择，会损失多少真实后端收益 |

“去掉最终验证”不能作为运行系统消融，因为会输出不安全结果；只能离线统计如果省略该门会产生多少误接纳。

## 6. 指标定义

### 6.1 正确性

- LLVM parse/verify pass rate；
- Alive2 `VERIFIED/REFUTED/UNKNOWN`；
- 最终直接 refinement 通过率；
- 测试/差分 fuzzing 结果作为补充；
- verifier/tool failure 独立报告。

### 6.2 恢复效果

```text
UsefulSalvage = I[final verified and K(final) < K(source)]
```

- 条件有益恢复率；
- 端到端有益率；
- oracle 可恢复率；
- 恢复后的相对成本下降；
- 相对完整错误候选保留的目标编辑比例，仅作描述性指标。

### 6.3 成本

- Alive2 调用次数及时间；
- LLVM 编译次数及时间；
- 搜索状态数；
- 峰值内存；
- LLM 请求、Token、费用；
- 总 wall-clock。

### 6.4 性能

主指标冻结为：源 IR 先经 `default<Oz>`，随后在固定 x86-64 generic 后端生成 ELF 对象，由 `llvm-readobj --symbols` 读取目标函数 `ST_Size`。成功至少减少 `max(2 bytes, 1%)`。Prompt、搜索目标和最终评价均使用 code size，不得在结果不利后切换到 latency。

补充：

- x86-64 `llvm-mca`/TTI；
- 可执行子集真实时间；
- AArch64 代码大小和静态成本；
- RV64GC/RVV 代码大小和指令数；真实或模拟周期仅作探索性补充。

LLVM Opt Benchmark 明确提醒 IR 差异只是代理，真实运行性能可能因后端识别而相反，因此论文必须避免把 IR 指令数变化写成运行加速。[LLVM Opt Benchmark](https://github.com/dtcxzyw/llvm-opt-benchmark)

## 7. 统计计划

### 7.1 主要比较

主要终点是每个冻结候选上的二元 `UsefulSalvage`。SalvageIR 与每个基线是配对数据：

- 报告配对差值和 cluster bootstrap 95% CI；
- bootstrap 以源函数或项目为簇，避免同一函数的多个模型输出被当作独立样本；
- 可补充 McNemar 检验；
- 多基线次要显著性检验使用 Holm 校正。

### 7.2 模型效应

使用模型分层结果和模型宏平均。若样本允许，可拟合：

```text
success ~ method * model_family + component_count + failure_type
          + (1 | project/source_function)
```

回归只是辅助解释，不取代配对原始结果。

### 7.3 连续成本

- 成本比例使用几何均值与 bootstrap CI；
- 实际运行时间预热后重复，报告中位数、MAD/IQR 和 bootstrap CI；
- 预先规定异常值和计时失败规则；
- 无收益和失败候选在端到端聚合中按零收益，不从平均数中删除。

### 7.4 样本量

P0 后根据配对不一致比例做 power 分析。正式 N 不在观察结果前凭经验写死。论文同时报告功效假设、预计失访和实际可用候选数量。

## 8. P3：RISC-V 保留验证

RISC-V 不参与：

- 候选生成 Prompt；
- 组件风险排序；
- 主成本函数；
- 开发集阈值；
- 方法选择。

方法完全冻结后，把 `S`、最佳 x86-64 恢复结果 `R` 和各基线结果编译到：

- RV64GC 标量后端；
- 方法适用时的 RVV 配置；
- 固定 `-march/-mabi/-mcpu`。

报告：

- LLVM IR refinement 仍使用 target-independent 主检查；
- RISC-V 编译成功率；
- `.text` 大小；
- 静态指令数及关键指令类别；
- 可运行子集在真实板卡或固定模拟器上的周期；
- x86 收益在 RISC-V 上保持、持平、反转的比例。

RISC-V 结果用于回答可移植性，而不是把换平台写成方法创新。

## 9. 复现包

最终 artifact 至少包含：

- 冻结 manifest 和许可证；
- 不含敏感信息的 prompts/configs；
- 每个模型的原始输出哈希和可再分发内容；
- LLVM/Alive2 容器或构建脚本；
- 每个验证调用的输入、stdout/stderr 和状态；
- 组件图、搜索 trace 和成本日志；
- 表格生成脚本；
- 主结果一键小规模重放与完整重放说明。

## 10. 阶段退出表

| 阶段 | 进入条件 | 退出条件 | 失败后的动作 |
|---|---|---|---|
| P0 | artifact/模型可访问 | 证明现象存在且有搜索空间 | 停止主线或改做失败谱系数据论文 |
| P1 | 有 oracle 数据 | 超过依赖闭合强基线 | 降级工程工具，不扩范围 |
| P2 | 方法冻结 | 跨模型/项目配对优势 | 收窄到有效模型/IR 范围 |
| P3 | 主结论稳定 | 完成 AArch64/RISC-V 外部验证 | 如收益反转，明确 target-specific 边界 |
