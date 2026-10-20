---
type: concept
topic: "JMM、JVM 与故障排查"
status: reference
created: '2026-10-20'
updated: '2026-10-20'
tags: [技术储备, Java, JMM, JVM, GC, 故障排查]
related:
  - '[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]'
  - '[[part78-technical-skills/chapter02-java/section02-concurrency|Java 并发与异步任务]]'
  - '[[part78-technical-skills/chapter02-java/section04-spring-backend|Spring 后端基础]]'
---

# JMM、JVM 与故障排查

> 学习目标：把语言的并发规则、虚拟机的运行时区域和具体实现的诊断工具分开，能沿着证据解释问题，而不是看到慢就调堆、看到 OOM 就判定内存泄漏。

## 一、核心概念：区分 JMM 与 JVM 运行时区域

| 概念 | 主要问题 | 典型关键词 |
| --- | --- | --- |
| Java Memory Model，JMM | 一个线程的读操作允许看到哪个写入；线程如何建立可见性和顺序关系 | 数据竞争、happens-before、volatile、锁、安全发布 |
| JVM 运行时数据区 | 程序执行使用哪些逻辑内存区域，它们属于线程还是共享 | PC、虚拟机栈、堆、方法区、运行时常量池 |
| HotSpot 实现与操作系统资源 | 某个 JVM 如何落实规范、如何申请和回收实际资源 | Metaspace、Code Cache、直接内存、RSS、GC 实现 |

