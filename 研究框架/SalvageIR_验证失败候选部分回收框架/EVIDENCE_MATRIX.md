# SalvageIR 证据矩阵与创新边界

> 检索截止：2026-09-20。
> 说明：仓库 Paper_ID 用于定位本地笔记/PDF；外部链接优先指向论文、会议或官方项目。以下事实与候选创新分开陈述。

## 1. 核心语料证据

| Paper_ID / 工作 | 角色 | 已有机制与可定位事实 | 对 SalvageIR 的约束 | 证据等级 |
|---|---|---|---|---|
| 49，Towards LLM-Based Optimization Compilers | Translator/T2 | 用 LLM/CoT 做 LLVM peephole 优化并验证 | “LLM 直接改 IR”不是创新 | 中，本地 PDF/笔记 |
| 12，LLM-VeriOpt，CGO 2026 | Translator/T2 | 对 4,386 个基础 Qwen-3B 输出，论文报告 927 个 syntax error、185 个 semantic error；验证失败时评价流程回退源 IR | 证明失败池存在；其 RL 不进入本方法 | 高，[论文](https://samainsworth.github.io/LLM-VeriOpt-CGO2026.pdf) 与 [artifact](https://zenodo.org/records/17672452) |
| 13，LLM-Vectorizer，CGO 2025 | Translator/T1 | LLM 生成向量化代码；Alive2 验证 38.2% 的向量化 | 生成—验证—回退链已存在，不能只加 Alive2 | 高，[会议页](https://2025.cgo.org/details/cgo-2025-papers/14/LLM-Vectorizer-LLM-based-Verified-Loop-Vectorizer) |
| 16/45，IntOpt，2026 | Translator/T2 | 意图形成、细化和实现；作者报告 200 程序上 90.5% verified correctness | 意图分解和完整 IR 生成已被覆盖；SalvageIR 必须针对失败后部分提交 | 中，当前为[预印本](https://arxiv.org/abs/2602.18511) |
| 38，Compiler Generated Feedback，2024 | Translator/T5 | 把编译器反馈给 LLM 再生成一次完整候选 | 完整重试是强基线，不是新机制 | 中，[预印本](https://arxiv.org/abs/2403.14714) |
| N25，CoV，CGO 2026 | Translator/T1 | syntax→profiling→Alive2→运行时双版本验证，扩大向量化接纳范围 | 验证链和 unknown 的运行时兜底已被覆盖；SalvageIR 只处理明确 refuted | 高，[会议页](https://2026.cgo.org/details/cgo-2026-papers/17/Compiler-Runtime-Co-operative-Chain-of-Verification-for-LLM-Based-Code-Optimization) |
| C34，T-LLM Compiler，2026 | Translator/T1 | 源代码级优化和验证闭环，失败时重试完整结果 | 不与源级全函数修复混淆 | 中，本地 PDF/笔记 |
| C179，VERT，2024 | Translator/T3 | 跨语言翻译后以测试/反例反馈修复 | 反例反馈 Translator 已存在；差异必须来自 LLVM refinement-aware partial commit | 中，本地 PDF/笔记 |
| 14，Alive2，PLDI 2021 | Supporting/B2 | LLVM IR bounded translation validation；可给 counterexample；不支持跨过程变换 | 最终正确性底座和适用范围边界 | 高，[官方仓库](https://github.com/AliveToolkit/alive2) |
| 39，Enhancing Translation Validation，2024 | Supporting/B6 | Alive2 inconclusive 后让 LLM 判断并 fuzz | 处理 unknown，不是从 refuted 优化中回收子翻译 | 中，[预印本](https://arxiv.org/abs/2401.16797) |
| C08，LPO，ASPLOS 2026 | Generator/G2 | 从 IR 切片生成候选，经 opt/interestingness/Alive2 验证并反馈重试，最终归纳成规则 | “反例+重试+验证”已存在；LPO 输出可复用规则，SalvageIR 输出单个程序实例 | 高，本地全文笔记与论文 |
| C84，Optimuzz，PLDI 2025 | Generator/G4 | 面向连续翻译验证生成优化相关测试 | 可提供验证压力和测试方法，不是恢复算法 | 高，本地全文笔记 |

## 2. 2024—2026 近邻碰撞

| 工作 | 最近邻机制 | 与 SalvageIR 的实质差别 | 风险 |
|---|---|---|---|
| ITER，ICSE 2024 | 不忽略不可编译的 partial patch，迭代改进并构造多位置补丁 | Java bug repair，以测试可行性为目标；没有 LLVM 优化精化和盈利子翻译 | 高：不能声称“利用部分失败输出”是新想法。[会议页](https://conf.researchr.org/details/icse-2024/icse-2024-research-track/55/ITER-Iterative-Neural-Repair-for-Multi-Location-Patches) |
| Indivisible Multi-Hunk Bugs，FSE 2024 | 研究 partial patches 的相互关系和不可分多位置补丁 | 数据对象是软件 bug；结果提醒局部编辑不可独立假设 | 高：支持必须做依赖闭包和最终全局验证。[论文](https://doi.org/10.1145/3660828) |
| MENTOR，AAAI 2025 | MaxSAT 故障定位、bug-free sketch、LLM 填洞、CEGIS | 学生程序修复与测试/reference solution；不是从错误优化翻译中恢复性能 | 高：不能把“定位后让 LLM 填补”作为贡献。[论文](https://ojs.aaai.org/index.php/AAAI/article/view/32046) |
| PReMM，OOPSLA 2025 | 依赖聚类、分治、多方法 partial patches、最终组合验证 | 多方法 bug repair；没有 LLVM UB/refinement 与目标成本 | 高：组件聚类和组合验证本身不新。[论文页](https://ira.lib.polyu.edu.hk/handle/10397/117462) |
| CUDABeaver，2026 | 评价修复是否因退化而丢失 CUDA 优化结构 | 是 benchmark/评价问题；CUDA 测试正确性，不做 LLVM 子变换回收 | 中：证明“修对但变慢”已是已知问题。[预印本](https://arxiv.org/abs/2605.08455) |
| Trivet，2026 | LLM+Lean 扩大 LLVM 翻译验证覆盖，可证明或反驳 147/148 个案例 | 目标是验证，不修改或回收候选 | 中：未来可替换/补充 Alive2，但不碰撞恢复机制。[预印本](https://arxiv.org/abs/2609.19583) |
| llvm-reduce | delta passes 删除 IR 内容并维持 interestingness | 缩减测试用例，不从源/目标编辑中构造 verified profitable 程序 | 中：必须作为 reducer 基线。[官方文档](https://llvm.org/docs/CommandGuide/llvm-reduce.html) |
| opt-bisect | 关闭优化流水线中某索引之后的 pass/变换以定位错误 | 只作用于 LLVM pass pipeline，不分解任意 LLM 目标 IR | 低到中：[官方文档](https://llvm.org/docs/OptBisect.html) |

## 3. 已不能声称的创新

- 首次使用 Alive2 验证 LLM 优化；
- 首次将反例反馈给 LLM；
- 首次利用部分错误输出；
- 首次进行多位置/联合修复；
- 首次同时关注正确性和性能；
- 首次用依赖聚类划分补丁；
- 首次在 RISC-V 上验证 LLM 编译优化；
- 首次用 delta debugging 定位错误修改。

## 4. 仍可竞争的机制增量

【候选创新】以下组合是当前尚未找到直接同构工作的部分：

1. 把失败的 LLM LLVM-IR 优化表示为可重放、依赖闭合的编辑组件，而非源代码语句或文本 hunk；
2. 明确处理 LLVM 的 poison/undef/freeze、semantic flags、SSA、CFG 和 MemorySSA；
3. 用 refinement counterexample 形成组件命中排序，执行部分回滚；
4. 在找到 verified 状态后重新加入组件做利润恢复；
5. 所有输出均通过源到最终候选的直接整函数验证；
6. 在固定候选与预算下，跨模型评价条件恢复率和端到端收益。

核心增量必须通过 B5 与消融证明。若去掉反例切片后效果不变，创新缩减为 LLVM-aware structured rollback；若连结构化回滚也不优于 ddmin，则算法主张失败。

## 5. 数据与评价支撑

- LLM-VeriOpt artifact 可作为候选和日志来源，但完整档案需要下载核验。[GitHub](https://github.com/carrotProgrammer/llmveriopt-AE)
- IR-OptSet 提供 170K LLVM IR 样本、来自 1,704 个仓库，可作为受控生成的源程序池，但不能直接提供 LLM 失败候选。[NeurIPS 2025](https://papers.nips.cc/paper_files/paper/2025/hash/a4ab7aefc004bed00e577164c57eafd7-Abstract-Datasets_and_Benchmarks_Track.html)
- LLVM Opt Benchmark 是真实 IR 数据来源，并明确区分 IR 代理和后端真实性能。[官方仓库](https://github.com/dtcxzyw/llvm-opt-benchmark)
- Meta LLM Compiler 提供 LLVM IR/汇编领域模型，但模型访问许可和版本锁定需纳入复现风险。[模型卡](https://huggingface.co/facebook/llm-compiler-7b/blob/main/README.md)

## 6. 可复现检索记录

检索日期：2026-09-20。来源：本地 Taxonomy v2、论文 PDF/笔记、ACM/IEEE 会议页、arXiv、OpenReview、LLVM 官方文档、论文 artifact。

主要检索式：

```text
LLM LLVM IR optimization repair Alive2 counterexample
partial salvage invalid compiler optimization candidate
LLVM IR partial rollback translation validation
LLM generated LLVM IR failed candidate repair
partial patch multi-location repair counterexample 2024 2025 2026
verified compiler optimization component rollback
```

未完成的检索：

- Semantic Scholar/OpenAlex 的系统引文追踪；
- LLM-VeriOpt 1.1 GB artifact 的完整文件清点；
- 2026 年 9 月以后可能出现的新工作；
- Trivet 的正式同行评审状态。

因此新颖性等级为“中等偏高但未封闭”，不得写“首次”。
