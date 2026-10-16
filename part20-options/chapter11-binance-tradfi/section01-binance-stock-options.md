---
type: reference
topic: 币安股票期权
aliases: [币安传统金融期权, Binance Stock Options]
status: done
created: "2026-10-16"
updated: "2026-10-16"
tags: [期权, 币安, 美股, 产品规则]
related: "[[section02-binance-small-capital]], [[section03-binance-options-workflow]], [[section06-binance-other-derivatives]]"
---

# 币安传统金融期权：先辨产品，再看规则

> 本卷为中文学习与操作核对资料，不提供具体合约买卖建议或收益承诺。规则核验日：**2026-10-16**。产品可用性与权限依地区、签约主体和账户而异；页面日期与实盘界面出现差异时，先核实后操作。

## 先确认自己看到什么

“传统金融”“黄金”“股票”描述资产类别，不能据此判断合约机制。

| 界面产品 | 买到什么 | 本卷处理方式 |
|---|---|---|
| Binance **Stock Options** | 美国股票/ETF 的期权合约 | 本卷重点：权利金、100股标准合约、实物交割、行权与指派 |
| 加密资产期权 | BTC、ETH 等相关的期权 | 另见 [[section06-binance-other-derivatives\|其他衍生品辨别]]；按加密期权规格核对 |
| 黄金/白银 Commodity Options | 商品价格相关的欧式现金结算期权 | 不能套用股票交割和100股乘数 |
| TradFi Perpetuals | 跟踪传统资产价格的永续合约 | 保证金、资金费、强平机制；不是期权 |

产品入口相邻不表示账户、清算、交易时段或风险机制相同。不要把“USDT现金结算”写成所有币安期权的共同特点。

## Stock Options 的服务链与账户边界

官方 FAQ 及上线公告说明：Stock Options 由 **Nest Trading** 引介，**Alpaca** 承担执行、清算、交收及托管相关服务。美国股票/ETF期权行权后涉及股票实物交割；它不是单纯押注股价的 USDT 现金差价合约。具体法律关系、资产持有及适用条款应阅读自己的开户协议。[Stock Options FAQ](https://www.binance.com/en/support/faq/detail/3dddc209bb564b448c0948fc4c5e6d80)、[2026-10-15 上线公告](https://www.binance.com/en/support/announcement/detail/9fe16193d2ca442ca4d0fca0d608c910)

查看入口不代表已获交易许可。操作前应自行确认地区资格、身份验证、期权权限/适当性评估、风险披露、行情授权和资金可用状态；如被拒绝或缺失，停止该产品流程，不填写虚假的投资经验，不通过其他产品代替。

## 规则有版本差异：不能只看旧教程

| 官方资料 | 页面日期 | 核验发现与阅读边界 |
|---|---|---|
| [Stock Options FAQ](https://www.binance.com/en/support/faq/detail/3dddc209bb564b448c0948fc4c5e6d80) | 发布2026-09-01；更新2026-10-09 | 仍写 long-only；可用于核对买方、报价、时段、行权和到期处理 |
| [备兑 Call / 现金担保 Put FAQ](https://www.binance.com/en/support/faq/detail/4b3552fd56b24af996c8377d2fea904f) | 发布2026-10-15 | 新增充分担保的卖方结构、抵押与指派说明 |
| [卖方功能上线公告](https://www.binance.com/en/support/announcement/detail/9fe16193d2ca442ca4d0fca0d608c910) | 2026-10-15 | 佐证功能扩展；注明地区可用性限制 |

**当前写法：已推出备兑 Call 和现金担保 Put，但实际可用范围必须核对地区、账户权限及现行条款。** 旧页的 long-only 不能证明现在所有账户均只能买，也不能因为新页有卖方教程就断言所有人都能卖。裸卖 Call/Put、复杂价差的可用性不能从这次上线推导。

## 时段、报价与费用要分开核对

- 普通美国股票期权通常为 **09:30–16:00 美东时间（ET）**；部分 ETF/ETN 合约可至16:15。盘前盘后一般不支持；休市、提前收市及停牌需另查。
- 免费报价可能**延迟15分钟**，注意时钟/延迟标识；OPRA 实时行情需激活并核对生效状态。标的最新价、Last和期权Bid/Ask的时间戳可能不同。
- 官方当前说明采用**限价单**。限价保护可接受价格，不能保证成交。订单有效期、最小报价单位、部分成交及拒单原因以确认页为准。
- 权利金报价是每股还是整张、币种是什么、乘数多少，均要在合约详情确认。资金充值/换汇与期权成交是不同环节，不能凭现货钱包余额认定已满足期权资金或交割要求。

以上时段与行情依据 [Stock Options FAQ](https://www.binance.com/en/support/faq/detail/3dddc209bb564b448c0948fc4c5e6d80)；ET随夏令时变化，记录时同时保留 `America/New_York` 与 UTC 时间。不要用固定北京时间替代全年时钟。

费用入口：[股票与期权费用页](https://www.binance.com/en/fee/stocks?type=stockOptions)。本次读取的可见正文仍标注 **Direct Equities**，不足以确认股票期权费率，故不抄录其数字。操作时查看期权专属现行费表和订单预览，记录交易、监管、行权/指派、换汇及其他适用费用；无法确认的项目写“待核实”，不能按零计算净收益。

## 本卷阅读顺序

1. [[section02-binance-small-capital|小资金参与的原理与盈亏]]：先理解权利金、乘数及归零风险。
2. [[section03-binance-options-workflow|从核查到开平仓的操作流程]]：区分买开、卖平、卖开、买平。
3. [[section04-binance-expiry-collateral|行权、到期与充分担保卖方]]：把有限权利金损失与股票义务分开。
4. [[section05-binance-options-journal|检查清单与日志]]：保留报价、规则、成交和交割证据。
5. [[section06-binance-other-derivatives|其他衍生品辨别]]：加密期权、商品期权、TradFi永续分别处理。
6. [[section07-binance-ibkr-backtest|IBKR对照与真实期权回测]]：理论共通，执行独立核查。

## 官方来源维护台账

| 来源 | 日期口径 | 用途 |
|---|---|---|
| 上方3篇 Binance Stock Options 官方资料 | 已分别注明发布/更新日 | 产品、权限变更、操作规则 |
| [OIC/OCC：Equity vs. Index Options](https://www.optionseducation.org/advancedconcepts/equity-vs-index-options) | 页面未列更新日；2026-10-16读取 | 股票标准合约与指数期权的机制区别；调整合约需另查 |
| [OIC：Understanding Options Greeks](https://www.optionseducation.org/advancedconcepts/understanding-options-greeks) | 页面未列更新日；2026-10-16读取 | 定价敏感度及模型边界 |
| [IBKR：Delivery, Exercise and Corporate Actions](https://portal.interactivebrokers.com/en/trading/delivery-exercise-actions.php) | 页面未列总更新日；2026-10-16读取 | IBKR行权、到期及风险处置；页面局部旧假日表不能当2026日历 |
| [[section06-binance-other-derivatives\|其他币安产品的独立官方来源]] | 各产品单列日期 | 禁止跨产品套用规则 |

维护时保存“查阅日期、页面发布/更新日期、适用主体、规则差异、界面确认结果”。日期较新是线索，不能仅凭新旧关系消除地区或产品范围差异。若冲突影响下单、抵押或到期结果，先向该产品官方支持核实。

---
**返回：** [[chapter00-期权学习]] | [[part03-目录]]
