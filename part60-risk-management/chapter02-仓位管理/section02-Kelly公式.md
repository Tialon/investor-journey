---
type: concept
topic: 仓位管理
aliases: [Kelly Criterion]
status: done
created: "2026-06-28"
updated: "2026-10-15"
tags: [风险管理, 仓位管理]
related: ""
---

# Kelly公式

## 理想化公式与实际边界

二元结果、输时损失全部下注额、赢时净赚b倍、胜率p已知时：f*=(bp−(1−p))/b。f是风险资金比例，并非持仓市值、名义本金或保证金比例；负值不等于授权做空。

期权有连续盈亏、跳空、成本及相关交易，短期胜率不足以估计Kelly。分数Kelly不能修复错误参数或遗漏尾部事件。没有可信收益分布时，优先用损失预算和压力约束。

例：p=0.6、b=1得到20%，只是理想化算术，不能作为真实账户配置建议。详细边界见 [[section02-position-sizing]]，执行见 [[section01-固定比例法]]。

---
**返回：** [[chapter00-风险管理]] | [[part00-主页]]
