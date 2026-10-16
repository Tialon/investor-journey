---
type: index
topic: 期权
aliases: [Options Learning, Options Trading]
status: done
created: "2026-06-28"
updated: "2026-10-16"
tags: [期权, MOC]
related: "[[chapter00-风险管理]], [[chapter00-量化投资]], [[chapter00-Python投资]]"
---

# 📈 期权学习

> 从基础到高级，系统掌握期权交易

## 速查参考
- [[section01-binance-stock-options|币安Stock Options：产品、规则与操作入口]]
- [[section02-binance-small-capital|为什么小资金能参与期权：权利金、乘数与归零风险]]
- [[section00-options-glossary|美股期权术语速查表]]
- [[section08-strategy-payoffs|策略盈亏与损失对照]]
- [[section06-execution-checklist|开仓、持仓、到期与复盘检查清单]]

## 第1卷：基础篇
1. [[section01-what-is-option|什么是期权]]
2. [[section02-call-and-put|看涨期权与看跌期权]]
3. [[section03-option-contract|期权合约要素]]
4. [[section04-intrinsic-time-value|内在价值与时间价值]]
5. [[section05-moneyness|实值、平值、虚值]]
6. [[section06-exercise-assignment|行权与指派]]
7. [[section07-option-chains|期权链解读]]
8. [[section08-strategy-payoffs|策略盈亏对照]]

## 第2卷：Greeks
1. [[section01-delta|Delta]]
2. [[section02-gamma|Gamma]]
3. [[section03-theta|Theta]]
4. [[section04-vega|Vega]]
5. [[section05-other-greeks|Rho等其他Greeks]]
6. [[section06-greeks-in-practice|Greeks综合实战]]

## 第3卷：买方策略
1. [[section01-long-call|买入看涨期权]]
2. [[section02-long-put|买入看跌期权]]
3. [[section03-long-straddle|买入跨式策略]]
4. [[section04-expiry-strike|到期日与行权价选择]]
5. [[section05-buyer-risk|买方风险管理]]

## 第4卷：卖方策略
1. [[section01-covered-call|备兑看涨期权]]
2. [[section02-cash-secured-put|现金担保看跌期权]]
3. [[section03-credit-spread|价差策略]]
4. [[section04-iron-condor|铁鹰策略]]
5. [[section05-iron-butterfly|铁蝶策略]]
6. [[section06-seller-risk|卖方风险管理]]

## 第5卷：波动率
1. [[section01-what-is-volatility|什么是波动率]]
2. [[section02-hv-vs-iv|历史波动率vs隐含波动率]]
3. [[section03-volatility-surface|波动率曲面]]
4. [[section04-volatility-smile|波动率微笑与偏斜]]
5. [[section05-vix|VIX指数详解]]
6. [[section06-vol-trading|波动率交易策略]]

## 第6卷：财报交易
1. [[section01-earnings-and-options|财报与期权的关系]]
2. [[section02-pre-earnings|财报前策略]]
3. [[section03-post-earnings|财报后策略]]
4. [[section04-iv-crush|IV Crush详解]]
5. [[section05-earnings-playbook|财报交易实战流程]]

## 第7卷：组合管理
1. [[section01-risk-metrics|期权组合的风险度量]]
2. [[section02-position-sizing|仓位管理与资金分配]]
3. [[section03-dynamic-hedging|动态对冲与再平衡]]
4. [[section04-stress-testing|压力测试与情景分析]]
5. [[section05-trading-journal|交易日志与绩效评估]]
6. [[section06-execution-checklist|交易检查清单]]

## 第8卷：IBKR实战
1. [[section01-tws-options|IBKR期权交易界面]]
2. [[section02-order-types|期权订单类型]]
3. [[section03-margin-risk|保证金与风控设置]]
4. [[section04-analytics-tools|期权分析工具]]
5. [[section05-api-basics|API与自动化基础]]

## 第9卷：专业交易
1. [[section01-market-makers|做市商与流动性]]
2. [[section02-unusual-activity|大单与异常交易识别]]
3. [[section03-term-structure|期限结构与日历价差]]
4. [[section04-vol-arbitrage|波动率套利]]
5. [[section05-portfolio-greeks|组合希腊字母管理]]

