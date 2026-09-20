# 历史归档：v2，不得用于新实验执行

原文件：THREATS_AND_REVIEW.md。2026-09-21 被 v3 替代；保留原文用于追溯，不代表当前约束。

# SalvageIR 反方审查、风险与可证伪条件

## 1. 最强反对意见

### 反对意见 A：这只是把程序修复方法搬到 LLVM IR

该意见目前成立一半。ITER、MENTOR、PReMM 已覆盖部分补丁、多位置定位、CEGIS 和依赖聚类。SalvageIR 只有在以下部分被实验证明确有独立作用时，才不是简单迁移：

- LLVM refinement 而非测试通过；
- poison/undef/flag/SSA/CFG/MemorySSA 的语义闭包；
- 从错误优化中保留性能，而不是修复原程序 bug；
- 回滚后重新加入组件的利润恢复；
- 整函数直接验证和目标后端收益。

LLVM SandboxIR 已提供事务 save/accept/revert 和 region 粒度盈利接受，因此“IR 可回滚”也不能作为创新。

**裁决实验：**B5 无反例 best-first、B6 同组件图 hierarchical ddmin、B7 反例 hitting-set MaxSAT/ILP，以及 `-UBClosure/-ProfitRecovery` 消融。若无实质差异，删掉算法新颖性主张。

### 反对意见 B：失败候选可能根本没有可救部分

LLM-VeriOpt 证明失败候选数量不为零，但没有证明其内部存在 profitable verified strict subset。

**裁决实验：**P0 oracle 穷举。若现象罕见或只存在于合成案例，停止主线。

### 反对意见 C：所谓组件来自不可靠的 IR diff

LLVM 优化会重命名、重排、消除和合并 SSA 值，源/目标不存在唯一 diff。对齐错误会制造虚假的“子变换”。

**缓解：**结构指纹、多层置信度、低置信区域整体合并、`Build(all source/target)` 自检、人工抽样审计。论文报告对齐覆盖与错误，不只报告恢复成功样本。

### 反对意见 D：反例不能定位错误组件

Alive2 的反例说明整体不精化，不保证某个 slice 内组件就是原因；两个单独错误组件组合后也可能正确。

**缓解：**反例仅排序，不作为 soundness 硬剪枝；所有候选仍完整验证；记录 `CE_LOCALIZABLE/STATIC_ONLY/UNLOCALIZABLE`；真实反例必须优于同失败类别内打乱归属的安慰剂反例和纯静态切片。

### 反对意见 E：删除修改后得到正确程序只是回到源程序

如果主要结果是回退，大部分“恢复率”没有研究价值。

**缓解：**只有 `K(R)<K(S)` 才算主成功；返回 S、与 S 持平或只优于错误 T0 均不计成功。

### 反对意见 F：强模型错误太少，方法只服务弱模型

这是最重要的外部有效性风险。

**缓解：**模型漏斗、模型宏平均、留一模型实验和 method×model 交互；若只在弱模型有效，标题、摘要和结论明确收窄。

### 反对意见 G：方法赢只是因为调用更多验证器

搜索式方法天然可以通过更多尝试提高成功率。

**缓解：**同时给出 16/32/64/128/256 验证预算性能剖面、相同 wall-clock 和等美元成本曲线；与 best-of-k、随机组件和全部强结构搜索对照。

### 反对意见 H：Alive2 本身有边界或 bug

Alive2 是实用的 bounded translation validator，但不支持跨过程变换，复杂特性可能 unsupported 或 timeout。

**缓解：**只把 `VERIFIED/REFUTED` 当明确结果；锁定 commit；保存原始日志；对成功结果补充 differential testing；抽样交叉验证；Trivet 等工具成熟后可作为补充而不是事后替换裁决器。

### 反对意见 I：代码大小收益不能代表速度

成立。主实验若选择 `.text` 大小，只能主张 code-size salvage。

**缓解：**标题中的“optimization”在摘要中说明主目标；有 harness 的子集实测速度；不把 IR 指令减少或静态延迟写成实际加速。

### 反对意见 J：RISC-V 只是换平台

成立，因此 RISC-V 不列为贡献机制。

**缓解：**方法冻结后才运行 RISC-V；只在同一源代码独立生成的 RV64GC IR 上评价，区分冻结决策投影与冻结算法重跑，禁止直接改 x86 IR triple。

### 反对意见 K：这不是 LLM 论文，而是通用 IR repair

如果只把 LLM 当作失败输入生成器，且方法对匹配的随机/传统变异同样有效，那么 LLM 不是机制的一部分。

**裁决实验：**构造编辑数、组件数、CFG 变化和 semantic-flag 类别匹配的非 LLM 负对照。若候选来源与方法无交互，改投通用 RCES/IR repair，不声称 LLM 特异贡献。

### 反对意见 L：`-Oz` 与函数符号大小可能是测量假象

`opt default<Oz>` 加任意 `llc -O` 不自动等价于 `clang -Oz`；函数 `ST_Size` 也忽略链接布局、ICF、常量池共享和 relaxation。

**裁决实验：**先通过 `clang -Oz`/IR 路径和原模块/抽取 harness 双校准；函数级结果再由 D-H 链接后二进制验证。任一校准失败都停止 code-size 主张。

## 2. 风险登记表

