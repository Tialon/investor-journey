---
type: index
topic: 技术技能储备
aliases: [后端与AI工程学习, Technical Skills]
status: reference
created: "2026-10-20"
updated: "2026-10-20"
tags: [技术储备, 索引, AI, Java, 分布式, 数据, 工程]
related: "[[part00-index/part00-主页]], [[part00-index/part03-目录]], [[part75-ai/chapter00-AI投资]]"
---

# 技术技能储备

> 用于个人学习与后续实践，按专业技能主题整理原理、场景、边界、面试简答和练习。这里的材料不构成个人项目经历或能力背书；所有练习当前均为**未执行**。资料核验日期为 **2026-10-20**，各页列出官方来源与版本敏感项。

本模块补充通用后端与AI工程基础。投资应用仍从[[part75-ai/chapter00-AI投资|AI投资]]、[[part70-python/chapter00-Python投资|Python投资]]和[[part80-data-analysis/chapter00-数据分析|数据分析]]进入；这里通过链接连接已有内容，不重复复制投资笔记。

## 先读什么

| 当前目标 | 建议顺序 | 应留下的学习证据 |
|---|---|---|
| 理解为什么用RAG、为何仍会答错 | AI 01 → 02 → 03 → 04 | 样本文档、检索标注、引用与冲突溯源记录 |
| 理解HTTP与异步阻塞 | Java 01 → 02 → 05，再读AI 05 | 调用/线程时间线、阻塞位置、池容量与超时表 |
| 搭建后端知识框架 | Java 01 → 03 → 04 → 分布式 01/02 → 数据 01/02 | 设计取舍、约束与失败场景清单 |
| 学习服务可靠性 | 分布式 02 → 03 → 04 → 工程 01 | 幂等键、状态机、补偿与重试流程 |
| 知识速记与复述 | 阅读核心解释，再用一句话概括并展开例子和边界 | 原理、取舍、反例各一个，不虚构工作经历 |

每篇使用“核心解释 → 场景 → 边界/误区 → 简短速记 → 练习与成果”的结构；面试简答仅作为知识速记附项。阅读后能复述与完成练习是不同阶段，需用实际产出更新学习状态。

## chapter01-ai · AI应用与RAG

1. [[part78-technical-skills/chapter01-ai/section01-rag-foundations|RAG的引入原因、场景与局限]] — 参数知识与外部证据；何时用检索、工具或微调。
2. [[part78-technical-skills/chapter01-ai/section02-document-ingestion|多源文档解析、OCR与父子块]] — Word/PPT/Excel阅读顺序、表格单位、结构与原文定位。
3. [[part78-technical-skills/chapter01-ai/section03-hybrid-retrieval|稠密/稀疏检索、融合与重排]] — BGE-M3、Milvus、WeightedRanker、BGE-Reranker的职责。
4. [[part78-technical-skills/chapter01-ai/section04-rag-troubleshooting|开发与生产答案矛盾的溯源]] — 文档、解析、索引、过滤、Prompt、模型与缓存证据链。
5. [[part78-technical-skills/chapter01-ai/section05-spring-ai-and-tools|Spring AI、SSE、结构化输出与工具/MCP]] — ChatClient、Advisors、记忆、结果记录与工具执行边界。
6. [[part78-technical-skills/chapter01-ai/section06-routing-evaluation-deployment|意图路由、微调、评估与模型部署]] — Prompt/Nacos、模板方法/EnumMap、LangChain/FastAPI、BERT/Hugging Face、RAGAS、DashScope/Ollama。

## chapter02-java · Java与Web后端

1. [[part78-technical-skills/chapter02-java/section01-java-and-design|Java基础与设计模式]] — 集合、反射、动态代理、泛型、Lambda，以及策略/工厂/模板方法。
2. [[part78-technical-skills/chapter02-java/section02-concurrency|线程池、锁与异步组合]] — synchronized/ReentrantLock、CAS/AQS、ThreadLocal、CompletableFuture。
3. [[part78-technical-skills/chapter02-java/section03-jmm-jvm-and-diagnostics|JMM、JVM、GC与故障诊断]] — 内存可见性与运行时区域分层解释，OOM/CPU排查。
4. [[part78-technical-skills/chapter02-java/section04-spring-backend|Spring后端与数据访问基础]] — IoC/AOP/注解/SpEL、自动配置/Starter、MVC REST、异常响应、MyBatis-Plus、Security/JWT/RBAC与多级缓存。
5. [[part78-technical-skills/chapter02-java/section05-http-and-async|HTTP流程、MVC与FastAPI并发模型]] — TCP/HTTP/SSE/WebSocket、同步helper、线程卸载和异步阻塞陷阱。

