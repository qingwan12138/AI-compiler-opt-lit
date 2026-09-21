# ProbeTrans：反事实探针驱动的 LLVM IR 翻译框架

> ARIS IR 修订版，2026-09-22。当前状态：**CONDITIONAL_GO；先完成 P0 预实验，不宣称方法已有效。**

## 一句话定位

ProbeTrans 是一个严格的 **LLVM IR → LLVM IR Translator**：它从标准前端产生的、进入 `LoopVectorize` 前的 LLVM IR 出发，用只供诊断的反事实 IR 探针找到能翻转向量化决策的预算内包含最小证书，再让 LLM 将证书翻译为一个新的 LLVM IR 函数，最后验证语义、向量化效果、真实性能和机制归因。

```text
C/C++ benchmark
  → clang 前端
  → 固定规范化前缀
  → pre-LoopVectorize LLVM IR
  → ProbeTrans（IR→IR）
  → LLVM LoopVectorize 与后端
  → x86 主评测 / RV64GCV 外部验证
```

## 代码与算法基座

- **开源代码基座**：[IR-OptSet（NeurIPS 2025 Datasets and Benchmarks）](https://github.com/yilingqinghan/IR-OptSet)，MIT License。
- **直接复用**：LLVM wrapper、IR 预处理、`opt -verify`、Alive2、`llvm-mca`、LLM 调用、日志和数据管线。
- **ProbeTrans 新增**：missed-loop selector、IR probe registry、minimal certificate search、function-level IR translator、semantic-strengthening audit、LoopVectorize attribution gate。
- **不声称继承 IntOpt/VecTrans/LLM-Vectorizer 代码**：截至本轮核查，其官方论文页面没有提供与本任务可直接复用的完整开源实现；它们作为方法近邻和基线。

## Benchmark 冻结

| 角色 | Benchmark | 用途 |
|---|---|---|
| 诊断校准 | 现代 LLVM 重认证的 `autovec-benchmark` | blocker 标签、探针准确性、证书查询成本 |
| 主测试 | LLVM test-suite 的完整 TSVC，经预注册规则筛选 missed loops | 端到端主结论和近邻可比性 |
| 外部泛化 | PolyBench/C + 3–5 个开源应用热点 | 排除 TSVC 专用模板 |
| 静态压力 | IR-OptSet HPC/Multimedia/Embedded 子集 | IR 合法率、证书覆盖、工具稳定性；不冒充真实运行时间 |
| 跨 ISA | 主实验已冻结且通过的相同 IR | RV64GCV 代码生成与真机性能外部验证 |

## 主创新与支撑机制

1. **主创新：Intervention-Certified IR Translation（ICIT）**
   在固定 LLVM 配置与有限探针空间中，搜索能稳定翻转同一循环 `LoopVectorize` 决策的包含最小证书，并以此约束 IR 翻译。
2. **支撑 A：Semantic-Obligation Contract（SOC）**
   禁止把诊断时临时加入的 `noalias`、强 alignment、`nsw/nuw`、`inbounds`、fast-math、`llvm.assume` 等语义强化直接洗入最终 IR；除非能由原 IR 证明或通过运行时 guard + fallback 实现。
3. **支撑 B：Vectorization Attribution Gate（VAG）**
   对原始/改写 IR 分别开关 LoopVectorize，排除仅由标量简化、其他 pass 或噪声带来的表面加速。

支撑机制服务于同一个主张，不作为三篇并列贡献。

## 明确不做

- 不做 source-to-source；不输出 C/C++ patch。
- 不做 RL；P0 不训练或微调模型。
- 不生成 x86/RVV intrinsic；输出保持 target-independent LLVM IR。
- 不从 `default<Oz>` 后开始；研究对象是进入目标向量化阶段前的 IR。
- 不扩展到多个 LLVM Pass；首版只研究 LoopVectorize。
- 不把 `llvm-mca` 改善写成真实性能提升。

## 阅读顺序

1. [研究契约](RESEARCH_CONTRACT.md)
2. [最终方法提案](FINAL_PROPOSAL.md)
3. [完整预实验协议](PREEXPERIMENT_PROTOCOL.md)
4. [正式实验计划](EXPERIMENT_PLAN.md)
5. [实验追踪表](EXPERIMENT_TRACKER.md)
6. [严苛审稿记录](REVIEW_SUMMARY.md)
7. [证据矩阵](EVIDENCE_MATRIX.md)
8. [ARIS 流水线总结](PIPELINE_SUMMARY.md)

## 当前最关键的失败条件

如果证书不能稳定预测向量化决策、绝大多数证书只能依赖不可证明的语义强化、确定性模板与 LLM 表现相当，或 VAG 表明多数加速并非来自 LoopVectorize，则停止把 ProbeTrans 作为 LLM Translator 论文主线。
