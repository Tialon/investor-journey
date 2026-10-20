---
type: concept
topic: "意图路由、效果评估与模型部署"
status: reference
created: '2026-10-20'
updated: '2026-10-20'
tags: [技术技能储备, AI, 意图分类, Prompt, RAGAS, 模型部署]
related:
  - "[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]"
  - "[[part78-technical-skills/chapter01-ai/section05-spring-ai-and-tools|Spring AI、流式响应与工具调用]]"
---

# 意图路由、效果评估与模型部署

本篇把模型接入之后的三个问题串起来：请求交给谁，怎样证明改动有效，怎样稳定发布。资料核验日期：**2026-10-20**。Nacos、LangChain、Transformers、RAGAS 和模型平台文档会更新，类名、配置项、能力与配额应以项目锁定版本及账号所在区域为准。

## 1. 意图路由：先决定流程，再组织回答

路由将请求送入 FAQ、知识检索、业务查询或澄清等流程，可用规则、小型分类模型或大模型结构化输出实现。规则适合确定信号，分类模型适合稳定标签与有标注数据的领域，大模型适合冷启动和复杂语义；选择应比较误路由代价、延迟和维护成本。对未知意图、多意图和低置信度输入，应保留澄清或兜底，不能强制所有句子落入已有分类。

模板方法可固定“检查输入—准备上下文—执行—校验—记录”骨架，让具体流程实现差异步骤；策略负责可替换行为，工厂负责选择策略。`EnumMap<Intent, Handler>` 适合有限枚举路由，启动时检查漏注册和重复注册，再安全发布只读映射。EnumMap 本身不保证并发修改安全，枚举新增也不等于策略自动实现。[Java EnumMap 文档](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/EnumMap.html)

例如“能退吗，顺便查物流”包含两项任务，可以拆分、澄清或按明确业务优先级处理。分类概率不天然等于真实正确率，阈值需用验证集校准；路由通过也不代表工具操作已经获得授权。

**一句面试回答：**意图路由将语义判断映射到可控制的流程，策略实现差异、模板方法固定步骤，未知和低置信度输入保留澄清出口。

## 2. Prompt 与 Nacos：热更新也需要版本边界

