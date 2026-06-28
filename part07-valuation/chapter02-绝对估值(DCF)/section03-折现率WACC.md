---
type: concept
topic: 绝对估值
aliases: [WACC, Weighted Average Cost of Capital, 加权平均资本成本]
status: done
created: "2026-06-28"
updated: "2026-06-28"
tags: [估值, DCF, 核心]
related: "[[DCF基础]], [[section04-终值计算]]"
---

# 折现率WACC


## 公式

1902WACC = \frac{E}{E+D} \times R_e + \frac{D}{E+D} \times R_d \times (1-t)1902

其中：
- E = 股权市值
- D = 有息负债
- R_e = 股权成本（CAPM）
- R_d = 债务成本
- t = 税率

## CAPM模型

1902R_e = R_f + \beta \times (R_m - R_f)1902

- R_f = 无风险利率（10年期国债）
- β = 股票相对市场的波动性
- R_m - R_f = 市场风险溢价

## WACC水平参考

| WACC | 评价 |
|------|------|
| 6-8% | 低风险（稳定公司） |
| 8-12% | 中等风险 |
| 12-15% | 高风险 |
| > 15% | 极高风险 |

## 思考题

1. WACC越高，估值越低，为什么？
2. 如何确定合理的β值？
3. 为什么债务成本要乘以(1-t)？

---

> **下一节**：[终值计算](终值计算.md)




---
**相关笔记：** [[DCF基础]], [[section04-终值计算]]
**返回：** [[chapter00-绝对估值(DCF)]] | [[chapter01-主页]]
