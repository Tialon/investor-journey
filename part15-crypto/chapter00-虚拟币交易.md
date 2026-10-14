---
type: index
topic: 虚拟币
aliases: [虚拟币, 加密资产, Crypto Trading]
status: done
created: "2026-10-15"
updated: "2026-10-15"
tags: [虚拟币, 现货, 合约, MOC]
related: "[[chapter00-风险管理]], [[chapter00-量化投资]], [[part01-实践工作台]]"
---

# 虚拟币现货与合约交易

> 通用知识储备，覆盖交易、成本、账户证据和自动交易设计；不绑定任何项目实现、账户默认值或交易授权。所有数值示例均为教学假设。

## 优先入口

- [[section02-crypto-top-down-analysis|日线到15分钟的分工]]、[[section04-crypto-mtf-trade-cases|多周期交易时间轴]]、[[section05-crypto-mtf-conflicts|冲突与假突破]]。

- [[section06-crypto-indicator-workflow|技术指标配合策略的决策流程]]、[[section07-crypto-indicator-cases|完整指标交易与失效案例]]。

- [[section04-crypto-strategy-playbook|常用策略对照与案例计算口径]] — 8类策略的入场、加减仓、退出及费用后结果。

- [[section05-crypto-checklist|交易检查清单]]、[[虚拟币交易复盘模板|虚拟币复盘模板]]。
- [[section03-crypto-fees-accounting|费用后收益与成本]]、[[section01-crypto-sizing|损失预算与数量]]。
- [[section05-crypto-liquidation|价格与强平]]、[[section06-crypto-funding|资金费]]。
- [[section02-crypto-order-reconciliation|未知订单与对账]]、[[section03-crypto-api-recovery|API与网络恢复]]。

## 建议路径与验收

1. **现货基础**：分清资产/报价币、托管、订单角色和净成交数量；独立算出基础币扣费后的持仓与卖出净收益。
2. **风险与账本**：按损失预算算数量，核对在途资金、外部资金流与历史成本；不能把未知记为零。
3. **合约机制**：分清线性/反向、保证金币与结算币、开平模式、标记价、维护档位和资金费；独立核算合约净收益。
4. **策略与自动交易**：完整行情→已核实账户→成本/风险→最终权限→订单→成交→对账，演练超时、部分成交和重启；历史回测不等于真实执行验收。

## 基础、托管与资产研究

- [[section01-crypto-products|虚拟币资产、现货与合约]]
- [[section02-crypto-custody|交易所托管、钱包与密钥安全]]
- [[section03-crypto-transfers|充值提现、网络与链上授权]]
- [[section04-crypto-stablecoins|稳定币、抵押资产与收益产品]]
- [[section05-crypto-token-research|代币研究、供给解锁与基本面]]

## 现货交易与成本

- [[section01-crypto-spot-mechanics|现货数量、余额与成交现金流]]
- [[section02-crypto-orders|订单类型、盘口与成交约束]]
- [[section03-crypto-fees-accounting|手续费、滑点、成本基础与净收益]]
- [[section04-crypto-margin-borrow|现货杠杆、借币与利息]]

## 合约机制与结算

- [[section01-crypto-contract-types|永续、到期与线性反向合约]]
- [[section02-crypto-contract-pnl|合约盈亏、线性与反向算例]]
- [[section03-crypto-leverage|杠杆、名义敞口与初始保证金]]
- [[section04-crypto-position-modes|逐仓、全仓、单向与双向持仓]]
- [[section05-crypto-liquidation|指数价、标记价与强平机制]]
- [[section06-crypto-funding|资金费率、结算与持仓成本]]
- [[section07-crypto-adl|保险基金、ADL与极端市场]]

## 仓位、保护与应急

- [[section01-crypto-sizing|现货与合约的损失预算和仓位]]
- [[section02-crypto-protective-exits|止损止盈、减仓与保护单生命周期]]
- [[section03-crypto-portfolio-risk|账户权益、底仓与组合集中度]]
- [[section04-crypto-incident-response|交易所故障、异常账户与应急处理]]

## 信号与策略研究

