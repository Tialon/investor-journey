---
type: concept
topic: 虚拟币实践
aliases: ["趋势、波动、成交量、OI与市场状态"]
status: done
created: "2026-10-15"
updated: "2026-10-15"
tags: [虚拟币, 策略]
related: "[[chapter00-虚拟币交易]]"
---

# 趋势、波动、成交量、OI与市场状态

## 常用指标与策略入口

- [[section01-crypto-moving-averages|均线]]：趋势、回调与退出。
- [[section02-crypto-rsi|RSI]]：动量、回归、钝化与背离边界。
- [[section03-crypto-macd|MACD]]：柱体、交叉与重复证据。
- [[section04-crypto-atr|ATR]]：波动距离、数量与保护。
- [[section05-crypto-volume|成交量]]：相对量、量价与深度区别。
- [[section06-crypto-indicator-workflow|策略配合流程]]、[[section07-crypto-indicator-cases|完整交易与失效案例]]。

## 多周期分析入口

[[section02-crypto-top-down-analysis]] | [[section03-crypto-mtf-position-exits]] | [[section04-crypto-mtf-trade-cases]] | [[section05-crypto-mtf-conflicts]]。

## 指标各自回答什么

趋势看价格方向与持续性；实现波动看历史变化幅度；成交量看换手；OI看未平仓合约数量；资金费与基差看定价和持仓成本。高波动不自动给出多空方向。

| 数据 | 必须检查 |
|---|---|
| 均线/动量 | 完整收盘、窗口长度、跨交易所口径 |
| 波动/ATR | 频率、跳空、年化口径，不直接当盈利信号 |
| 成交量 | 基础币/报价币、是否汇总、异常刷量 |
| OI | 张数/币数量/美元值；币价变化可改变美元OI |
| 清算数据 | 来源覆盖、不完整统计与时间延迟 |
| 资金费/基差 | 合约、期限、预计值与已结算事实 |

OI增长不证明新资金只在做多，每份合约有多空双方。价格上涨与OI上涨可形成研究假设，但需结合持仓/成交结构并承认动机不可直接观察。

制定TREND/震荡/高波动等状态时，说明计算规则与允许行为。状态只是输入，仍需账户、费用与风险检查；市场状态不同于开仓授权。

---
**返回：** [[chapter00-虚拟币交易]] | [[part01-实践工作台]]