JMM 不是“堆、栈、方法区的划分”，也不能简单等同“主内存与工作内存两张物理内存图”。JMM 是语言规范对并发执行允许行为的约束；运行时区域来自 JVM 规范，内存布局和许多优化由实现决定。[JLS 第 17 章](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html)、[JVMS 第 2 章](https://docs.oracle.com/javase/specs/jvms/se25/html/jvms-2.html#jvms-2.5)

## 二、JMM：原子性、可见性和顺序分别回答

原子性关注操作是否可被其他线程观察为不可分割步骤；可见性关注写入如何被其他线程可靠观察；顺序关注哪些执行结果被规范允许。单线程看似等价的编译优化，若没有线程间同步关系，可能产生与直觉不同的跨线程观测结果。

常用 happens-before 关系包括：同一线程中的程序顺序；某监视器解锁先于后续对同一监视器加锁；对 volatile 字段的写先于后续对该字段的读；调用线程的 `start` 先于被启动线程的动作；线程结束先于另一个线程成功 `join` 返回。关系具有传递性；它不是所有操作实际按墙上时钟串行执行的要求。[JLS 17.4.5](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html#jls-17.4.5)

```java
// 示意：发布一个初始化后不再修改的结果。
int answer;
volatile boolean ready;

// 写线程：
answer = 42;
ready = true;
// 读线程：
if (ready) {
    use(answer);
}
```

在没有其他写入的前提下，读线程读到这次发布的 `ready == true`，就能通过 volatile 建立的关系观察先前写入的 `answer`。若去掉 volatile 且没有其他同步，不能依赖这个发布过程。反过来，`volatile int count` 的 `count++` 仍由多个步骤组成，多个写线程可能丢更新。`sleep()` 也不能用来建立可见性保证。

适用场景包括单例安全发布、缓存快照替换、停止标记和并发状态机。优先使用不可变对象、安全发布和成熟并发容器；只有在能解释同步关系时才自行设计无锁协议。

## 三、运行时数据区：逻辑区域与具体实现

- 每线程的 PC 记录执行位置；虚拟机栈由方法调用的栈帧组成，包含局部变量、操作数栈等。规范也定义本地方法栈，但具体实现可以采用不同组织方式。
- 堆由线程共享，逻辑上供类实例和数组分配，通常由垃圾收集器管理。方法里的局部引用不等于对象本体在栈上；逃逸分析、标量替换等优化又使“每个对象一定有一块可观察的堆内存”不够严谨。
- 方法区是规范中的逻辑共享区域，保存类结构等信息，运行时常量池属于其中。HotSpot 的 Metaspace 是类元数据的一种实现方式，不能把二者当成所有 JVM、所有版本完全相同的物理区域；也不能把字符串常量、静态字段和全部类信息一概归入 Metaspace。[JVMS 2.5–2.6](https://docs.oracle.com/javase/specs/jvms/se25/html/jvms-2.html#jvms-2.5)

## 四、GC：回收不可达对象，围绕目标调优

GC 从根集合出发追踪可达对象，因此互相引用但整体不可达的对象仍可被回收；“有循环引用就泄漏”是误区。Java 中常见泄漏是对象已无业务用途，却仍被长生命周期容器、监听器或线程上下文引用。

评价 GC 需要看吞吐、停顿分布、分配速率、回收后存活量和容器内存上限。堆大可能降低回收频率，也可能增加某些阶段成本，并挤占本地内存。不同收集器的分代方式、并发阶段、停顿目标和支持平台存在差异；选择 G1、ZGC 等前先固定 JDK、工作负载与延迟目标，不能用“某收集器永不停顿”概括。[GC 调优入口](https://docs.oracle.com/en/java/javase/25/gctuning/introduction-garbage-collection-tuning.html)

## 五、OOM 与 CPU 问题的证据路径

OOM 先保留错误类型和现场：`Java heap space` 关注堆容量、存活对象和分配；Metaspace 关注类加载器与动态生成类；直接内存、无法创建本地线程等还涉及堆外空间、线程数、系统限制。`StackOverflowError` 通常从调用深度或递归排查，不应直接归为“堆不够”。内存不足既可能是泄漏，也可能是合法工作集过大或并发失控。[Oracle 内存故障说明](https://docs.oracle.com/en/java/javase/25/troubleshoot/troubleshooting-memory-leaks.html)

排查时先记录版本、启动参数、容器限制、请求变化和时间线；再比较多次 GC 后的存活量、类直方图或堆转储中的支配关系与到根路径。RSS 高于堆使用量是可能的，不能据此直接判定堆泄漏。堆转储可能包含业务数据，且大对象分析、转储等操作有资源成本，应先评估影响。

CPU 高时先确认是哪个进程、哪个线程、持续还是突发，再多次采集线程栈并结合 JFR 等采样看热点。Linux 上可用系统工具观察线程 CPU，再与栈中的本地线程标识对应；`RUNNABLE` 只表示线程状态，不能单凭它断言正在消耗 CPU。常见方向是热循环、序列化、正则、锁竞争自旋、GC 或重试风暴；CPU 不高但接口慢则还要看排队、数据库与网络等待。

以下是待用命令模板，`<pid>` 必须替换为已确认的目标进程；并非本次执行记录：

```text
jcmd <pid> help
jcmd <pid> VM.version
jcmd <pid> VM.flags
jcmd <pid> GC.heap_info
jcmd <pid> Thread.print -l
jcmd <pid> JFR.start name=investigation settings=profile duration=60s filename=investigation.jfr
```

命令可用性、参数与影响以目标 JVM 的 `help` 为准；虚拟线程的诊断方式也与版本相关。[Oracle 诊断工具](https://docs.oracle.com/en/java/javase/25/troubleshoot/diagnostic-tools.html)

## 六、一句话面试回答

> JMM 定义线程间读写的可见性与顺序规则，JVM 运行时区域描述执行所需的逻辑内存；排查 OOM 和 CPU 时，先确认故障类别与时间线，再用 GC、线程栈和采样证据定位原因。

## 七、实操练习与预期成果（未执行）

1. 分别设计 volatile 发布与自增实验，解释为什么一个解决可见性、另一个仍丢更新；一次没复现错误不能证明无数据竞争。
2. 在独立练习程序中构造持续持有对象与正常释放对象两组负载，观察 GC 后存活量，并从引用链解释差异；不要在业务应用中制造 OOM。
3. 在本地练习程序中构造一个可停止的 CPU 热循环和一个等待任务，比较系统 CPU、线程栈及采样结果。交付故障假设、证据、排除过程、修复建议与版本记录。
4. 本笔记未启动应用、未执行诊断命令、未取得转储文件；这些是后续学习任务，没有虚构故障复现或性能数据。

返回：[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]

资料核验日期：**2026-10-20**。规范与工具链接以 Java SE/JDK 25 为参考；HotSpot 区域实现、收集器、容器支持与诊断命令均须按实际运行版本核验。
