---
type: concept
topic: 绝对估值
aliases: [FCF Forecast, FCF预测]
status: done
created: "2026-06-28"
updated: "2026-10-15"
tags: [估值, DCF]
related: "[[part45-valuation/chapter02-绝对估值(DCF)/section01-DCF基础|DCF基础]], [[section03-折现率WACC]]"
---

# 自由现金流预测

## 避免重复扣减CapEx

$$FCFF=EBIT(1-T)+D\&A-CapEx-\Delta NWC$$

NWC在此为非现金经营性营运资本；ΔNWC为本期减上期，增加通常消耗现金。非现金加回不代表没有经济成本，特别要另查SBC与稀释。

如果已经用“收入×FCF利润率”直接预测FCF，该利润率已包含资本开支和营运资本，就不能再扣CapEx与ΔNWC。选择驱动法或完整FCF利润率法，注明口径。

| 驱动项 | 输入依据 |
|---|---|
| 收入 | 分部销量×售价、客户数×消费、订阅数×单价 |
| EBIT率 | 毛利、费用、规模与竞争假设 |
| 税率 | 有效/正常化税率与亏损抵扣 |
| 再投资 | 维持/扩张CapEx、折旧、账期与存货 |
| SBC | 经济成本与股数处理，防止遗漏或重复计成本 |

教学例：EBIT100、税25%、折旧10、CapEx20、ΔNWC5，FCFF=60。增长需要再投资，不能同时假设收入高速增长和CapEx/营运资本几乎不变。

模型框架参考：[Damodaran 估值课程](https://pages.stern.nyu.edu/~adamodar/New_Home_Page/lectures/val.html)；示例为手工构造，非实时数据。

---
**返回：** [[chapter00-估值方法]] | [[part00-主页]]
