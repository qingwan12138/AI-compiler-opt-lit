# ProbeTrans 实现前终审变更记录

**日期**：2026-09-22
**范围**：只修改研究设计、协议、tracker、状态和 schema；未实现源码，未运行实验。

## 修改了什么以及为什么

1. 将项目正文统一为“LLVM IR 翻译框架”，并保留历史目录名以避免路径破坏。
2. 新增 `P0_IMPLEMENTATION_SPEC.md` 作为唯一机器协议来源，避免搜索预算、schema、lineage 和 gate 在多份文档中分叉。
3. 将 P0 probe space 收缩到 profitability、alias/dependence、trip-count，并把 family 展开成有限、可 hash、可计费的 instance。
4. 将 16 次预算闭合为 11 次 discovery（含 baseline）+3 次 deletion+2 次无缓存 replay；未完成全部删除检查不得声称 inclusion-minimal。
5. 移除 D0 的“必须可映射到 registry”纳入条件；OUT_OF_REGISTRY 留在总分母，ground truth 与 probe 结果解耦。
6. 将 baseline 拆成 generation-matched、natural-cost 和 query-matched feedback 两条成本轨，并强制报告完整成本。
7. 收紧 replacement function 边界；module edit 和跨函数事实明确退出当前 Translator。
8. 新增一对多 loop lineage 与歧义失败状态，防止把其他循环或 remainder 当成目标成功。
9. 固定 Alive2 的 source→target refinement 方向，区分 proved、differential-only、disproved、unresolved。
10. 冻结 VAG：主协议只改变目标 LoopVectorize，SLP 全局关闭仅作敏感性分析。
11. 将 target-independent 改为 target-aware but ISA-intrinsic-free，并收窄 RV64GCV 主张。
12. 重排 M0–M6，并加入 M2b vertical slice；M2b 前禁止接 LLM。

## 被收窄的主张

- certificate 是预算内、冻结 registry 内的可重放决策翻转证据，不是唯一根因或完整因果证明。
- profitability probe 只区分成本/启发式决策，不代表缺失语义事实。
- Alive2 未解决但差分通过的候选不算 formal proof。
- VAG 只提供机制归因证据，不构成完整因果证明。
- 跨 ISA 只测 portability/robustness，不承诺同一向量化决策或普遍加速。
- P0 是工程 kill-gate，不提供论文级统计结论。

## 尚未解决的风险

- 冻结 registry 是否覆盖足够多的真实 blocker；
- runtime non-overlap/trip guard 是否能在函数级边界内安全构造；
- Alive2 对循环、内存和版本化 CFG 的支持率；
- template 是否追平 LLM；
- AutoDL CPU 是否能支持可靠的 VAG 性能测量；
- canonical prefix 与真实 `default<O3>` 中目标 pass 输入的差距。

## 必须由实验回答的问题

- M2b 是否能在不逐案例改 probe 的情况下达到 100% replay 且不超过 16 次查询；
- D0 的 registry coverage、overall accuracy 与 remark baseline 差值；
- 安全可实现证书比例及各 realization class 分布；
- strict semantic 口径下 LLM 是否超过 Remark、Iterative Remark 和 Template；
- VAG 是否会改变表面性能成功的接受集合。

## 本轮结果声明

本轮没有运行正式实验，没有生成性能、正确率、覆盖率或成功率数据；文档中的数字均为预算、样本计划或预注册 gate 阈值。
