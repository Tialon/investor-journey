---
type: concept
topic: "Java 并发与异步任务"
status: reference
created: '2026-10-20'
updated: '2026-10-20'
tags: [技术储备, Java, 并发, 线程池, CompletableFuture]
related:
  - '[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]'
  - '[[part78-technical-skills/chapter02-java/section03-jmm-jvm-and-diagnostics|JMM、JVM 与故障排查]]'
  - '[[part78-technical-skills/chapter02-java/section05-http-and-async|HTTP 与异步并发模型]]'
---

# Java 并发与异步任务

> 学习目标：能区分任务调度、共享状态保护、线程上下文和异步结果编排，并说明过载、超时与失败时会发生什么。

## 一、线程池：管理有限执行资源

线程池复用工作线程，并用队列缓冲任务。`ThreadPoolExecutor` 的典型接收路径是：不足核心线程数时尝试建线程；否则尝试入队；队列放不下时再尝试增加到最大线程数；仍无法接收则调用拒绝策略。无界队列通常让任务积压，而不是持续扩到 `maximumPoolSize`。因此“把最大线程数设大”不一定改变实际并发度。[ThreadPoolExecutor API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)

场景是批处理、并行调用和后台任务，但池大小须同时受 CPU、数据库连接数、下游并发配额和可接受延迟限制。CPU 密集任务增加线程可能主要增加切换成本；阻塞 I/O 可以需要更多并发，但应由压测和下游容量确定。队列应有容量、等待时限与拒绝后的业务语义，观察活跃线程、排队长度、拒绝量及端到端耗时。

拒绝策略也有代价：`CallerRunsPolicy` 会占用提交者线程，能产生背压，却可能拖慢 HTTP 接收线程或阻塞事件循环；静默丢弃不适合必须完成的订单任务。线程池内的任务等待同一饱和池里的子任务，还可能造成饥饿。异步化不能替代持久化消息与任务状态机。

## 二、锁、CAS 与 AQS：保护哪一个不变量

`synchronized` 使用对象监视器，支持可重入，退出代码块时自动释放锁；同一监视器的释放与后续获取建立可见性关系。`ReentrantLock` 提供显式加解锁，以及可中断等待、限时 `tryLock`、多个 `Condition` 等能力。它必须在成功获锁后用 `finally` 解锁。公平锁通常减少插队，但可能损失吞吐，也不承诺线程获得 CPU 的严格顺序；无参 `tryLock()` 还有其自身的公平性规则。[ReentrantLock API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)、[JLS 线程与锁](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html)

CAS 比较当前位置是否仍为预期值，匹配才原子更新，适合计数、状态迁移等较小范围的更新。失败可以重试，但高竞争时会消耗 CPU；ABA 问题表示值经历 A→B→A 后看似没变，需要按业务选择版本号等方案。一个 CAS 操作不能自动保证多个变量组成的不变量，也不能代替数据库事务。

AQS 是构建同步器的框架，以同步状态、获取/释放协议和等待队列协调线程，子类定义状态含义；`ReentrantLock`、信号量等可基于它实现不同语义。它会结合原子状态修改与线程挂起/唤醒，不是“所有线程一直自旋”，也不是“一套代码等同所有锁”。[AbstractQueuedSynchronizer API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/AbstractQueuedSynchronizer.html)

选择锁时先描述不变量，例如“可售数量不能为负、扣减与状态迁移必须一起成立”，再确定保护范围。避免持锁调用慢网络，也要固定多锁获取顺序。进程内的锁只协调当前进程，不能保护多个服务实例。

## 三、ThreadLocal：线程私有不等于请求私有

`ThreadLocal` 把值与当前线程关联，适合临时保存请求上下文；它不提供共享数据同步。线程池会复用线程，如果请求结束时不清理，下一个任务可能读到旧租户、用户或追踪信息，长期存活线程也可能保留不再需要的对象。[ThreadLocal API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/ThreadLocal.html)

```java
requestContext.set(context);
try {
    handleRequest();
} finally {
    requestContext.remove();
}
```

线程切换不会自动传递普通 `ThreadLocal`。`InheritableThreadLocal` 的继承发生在线程创建时，也不能普遍解决线程池任务的上下文传播。显式传参最直观；确需任务包装时，应复制必要上下文，并在执行结束后清理或恢复原值。不要把带权限含义的上下文默认为跨线程可靠。

## 四、CompletableFuture：编排结果，不消除阻塞

`thenApply` 做结果转换，`thenCompose` 串联一个返回 Future 的后续步骤，`thenCombine` 合并两个独立结果；异常可以用 `exceptionally`、`handle`、`whenComplete` 按不同目的处理。非 Async 阶段可能由完成前一阶段的线程执行；无显式执行器的 Async 阶段通常使用 `ForkJoinPool.commonPool()`，并非总会新建专属线程。[CompletableFuture API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CompletableFuture.html)

```java
// 示例片段：ioExecutor 应是按下游容量设置的受控执行器。
CompletableFuture<Price> price =
    CompletableFuture.supplyAsync(() -> priceClient.fetch(id), ioExecutor);
CompletableFuture<Stock> stock =
    CompletableFuture.supplyAsync(() -> stockClient.fetch(id), ioExecutor);
CompletableFuture<View> result = price.thenCombine(stock, View::new);
```

若 `fetch` 是同步 HTTP 调用，它仍会阻塞工作线程。把大量阻塞 I/O 塞入公共池会与其他使用者争用资源；显式隔离执行器也仍需要连接超时、读取超时与并发上限。请求线程立刻 `join()` 依然在等待；`allOf` 也不直接收集各结果。Future 的超时或取消不等于底层 HTTP 请求、数据库语句已经停止，必须理解客户端的取消语义并观察残留任务。

## 五、边界与常见误区

- `volatile count++` 仍是读、改、写组合，不能代替原子计数或锁；可见性与原子性分开讨论。
- `CompletableFuture` 不是事务传播工具，新线程不会自动继承当前线程绑定的数据库事务。
- 把异常转成默认值要有业务依据，否则“全部成功”的聚合结果可能掩盖下游故障。
- JDK 虚拟线程改变阻塞任务的线程成本，不增加数据库连接或模型接口的容量；其行为与支持情况需按 JDK 版本核对。

## 六、一句话面试回答

> 使用线程池限制并发和排队，用锁或原子操作保护明确的不变量，及时清理 ThreadLocal，再用 CompletableFuture 编排结果；异步调用仍要设计执行器、超时、失败和下游容量。

## 七、实操练习与预期成果（未执行）

1. 建一个小容量线程池，以快慢任务混合提交，记录提交、开始、完成与拒绝时刻，比较有界队列和无界队列的差异。
2. 用普通自增、原子计数和锁保护计数做正确性实验，再扩展为“库存与状态”两个字段，解释为何单个原子变量不足以保护组合规则。
3. 用单工作线程复用请求上下文，复现漏清理造成的串值，再加入 `finally remove()`；另验证跨线程不能直接读取上下文。
4. 编排两个模拟阻塞下游，比较公共池与独立池；加入一个失败和一个超时，说明调用方完成后底层任务是否继续。交付日志、时序图和参数理由，不预设吞吐数字。本笔记未运行这些练习。

返回：[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]

资料核验日期：**2026-10-20**。链接采用 JDK 25；示例只用于解释结构。虚拟线程、Future 新增方法和运行时调度细节具有版本敏感性，实施前按实际 JDK 文档核验。
