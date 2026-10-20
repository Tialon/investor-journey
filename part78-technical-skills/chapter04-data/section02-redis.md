---
type: concept
topic: Redis结构与一致性
aliases: [Redis学习提纲, Redis缓存与合并写]
status: reference
created: "2026-10-20"
updated: "2026-10-20"
tags: [技术储备, 数据, Redis, 缓存, 一致性]
related: "[[part78-technical-skills/chapter04-data/section01-mysql|MySQL索引与事务]]"
---

# Redis：结构选择、缓存故障与数据边界

> 学习参考，练习未执行。命令能力、内部编码、持久化文件组织均有版本差异；“五类结构”是基础学习范围，不表示 Redis 只有五种类型。

## 核心解释与应用场景

| 基础类型 | 原理与常用场景 | 要先确定的边界 |
|---|---|---|
| String | 字节串，可存序列化对象、计数器；整数增量可用原生命令 | 整对象覆盖易产生并发丢更新，值过大会放大传输成本 |
| Hash | 一个键内保存字段和值，适合对象局部字段更新 | 字段更新与数据库提交不是同一个事务 |
| List | 有序序列，可从两端操作，适合简单待处理列表 | 简单弹出不自动提供确认、重投和消费幂等 |
| Set | 不重复成员集合，适合标签、集合运算和短期防重 | 容量、过期和故障丢数据决定防重范围 |
| ZSet | 成员唯一且带分值，按分值组织次序，适合排行榜 | 同分排序规则、分值精度与业务排名定义要一致 |

类型决定可用的原子操作，不能只凭“都在内存”选择。内部可能采用不同紧凑编码或通用结构，阈值随版本和配置变化。[数据类型](https://redis.io/docs/latest/develop/data-types/)。

### 从业务映射到命令

签到可以用 Bitmap：以用户和月份组成键，以日号减一作为 bit 偏移，`SETBIT` 标记、`GETBIT` 查询、`BITCOUNT` 统计。Bitmap 基于 String，适合稠密且上界可控的编号；直接用很大的稀疏用户 ID 当偏移会浪费空间。连续签到还要定义时区、跨月和补签规则。[SETBIT](https://redis.io/docs/latest/commands/setbit/)。

排行可用 `ZADD` 设置积分、`ZINCRBY` 增量更新，再按分值取区间。成员防重不等于重复积分事件防重：同一“加10分”重放两次仍会多加，需事件标识与幂等逻辑。金额不宜直接依赖浮点分值作精确账本。[ZADD](https://redis.io/docs/latest/commands/zadd/)。

Set 的 `SADD` 返回新增数量，可判断本次是否首次加入。它适合有限时间内的消息去重，但键过期后同一事件会再次被接受；关键订单仍需数据库唯一约束与状态机。[Set](https://redis.io/docs/latest/develop/data-types/sets/)。

### Hash“合并写”必须说明一致性

一次 `HSET key f1 v1 f2 v2` 可以原子设置多个字段；客户端先 `HGET` 再计算再写回则可能覆盖别人的更新，计数优先用 `HINCRBY`，条件更新可用版本号与 Lua。Pipeline 主要减少网络往返，不把多命令变成一个原子业务操作。[HSET](https://redis.io/docs/latest/commands/hset/)。

若“合并写”指把多次修改先放 Hash，再定时批量写 MySQL，这是 write-behind 设计，需要另定：允许丢多少数据、累计增量还是最终值、如何防重复刷入、怎样处理刷入期间的新写入。可用带批次号的不可变增量与数据库去重记录；数据库提交成功后再确认该批，不能简单“读 Hash → 写库 → 删 Hash”，否则新写入会被删除。Redis 原子性不涵盖这次数据库事务；这是设计练习方案，尚未运行验证。

Lua 使一段读判断写在执行期间不被其他命令穿插，但长脚本会阻塞服务，也不提供 SQL 式错误回滚。先校验参数和键类型，再写入；Cluster 中多键脚本通常要落同一哈希槽，所有键应显式传入。还要处理重启或切换后的 `NOSCRIPT`，不能假设脚本缓存永久存在。[Lua脚本](https://redis.io/docs/latest/develop/programmability/eval-intro/)。

## 缓存故障与边界

- **穿透**：大量请求不存在的数据。入口校验、短期空值缓存或布隆过滤可减少回源；空值过期及数据新建时的失效策略必须配套，布隆过滤“可能存在”也不证明真实存在。
- **击穿**：热点键失效后并发回源。可用请求合并、有界互斥重建或允许短期旧值；必须限定等待、重试和旧值最大年龄。
- **雪崩**：大量键同时过期或缓存集群不可用。过期时间加抖动只能缓解集中失效，还需数据库限流、降级及容量预算。

以上是工程故障分类，实际请求可能同时触发多种情况。缓存与数据库双写没有天然原子性；删除缓存、异步失效或版本校验都要分析竞争窗口，不能承诺“绝无不一致”。

## 持久化与复制

RDB 保存时间点快照，便于备份与恢复，但可能丢快照之后的写；AOF 记录写操作，可靠性取决于 fsync 策略及故障条件，`everysec` 通常有约一秒的数据损失窗口。AOF 重写压缩的是恢复状态所需操作，不能当永久审计日志。Redis 7 起的多段 AOF 与旧版单文件不同，恢复要按实际版本操作。[持久化](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)。

主从默认异步复制，副本读可能落后，切换也可能丢已确认写。`WAIT` 增加副本确认，不把 Redis 变成强一致数据库。复制会传播误删除，不能替代独立备份。[复制](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)。

## 一句话面试回答

Redis 选型要把业务操作映射为结构和原子命令，并分别设计缓存失效、持久化、复制与跨数据库一致性，不能把“单线程”当作所有操作都安全的理由。

## 实操练习（未执行）

在隔离的本地练习实例，用 `redis-cli` 执行以下命令；键前缀仅用于本练习：

```text
SETBIT ij:lab:sign:u1:202610 19 1
SETBIT ij:lab:sign:u1:202610 19 1
BITCOUNT ij:lab:sign:u1:202610
ZADD ij:lab:rank 100 u1 100 u2 90 u3
ZRANGE ij:lab:rank 0 -1 REV WITHSCORES
SADD ij:lab:seen event-001
SADD ij:lab:seen event-001
HSET ij:lab:counter count 0 version 0
```

1. 记录重复签到、防重和同分排序结果，并解释排行与事件防重的差别。旧版本若不支持 `ZRANGE ... REV`，查对应版本命令后调整。
2. 两客户端分别读取 Hash 的 `count`，各自加一再写回，观察丢更新；换成 `HINCRBY` 重做。再画出一次 write-behind 在“库已提交、确认前进程退出”时的恢复流程。
3. 模拟热点过期，给回源函数加计数，比较直接回源与请求合并的回源次数、等待时间；另写 RDB/AOF/复制的故障损失表，不把未做的断电演练写成结论。

应产出：命令与实际返回、并发时间线、合并写失败恢复图、容量和可接受旧值时限；仅整理笔记不代表完成演练。

## 官方资料（核验：2026-10-20）

上文链接均为已核验 Redis 官方文档。文档 `latest` 会更新，学习记录应另保存服务端版本、客户端版本与相关配置。

**返回：** [[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]
