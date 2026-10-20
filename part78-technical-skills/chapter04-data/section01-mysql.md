---
type: concept
topic: MySQL索引与事务
aliases: [MySQL学习提纲, InnoDB索引与MVCC]
status: reference
created: "2026-10-20"
updated: "2026-10-20"
tags: [技术储备, 数据, MySQL, 索引, 事务]
related: "[[part78-technical-skills/chapter04-data/section02-redis|Redis结构与一致性]]"
---

# MySQL：从索引访问到事务可见性

> 学习参考，练习未执行。以 MySQL 8.4 / InnoDB 为原理基线；执行计划、诊断字段及优化策略随版本变化，先记录实际版本与配置。

## 核心解释

### B+树解决什么问题

InnoDB 常规索引可按 B+树理解：内部节点保存导航信息，叶子节点保存索引记录，较高扇出降低树高；有序叶子有利于范围扫描。它优化的是按页访问与排序组织，不能把一次查询固定说成“三次磁盘 IO”，因为树高、缓存命中、回表和结果数量都在变化。

聚簇索引叶子保存整行，通常按主键组织；二级索引记录包含索引列和主键，查其他列时可能再用主键找整行，即回表。若所需列已在索引中，可以覆盖查询。长主键也会扩大二级索引，因此主键设计影响不止主键本身。[聚簇与二级索引](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)、[索引物理结构](https://dev.mysql.com/doc/refman/8.4/en/innodb-physical-structure.html)。

联合索引 `(user_id, status, created_at)` 按这组键的字典序组织。固定前两列后按时间取最近订单，通常比只按时间过滤更容易缩小扫描范围；顺序应由查询条件、排序和数据分布决定。左前缀是理解访问范围的基础，但“遇到范围后所有后续列都失效”过于绝对，后续列仍可能参与过滤或覆盖，具体看计划与版本能力。[索引优化](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)。

### 锁控制冲突，MVCC控制读到哪个版本

行锁在 InnoDB 中落实到索引记录。记录锁锁住记录；间隙锁限制某个索引间隙内的插入；临键锁是记录锁加其前方间隙锁，可表示为 `(前一键, 当前键]`。锁定读与更新的扫描路径会影响锁范围，不能只看最终返回几行。[InnoDB锁](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)。

例如键为 10、20、30，范围锁可能覆盖没有实际行的区间。完整唯一键等值查询命中已有记录时通常只需记录锁；查不到记录、只使用联合唯一键部分列、范围扫描等不能直接套用这个结论。RC 通常不对搜索与扫描加间隙锁，但外键和重复键检查仍有例外。

MVCC 借助版本信息、undo 版本链与 Read View 判断事务能看见哪个版本。普通一致性读通常读快照，`SELECT ... FOR UPDATE`、更新等采用不同的锁定访问规则。RR 的常规一致性读使用首次一致性读建立的快照，RC 每次一致性读建立新快照；事务自身写入仍可被自己看见。不能把“开始事务的时间”直接等同于快照建立时刻，也不能把 MVCC 理解成所有读都不加锁。[一致性读](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)。

RU、RC、RR、Serializable 是隔离级别，不是四种独立存储引擎。隔离越严格通常要付出更多并发代价；RR 的快照读与范围锁分别处理不同路径，不应凭一句“解决幻读”推导业务不会竞争。余额扣减、库存扣减还需要条件更新、约束或明确的锁定事务。[隔离级别](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)。

## 适用场景与排查顺序

订单查询、分页、库存和账户更新都应先明确访问条件及事务边界。慢 SQL 排查依次收集：真实 SQL 与参数、耗时分位数、调用次数、扫描/返回行数、锁等待，再看 `EXPLAIN` 的访问路径、索引、估计行数和排序。`EXPLAIN ANALYZE` 会实际运行语句并给出行数、耗时与循环次数，只在可控环境分析允许执行的语句，不能当作纯静态预览。[EXPLAIN](https://dev.mysql.com/doc/refman/8.4/en/explain.html)。

优化要比较同一数据量和参数分布下的前后结果。低选择性条件、大结果集、深分页、隐式转换或对索引列运算都值得检查；“出现全表扫描”不自动代表错误，小表或读取大部分数据时可能更合算。新增索引会增加写入与存储成本，不能为每列都建索引。

## 常见误区

- 查了一个主键就一定没有等待：其他事务可能持有冲突锁，事务也可能等待元数据锁。
- `COUNT` 后再插入可以防重：两次操作之间存在竞争，业务唯一性应有数据库唯一约束。
- 死锁意味着数据库坏了：它是并发事务可能出现的结果，应缩短事务、统一访问顺序，并对可重试事务做有界重试。
- 换隔离级别就修好慢 SQL：先区分扫描、锁等待、连接池和磁盘等瓶颈。

## 一句话面试回答

InnoDB 用 B+树组织数据与索引，用 MVCC 决定一致性读的版本，用索引记录和范围锁协调写冲突；调优必须结合隔离级别、执行计划和真实访问分布。

## 实操练习（未执行）

在一次性练习数据库开两个客户端，先执行：

```sql
CREATE TABLE ij_mysql_lab (
  id INT PRIMARY KEY, user_id INT NOT NULL,
  status INT NOT NULL, amount INT NOT NULL,
  KEY idx_user_status (user_id, status)
) ENGINE=InnoDB;
INSERT INTO ij_mysql_lab VALUES (10,1,0,100),(20,1,0,200),(30,2,1,300);
```

1. 客户端 A 设置 RR，`START TRANSACTION` 后普通查询 `id=10`；B 更新该行并提交；A 再普通查询，随后执行锁定读。记录查询顺序、读到的值和等待情况，最后提交。再用 RC 重做，解释快照建立时点。
2. A 开事务执行 `SELECT * FROM ij_mysql_lab WHERE id BETWEEN 10 AND 20 FOR UPDATE`；B 执行 `INSERT INTO ij_mysql_lab VALUES (15,1,0,150)`。观察后结束 A 的事务，记录 B 的后续结果；再比较唯一键命中查询，不能预先把锁范围当作实测结果。
3. 对 `WHERE user_id=1 AND status=0` 分别选择索引内列与 `amount`，保存 `EXPLAIN`，解释覆盖与回表。另生成至少一万行不同分布数据，比较索引前后的扫描量与耗时，避免用三行数据证明性能。

应产出：双客户端时间线、版本与隔离级别、执行计划对照、一个失败或反例，以及仍未解释的差异；没有运行记录时保持“未执行”。

## 官方资料（核验：2026-10-20）

上文链接均已核验为 MySQL 官方手册。版本敏感处包括执行计划格式、优化器选择和锁诊断字段；8.4 基线不能直接替代其他版本的验证。

**返回：** [[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]
