---
type: concept
topic: "RabbitMQ 与 XXL-JOB：可靠投递、延迟唤醒与补偿调度"
status: reference
created: '2026-10-20'
updated: '2026-10-20'
tags: [technical-skills, distributed, rabbitmq, xxl-job]
related:
  - "[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]"
  - "[[part78-technical-skills/chapter03-distributed/section02-transactions-and-idempotency|事务与幂等]]"
---

# RabbitMQ 与 XXL-JOB：可靠投递、延迟唤醒与补偿调度

返回 [[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]

## 核心解释：可靠性是分段确认

生产者、Broker、消费者、业务数据库之间有不同提交边界。Publisher confirm 说明 Broker 对本次发布的处理结果，consumer ack 则告诉 Broker 某次投递已处理，两者彼此独立；confirm 不证明消费者业务成功，TCP 写入成功也不证明消息已经安全保存。确认本身丢失时，发送方无法确定结果，重发就可能产生重复。[RabbitMQ 确认官方文档](https://www.rabbitmq.com/docs/confirms)

一条实用链路是：业务与 Outbox 本地提交 → 发布器发送并处理 confirm/return → 消费者提交业务事务 → 手工 ack。无法路由的消息还要检查 `mandatory` 与 return 等机制；持久化消息、持久化队列、合适的副本策略要共同设计。不同队列类型的确认条件与故障保证不同，不能只说“开了持久化就不会丢”。

消费者先 ack 再提交，崩溃可能造成业务遗漏；先提交再 ack，崩溃可能造成重复投递。因此通常让业务提交在前，用事件 ID 唯一约束或业务状态机处理重复，见 [[part78-technical-skills/chapter03-distributed/section02-transactions-and-idempotency|事务与幂等]]。delivery tag 只在 channel 内标识投递，不能充当长期业务幂等键。瞬时失败可有限重试，永久性格式错误应进入隔离流程，避免反复 requeue 形成热循环。

## 延迟与死信要分清

死信是消息因过期、拒绝且不重新入队、超出队列限制等原因，被重新发布到死信交换机；DLX 不是天然延时器，也不是错误消息必达的保险箱。默认死信转发有可靠性边界，部分 quorum queue 配置支持至少一次死信转发，应按版本和队列策略核验。[DLX 官方文档](https://www.rabbitmq.com/docs/dlx)

| 方案 | 工作方式 | 使用边界 |
| --- | --- | --- |
| TTL + DLX | 在等待队列过期后转发到目标交换机 | 过期处理受队列行为影响，混合每条 TTL 可能出现队头等待；常用固定延迟档位拆队列 |
| 延迟消息插件（历史方案） | `x-delayed-message` 交换机按 `x-delay` 延后路由 | 需匹配兼容版本；官方项目已声明不再维护，存量系统须核验迁移与消息安全 |
| 定时任务扫描 | 从数据库找出已经到期且仍待处理的业务记录 | 增加扫描与索引负担，但适合持久化兜底和长时间跨度任务 |

TTL 表示保留时间，不保证到期毫秒级触发；延迟插件也不提供业务截止时间的精确承诺。长周期、高价值任务应以数据库到期时间与状态为准，让消息或任务只负责唤醒。核验时插件官方 README 已声明不再维护，并提示 RabbitMQ 4.4 起 quorum queues 提供原生 delivery delay；具体可用能力须按集群版本、协议与对应文档确认，不能沿用旧插件教程直接选型。[TTL 官方文档](https://www.rabbitmq.com/docs/ttl)、[插件官方仓库](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange)

## XXL-JOB：调度中心决定何时，执行器决定怎样处理

XXL-JOB 将任务配置、触发、执行器路由与执行分离。Cron 决定触发时间，分片广播把一次任务分配给多个执行器，各执行器读取分片序号和总数，再由业务代码划分数据，例如 `id % shardTotal == shardIndex`。分片不会自动替业务查询去重，也不自动解决数据倾斜。

Cron 语法不能在 Linux、Quartz、Spring 与 XXL-JOB 之间盲目复制。官方文档示例 `0 0/1 * * * ?` 含秒字段，表示每分钟触发；上线前用目标版本的表达式校验与“未来触发时间”预览，检查时区、夏令时及月末边界。错过触发的补偿策略、阻塞策略和失败重试也必须写清。[XXL-JOB 官方文档](https://github.com/xuxueli/xxl-job/blob/master/doc/XXL-JOB%E5%AE%98%E6%96%B9%E6%96%87%E6%A1%A3.md)

执行器扩缩容可能改变分片总数，长任务不宜只凭取模假定永不重叠或遗漏；可以结合固定批次快照、游标、条件抢占与唯一处理记录。超时后发出中断也不代表外部 HTTP 调用或数据库副作用已经撤销。应把“任务成功回调”和“每条业务记录确实处理成功”分别监控。

## 适用场景与常见误区

MQ 适合事件驱动与削峰，调度器适合周期性对账、到期扫描和补偿；两者可互为唤醒与兜底，但不天然构成 exactly-once。至少监控未确认消息、队列积压、重试/死信数量、最老任务延迟、补偿成功率，并为无法自动恢复的记录保留可审计的人工处理入口。

## 一句话面试回答

“应分别保证发布确认、消费提交和确认顺序，用幂等接受重复，以数据库状态判断延迟任务是否仍需执行，再用调度扫描与告警补齐消息或执行失败的恢复路径。”

## 实操练习与预期成果（未执行）

设计三组故障：发布成功后丢 confirm、业务提交后断开 ack、任务执行中扩容分片。再比较 TTL/DLX 与目标版本支持的延迟能力对不同延迟消息的投递顺序；若研究旧插件，只在兼容的隔离环境实验。预览跨月 Cron。应产出重复事件处理记录、延迟分布、分片覆盖表与恢复步骤；本文未部署 Broker、插件或 XXL-JOB，也未执行这些实验。

## 官方资料核验

核验日期：**2026-10-20**。RabbitMQ 在线文档当前页面标示 4.4；XXL-JOB 与插件链接指向会变化的主分支。实际实施时固定版本、队列类型、插件兼容关系与 Cron 校验结果。
