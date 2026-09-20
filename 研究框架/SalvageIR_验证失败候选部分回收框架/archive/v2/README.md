# 历史归档：v2，不得用于新实验执行

原文件：README.md。2026-09-21 被 v3 替代；保留原文用于追溯，不代表当前约束。

# SalvageIR：验证失败候选部分回收框架

> 状态：完整研究设计，尚未实现、尚无实验结果。
> 日期：2026-09-20。
> 唯一主类：`TRANSLATOR / T2_IR_ASM_Optimization_Superoptimization`。
> 约束：不使用强化学习；RISC-V 仅作保留平台上的外部有效性验证。

## 一句话主线

SalvageIR 不把一个被 Alive2 明确证伪的 LLM 优化结果整体丢弃，而是把源 IR 到候选 IR 的变化表示为结构可重放组件，在反例软引导下搜索安全子翻译，并且只输出经过整函数 LLVM refinement 验证且确有机器码收益的 IR。真实反例是否比安慰剂反例和强结构搜索更有效，是待验证假设，不是既成事实。

## 与仓库中旧 Translator 后优化框架的关系

本目录不覆盖 [`../Translator_后优化框架/`](../Translator_后优化框架/)：

- 旧框架后期把“已验证正确候选的进一步优化”作为主任务；
- SalvageIR 的主任务严格限定为“可解析但被形式验证明确反驳的候选”；
- 两者不能混用分母、成功率或论文贡献。正确候选的二次优化不进入 SalvageIR 主实验。

## 阅读顺序

1. [`RESEARCH_CONTRACT.md`](RESEARCH_CONTRACT.md)：不可漂移的研究对象、主张和完成条件。
2. [`FINAL_PROPOSAL.md`](FINAL_PROPOSAL.md)：完整问题、架构、算法、研究问题和论文定位。
3. [`ALGORITHM_SPEC.md`](ALGORITHM_SPEC.md)：RCES 问题、编辑图、反例适配器、安全前沿和正确性不变量。
4. [`DATA_AND_FAIRNESS_PROTOCOL.md`](DATA_AND_FAIRNESS_PROTOCOL.md)：候选来源、多模型公平性和数据冻结协议。
5. [`PILOT_PROTOCOL.md`](PILOT_PROTOCOL.md)：测试集、顺序扩样、oracle、资源预算和红黄绿门控的可预注册规格。
6. [`GATE_CARD.md`](GATE_CARD.md)：执行时唯一使用的 F/I/S 单页裁决卡。
7. [`CODEX_P0A_EXECUTION_HANDOFF.md`](CODEX_P0A_EXECUTION_HANDOFF.md)：可直接交给 Codex/ARIS 实施 P0a 的执行工单。
8. [`EXPERIMENT_PLAN.md`](EXPERIMENT_PLAN.md)：正式基线、消融、统计和 RISC-V 验证。
9. [`EVIDENCE_MATRIX.md`](EVIDENCE_MATRIX.md)：语料库证据、2024—2026 近邻和创新碰撞。
10. [`THREATS_AND_REVIEW.md`](THREATS_AND_REVIEW.md)：反方审查、风险、降级与可证伪条件。
11. [`REVIEW_ROUND_1.md`](REVIEW_ROUND_1.md)：以 CGO/PLDI 标准给出的独立强拒稿意见。
12. [`REVISION_ROUND_1.md`](REVISION_ROUND_1.md)：每条拒稿意见对应的修订与失败降级。

## 当前结论

- **研究问题完整度：**已完成。
- **机制规格完整度：**已完成到可进入实现计划的程度。
- **候选来源：**公开来源可行，但 LLM-VeriOpt 的 1.1 GB 完整档案仍需下载后清点；公开仓库说明其保存 IR 输出、Alive2 日志和指标。
- **创新可信度：**暂定中等。LLVM SandboxIR 已覆盖事务回滚，程序修复已覆盖部分补丁、依赖聚类和反例/MaxSAT 定位；剩余贡献必须由“真实反例优于打乱反例与强结构搜索”证明，不能靠系统组合宣称。
- **最大未知量：**真实失败候选是否经常包含可单独组合成正确且有收益结果的子变换。P0 先导实验是继续或停止该主线的强制门。

## 明确不做

- 不训练 LLM，不使用 PPO、GRPO、MCTS 或任何强化学习。
- 不把 Alive2、RAG、Agent、换模型或换到 RISC-V 单独包装成创新。
- 不把 timeout、unsupported、fuzzing 未发现错误当作正确。
- 不以“保留编辑数最多”代替真实性能收益。
- 不把返回源 IR 计作恢复成功。
- 不以 RISC-V 为主要数据来源、方法条件或主实验背景。
- 不把 x86 datalayout IR 直接改 triple 当作 RISC-V 验证；RISC-V IR 必须从同一源代码独立生成。