## 第10卷：Python + AI 自动交易
1. [[section01-python-libraries|Python期权库入门]]
2. [[section02-pricing-greeks|期权定价与Greeks计算]]
3. [[section03-backtesting|回测框架搭建]]
4. [[section04-signal-strategy|信号生成与策略开发]]
5. [[section05-ibkr-api|IBKR API自动交易]]
6. [[section06-ai-options|AI在期权交易中的应用]]

## 第11卷：币安传统金融期权

1. [[section01-binance-stock-options|Stock Options产品与规则版本]]
2. [[section02-binance-small-capital|小资金原理、基础盈亏与Greeks/IV]]
3. [[section03-binance-options-workflow|操作核查、限价开仓与平仓退出]]
4. [[section04-binance-expiry-collateral|行权到期、备兑Call与现金担保Put]]
5. [[section05-binance-options-journal|检查清单与交易日志]]
6. [[section06-binance-other-derivatives|另章辨别加密/商品期权与TradFi永续]]
7. [[section07-binance-ibkr-backtest|IBKR对照与真实期权回测要求]]

## 内容覆盖与学习验收

| 卷 | 现有篇数（不含本目录） | 下一步验收 |
|---|---|---|
| 基础篇 | 9 | 看懂合约、区分实值与盈利、处理行权与指派 |
| Greeks | 6 | 带符号数量与乘数核算，分清局部近似和跳空风险 |
| 买方策略 | 5 | 计算成本、到期平衡点及 IV 回落情景 |
| 卖方策略 | 6 | 计算损失、现金义务及短腿指派后暴露 |
| 波动率 | 6 | 统一 IV/HV 口径，解释期限结构与偏斜 |
| 财报交易 | 5 | 比较方向、事件溢价和跳空的联合影响 |
| 组合管理 | 6 | 按损失预算算张数，完成压力表与复盘 |
| IBKR实战 | 5 | 核对当前权限、订单预览、保证金及到期政策 |
| 专业交易 | 5 | 区分持仓事实、交易动机推断和模型假设 |
| Python+AI | 6 | 验证数据与模型，再做模拟对账与故障演练 |
| 币安传统金融期权 | 7 | 区分产品、核算权利金与抵押，逐项确认权限、到期与交割 |

**现有内容：66篇正文/参考笔记，另有本目录1篇。** “内容已写”不等于“已经掌握”，也不等于已通过模拟或实盘验收。运行 API 教学代码前请先查看该页的运行边界。

## 建议学习路径

1. **机制与算术**：基础篇 → [[section08-strategy-payoffs|策略盈亏对照]] → 行权与指派。能解释“实值仍亏钱”和指派后的股票义务。
2. **敏感度与情景**：Greeks → 波动率 → 买卖方结构。能核算张数/乘数/符号，并解释高 IV 不能单独作为卖出理由。
3. **组合与执行**：仓位管理 → [[section06-execution-checklist|交易检查清单]] → IBKR 工具。先做模拟成交、交割和风险记录。
4. **事件与进阶**：财报 → 专业交易 → 回测/Python/AI。先检查数据时间、成本、样本外和订单对账，再讨论策略效果。

## 易混淆概念

- 实值 ≠ 盈利；Delta ≠ 真实盈利概率。
- 保证金 ≠ 最大亏损；收取权利金 ≠ 已实现利润。
- Volume/OI ≠ 净买入方向；高 IV ≠ 自动有卖方优势。
- Delta 中性 ≠ 全风险中性；价差到期最大损失 ≠ 所有路径下的账户损失上限。
- 展期 = 旧仓平仓 + 新仓开仓，旧亏损仍需记录。

## 官方资料与维护

合约机制与策略优先查 [OIC/OCC 投资者教育](https://www.optionseducation.org/)，VIX 产品查 [Cboe](https://www.cboe.com/tradable-products/volatility-trading/)，账户规则查 [IBKR 到期政策](https://www.interactivebrokers.com/en/trading/delivery-exercise-actions.php)。通用期权内容核对日期：2026-10-15；币安专章核对日期：2026-10-16，详见 [[section01-binance-stock-options|官方来源与规则差异台账]]。券商界面、权限和保证金规则仍需在使用时复核。


---
**相关笔记：** [[chapter00-风险管理]], [[chapter00-量化投资]], [[chapter00-Python投资]]
**返回：** [[part00-主页]] | [[part03-目录]]
