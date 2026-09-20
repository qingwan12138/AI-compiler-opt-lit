# SalvageIR：面向 LLM 生成 LLVM IR 优化的精化引导部分回滚

英文题目建议：**SalvageIR: Refinement-Guided Partial Rollback for LLM-Generated LLVM IR Optimizations**

## 1. 问题陈述

现有 LLM IR 优化系统通常遵循“生成完整候选—整体验证—接受或回退”的原子提交语义。一旦完整候选被形式验证反驳，候选中所有修改都会随之丢弃，即使其中一部分修改本身正确且有收益。

SalvageIR 研究一个更窄但可检验的问题：

> 被明确证伪的完整 LLVM IR 翻译，是否经常包含可重新组合为正确且有收益结果的子翻译；若存在，能否用 LLVM 结构依赖和反例证据在固定预算内找到它们？

该问题不是提高验证覆盖率，也不是修复任意程序 bug。源函数 `S` 是语义规范，目标 `T0` 是优化翻译，系统允许撤销目标中的错误变换，但不允许改变源函数契约。

## 2. 研究空白推导

### 2.1 已经解决的内容

- LLM-Vectorizer、LLM-VeriOpt、IntOpt 等工作已经证明 LLM 可以生成 LLVM IR 或优化程序，并用 Alive2 做验证。
- Compiler Generated Feedback、LPO 等工作已经把编译错误或验证反例反馈给 LLM 进行完整候选重试。
- CoV 已经用静态与运行时验证扩大 LLM 优化的可接纳范围。
- ITER、MENTOR、PReMM 等程序修复工作已经证明部分补丁、多位置修复、反例引导定位并非新概念。

### 2.2 仍未直接解决的内容

当前定向检索尚未发现同时满足以下条件的方法：

1. 输入是 LLM 生成的、已被明确证伪的 LLVM IR 优化翻译；
2. 不是让模型重新生成整个程序，而是把源到目标的变化显式分解为 LLVM-aware 编辑组件；
3. 在 poison、undef、SSA、CFG 和内存依赖约束下回滚或重放组件；
4. 用 LLVM refinement 逐步裁决组合；
5. 目标不是最小补丁，而是恢复 verified 且 profitable 的子翻译；
6. 对完全相同的失败候选进行跨模型、配对和预算相等的评价。

因此剩余空白不是“利用反例修复代码”，而是：

> **面向 LLVM 优化翻译的 refinement-aware partial commit。**

“尚未发现”仅代表截至 2026-09-20、在当前数据库和检索式下未找到直接同构工作，不构成“首次”声明。

## 3. 系统总览

```text
源 IR S ───────────────┐
                      ├─ M1 规范化与结构对齐
失败候选 T0 ──────────┘          ↓
                          候选编辑原子
                                ↓
                 M2 结构可重放闭包 → 候选编辑组件 CEC
                                ↓
Alive2 反例 ──→ M3 反例相关切片与回滚命中约束
                                ↓
                    M4 预算化回滚/重放搜索
                     ↙ refuted          ↘ verified
              累积反例与切片          收益评估
                     ↖                   ↓
                      └────────── 保留安全 incumbent
                                             ↓
                                  最终整函数 S -> R 验证
                                             ↓
                               verified 且 profitable 才输出
```

## 4. 四个核心模块

### M1：规范化与结构对齐

输入是 `S` 与 `T0`。系统移除不影响语义的调试噪声，稳定化匿名值和基本块标识，但保留数据布局、目标 triple、函数属性以及 `nsw/nuw/exact/inbounds` 等语义相关信息。

对齐不依赖行号，而使用：

- 参数、返回值和外部可观察位置作为锚点；
- CFG 前驱/后继、支配与后支配签名匹配基本块；
- 操作码、类型、常量、操作数来源和交换律规范化匹配指令；
- 无法可靠对齐的区域形成一个整体组件，不猜测细粒度对应关系。

输出是源/目标映射和候选编辑原子，包括增加、删除、替换指令，操作数变化，flag/attribute 变化，以及可选的 CFG/PHI 变化。

### M2：依赖闭合的候选编辑组件

单个文本编辑通常不是可执行单位。SalvageIR 对编辑原子求闭包：

