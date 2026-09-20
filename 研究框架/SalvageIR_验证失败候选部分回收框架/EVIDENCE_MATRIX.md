# SalvageIR v3 证据与创新边界

> salvageir_v3 · 核验截止2026-09-21。定向近邻检索，不是完整系统综述。
> 区分论文事实、合理推导和待检验机制；旧笔记不是优先于论文正文的证据。

## 1. 已回查原文的核心证据

| 本地ID / 工作 | 已核验事实及位置 | 对本框架的含义/边界 |
|---|---|---|
| 12 / LLM-VeriOpt，CGO2026 | §IV/V从未优化IR与instcombine参考构造数据；验证失败回退；Model-Zero亦属GRPO训练系列。[论文](https://samainsworth.github.io/LLM-VeriOpt-CGO2026.pdf) | 证明失败池存在；不证明存在盈利子集。论文方法的RL不进入我们的核心 |
| C08 / LPO，ASPLOS2026 | §3从已优化IR切片，经opt/interestingness/Alive2检查，错误/反例反馈重试；优化实例需后续分析与人工泛化。[论文](https://cs.uwaterloo.ca/~cnsun/public/publication/asplos26/asplos26.pdf) | 与其差异不是“用了反例”，而是不再次生成、搜索固定失败候选的子集。不能把LPO简化为自动直接输出通用规则 |
| 20 / Minotaur，OOPSLA2024 | §2.4–2.5提取cut，使用数据/控制信息及MemorySSA，处理多个基本块的局部问题。[论文](https://users.cs.utah.edu/~regehr/minotaur-oopsla24.pdf) | LLVM-aware切片与验证局部优化已有先例。不能复用后单独宣称创新 |
| 45 / IntOpt，2026预印本 | §3意图形成/细化/实现；Table1 Alive2-only为71.0%，Alive2+差分测试为90.5%。[论文](https://arxiv.org/pdf/2602.18511) | 更正v2的“90.5% verified”歧义，不能与严格形式验证接纳率混用；不因模板页眉称ICML录用 |
| 外部 / PReMM，OOPSLA2025 | §3按测试/调用依赖聚类，多方法补丁生成与组合验证。[论文](https://ira.lib.polyu.edu.hk/bitstream/10397/117462/1/Xie_PReMM_LLM-based_Program) | 依赖分组/组合验证不是新意；其对象是程序修复，不是失败优化收益回收 |
| 外部 / P³，OOPSLA2025 | 自动构造C程序版本的product program，用测试/符号执行推理补丁行为。[论文](https://daniel.schemmel.net/publication/2025-product-programs.pdf) | 双版本关系分析有先例，但不直接解盈利编辑子集搜索 |
| 外部 / Agentic Code Optimization via Compiler–LLM Cooperation，2026预印本 | 多抽象层LLM与编译器协作、测试及预算分配。[论文](https://arxiv.org/pdf/2604.04238) | LLM与编译器互补不是新主张；不可把检索聚合页误给的2604.04345当本文ID |
| 数据 / IR-OptSet，NeurIPS2025 | 数据卡区分original/preprocessed与O3目标；实际来源/revision仍需运行前锁定。[数据卡](https://huggingface.co/datasets/YangziResearch/IR-OptSet/blob/5b69324e8c51d153bb2cd37e75f8015edc7816f0/README.md) | 可提供未优化源候选，但不是现成LLM失败池，不证明本协议筛选后有足够样本 |

事实层证据较高，但对“自然可回收率”和“反例搜索优势”的推导证据仍低：这些需要本实验。

## 2. 工具语义证据

- LLVM lifetime标记影响对象生命周期，不是普通debug噪声；相关删除必须经过语义论证，v3不主动删。[LangRef](https://llvm.org/docs/LangRef.html)
- mem2reg是将可提升栈槽转换为SSA的pass，PRE_SSA必须披露此变换，不能写纯格式清理。[Passes](https://llvm.org/docs/Passes.html)
- Alive2适用范围和跨过程限制需要锁定版本并遵守；未知不等于错误或正确。[官方项目](https://github.com/AliveToolkit/alive2)
- LLVM SandboxIR已提供变更追踪与事务回滚；重放实现机制不是本论文独占贡献。[官方文档](https://llvm.org/docs/SandboxIR.html)
- llvm-reduce主要保留interestingness缩减测试用例，不自动解决带正确性/盈利约束的编辑回收。[文档](https://llvm.org/docs/CommandGuide/llvm-reduce.html)
- LLVM test-suite提供参考输出及构建/运行框架，适合后期系统与跨平台验证。[文档](https://llvm.org/docs/TestSuiteGuide.html)

LLVM在线文档会变化，执行依据必须是实际锁定release/commit语义，不用最新网页覆盖旧工具行为。

## 3. 剩余候选增量

【候选创新】在自然LLM失败优化实例上：
1. 建立有来源、禁止新计算的严格编辑空间；
2. 区分必须共变的结构约束与只影响优先级的语义风险；
3. 用当前状态的可信反例对回滚/加回排序；
4. 最终双重整函数验证并以统一Oz后机器码收益接纳；
5. 在同候选、同空间、同预算强基线下证明增量。

每一部件都有近邻，组合不自动等于创新。
若B5/B6/B7与真实/静态/匹配随机信号无法区分，删除反例算法主张。
如果只有规范化或更多预算带来收益，不归因于SalvageIR核心机制。
换成O0输入只是任务设定变化，不是创新点。

## 4. 检索记录与未覆盖项

日期：2026-09-21及前一轮2026-09-20；来源：本地taxonomy/PDF笔记、作者PDF、会议页、arXiv和LLVM官方文档。
本轮本地重点ID为12/20/45/C08；新近邻为P³与Compiler–LLM Cooperation，PReMM用于再次核查。
关键词族：
- LLM LLVM IR optimization repair Alive2 counterexample
- partial salvage invalid compiler optimization candidate
- LLVM IR partial rollback translation validation
- partial patch multi-location repair dependency clustering
- P³ Reasoning about Patches via Product Programs
- Agentic Code Optimization via Compiler-LLM Cooperation

本轮未执行全库系统引文追踪、完整artifact下载或AutoDL数据重跑。
旧表中未经本轮充分回查的条目保留在archive/v2，不作为当前唯一论据；不表示其不存在。
尚需补查：语义补丁拆分/最大正确补丁子集、非单调delta debugging、refinement-aware repair与编辑表示召回。
新颖性结论为“有待实验与进一步检索支持的候选增量”，不写“首次”或录用保证。
