# C49 Commuting Conversions and Join Points for Call-By-Push-Value 文献阅读总结

论文题目：**Commuting Conversions and Join Points for Call-By-Push-Value**

作者：Jonathan Chan、Madi Gudin、Annabel Levy、Stephanie Weirich。

发表信息：Proceedings of the ACM on Programming Languages, Volume 10, Issue OOPSLA1, Article 102（2026）；页码 289–313；DOI 10.1145/3798210。OOPSLA 2026 正式论文。
元数据核验来源：[ACM PACMPL DOI](https://doi.org/10.1145/3798210)；[Zenodo Lean artifact](https://doi.org/10.5281/zenodo.19288094)
代码/数据/工件：Lean 形式化工件：[Zenodo](https://doi.org/10.5281/zenodo.19288094)

来源：[OOPSLA 2026 论文页](https://2026.splashcon.org/details/oopsla-2026/11/Commuting-Conversions-and-Join-Points-for-Call-By-Push-Value)、[ACM DOI 记录](https://doi.org/10.1145/3798210)、[作者公开 PDF](https://ionathan.ch/assets/pdfs/ccnf.pdf)。

阅读材料：作者最终稿 PDF，共 25 页；论文提供的 Lean 4 证明发展作为补充材料。

关键词：call-by-push-value、commuting conversions、continuation、join point、编译变换、类型保持、语义保持、Lean 4。

## 1. 研究背景

编译器常用 commuting conversion 把计算从嵌套的求值上下文中移出，形成更适合后续内联和低层代码生成的规范形式。类似变换散见于 ANF、continuation-based lowering 和函数式编译器优化，但 call-by-push-value（CBPV）同时显式区分值与计算，且涵盖 call-by-value 与 call-by-name 的共同结构，因而需要系统地刻画其可交换规则与正确性。

## 2. 论文要解决的问题

论文回答两个相连的问题：如何完整描述 CBPV 中适用的 commuting conversions，并将其实现为一个单遍、类型保持且语义保持的规范化变换；以及如何避免把延续复制到多个分支时产生代码膨胀，同时不为每个延续创建闭包或 thunk。作者还希望给出可机械检查的证明，而不只依赖直觉式等式推导。

## 3. 核心方法概述

作者以 CBPV 求值上下文刻画所有相关的消去形式，并定义 commuting conversion normal form（CCNF）：求值上下文被持续推进到尾位置。变换以显式 continuation 参数递归遍历输入项，输出 CCNF。遇到需要跨越分支复用的 continuation 时，生成局部 join point 定义，并在尾位置通过 jump 传递结果；join-point 类型环境与普通变量环境分开管理。这样不需复制 continuation 主体，也不需将其封装为新的 thunk/闭包。

变换的规则涵盖 let、函数应用、force/thunk、积类型投影、case 等交互。其设计目标是单遍构造最终结果，而不是反复应用一系列可能产生中间非规范项的局部等式。正文给出输入输出语法、上下文、join/jump 的类型规则和操作语义，并对变换各构造逐一建立保持性质。

## 4. 实验框架与训练流程

本文是形式化语言与编译变换研究，没有模型训练、强化学习、数据集、硬件基准或性能评测。作者在 Lean 4 中机械化了语法、类型系统、动态语义、逻辑关系及主要定理（论文第 1、4–5、8 节；补充 Lean artifact）。因此不能把该论文描述为实测加速或 benchmark 证明。

## 5. 奖励函数、损失函数或关键公式

本文没有奖励函数或损失函数。核心正确性主张是：对封闭、良类型且求值为 ground value 的 computation，翻译后求值仍得到相同 ground value。定理 5.24 由语义等价 corollary 5.22 推出；上游证明先建立翻译的类型保持（Theorem 4.7）与语义等价（Theorem 5.21）。join point 只允许在尾位置被 jump 到，从语法/类型规则上限制其控制流用途。

## 6. 实验设置

没有数值实验设置。验证设置为 Lean 4.26 的机器检查证明；研究对象是无副作用 CBPV 语言，语义关系基于论文定义的值/计算类型和操作语义。论文讨论了 effect-safe 的直觉，但没有将 effect 纳入形式系统和逻辑关系，因此这一点不是已证明定理。

## 7. 实验结果与结论

作者形式化定义 CCNF，并给出一次遍历即可把良类型 CBPV computation 变换为 CCNF 的算法。机械化结果证明该变换保持类型与求值行为：翻译前若封闭 computation 求值为某 ground value，翻译后也求值为相同值（第 4–5 节，特别是 Theorems 4.7、5.21、5.24）。论文展示 case-of-case 如何借助 join point 保留共享分支而不复制 continuation（第 6.1 节）。这证明的是语言层面的语义正确性，不是运行时变快的经验结论。

## 8. 主要创新点

1. 对 CBPV 的 commuting conversions 作系统化刻画，并给出 CCNF。
2. 将单遍 continuation-based 变换与显式 join/jump 结合，在共享延续时避免代码复制和闭包构造。
3. 在 Lean 4 中机械证明变换的类型保持和语义保持，并给出 ground result 不变结论。

## 9. 局限性

1. 形式化语言不包含 effects；作者认为转换本身按原顺序执行子计算，但 effect safety 仍需把 effects 纳入逻辑关系后严格证明（第 6.4 节）。
2. 本文未测量代码大小、编译时间、运行时间或栈使用量；栈优化是讨论中的潜在性质，不是实验结果或已证明成本定理。
3. 仅研究 commuting 消去形式的变换；构造形式之间的交换因方向选择、代码复制和适用条件问题而未纳入（第 6.3 节）。
4. 后续内联若强制求值 thunk，可能破坏 CCNF，需要在相应场景重新规范化（第 6.2 节）。

## 10. 阅读后的研究方向反思

该工作提供了一个可复用的证明套路：先定义显式控制流构造与独立类型上下文，再以逻辑关系证明整个变换而非逐个局部重写。对编译器工程而言，下一步关键是将该语言级规范化与真实中间表示、内联/去 thunk 阶段连接起来，并测量代码尺寸、栈帧和编译成本，避免把语义正确性直接等同于性能收益。

## 11. 可进一步尝试的研究方向

1. 将 effects、异常或控制效应加入 CBPV 语义，证明 commuting 顺序保持性质。
2. 将 join point lowering 接到显式 CFG 或 stack-passing IR，并验证控制流与栈空间成本。
3. 对 inlining 后的 thunk forcing 设计增量 CCNF 检查和局部重规范化。
4. 与基线编译器变换比较编译时间、代码大小、栈峰值和执行时间。

## 12. 与当前批次文献的关系

与同批的 profile propagation 论文同属 OOPSLA 2026 的编译器相关工作，但层次不同：本文是形式化语义与控制流变换基础，关注 CBPV 规范化的正确性；另一篇是 LLVM 优化流水线中的 profile 预测与 CFG 映射，关注可用 profile 的准确性和代价。两篇均未把 LLM 作为核心编译系统角色。

## 13. 一页式总结

本文为 CBPV 建立了 commuting conversion normal form，并提出使用显式 continuation 与局部 join/jump 的单遍 CC-normalizer。Lean 4 机械化证明它保持类型和求值结果；case-of-case 例子说明 join point 如何避免复制延续。结论支持该变换在所定义语言中的语义正确性，不提供速度提升证据。effect 扩展、真实 IR 落地、栈/代码尺寸/性能测量仍待研究。