- SSA def-use 闭包；
- 删除定义与其剩余使用的闭包；
- PHI 与 CFG 边闭包；
- 控制依赖闭包；
- MemorySSA/别名相关闭包；
- poison、undef、freeze 和语义 flag 约束闭包；
- 函数属性与调用约定闭包。

闭包后的最小构建单位称为 **Candidate Edit Component（CEC，结构可重放候选编辑组件）**。“闭合”不表示语义独立或正确。只有当一个 CEC 或 CEC 组合已经通过整函数验证时，才称为 **Verified Optimization Component（VOC，已验证优化组件）**。

组件图 `G=(C,D)` 中，`C` 是 CEC 集合，`D` 表示必须同时存在或先于重放的依赖。任意搜索状态都必须是依赖闭合的组件子集。

### M3：反例相关切片

Alive2 的反例说明完整候选不满足 refinement，但通常不直接给出唯一错误指令。SalvageIR 不把反例解析包装成完备故障定位，而把它作为搜索引导：

1. 保存反例原始文本、版本、状态 ID 和哈希；
2. 双次重放后分为 `CE_LOCALIZABLE`、`CE_STATIC_ONLY` 和 `CE_UNLOCALIZABLE`；
3. 对可稳定物化的输入，在当前源/候选状态上复现可观察差异；
4. 从差异的返回值、内存或 UB/poison 事件逆向构造数据与控制依赖切片；
5. 把切片投影到当前仍活跃的 CEC，形成 state-conditioned 可疑集合；
6. 多个反例只形成软排序，不形成 soundness 剪枝。

切片只改变搜索优先级，不能声称未进入切片的组件必然正确，也不能据此跳过最终验证。真实反例必须与静态切片、同失败类别内打乱归属的反例和无反例进行四方对照；否则无法证明反例信息本身有效。

### M4：部分回滚与利润恢复

搜索从完整失败目标开始。每一步选择一个依赖闭合的回滚集，把相应区域恢复到源版本并重建合法 IR。

- 若候选无法通过 LLVM verifier：记录结构失败并扩大闭包。
- 若 Alive2 再次 refuted：累积反例和切片，更新命中约束。
- 若 Alive2 verified：加入最多 8 个非支配状态组成的安全前沿；继续探索其他安全盆地，并从前沿轮转重加组件以恢复利润。
- 若 timeout/unsupported：标记 unknown，不更新安全集合。

搜索目标不是保留最多编辑，而是：

```text
maximize Gain(R) - lambda_v * VerifyCost(R) - lambda_s * SearchCost(R)
subject to Alive2(S, R) = VERIFIED
```

实际实现不声称精确求全局最优。第一版用冻结评分的确定性 best-first/beam search，并在组件数较小时提供穷举 oracle 作为上界。实现优先复用 LLVM SandboxIR 的事务 save/accept/revert；事务回滚本身不列为创新。

## 5. 安全不变量

任何阶段都必须保持：

1. 安全 incumbent 只包含整函数验证成功的候选；
2. unknown 永远不进入 verified 集合；
3. 搜索代理成本不能替代实际后端成本；
4. 局部或逐步证明不能替代最终 `Alive2(S,R)`；
5. 最终收益必须相对源 `S`，不是相对错误候选 `T0`；
6. 若没有 verified 且 profitable 的结果，返回 `NO_SALVAGE`，系统部署时才安全回退 `S`，但论文不把回退计作成功。

## 6. 研究问题与假设

### RQ1：现象是否存在

被明确证伪的 LLM LLVM-IR 优化中，有多少包含至少一个能组成 verified 且 profitable 结果的严格子集？该比例如何随模型、失败类型和编辑复杂度变化？

**H1：**该现象在至少三个未由本研究进行任务特定训练/RL 的模型家族中均可观察；若仅弱模型或单一模板出现，则不支持一般性主张。

### RQ2：方法是否有效

在相同候选与预算曲线下，SalvageIR 是否比整体丢弃、整函数重试、文本 ddmin、同组件图 hierarchical ddmin、无反例 best-first 和反例 hitting-set MaxSAT/ILP 恢复更多有益候选？