- [[section01-crypto-market-signals|趋势、波动、成交量、OI与市场状态]]
- [[section02-crypto-grid-dca|网格、分档买入与有限周期预算]]
- [[section03-crypto-basis-hedging|基差、资金费交易与现货合约对冲]]
- [[section04-crypto-strategy-playbook|常用策略选择与案例计算口径]]
- [[section05-crypto-spot-breakout|现货突破：确认后加仓与跟踪止盈]]
- [[section06-crypto-spot-pullback|现货趋势回调：有限加仓与支撑失效]]
- [[section07-crypto-futures-trend|合约趋势多头：顺势加仓与分批保护]]
- [[section08-crypto-futures-short|合约跌破空头：反抽确认与分批回补]]
- [[section09-crypto-futures-range|合约区间反转：边界确认与严格退出]]

所有案例包含规则、实际成交和费用口径；盈利路径与失败路径分别演练。先读统一计算口径，再逐页核算现金流与剩余仓位。

## 数据、订单、恢复与验证

- [[section01-crypto-data-quality|行情窗口、K线闭合与数据新鲜度]]
- [[section02-crypto-order-reconciliation|订单状态、部分成交与幂等对账]]
- [[section03-crypto-api-recovery|API权限、校时、限流与网络恢复]]
- [[section04-crypto-backtest|成本后回测、回放与验证层级]]
- [[section05-crypto-checklist|现货与合约交易检查清单]]

## 常用技术指标与策略配合

- [[section01-crypto-moving-averages|均线：趋势过滤、回调与跟踪退出]]
- [[section02-crypto-rsi|RSI：动量状态、回归确认与钝化]]
- [[section03-crypto-macd|MACD：趋势动量、柱体与交叉边界]]
- [[section04-crypto-atr|ATR：波动距离、仓位与保护退出]]
- [[section05-crypto-volume|成交量：量价、相对量与执行确认]]
- [[section06-crypto-indicator-workflow|技术指标配合策略的决策流程]]
- [[section07-crypto-indicator-cases|指标配合策略的完整交易与失效案例]]

先分工，再配合：趋势/动量不保证方向，ATR用于风险尺度，成交量不代替执行深度。研究用已闭合信息，保护退出独立执行。

## 多周期分析与保护协调

- [[section01-crypto-timeframe-alignment|多周期K线闭合、时间对齐与信号有效期]]
- [[section02-crypto-top-down-analysis|日线到15分钟：趋势、结构与入场分工]]
- [[section03-crypto-mtf-position-exits|多周期加减仓、目标与保护如何协调]]
- [[section04-crypto-mtf-trade-cases|多周期协同交易：趋势、触发与分批退出案例]]
- [[section05-crypto-mtf-conflicts|周期信号冲突、未闭合确认与假突破案例]]

大周期负责背景，小时定义结构，15分钟找执行；保护实时独立，预算和真实数量统一。完整闭合、数据收到和信号有效期分别核验。

## 最易混淆的边界

- 名义仓位、初始保证金、账户权益和最大可能损失是不同量。
- 市价优先成交，限价约束价格；触发、受理、成交和对账是不同状态。
- 正资金费通常多付空，结算间隔和实际金额按具体合约核对，不能固定套8小时。
- 单向/双向与逐仓/全仓是不同设置；BUY/SELL不足以说明开平仓。
- 现货合约对冲可能因基差、保证金、ADL、平台或稳定币风险失效。
- 只读恢复不等于交易授权，订单超时不等于未成交，余额一致不证明全部事件完整。

## 官方资料与维护

以下资料用于核对机制，不是平台推荐或对所在地资格的判断。复核日期：2026-10-15；规则变化、权限、费率和具体产品参数使用时重新核对。

- [CFTC 虚拟币交易风险说明](https://www.cftc.gov/LearnAndProtect/AdvisoriesAndArticles/understand_risks_of_virtual_currency.html)。
- [Ethereum 安全与防骗](https://ethereum.org/security/)、[稳定币](https://ethereum.org/stablecoins/)。
- [Binance 现货过滤器](https://developers.binance.com/en/docs/products/spot/filters)、[现货交易接口](https://developers.binance.com/en/docs/catalog/core-trading-spot-trading/api/rest-api/trade)。
- [Binance 合约接口](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/trade)、[资金费](https://www.binance.com/en/support/faq/detail/360033525031)、[清算机制](https://www.binance.com/en/support/faq/detail/360033525271)。
- [Bybit 反向合约损益](https://www.bybit.com/en/help-center/article/Profit-Loss-calculations-Inverse-Contracts)。

**内容覆盖：46篇正文，另有本目录1篇及复盘模板1篇。** 内容完成只指笔记已写，未运行交易程序或验证盈利。未知参数不自动补默认值。

---
**返回：** [[part00-主页]] | [[part03-目录]] | [[part01-实践工作台]]
