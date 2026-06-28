---
type: concept
topic: 三大报表
aliases: [FCF Calculation]
status: done
created: "2026-06-28"
updated: "2026-06-28"
tags: [财务报表, 会计基础]
related: "[[section10-经营现金流vs净利润]], [[自由现金流FCF]]"
---

# 自由现金流计算

## 两种FCF

### FCFF（公司自由现金流）

$$\text{FCFF} = \text{NOPAT} + \text{D\&A} - \text{CapEx} - \Delta\text{NWC}$$

其中：
- NOPAT = EBIT × (1 - 税率)
- D&A = 折旧摊销
- CapEx = 资本支出
- ΔNWC = 营运资本变动

### FCFE（股权自由现金流）

$$\text{FCFE} = \text{FCFF} - \text{利息} \times (1 - \text{税率}) + \text{净借款}$$

## 简化计算

实际分析中最常用：

$$\text{FCF} = \text{经营现金流（CFO）} - \text{资本支出（CapEx）}$$

## CapEx的两种类型

| 类型 | 定义 | 对FCF的影响 |
|------|------|------------|
| **维护性CapEx** | 维持现有产能 | 必须扣除 |
| **扩张性CapEx** | 扩大产能 | 可以选择性扣除 |

$$\text{维护性FCF} = \text{CFO} - \text{维护性CapEx}$$

$$\text{总FCF} = \text{CFO} - \text{总CapEx}$$

## 计算实例

### Apple 2024年

| 项目 | 金额 |
|------|------|
| 经营现金流（CFO） | $1,100亿 |
| 资本支出（CapEx） | $110亿 |
| **自由现金流** | **$990亿** |

### Amazon 2024年

| 项目 | 金额 |
|------|------|
| 经营现金流（CFO） | $1,000亿 |
| 资本支出（CapEx） | $750亿 |
| **自由现金流** | **$250亿** |

**解读**：Amazon CapEx极高（建数据中心、仓库），FCF相对较低。

## FCF的衍生指标

### FCF Yield

$$\text{FCF Yield} = \frac{\text{FCF}}{\text{市值}} \times 100\%$$

| 水平 | 评价 |
|------|------|
| > 8% | 高（可能被低估） |
| 5-8% | 合理 |
| < 3% | 低（可能被高估） |

### FCF Margin

$$\text{FCF Margin} = \frac{\text{FCF}}{\text{收入}} \times 100\%$$

### FCF vs 净利润

$$\text{FCF / 净利润} > 1 \text{（优秀）}$$

## 不同类型公司的FCF特征

| 公司类型 | FCF特征 |
|----------|---------|
| 成熟科技 | 高FCF，低CapEx |
| 成长科技 | 低FCF，高CapEx |
| 重资产 | FCF受限于高CapEx |
| 轻资产 | 高FCF，低CapEx |

## 常见错误

| 错误 | 正确做法 |
|------|----------|
| 用净利润代替FCF | 用CFO - CapEx |
| 忽略CapEx类型 | 区分维护性和扩张性 |
| 不考虑营运资本 | 关注应收/存货/应付变动 |

## 思考题

1. FCFF和FCFE有什么区别？分别用于什么估值？
2. 为什么Amazon的FCF比Apple低很多？
3. 一家公司FCF很高但不分红不回购，钱去哪了？

---

> **下一节**：[现金流陷阱](现金流陷阱.md)




---
**相关笔记：** [[section10-经营现金流vs净利润]], [[自由现金流FCF]]
**返回：** [[chapter00-三大报表]] | [[chapter01-主页]]
