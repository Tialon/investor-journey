---
type: concept
topic: 绝对估值
aliases: [DCF Sensitivity Analysis]
status: done
created: "2026-06-28"
updated: "2026-10-15"
tags: [估值, DCF]
related: "[[part45-valuation/chapter02-绝对估值(DCF)/section04-终值计算|终值计算]], [[section03-折现率WACC]]"
---

# DCF敏感性分析

## 同时改变经营与估值假设

先建悲观、基准、乐观经营情景，改变收入、利润率、再投资、股数和债务；再在每个情景内做WACC×g二维表。只改变折现率而不改变业务，可能低估经营风险。

示例：FCFFₙ₊₁=10，在WACC10%下，g从1%→2%→3%，第n年终值从111.11→125→142.86。所有格子必须满足WACC>g，并单独折现与桥接股权，不能把终值直接当每股价值。

输出：价值区间、主要敏感项、当前价格隐含的增长/利润条件、哪项证据会推翻模型。每个输入保留来源和日期；“合理区间”也只是条件结果。

模型框架参考：[Damodaran 估值课程](https://pages.stern.nyu.edu/~adamodar/New_Home_Page/lectures/val.html)；示例为手工构造，非实时数据。

---
**返回：** [[chapter00-估值方法]] | [[part00-主页]]
