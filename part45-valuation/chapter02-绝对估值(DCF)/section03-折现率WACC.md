---
type: concept
topic: 绝对估值
aliases: [WACC, Weighted Average Cost of Capital, 加权平均资本成本]
status: done
created: "2026-06-28"
updated: "2026-10-15"
tags: [估值, DCF, 核心]
related: "[[part45-valuation/chapter02-绝对估值(DCF)/section01-DCF基础|DCF基础]], [[section04-终值计算]]"
---

# 折现率WACC

## 公式与输入

$$WACC=\frac{E}{E+D}K_e+\frac{D}{E+D}K_d(1-T)$$
$$K_e=R_f+\beta ERP$$

E/D用一致的市场价值或有依据的目标资本结构；Kd是当前相应风险融资成本，不直接用旧债票息。税盾是否可用要核对亏损、税务及限制，不无条件享受。

Rf与现金流币种及期限匹配，ERP与Beta记录来源、估计期间和方法。历史Beta不是未来风险保证；WACC不直接用来折现FCFE。不能把6%—8%固定解释成低风险、12%—15%固定解释成高风险。

教学例：E权重80%、D权重20%、Ke10%、Kd5%、T25%，WACC=8.75%。用多个有依据的折现率情景检查模型，而不是通过调低WACC追求目标价。

模型框架参考：[Damodaran 估值课程](https://pages.stern.nyu.edu/~adamodar/New_Home_Page/lectures/val.html)；示例为手工构造，非实时数据。

---
**返回：** [[chapter00-估值方法]] | [[part00-主页]]