提示词宜拆分角色约束、任务模板、变量和输出契约，检索文本与用户输入明确作为数据。用 Nacos 配置监听可在应用运行中获知内容变化，但回调收到新文本，并不代表模板、缓存和所有实例已经一致生效。[Nacos 配置发布与监听](https://nacos.io/en/docs/next/manual/user/config/publish-query-listen/)

一种发布设计是：监听后先解析变量、校验长度与输出 schema，再构造不可变配置快照，原子替换；在途请求继续使用进入时的版本。记录 `promptVersion`、摘要及生效时间，配套灰度、回滚和离线样例检查。坏配置应保留上一版并报警，避免把半份新模板与旧变量拼接。Nacos 的 `next` 文档代表预览分支，接入时应切换至实际服务端与 SDK 版本，不能照抄其中接口。

**一句面试回答：**Prompt 热更新是有校验、版本和回滚的配置发布；每次请求固定配置快照，才能解释同一问题为何前后回答不同。

## 3. LangChain、FastAPI 与 BERT 各解决什么问题

LangChain 提供模型、工具、提示词与代理编排抽象；FastAPI 提供 HTTP 路由、参数校验和接口服务，两者并不互相替代。可将分类、检索或模型能力封装成 Python 服务，由 Java 服务按契约调用，也可在一种语言内完成，是否拆服务取决于团队和运行成本。[LangChain 官方概览](https://docs.langchain.com/oss/python/langchain/overview)

并发模型必须沿调用链检查：FastAPI 的 `async def` 直接调用同步阻塞 SDK 会阻塞事件循环；普通 `def` 路由和同步依赖由框架线程池执行，但业务自己调用的普通 helper **不会自动卸载**。应选异步库、显式卸载阻塞 I/O，或把重计算放入适当工作进程，并设置并发和超时上限。[FastAPI async 文档](https://fastapi.tiangolo.com/async/)

BERT 中文意图分类是在适合中文的预训练编码器上添加分类头，用标注句子优化任务参数，并非训练一个生成聊天模型。Hugging Face 的常见流程是：建立 `label2id/id2label`，划分训练/验证/测试集，用对应 tokenizer 截断编码，加载 `AutoModelForSequenceClassification`，配置 Trainer、训练和评估，再保存模型及 tokenizer。[BERT 原理与任务头](https://huggingface.co/docs/transformers/en/model_doc/bert)、[文本分类流程](https://huggingface.co/docs/transformers/en/tasks/sequence_classification)

领域实践还应去重，并按用户、来源或时间分组，防止近似句泄漏到测试集；观察 macro-F1、混淆矩阵与未知意图拒识。类不均衡、标签边界含糊、业务新增意图会限制效果，不能只报总体准确率。

**一句面试回答：**LangChain 编排 AI 调用，FastAPI 暴露服务，BERT 处理有监督分类；模型效果和 HTTP 并发分别验证，不能由框架名推断。

## 4. RAGAS 四类指标：定位问题所在层

评估集至少保存问题、实际回答、检索上下文，以及适用指标需要的参考答案或参考证据。四类指标关注点不同：

| 指标 | 要回答的问题 | 边界 |
|---|---|---|
| [Faithfulness 忠实度](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) | 回答中的主张是否由检索上下文支持？ | 忠于错误资料仍可能事实错误。 |
| [Response / Answer Relevancy 答案相关性](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/answer_relevance/) | 回答是否回应用户问题？ | 相关不代表正确或证据充分。 |
| [Context Precision 上下文精确度](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/) | 取回的证据是否相关，相关证据是否靠前？ | 有参考/无参考及 ID 等变体，须记录具体实现。 |
| [Context Recall 上下文召回率](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) | 参考答案或参考证据中的必要信息找全了吗？ | 需有参考；不能由少量取回内容推断没有漏检。 |

若召回低，先查资料覆盖、解析、切块和召回配置；证据齐全而忠实度低，再查提示词、上下文使用和生成。这个定位是排查假设，需结合失败样本人工核验。相似度分数不是答案正确率，四项指标也不能随意平均成唯一质量结论。LLM 评委有偏差和费用，应固定评委版本、参数、样本及代码版本，抽样人工复核。**版本敏感：**RAGAS 文档同时存在新旧指标 API，应按安装版本核对类名和必需字段。

**一句面试回答：**四类指标分别检查证据支持、回答相关、检索噪声与证据遗漏，结合人工样本才能判断问题在检索还是生成。

## 5. DashScope、Ollama 与模型切换

DashScope / 百炼提供云端模型接口；Ollama 可用于运行本地模型，接入时仍需明确实际使用的是本机还是云端。兼容接口降低迁移成本，但工具调用、结构化输出、上下文上限和流事件不一定相同，应维护能力表并回归验证。[百炼兼容接口](https://help.aliyun.com/zh/model-studio/compatibility-of-openai-with-dashscope)、[Ollama 模型导入](https://docs.ollama.com/import)

私有部署需要评估显存、量化精度、并发、冷启动、监控和许可证；“本地运行”不自动证明整个链路无外发，仍要核对日志、Embedding 和其他外部服务。Embedding 切换尤其不能只改模型名称：维度相同也不表示向量空间兼容。通常应使用新模型重建文档向量、核对索引度量与归一化，查询侧同步切换，并通过新索引灰度和回滚。[Ollama Embeddings](https://docs.ollama.com/capabilities/embeddings)、[百炼 Embedding](https://help.aliyun.com/zh/model-studio/embedding)

**一句面试回答：**切换模型要回归能力、质量、延迟和成本；Embedding 还涉及向量空间兼容，通常需要重建索引而非替换配置。

## 6. 实操练习与验收成果

以下练习**尚未执行**：

1. 构造 FAQ、检索、查询、未知四类脱敏问题，包含多意图和边界样例。验收：路由表、混淆矩阵、拒识阈值说明，以及缺失策略时的启动校验结果。
2. 用模拟配置事件发布正确、缺变量和回滚三版 Prompt。验收：保存版本生效表，证明无效配置不替换有效快照，在途请求可关联原版本。
3. 准备一组固定 RAG 问答，人工标出缺证据、答非所问和无据扩写，再对比两种检索或模型配置。验收：逐例四项评估、人工差异说明及成本记录；不以单次均值宣称改进。若练习 BERT 微调，另交付数据划分清单、模型卡及独立测试报告。

返回：[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]
