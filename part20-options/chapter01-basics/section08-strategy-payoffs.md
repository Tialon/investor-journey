---
type: reference
topic: 期权策略
aliases: [期权策略盈亏对照, Strategy Payoffs]
status: done
created: "2026-10-15"
updated: "2026-10-15"
tags: [期权, 策略, 风险, 速查]
related: "[[section02-call-and-put]], [[section02-position-sizing]], [[section06-execution-checklist]]"
---

# 期权策略盈亏对照：先算损失，再选结构

## 统一口径

以下是到期损益，假设普通非负价格股票、标准未调整合约、忽略利息、分红、税费和交易成本。Sₜ 是到期股价，K 是行权价，p 是单股报价的权利金，m 是乘数，n 是正数张数。美股标准个股合约通常 m=100，调整合约需另查交割物。

到期公式不是提前平仓报价。提前退出要用可成交的组合价格重算，不能拿模型中间价当成交价。

## 单腿与持股策略

| 策略 | 每股到期损益 | 最大收益 | 最大亏损 | 到期盈亏平衡 |
|---|---|---|---|---|
| Long Call | max(Sₜ−K,0)−p | 理论无限 | p | K+p |
| Long Put | max(K−Sₜ,0)−p | K−p | p | K−p |
| 裸 Short Call | p−max(Sₜ−K,0) | p | 理论无限 | K+p |
| Short Put / CSP | p−max(K−Sₜ,0) | p | K−p | K−p |
| Covered Call，股票成本 B | Sₜ−B+p−max(Sₜ−K,0) | K−B+p | B−p | B−p |
| Protective Put，股票成本 B | Sₜ−B−p+max(K−Sₜ,0) | 理论无限 | B+p−K | 通常 B+p，须检查是否落在 Sₜ≥K 区域 |

美元损益 = 每股损益 × m × n。CSP 与裸卖 Put 的期权损益相同，区别在现金准备；CSP 接股需预留 K×m×n，不可只按损失预算预留现金。

Covered Call 仍承受股票下跌，权利金只缓冲一部分。上表按整个持股周期成本 B 算；评估“现在是否卖 Call”时，也应按当前市值核对机会成本。

## 同到期垂直价差

K₁<K₂，宽度 W=K₂−K₁；d 为净支出，c 为净收入，均是每股报价且介于 0 和 W。两腿等数量、相同乘数和交割物。

| 策略 | 构造 | 最大收益/股 | 最大亏损/股 | 到期盈亏平衡 |
|---|---|---|---|---|
| Bull Call Debit Spread | 买 K₁ Call，卖 K₂ Call | W−d | d | K₁+d |
| Bear Put Debit Spread | 买 K₂ Put，卖 K₁ Put | W−d | d | K₂−d |
| Bull Put Credit Spread | 卖 K₂ Put，买 K₁ Put | c | W−c | K₂−c |
| Bear Call Credit Spread | 卖 K₁ Call，买 K₂ Call | c | W−c | K₁+c |

同到期 Iron Condor：买更低 Put、卖低 Put、卖高 Call、买更高 Call，两边宽度 Wₚ、W꜀。最大收益 c，最大亏损 max(Wₚ,W꜀)−c，平衡点是短 Put 行权价−c、短 Call 行权价+c。前提是四腿等量、交割物一致、短行权价有正确排序。

Short Iron Butterfly 的两个短腿同一 K、两翼距离分别为 Wₚ、W꜀；最大亏损仍为 max(Wₚ,W꜀)−c，平衡点 K±c。不可把铁蝶与普通 Long Call Butterfly 混为一谈。

## 方向未知的买方组合

| 策略 | 构造 | 最大亏损/股 | 到期平衡点 | 主要代价 |
|---|---|---|---|---|
| Long Straddle | 买同一 K 的 Call 与 Put | 总支出 d | K−d、K+d | 两腿权利金、Theta、IV 回落 |
| Long Strangle | 买较低 Kₚ Put 与较高 K꜀ Call | 总支出 d | Kₚ−d、K꜀+d | 需要更大幅度移动 |

两者上方收益理论无限，下方受股价零下限限制。若下方平衡点小于零，在非负股价范围内就不存在该侧盈利区间。

日历与对角价差跨到期，不能套用 W−c 作为统一最大亏损；须检查提前指派、近月到期后的剩余长腿和两段期限 IV。

## 三个算术练习

1. Long Call：K=100、p=5、m=100、n=1。到期 Sₜ=103，期权实值，但总损益是 (3−5)×100=−$200；到期 Sₜ=110，盈利 $500。
2. Bull Put：卖 100 Put、买 95 Put，净收 1.20。最大收益 $120，最大亏损 $380，平衡 98.80；到期 Sₜ=97，损益 (1.20−3)×100=−$180。
3. Covered Call：持股成本 100、卖 105 Call 收 2。到期 Sₜ=80，损益 −$1,800；到期 Sₜ=120，损益仍封顶 $700。

## 到期公式之外的风险

公式假设组合完整保留并正确结算。美式短腿可提前指派；长腿不自动替短腿履约。拆腿、到期临界指派、交割后股票跳空、交易成本与强制清算可能改变实际损失。策略比较必须同时看损益、资金占用、路径和退出方式。

官方延伸阅读（核对：2026-10-15）：[OIC 策略库](https://www.optionseducation.org/strategies/all-strategies)、[Covered Call](https://www.optionseducation.org/strategies/all-strategies/covered-call-buy-write)、[Cash-Secured Put](https://www.optionseducation.org/strategies/all-strategies/cash-secured-put)、[Bear Call Spread](https://www.optionseducation.org/strategies/all-strategies/bear-call-spread-credit-call-spread)。

---
**相关笔记：** [[section02-call-and-put]], [[section02-position-sizing]], [[section06-execution-checklist]]
**返回：** [[chapter00-期权学习]] | [[part00-主页]]
