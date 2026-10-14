---
type: concept
topic: 绝对估值
aliases: [Terminal Value, 永续价值]
status: done
created: "2026-06-28"
updated: "2026-10-15"
tags: [估值, DCF]
related: "[[part45-valuation/chapter02-绝对估值(DCF)/section01-DCF基础|DCF基础]], [[section03-折现率WACC]], [[section05-DCF敏感性分析]]"
---

# 终值计算

## 永续增长法

$$TV_n=\frac{FCFF_{n+1}}{WACC-g},\qquad WACC>g$$

终值在第n年末，仍需折现回估值日。FCFFₙ₊₁必须反映稳定期增长所需再投资；不能只增加收入却不增加资金需求。g、通胀、币种及长期经营环境一致，2%—3%不是所有公司的固定标准。

退出倍数法TVₙ=EBITDAₙ×退出EV/EBITDA，倍数需对应稳定期增长、风险和利润质量。采用倍数并不消除相对估值假设。

教学例：稳定期下一年FCFF10、WACC10%、g2%，TV=125；若第5年末得此价值，现值≈77.62。终值占比=终值现值/全部经营价值，必须实际计算，不套用“5年就60%—80%”。

模型框架参考：[Damodaran 估值课程](https://pages.stern.nyu.edu/~adamodar/New_Home_Page/lectures/val.html)；示例为手工构造，非实时数据。

---
**返回：** [[chapter00-估值方法]] | [[part00-主页]]
