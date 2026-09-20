# SalvageIR v3 裁决卡

> salvageir_v3 · 2026-09-21。摘要；详细定义与数值以 PILOT_PROTOCOL.md 和 configs/pilot_v3.json 为准。

## 不可偷换的接纳门

严格编辑子集；LLVM verifier通过；直接X→R及X→Oz(R)都VERIFIED；
同流水线下Oz(R)的目标函数机器码至少减少max(2 bytes,ceil(1%基线))；
allocatable非BSS字节不增加。UNKNOWN、仅修正确、回到源、仅Oz前收益均不算主成功。

## 分阶段裁决

| 阶段 | 允许裁决 | 下一步 |
|---|---|---|
| P0a fixtures/测量未过 | BLOCKED_TOOLCHAIN 或 FAIL_INFRA | 修复并重跑受影响测试；不生成新数据 |
| P0a smoke通过但无自然输入 | SMOKE_PASS_NATURAL_MISSING | 给出raw/source/target/log/模型信息缺件清单 |
| P0a smoke及自然审计完成 | P0A_AUDITED | 如实报告正例、负例或未决；P0b需资源授权 |
| P0b ≥2个项目有自然自动正例 | PROCEED_MECHANISM_DEV | 只支持P1开发，不代表统计确认 |
| P0b 少量/集中/未决 | LIMITED_OR_INCONCLUSIVE | 说明瓶颈，固定批次停止，不无限加样 |
| P0b 预算内零正例 | NO_POSITIVE_WITHIN_BUDGET | 停止自动扩大；不是证明普遍不存在 |
| P1 不优于结构/静态基线 | MECHANISM_UNSUPPORTED | 删除反例贡献，保留工具/现象的范围内结论 |
| P2 独立确认支持 | CONFIRMED_IN_SCOPE | 仅在预声明范围内报告效果 |
| RISC-V正例但总体不确定 | RV_CASES_ONLY | 只能说存在案例，不能宣称总体改善 |

## Oracle语义

POSITIVE_COMPLETE、POSITIVE_PARTIAL均证明表示内存在有益状态。
NEGATIVE_COMPLETE需所有合法严格状态有决定性结论。
UNRESOLVED不等于negative；表示失败仍在核心分母。
枚举最优只对冻结表示空间有效，不是所有可能优化的全局最优。

## 不再使用的门

不以三个模型120个失败为P0a前置条件；不以固定60%动态反例覆盖决定是否存在现象；
不要求两个求解返回相同witness；不要求clang -Oz与IR路径95%字节相同；
不把0/小样本或稀疏bootstrap区间当作不存在证据。
文档检查通过不是LLVM集成测试通过；集成测试通过也不是自然样本研究通过。

## 每次报告必备

版本/哈希；实际阶段；分子分母；项目/模型/设置；表示覆盖；
两道证明与精确成本；所有unknown/超预算/缺件；实际调用和时间；
允许主张、禁止主张、唯一下一阶段。没有证据一律未验证。
