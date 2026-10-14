---
type: concept
topic: 数据源
aliases: [SEC EDGAR Data]
status: done
created: "2026-06-28"
updated: "2026-10-15"
tags: [数据分析, 数据源]
related: ""
---

# SEC EDGAR数据

## 找到原文与结构化数据

先确认CIK和提交文件。SEC官方提供提交历史与XBRL JSON，无需API密钥；结构化数据不能替代附注和分部披露。

```text
https://data.sec.gov/submissions/CIK##########.json
https://data.sec.gov/api/xbrl/companyfacts/CIK##########.json
```

CIK补齐10位。每个事实保留taxonomy/tag、unit、start/end、filed、form、accession与来源链接。按期间、单位和公开日期筛选，不能取数组最后一项就当最新可比数据；自定义标签和分部维度可能不在聚合API中。

批量访问遵守SEC当前公平访问规则，设置可识别User-Agent、缓存与退避；不要在异常时无限重试。使用时核对[SEC API官方文档](https://www.sec.gov/search-filings/edgar-application-programming-interfaces)（本轮核对2026-10-15）。

---
**返回：** [[chapter00-数据分析]] | [[part00-主页]]
