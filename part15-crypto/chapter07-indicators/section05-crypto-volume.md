---
type: concept
topic: 虚拟币技术指标
aliases: ["成交量：量价、相对量与执行确认"]
status: done
created: "2026-10-15"
updated: "2026-10-15"
tags: [虚拟币, 技术指标, 策略]
related: "[[section06-crypto-indicator-workflow]], [[section07-crypto-indicator-cases]]"
---

# 成交量：量价、相对量与执行确认

## 成交量不是可用资金净流入

每笔成交同时有买卖双方。基础币量、报价币成交额、张数、美元估值和跨平台汇总不是同一单位；币价上涨本身可增加报价币成交额。现货量与合约量分别看，OI也不是成交量。

相对量的一种自定口径：

```text
RVOL20_t = 当前完整K线Volume_t / 前20根完整K线Volume的均值
```

分母排除当前根，不用未来值。20是本组教学选择，不声称和所有平台默认一致。前20根总量2,000、均值100、本根180，RVOL=1.8；未完成半小时的量不能直接与完整1小时相比。

24小时币市也有时段/周末差异。若按同星期/同小时基准比较，要单独定义历史样本和缺失处理，不用股票常规交易时段规则直接套用。

## 配合策略的作用

| 场景 | 可检查的量价条件 | 行为边界 |
|---|---|---|
| 突破 | 闭合突破、较历史活跃且盘口可执行 | 量大不保证真突破，仍留无效位 |
| 回调 | 回撤量较低，确认反弹有合理活跃度 | 缩量也可能买盘枯竭，不能自动加仓 |
| 跌破 | 下行闭合结构与活跃卖出换手 | 放量也可能耗竭，不能据此无限追空 |
| 区间 | 边界拒绝与回归换手 | 突破放量时先重估区间，而非新增网格 |

手工突破快照：RVOL20=1.8、收盘99.8突破先前99.5结构，才与趋势/动量/风险一起研究买入。之后RVOL=1.4并有新价格确认，可研究有限加仓；若价格跌回结构、下跌RVOL=2.2，取消追加并处理保护，不因“量很大”继续买。

## OBV的辅助读法

OBV在收盘上升时加该根量，下跌时减，同价时按本文约定不变。以初值0，价格100→101、量180得180；再到100、量220得−40；下一根仍100、量200仍为−40。这是价格方向归类的累积量，不是净流入−40币。

## 数据与失效

刷量、异常单笔、平台覆盖变化、合约/现货混拼、未闭合时段和零分母会失真。单平台Volume很高也不证明当前Bid/Ask深度够；执行还需当前报价、数量、稳定性及退出成本。

放量、RSI与MACD可以同时好看却仍失败。见 [[section07-crypto-indicator-cases]]，不要把指标一致解释为真实盈利概率。

资料（核对2026-10-15）：[相对量的口径差异](https://www.tradingview.com/support/solutions/43000635874-how-do-we-calculate-relative-volume-and-relative-volume-at-time/)、[Fidelity OBV定义](https://www.fidelity.com/learning-center/trading-investing/technical-analysis/technical-indicator-guide/obv)。

## 与支撑压力及K线配合

突破与回踩的同周期相对量见 [[section03-crypto-breakout-retest]]；4根15分钟量聚合成小时量见 [[section05-crypto-price-action-cases]] 案例A。放量收盘突破仍可能失败，见 [[section06-crypto-price-action-failures]]。

---
**返回：** [[chapter00-虚拟币交易]] | [[part01-实践工作台]]
