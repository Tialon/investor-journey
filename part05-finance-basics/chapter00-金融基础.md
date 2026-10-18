---
type: index
topic: 金融基础
aliases: [金融基础导航, Finance Foundations]
status: done
created: "2026-10-16"
updated: "2026-10-16"
tags: [金融基础, 学习路线, MOC]
related: "[[part05-学习路线]], [[chapter00-资产配置]], [[chapter00-期权学习]]"
---

# 金融基础：从概念、现货到衍生品

先回答三个问题：接触什么资产、通过什么产品取得权利或义务、由谁交易和交割。这三个分类维度可以同时描述同一笔投资，不能把股票、现货、期权、券商混成一张互斥清单。

本模块提供4篇基础正文，已有专题通过链接继续学习，不以创建文件代表掌握或策略有效。只讲机制、核算与核查方法，不给具体买卖建议或收益承诺。资料读取日：2026-10-16；平台动态规则另查当时条款。

## 四篇入口

1. [[section01-financial-map|金融分类地图]]：资产类别、基金/ETF载体、现货与衍生品、交易所/券商/清算托管。
2. [[section01-capital-risk-cost|资金、杠杆与交易成本]]：名义本金、保证金、权利金、抵押、流动性和结算。
3. [[section02-cashflow-total-return|现金流与总收益]]：反复收权利金、账户损益、展期、复利和回撤。
4. [[section01-finance-glossary-quiz|术语速查与自测]]：先用自己的话说明，再独立复算例题。

## 建议学习顺序与小成果

| 顺序 | 阅读入口 | 应能留下的成果 |
|---|---|---|
| 1. 概念与分类 | 本模块金融地图、术语速查 | 给一个产品写出资产、结构、交易/清算/托管关系 |
| 2. 现金与无借贷现货 | [[section06-现金]]、[[chapter00-股票投资]]、[[part15-crypto/chapter01-basics/虚拟币基础、托管与资产研究#虚拟币资产、现货与合约|虚拟币资产、现货与合约]]、[[part15-crypto/chapter02-spot/现货交易、订单与成本核算#现货数量、余额与成交现金流|现货数量、余额与成交现金流]] | 说明实际持有什么；核算数量、买卖价差、费用与可用余额 |
| 3. 组合、估值与现金流 | 本模块资金风险与总收益；[[chapter00-风险管理]]、[[chapter00-估值方法]]、[[chapter00-财务分析]] | 区分入金、收款和盈利；解释持仓集中度、估值假设与现金用途 |
| 4. 债券、利率与配置 | [[chapter00-资产配置]]、[[part30-macro/chapter03-货币政策/section02-收益率曲线]]、债券提纲 | 认识债权/股权、期限和利率风险；比较不同资产在组合中的用途 |
| 5. 衍生品与期权工具 | [[part15-crypto/chapter03-derivatives/合约机制、保证金与盈亏结算#永续、到期与线性反向合约|永续、到期与线性反向合约]]、[[chapter00-期权学习]]、[[section08-strategy-payoffs]] | 先比较保护Put、备兑Call、现金担保Put，再理解价差；复算权利金、担保与指派 |
| 6. 验证与复盘 | [[section03-绩效评估]]、[[section03-最大回撤]]、[[section07-binance-ibkr-backtest]] | 将理论情景、模拟与真实记录分开，保留数据和成本缺口 |

股票与加密资产是并行学习分支，并非每个人都要交易全部资产。债券和ETF的完整专题目前仍是待补页：[[part65-portfolio/chapter02-资产类别/section02-债券|债券提纲]]、[[part92-institutional/chapter02-共同基金与ETF/section02-ETF|ETF提纲]]；本模块只补足入门识别，不将它们标为完整指南。

已有美股交易经验，可以用熟悉标的做“直接持有与不同期权结构”的纸面对照，先看组合总收益与风险，再看权利金。期权可以系统学习，但是否优先取决于概念和资金义务是否已掌握，并不天然优于组合管理、估值现金流或债券利率的基础学习。[FINRA：Options](https://www.finra.org/investors/investing/investment-products/options)

需要查询具体平台时，再去 [[section01-binance-stock-options|币安Stock Options]]、[[section06-binance-other-derivatives|币安其他衍生品辨别]] 与 [[section01-tws-options|IBKR专题]]；本模块保持通用概念层级。

## 币安股票/ETF期权的理论学习路线

这里沿用Stock Options语境；加密和商品期权先到 [[section06-binance-other-derivatives]] 辨别规则。可以用6–8周安排下列六阶段，掌握程度优先于周数，这只是学习日历示例，不是盈利承诺。

| 阶段 | 复用笔记与独立自测 |
|---|---|
| 1. 识别合约 | [[section03-option-contract]]、[[section07-option-chains]]；说明标的、Call/Put、K、到期、乘数，独立算出报价1.20×100×2张=240美元权利金，未计费用 |
| 2. 核算盈亏 | [[section02-binance-small-capital]]、[[section08-strategy-payoffs]]；分别解释到期与到期前盈亏、平仓与行权、实值与盈利 |
| 3. 理解定价输入 | [[section06-greeks-in-practice]]、[[section02-hv-vs-iv]]；用自己的话说明Delta、Theta、Vega、Gamma，以及IV与历史波动率的区别 |
| 4. 比较策略 | [[section01-long-call]]、[[section02-long-put]]、[[section08-strategy-payoffs]]、[[section04-binance-expiry-collateral]]；先买Call/Put与保护Put，再备兑Call/担保Put，最后研究垂直价差及短腿指派；比较整体收益与资金义务 |
| 5. 核对平台执行 | [[section03-binance-options-workflow]]、[[section04-binance-expiry-collateral]]；解释实时/延迟行情、限价开平仓、股票交割及不能默认实值自动行权；策略理论不代表账户均可执行 |
| 6. 纸面复盘 | [[section05-binance-options-journal]]、[[section07-binance-ibkr-backtest]]；记录报价与成本，区分方向、时间、IV和流动性对结果的影响，不能只看标的K线或权利金收入 |

每阶段先独立解释与复算，再进入下一阶段；模拟、纸面研究和真实账户结果分别记录。具体期权规则的日期与冲突复用 [[section01-binance-stock-options|官方来源台账]]，不在基础导航重复维护平台手册。

## 知识与实施的归属

知识原理、术语解释、学习笔记和讨论结论沉淀在 **investor-journey（ij）**；策略验证、实施记录和代码归 **adaptiveTrading（at）**。实施讨论再引用当时确认存在的at资料与代码路径；本导航仅连接已核实的ij笔记。

知识解释正确、模拟有收益和真实执行可行是不同的验收结果。进入实施前还需单独明确数据、账户权限、风险预算与执行证据。

---
**返回：** [[part00-主页]] | [[part03-目录]] | [[part01-实践工作台]]
