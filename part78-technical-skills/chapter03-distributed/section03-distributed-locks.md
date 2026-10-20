---
type: concept
topic: "Redisson 分布式锁：协调临界区并守住存储边界"
status: reference
created: '2026-10-20'
updated: '2026-10-20'
tags: [technical-skills, distributed, redis, redisson]
related:
  - "[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]"
  - "[[part78-technical-skills/chapter03-distributed/section02-transactions-and-idempotency|事务与幂等]]"
---

# Redisson 分布式锁：协调临界区并守住存储边界

返回 [[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]

## 核心解释

JVM 内的 `synchronized` 只协调同一进程中的线程。多个服务实例更新同一共享资源时，可以使用统一锁服务协调，但前提是所有相关路径都遵守相同协议与锁键。锁是一种协作约定，不能阻止绕过它的脚本、管理后台或其他服务直接写库。

Redisson 的 `RLock` 在 Redis/Valkey 中维护锁所有者与重入信息，并用原子操作处理竞争；当前持有者线程可以重复进入，释放次数要与成功进入次数对应。默认锁不承诺公平，公平锁按排队规则分配，但需要额外维护等待队列，会增加延迟，不能据此保证 HTTP 请求完成顺序。读写锁允许多个读者并发，写者排他；只有全部读写路径使用同一把读写锁，才能得到预期协调。[Redisson 锁官方文档](https://redisson.pro/docs/data-and-services/locks-and-synchronizers/)

## 租期、看门狗与旧持有者

显式 `leaseTime` 表示锁在指定时间后到期，业务不会因为锁到期就自动停止。未指定固定租期的常用获取方式可由看门狗续租；它降低“业务尚未结束而租期已到”的概率，但依赖客户端、调度线程、网络与服务端正常工作。默认超时、续租策略和具体重载行为要按所用版本核验。

一个重要反例：A 获锁后发生长时间 GC 停顿，续租失败，锁过期；B 获锁写入；A 恢复后仍执行旧代码，随后覆盖 B。**看门狗不能防止任意停顿后的旧持有者写入。**“写之前查一下自己是否持锁”也有检查到实际写入之间的竞争窗口。

可使用 fencing token：每次成功获取锁得到递增令牌，受保护存储记录已经接受的最大令牌，并原子拒绝更旧的令牌。只在客户端生成、却不由资源校验的令牌没有这个作用。实际项目也可结合数据库版本条件更新、状态条件和唯一约束；适用方案取决于能否修改受保护资源的写入协议。

## 场景与实现取舍

缓存热点重建、同一批次任务协调、低频配置更新适合考虑锁。订单“最多扣款一次”还需要业务流水与 [[part78-technical-skills/chapter03-distributed/section02-transactions-and-idempotency|幂等机制]]，余额非负还需要数据库条件更新。锁粒度优先对应业务资源，如 `stock:sku:42`；全局锁会压低并发，过细的锁则可能遗漏跨资源不变量。需要获取多把锁时，统一排序以降低死锁风险。

调用流程示意如下，具体 API 与默认策略需按版本确认：

```java
RLock lock = redisson.getLock("rebuild:product:42");
boolean acquired = lock.tryLock(2, TimeUnit.SECONDS);
if (!acquired) {
    return; // 示例：退出本次重建；真实接口要返回明确业务结果
}
try {
    // 再检查缓存，再执行重建；重要写入还需存储层约束
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

`finally` 降低异常路径漏释放的风险，不会让进程崩溃后的代码继续执行。锁超时、被中断、获取失败必须有业务处理；解锁检查也不是 fencing。异步或响应式任务可能跨线程，不能直接假定获取和释放拥有同一线程身份。

## AOP 封装与常见误区

AOP 锁注解可统一解析 SpEL 锁键、等待时间、租期和异常策略，但封装之后仍应能追踪实际 key、等待耗时与持有时间。锁切面与事务切面的顺序必须验证：通常需要让锁覆盖事务提交，否则锁已释放而数据尚未提交，后续请求仍可能读到旧状态。代理模式中的同类自调用会绕过切面，不能仅靠“方法上有注解”判断锁已生效。[Spring AOP 代理官方说明](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

读写锁不一定比普通锁更快，读少写多时队列与网络开销可能更大。公平锁不等于无饥饿的业务 SLA，Redis 高可用也不等于锁在所有故障条件下都提供绝对互斥。**版本敏感：**副本同步检查、锁类型、线程身份 API 和 fencing 支持应查已安装 Redisson 版本，不能直接照搬最新在线示例。

## 一句话面试回答

“Redisson 锁用于跨实例协调临界区，应选择业务粒度的 key、明确等待与租期、确保异常释放，并用 fencing 或数据库约束处理旧持有者与业务正确性，锁本身不能替代幂等。”

## 实操练习与预期成果（未执行）

启动两个隔离进程竞争相同 key，分别验证可重入、公平排队和读写互斥；再用可控暂停让 A 超过租期，观察 B 接手后 A 恢复的行为。加入数据库版本或 fencing 校验，比较旧写入是否被拒绝；另验证 AOP 自调用及事务提交顺序。

应留下时间线、锁等待/持有指标、旧写入的拒绝记录及代理调用图。本笔记未启动 Redis、Java 程序或故障实验。

## 官方资料核验

核验日期：**2026-10-20**。本文依据上述 Redisson 与 Spring 官方资料整理；练习中的可靠性结论需在实际部署拓扑和明确版本下验证。
