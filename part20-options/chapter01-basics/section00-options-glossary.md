---
type: reference
topic: 期权术语
aliases: [Options Glossary, 期权速查表, 期权术语表]
status: done
created: "2026-07-05"
updated: "2026-07-05"
tags: [期权, 术语, 速查, 参考]
related: "[[什么是期权]], [[看涨期权与看跌期权]], [[期权合约要素]]"
---

# 美股期权术语速查表（Options Glossary）

> 覆盖学习期权最常见的 90% 术语，作为后续学习的"字典"使用。

---

## 一、基础概念

| 英文              | 中文       | 含义                   |
| --------------- | -------- | -------------------- |
| Option          | 期权       | 一种赋予买方权利、卖方承担义务的金融合约 |
| Contract        | 合约       | 一份标准化期权合约，通常代表100股股票 |
| Underlying      | 标的资产     | 期权对应的股票或ETF          |
| Premium         | 权利金      | 买方支付、卖方收取的价格         |
| Strike Price    | 行权价      | 买卖股票的约定价格            |
| Expiration Date | 到期日      | 期权失效日期               |
| Exercise        | 行权       | 买方执行期权权利             |
| Assignment      | 指派       | 卖方被要求履行义务            |
| OCC             | 美国期权清算公司 | 负责美股期权清算与履约          |

---

## 二、四种最基础策略（必须掌握）

| 英文 | 中文 | 市场观点 | 风险特点 |
|------|------|---------|---------|
| Long Call | 买入看涨期权 | 看涨 | 最大亏损为权利金，收益理论无限 |
| Short Call | 卖出看涨期权 | 看不涨 | 收取权利金，裸卖风险理论无限 |
| Long Put | 买入看跌期权 | 看跌 | 最大亏损为权利金，股价越跌越有利 |
| Short Put | 卖出看跌期权 | 看不跌 | 收取权利金，可能被要求买入股票 |

**记忆口诀：**
- **Call = 看涨**
- **Put = 看跌**
- **Long = 买入期权（拥有权利）**
- **Short = 卖出期权（承担义务）**

---

## 三、交易动作（Order Actions）

| 英文 | 中文 | 作用 |
|------|------|------|
| Buy | 买入 | 交易动作 |
| Sell | 卖出 | 交易动作 |
| Open | 开仓 | 建立新仓位 |
| Close | 平仓 | 结束已有仓位 |

组合后形成四种常见指令：

| 英文 | 中文 |
|------|------|
| Buy to Open (BTO) | 买入开仓 |
| Sell to Close (STC) | 卖出平仓 |
| Sell to Open (STO) | 卖出开仓 |
| Buy to Close (BTC) | 买入平仓 |

---

## 四、Long / Short 的真正含义

> **Buy / Sell 描述的是交易动作；Long / Short 描述的是持仓状态。**

| 英文 | 中文 | 本质 |
|------|------|------|
| Long | 多头仓位 | 持有权利，通常受益于价格上涨（股票）或持有期权 |
| Short | 空头仓位 | 承担义务或借券卖出 |

**注意：**
- Sell 不一定是 Short（可能只是平仓）
- Buy 不一定是 Long（可能是回补空头）

详见：[[看涨期权与看跌期权]] 第 0 节

---

## 五、期权价值相关术语

| 英文 | 中文 | 说明 |
|------|------|------|
| Intrinsic Value | 内在价值 | 立即行权可获得的价值 |
| Extrinsic Value | 时间价值 | 超出内在价值的部分 |
| Time Value | 时间价值 | 距离到期越远通常越高 |
| In the Money (ITM) | 实值 | 已具有内在价值 |
| At the Money (ATM) | 平值 | 行权价接近当前股价 |
| Out of the Money (OTM) | 虚值 | 尚无内在价值 |

详见：[[内在价值与时间价值]]

---

## 六、期权 Greeks（希腊字母）

| 英文 | 中文 | 作用 |
|------|------|------|
| Delta | Delta（方向敏感度） | 股价每变动1美元，期权价格预计变化多少 |
| Gamma | Gamma | Delta 的变化速度 |
| Theta | Theta（时间衰减） | 每天因时间流逝导致的价值变化 |
| Vega | Vega | 对隐含波动率（IV）的敏感度 |
| Rho | Rho | 对利率变化的敏感度 |

**学习顺序建议：** Delta → Theta → Vega → Gamma → Rho

详见：[[Delta]]

---

## 七、波动率术语

| 英文 | 中文 | 说明 |
|------|------|------|
| IV (Implied Volatility) | 隐含波动率 | 市场预期未来波动程度 |
| HV (Historical Volatility) | 历史波动率 | 过去实际波动情况 |
| IV Rank | IV 排名 | 当前 IV 在历史区间中的位置 |
| IV Percentile | IV 百分位 | 当前 IV 高于历史多少比例 |

**经验：**
- IV 高：卖方通常更有优势（权利金较高）
- IV 低：买方成本相对较低

---

## 八、常见策略名称

| 英文 | 中文 |
|------|------|
| Covered Call | 备兑看涨 |
| Cash-Secured Put | 现金担保卖出 Put |
| Protective Put | 保护性 Put |
| Bull Call Spread | 牛市看涨价差 |
| Bear Put Spread | 熊市看跌价差 |
| Iron Condor | 铁鹰策略 |
| Iron Butterfly | 铁蝶策略 |
| Straddle | 跨式策略 |
| Strangle | 勒式策略 |
| Calendar Spread | 日历价差 |
| Diagonal Spread | 对角价差 |

---

## 九、风险管理术语

| 英文 | 中文 |
|------|------|
| Margin | 保证金 |
| Buying Power | 购买力 |
| Max Profit | 最大收益 |
| Max Loss | 最大亏损 |
| Break-even | 盈亏平衡点 |
| Probability of Profit (POP) | 盈利概率 |
| Risk/Reward Ratio | 风险收益比 |

---

## 十、专业交易员常用表达

| 英文 | 中文 |
|------|------|
| Open a Position | 建仓 |
| Close a Position | 平仓 |
| Roll the Option | 展期 |
| Expire Worthless | 到期归零 |
| Assigned | 被行权 |
| Exercised | 主动行权 |
| Deep ITM | 深度实值 |
| Deep OTM | 深度虚值 |
| Premium Collected | 收取权利金 |
| Premium Paid | 支付权利金 |

---

## 十一、核心思维模型

```
股票
│
├── Buy → Long Stock（持有股票）
└── Sell → Flat（平仓）或 Short Stock（做空）

期权
│
├── Long Call  = 买入看涨期权
├── Short Call = 卖出看涨期权
├── Long Put   = 买入看跌期权
└── Short Put  = 卖出看跌期权
```

---

## 十二、一句话口诀

```
Call = 看涨
Put = 看跌

Buy = 买入（动作）
Sell = 卖出（动作）

Long = 多头（持仓）
Short = 空头（持仓）

Open = 开仓
Close = 平仓

BTO = 买入开仓
STC = 卖出平仓
STO = 卖出开仓
BTC = 买入平仓
```

---

## 建议学习顺序

1. 四种基础期权（Long Call、Short Call、Long Put、Short Put）
2. ITM / ATM / OTM
3. Delta、Theta、IV
4. Covered Call、Cash-Secured Put
5. 各类价差策略（Spread）
6. 波动率交易与组合策略

---

**相关笔记：** [[什么是期权]], [[看涨期权与看跌期权]], [[期权合约要素]], [[内在价值与时间价值]]
**返回：** [[chapter00-期权学习]] | [[part00-主页]]
