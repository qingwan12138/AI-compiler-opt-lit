# SalvageIR：验证失败候选部分回收框架

> 状态：完整研究设计，尚未实现、尚无实验结果。
> 日期：2026-09-20。
> 唯一主类：`TRANSLATOR / T2_IR_ASM_Optimization_Superoptimization`。
> 约束：不使用强化学习；RISC-V 仅作保留平台上的外部有效性验证。

## 一句话主线

SalvageIR 不把一个被 Alive2 明确证伪的 LLM 优化结果整体丢弃，而是把源 IR 到候选 IR 的变化表示为依赖闭合的编辑组件，在反例引导下回滚错误组件、重放仍有价值的组件，并且只输出经过整函数 LLVM refinement 验证且确有性能收益的 IR。

## 与仓库中旧 Translator 后优化框架的关系

本目录不覆盖 [`../Translator_后优化框架/`](../Translator_后优化框架/)：

- 旧框架后期把“已验证正确候选的进一步优化”作为主任务；
- SalvageIR 的主任务严格限定为“可解析但被形式验证明确反驳的候选”；
- 两者不能混用分母、成功率或论文贡献。正确候选的二次优化不进入 SalvageIR 主实验。

## 阅读顺序

1. [`RESEARCH_CONTRACT.md`](RESEARCH_CONTRACT.md)：不可漂移的研究对象、主张和完成条件。
2. [`FINAL_PROPOSAL.md`](FINAL_PROPOSAL.md)：完整问题、架构、算法、研究问题和论文定位。
3. [`ALGORITHM_SPEC.md`](ALGORITHM_SPEC.md)：编辑图、反例引导回滚、正确性不变量和伪代码。
4. [`DATA_AND_FAIRNESS_PROTOCOL.md`](DATA_AND_FAIRNESS_PROTOCOL.md)：候选来源、多模型公平性和数据冻结协议。
5. [`EXPERIMENT_PLAN.md`](EXPERIMENT_PLAN.md)：先导实验、正式基线、消融、统计和 RISC-V 验证。
6. [`EVIDENCE_MATRIX.md`](EVIDENCE_MATRIX.md)：语料库证据、2024—2026 近邻和创新碰撞。
7. [`THREATS_AND_REVIEW.md`](THREATS_AND_REVIEW.md)：反方审查、风险、降级与可证伪条件。

## 当前结论

- **研究问题完整度：**已完成。
- **机制规格完整度：**已完成到可进入实现计划的程度。
- **候选来源：**公开来源可行，但 LLM-VeriOpt 的 1.1 GB 完整档案仍需下载后清点；公开仓库说明其保存 IR 输出、Alive2 日志和指标。
- **创新可信度：**中等偏高。尚未发现“对被证伪的 LLM LLVM-IR 优化做 refinement-aware 部分回滚并保留盈利子翻译”的直接同构工作，但通用程序修复已有部分补丁、多位置修复和反例引导定位，故不能声称“首次保留部分修改”。
- **最大未知量：**真实失败候选是否经常包含可单独组合成正确且有收益结果的子变换。P0 先导实验是继续或停止该主线的强制门。

## 明确不做

- 不训练 LLM，不使用 PPO、GRPO、MCTS 或任何强化学习。
- 不把 Alive2、RAG、Agent、换模型或换到 RISC-V 单独包装成创新。
- 不把 timeout、unsupported、fuzzing 未发现错误当作正确。
- 不以“保留编辑数最多”代替真实性能收益。
- 不把返回源 IR 计作恢复成功。
- 不以 RISC-V 为主要数据来源、方法条件或主实验背景。
