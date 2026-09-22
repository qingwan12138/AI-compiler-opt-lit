# ProbeTrans IR-v2 证据与代码基座矩阵

**检索截止**：2026-09-22

| 工作 | 场所 | 层级/产物 | 开源状态与可复用性 | 对 ProbeTrans 的约束 |
|---|---|---|---|---|
| IR-OptSet | NeurIPS 2025 D&B | LLVM IR 数据、分析/生成任务、验证与扩展工具链 | [MIT 代码](https://github.com/yilingqinghan/IR-OptSet)；可复用 LLVM wrapper、preprocess、verify、Alive2、mca、LLM modules | **唯一正式代码基座**；ProbeTrans 必须清楚列出新增模块 |
| Meta LLM Compiler | 2024 | LLVM IR/assembly foundation model，code-size/pass tasks | 模型公开但有专用许可/访问要求 | 可作 backbone/基线，不是 ProbeTrans 工程基座 |
| Compiler Generated Feedback | 2024 arXiv | LLVM IR、pass、instruction count feedback | 论文方法明确；未发现可直接继承的完整系统仓库 | “编译反馈给 LLM”已被覆盖，不能作创新 |
| LLM-Vectorizer | CGO 2025 | scalar program→显式 SIMD，FSM multi-agent，Alive2 | 官方论文页未发现可直接复用完整代码 | LLM vectorization+verification 已覆盖；ProbeTrans 使用 target-aware but ISA-intrinsic-free IR |
| VecTrans | 2025 arXiv | source→source 重构触发 auto-vectorization | 官方论文页未发现公开完整实现 | 源码层路线；ProbeTrans 区别必须是 IR 层证书条件化翻译 |
| IR-OptSet paper | NeurIPS 2025 D&B | 170K IR、4.3M optimization annotations、验证/静态性能工具 | 数据 CC-BY-4.0；代码 MIT | IR-OptSet 样本多数无完整 runtime harness，不可作主要真实性能集 |
| IntOpt | 2026 arXiv | intent formulation/refinement/realization 的一般 IR 优化 | 论文页未给公开代码 | explicit intent 已覆盖；certificate 必须是可重放 compiler intervention evidence，而非换名 intent |
| AI Coding Agents Need Better Compiler Remarks | 2026 arXiv | 精确 remarks 改善 agent | 论文证据；无关代码继承 | precise remark 必须是强基线 |
| Objective Withdrawal of Useful Information | ACM TACO 2019 | 隐藏/暴露编译期信息的 TSVC+微核 | [开源 benchmark](https://github.com/sergisiso/autovec-benchmark) | 信息撤回机制已存在；只作现代重认证诊断校准 |
| CoV | CGO 2026 | LLM vectorization 的静态/符号/运行时验证链 | 论文/preprint | 多层 verification 不是创新；VAG 必须测机制归因 |

## 代码继承结论

论文中必须写：

> We implement ProbeTrans by extending the open-source IR-OptSet toolchain (NeurIPS 2025). We reuse its LLVM IR extraction, preprocessing, verification, Alive2, static-analysis, model-adapter, and logging infrastructure, and add vectorization-specific counterfactual probes, certificate search, function-level IR translation, semantic-strengthening audits, and attribution gates.

不得写“基于 IntOpt/VecTrans/LLM-Vectorizer 代码修改”，除非未来出现并实际采用官方仓库。

## Benchmark 证据边界

- TSVC 是与 LLM-Vectorizer、VecTrans、CoV 对齐的主锚点；
- autovec-benchmark 只提供已知信息条件，不继承 2019 性能数字；
- IR-OptSet 是静态规模与工程压力集，不替代 executable benchmark；
- PolyBench/C 和真实应用承担外部泛化；
- RVV 只检验冻结 IR 的跨后端表现。

## 剩余新颖性风险

中等偏高。可防守的主张仅是：**用可重放、最小化的 compiler interventions 形成翻译条件，并将诊断假设与最终 IR 的语义义务显式分离。** 如果证书只等价于更详细 remark，或者 template 已能机械实现，论文主张失效。