| 风险 | 概率 | 影响 | 早期信号 | 降级方案 |
|---|---:|---:|---|---|
| 语义 refuted 样本不足 | 中 | 高 | 多数失败是 syntax invalid | 扩充受控多模型生成；不混入 syntax 主任务 |
| 可回收严格子集罕见 | 中高 | 致命 | P0 oracle 接近零 | 停止算法主线，转为失败候选可分解性负结果 |
| 对齐失败率高 | 高 | 高 | 同构自检失败、coarse component 多 | 主张收窄到同 CFG/单块；发布覆盖率 |
| Alive2 timeout/unsupported 高 | 中 | 高 | unknown 漏斗大 | 缩小语言子集；不把 unknown 转为失败样本 |
| 搜索成本过大 | 中 | 高 | 验证次数接近穷举 | 强化闭包、缓存和预算；只主张离线恢复 |
| B5/B6/B7 已达到相同效果 | 中高 | 致命 | 预算曲线和反例消融无差异 | 删除反例引导贡献，降为结构化回滚工具 |
| 真实反例≈打乱反例 | 中 | 致命 | real/shuffled AUC 无差异 | 删除动态反例定位主张 |
| SFT 与 code-size 目标错配 | 高 | 高 | Track A/B 错误谱显著不同 | Track B 只用原始 instruct，SFT 单列 |
| `-Oz` 成本管线未校准 | 中 | 致命 | 与 clang/reference 不一致 | 停止主实验，先修测量 |
| 后端真实收益消失 | 中 | 高 | IR 减少但 `.text`/runtime 不变 | 改用真实性能排序；若仍无效则停止盈利主张 |
| 只对一个模型有效 | 中 | 高 | method×model 强交互 | 收窄适用模型，禁止一般化 |
| artifact 许可/内容不足 | 中 | 中 | 缺 raw IR 或日志 | 受控重新生成并公开 manifest |
| API 模型漂移 | 高 | 中 | 同配置输出变化 | 开源权重为主，API 只作时间戳外测 |
| RISC-V 收益反转 | 中 | 中 | 大量 `.text` 或周期退化 | 如实给出 target-specific 边界，不改主算法 |

## 3. 可证伪条件

以下任一结果足以否定或显著收缩主张：

1. P0 中几乎没有 `REFUTED_ELIGIBLE` 候选存在 profitable verified strict subset；
2. SalvageIR 在预算剖面下不优于 B5/B6/B7 的最强者；
3. 真实反例不优于打乱反例，或去掉反例切片、UB 闭包、利润恢复后结果基本不变；
4. 优势完全由更多 Alive2 调用、更多 wall-clock 或额外 LLM 请求解释；
5. 结果只在单一弱模型、单一项目或近重复模板上成立；
6. 模型宏平均或留一模型结果不支持方向一致性；
7. 主性能收益在固定后端真实测量中消失；
8. 结构对齐覆盖率低到使结果只适用于非常小的人工子集；
9. 成功依赖把 timeout/unsupported 当作错误或正确；
10. 从同一源独立生成的 RV64GC IR 上收益大量反转，且分析表明主成本模型只过拟合 x86 后端；
11. 匹配非 LLM 负对照与 LLM 候选无可区分差异，却仍声称 LLM 特异贡献；
12. `clang -Oz`/IR 路径或原模块/抽取 harness 校准失败。

负结果不能改名为“发现边界”后继续保持原贡献措辞。只有在预先计划的 P0 现象研究本身规模和分析足够时，才能单独形成负结果或数据论文。

## 4. 审稿人会追问的证据

### 新颖性

- 与 ITER、MENTOR、PReMM 的逐项机制差异；
- 为什么不是 llvm-reduce + Alive2；
- B5/B6/B7、打乱反例和 oracle 是否共同证明反例引导有贡献；
- 是否搜索到同构的 2026 新工作。

### 正确性

- refinement 方向是否一致；
- poison/undef/attribute 怎么处理；
- final direct check 是否始终执行；
- Alive2 unknown 和 tool error 怎么统计；
- 成功 IR 是否可公开复核。

### 公平性

- 初始候选是否完全相同；
- 强/弱模型是否分开；
- 是否只筛选有利失败；
- Token、验证次数与墙钟是否匹配；
- 同一源函数的多个候选是否错误地当作独立样本。

### 实用性

- 每个成功恢复花多少验证时间；
- 为什么不直接重采样；
- `.text` 或 runtime 是否真的改善；
- 支持多少真实函数和 IR 特性；
- RISC-V 结果是否只是装饰性表格。

## 5. 回答策略

论文应采用保守措辞：

- 写“recover profitable verified sub-translations within the supported scope”，不写“repair arbitrary LLVM IR”；
- 写“counterexample-guided prioritization”，不写“counterexample precisely localizes the fault”；
- 写“Alive2-verified under the configured semantics and bounds”，不写“universally proven equivalent”；
- 写“cross-model evidence over the evaluated families”，不写“model agnostic”；
- 写“held-out RISC-V evaluation”，不写“RISC-V-aware framework”。

## 6. 最终 go/no-go 决策表

| 观察 | 决策 |
|---|---|
| 现象存在，M 优于 B5/B6/B7，且 real CE 优于 shuffled CE | 继续完整论文 |
| 现象存在，但 M≈强结构基线或 real≈shuffled | 改成 LLVM-aware structured rollback，降低创新等级 |
| 只有 LLM 局部修复有效 | 改为局部修复论文，重新做模型公平协议 |
| 现象只在弱模型存在 | 收窄到低资源模型后处理或停止 |
| 只有 IR 指令收益 | 不声称 code size/runtime；评估是否仍值得投稿 |
| 现象罕见/不存在 | 停止 SalvageIR 主线 |