**H2：**SalvageIR 的配对有益恢复率和预算曲线 AUC 高于最强非 oracle 基线，且真实反例优于打乱反例安慰剂。

### RQ3：哪个机制贡献收益

结构对齐、依赖闭包、反例切片和利润恢复各自有什么影响？

**H3：**去掉反例切片后验证调用数上升；去掉依赖闭包后非法 IR 比例上升；去掉利润恢复后 verified 率可能相近但实际收益下降。

### RQ4：是否跨模型稳定

效果是否只来自较弱模型的明显错误？

**H4：**在模型宏平均和留一模型测试中，方法仍优于最强基线；若模型×方法交互完全由一个模型驱动，则收窄主张。

### RQ5：成本是否合理

恢复收益是否值得额外的编译、验证和搜索成本？

**H5：**在固定 wall-clock/验证调用预算下仍保持优势；若无限预算才有效，则不支持实用性主张。

### RQ6：是否具有跨后端外部有效性

在同一源代码独立生成的 x86-64 与 RV64GC/RVV IR 上，冻结编辑决策能否迁移，冻结算法重新搜索能否保持正确性与收益方向？

**H6：**RISC-V 只验证可移植性和收益方向；它不参与方法开发、Prompt、组件规则或 P0/P1 门控。仅在同一源代码独立生成的 RV64GC IR 上评价，并区分冻结编辑决策迁移与冻结算法重新搜索；不允许把 x86 datalayout IR 直接改 triple。

### RQ7：LLM 是否是必要研究对象

相对编辑数、组件数、CFG 变化和 semantic-flag 类别匹配的非 LLM 变异，LLM whole-function 失败是否具有不同的可分解性、协同编辑结构或回收难度？

**H7：**候选来源与方法存在可解释交互。若没有交互，方法仍可能成立，但论文必须改写为通用 RCES/IR repair，不把 LLM 特异性写成贡献。

## 7. 预期论文贡献

只有实验支持时，才能写成贡献：

1. **问题刻画：**系统性量化“被证伪的 LLM IR 优化中可回收有效子翻译”的数据集与失败谱系；在完成穷尽检索前不使用“首次”。
2. **方法候选：**LLVM refinement-aware 的依赖闭合部分回滚与利润恢复算法。
3. **公平评价协议：**固定候选、模型分层、预算相等、条件恢复率与端到端率并报。
4. **跨后端证据：**主实验不依赖 RISC-V，但在 RV64GC/RVV 上验证正确性和收益保持边界。

若 H2 或 H3 不成立，贡献 2 必须删除；不能以数据集和工程量掩盖算法没有增益。

## 8. 论文结构建议

1. Introduction：原子接受/丢弃造成的浪费与研究问题。
2. Background：LLVM refinement、LLM IR 优化、Alive2 边界。
3. Empirical Motivation：多模型失败候选和可分解性先导研究。
4. SalvageIR：对齐、CEC、反例切片、回滚/重放和安全不变量。
5. Experimental Methodology：候选冻结、公平性、指标、统计。
6. Evaluation：RQ1—RQ6。
7. Related Work：Translator、验证、APR、多位置补丁、reducer。
8. Threats and Limitations。
9. Conclusion。

## 9. 论文摘要骨架

> LLM-based IR optimizers generate whole-function transformations that must be rejected when translation validation finds a counterexample. Existing systems typically retry or fall back to the source, discarding all edits in the failed candidate. We investigate whether such candidates contain profitable sub-translations that can be recovered safely. We present SalvageIR, a refinement-constrained edit-salvage framework that aligns source and target LLVM IR, groups edits into structurally replayable components, and uses state-conditioned counterexamples to prioritize a budgeted search over directly validated whole functions. [实验规模与结果待填] Across [模型] and [数据集], SalvageIR recovers [结果待填] under matched budget profiles and outperforms structured ddmin, no-counterexample best-first search, MaxSAT/ILP diagnosis, and shuffled-counterexample controls [结果待填]. Every emitted result is directly validated against the source. A held-out evaluation independently regenerates RV64GC IR from the same source programs to examine cross-backend profitability without making RISC-V part of the method.

方括号内容必须由真实实验填入，当前不得删除“待填”后当作论文摘要使用。