## chapter03-distributed · 分布式与可靠性

1. [[part78-technical-skills/chapter03-distributed/section01-service-governance|服务治理与一致性取舍]] — Nacos、Feign、Gateway、Sentinel、CAP/BASE。
2. [[part78-technical-skills/chapter03-distributed/section02-transactions-and-idempotency|事务、补偿与接口幂等]] — Seata AT、本地事务、延迟消息与外部副作用。
3. [[part78-technical-skills/chapter03-distributed/section03-distributed-locks|Redisson分布式锁与租约]] — 可重入、公平、读写锁、看门狗与AOP封装。
4. [[part78-technical-skills/chapter03-distributed/section04-rabbitmq-and-jobs|RabbitMQ可靠消息与XXL-JOB]] — 确认、幂等、延迟/死信、cron与分片。
5. [[part78-technical-skills/chapter03-distributed/section05-elasticsearch|Elasticsearch检索与排序]] — 倒排、复合查询、高亮和FunctionScore。

## chapter04-data · 数据存储与平台接入

1. [[part78-technical-skills/chapter04-data/section01-mysql|MySQL索引、锁与MVCC]] — B+树、行/间隙/临键锁、隔离级别、索引与慢SQL。
2. [[part78-technical-skills/chapter04-data/section02-redis|Redis结构、缓存与持久化]] — 五类基础结构、Bitmap/ZSet/Hash/Set业务选型、Lua、缓存故障与复制。
3. [[part78-technical-skills/chapter04-data/section03-iot-and-platforms|IoT、OSS、内容审核与微信集成]] — SDK/AMQP/物模型/快照、对象存储、登录与支付确认。

## chapter05-engineering · 工程交付与协作

1. [[part78-technical-skills/chapter05-engineering/section01-delivery-and-troubleshooting|Linux、Git、Maven、Jenkins与Docker]] — 可追溯交付、流水线、资源与依赖排查。
2. [[part78-technical-skills/chapter05-engineering/section02-api-and-client-collaboration|接口契约、若依与前端协作]] — OpenAPI/Swagger/Knife4j、Apifox/Postman、Vue3/Element Plus、HTML/CSS/JS与小程序。
3. [[part78-technical-skills/chapter05-engineering/section03-ai-assisted-development|Cursor/Codex与代码验证]] — 任务约束、上下文、diff、行为检查及如实报告。

## 必须能说清的边界

- RAG相似度反映匹配信号，不等于答案正确率；检索到过时或冲突证据仍可能答错。
- Word/PPT/Excel“抽出文字”不代表恢复了业务语义，单位、表头、阅读顺序和版本要可追溯。
- WeightedRanker融合多路分数，cross-encoder重排重新联合理解query与候选文本，两者不是同一过程。
- `async def`直接调用同步阻塞库会堵住事件循环；普通`def`路由可由框架线程池执行，普通helper不会自动卸载。Spring也有异步、WebFlux与虚拟线程方案。
- Seata AT处理受支持资源的事务协调，不能撤销已经完成的第三方支付；补偿是新的业务动作。
- 分布式锁不能替代幂等、数据库唯一约束和状态校验；SSE断开也不保证远端模型停止生成与计费。

## 练习记录方式

完成某篇练习后，用[[templates/学习笔记模板|学习笔记模板]]记录：实际环境和版本、输入与预期、执行步骤、原始输出、反例、结论与剩余问题。状态可区分“已阅读 / 已复述 / 已练习 / 已复现故障”，没有执行证据时保持“未执行”。不要把材料中的示例当作真实项目业绩。

本模块包含 **22篇主题笔记 + 1篇索引**；每篇提供官方来源，滚动文档应切换到实际依赖版本后再照写API。当前只完成文档整理及静态核查，没有启动服务、访问业务平台或运行教学练习。

**返回：** [[part00-index/part00-主页|主页]] | [[part00-index/part03-目录|完整目录]]
