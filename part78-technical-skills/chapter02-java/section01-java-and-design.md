---
type: concept
topic: "Java 基础与可扩展设计"
status: reference
created: '2026-10-20'
updated: '2026-10-20'
tags: [技术储备, Java, 集合, 泛型, 设计模式]
related:
  - '[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]'
  - '[[part78-technical-skills/chapter02-java/section02-concurrency|Java 并发与异步任务]]'
  - '[[part78-technical-skills/chapter02-java/section04-spring-backend|Spring 后端基础]]'
---

# Java 基础与可扩展设计

> 学习目的：能说明数据结构与抽象方式的选择依据，并能解释框架如何在运行时发现、包装和调用对象。以下是学习资料与待做练习，不表示已有项目经历或已验证的能力。

## 一、集合核心原理：先看约束，再看复杂度

集合选型先问：是否有序、是否允许重复、需要按下标还是按键访问、是否有并发写入。`List` 维护序列，`Set` 维护唯一性，`Map` 维护键值关系；接口约定与具体实现的代价要分开理解。

| 需求 | 常见选择 | 必须一起说明的代价 |
| --- | --- | --- |
| 连续遍历、按下标读取 | `ArrayList` | 可变数组；尾部追加是均摊 O(1)，中部插入或删除通常移动元素 |
| 按键查询 | `HashMap` | 哈希分布良好时基本操作平均 O(1)；无顺序保证，扩容与碰撞影响代价 |
| 有序范围查找 | `TreeMap` | 比较排序，主要查改操作 O(log n)；比较规则须与业务一致 |
| 多线程共享映射 | `ConcurrentHashMap` | 支持并发访问；单次操作安全不代表多个操作组成的业务流程原子化 |

`ArrayList` 与 `HashMap` 都不是并发容器。fail-fast 迭代器只是尽力发现错误，不能作为同步方案。键放入哈希容器后，参与 `equals/hashCode` 的字段应保持稳定；否则可能出现“对象还在集合里，却查找不到”。需要顺序时应明确使用支持该顺序的实现，而不是依赖当前观察到的 `HashMap` 遍历结果。[ArrayList API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/ArrayList.html)、[HashMap API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/HashMap.html)、[TreeMap API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/TreeMap.html)、[ConcurrentHashMap API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

例如，“不存在才登记”应优先考虑 `putIfAbsent` 或 `computeIfAbsent`，而不是先 `containsKey` 再 `put`。但映射值若是一个可变 `ArrayList`，容器不会自动保护这个列表内部的并发修改；数据库唯一约束也不能由一个进程内的 Map 替代。

## 二、泛型与 Lambda：把约束交给类型系统

泛型让编译器检查类型关系，减少强制转换。`List<Integer>` 不是 `List<Number>` 的子类型，否则向后者加入 `Double` 会破坏前者的约束。方法只读取一组数字，可接收 `List<? extends Number>`；向目标集合写入整数，可考虑 `List<? super Integer>`。PECS 是“生产者 extends、消费者 super”的记忆法，仍要按具体读写需求设计接口。[通配符说明](https://docs.oracle.com/javase/tutorial/java/generics/wildcards.html)

Java 泛型主要通过类型擦除实现：编译器实施类型检查，并按需要插入转换或桥接方法，不是为每种参数生成独立类。不能直接写 `new T()`；也不能从任意 `Object` 引用用 `instanceof List<String>` 验证元素类型，通常先用 `instanceof List<?>`，再按业务检查元素。**版本敏感：**JDK 16 起允许部分无需未检查转换的参数化类型检查，例如从 `List<Integer>` 判断 `instanceof ArrayList<Integer>`，这不等于恢复运行时元素类型信息。部分声明的泛型签名仍能通过反射读取，不能笼统说“运行时所有泛型信息都消失”。[类型擦除](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html)、[JLS 15.20.2](https://docs.oracle.com/javase/specs/jls/se25/html/jls-15.html#jls-15.20.2)

Lambda 表达的是函数式接口的行为，例如 `Predicate<Order>` 表示筛选条件。捕获的局部变量须为 `final` 或事实上不再赋值；被捕获的对象本身仍可能可变，因而不自动具备线程安全。Lambda 也不意味着异步执行：`list.forEach(x -> save(x))` 仍按调用方式执行，是否并发取决于调用它的 API 和执行器。[JLS 15.27](https://docs.oracle.com/javase/specs/jls/se25/html/jls-15.html#jls-15.27)

## 三、反射与动态代理：理解框架的调用入口

反射通过 `Class`、`Method`、`Field` 等元数据发现类型、读取注解、创建实例和调用方法，适合序列化、依赖注入、插件注册等通用基础设施。业务代码能直接调用时通常保持直接调用，更利于编译检查和重构。反射访问受可见性、模块导出和开放规则约束，`setAccessible(true)` 不是绕过一切限制的通行证。[AccessibleObject API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/reflect/AccessibleObject.html)

JDK 动态代理实现一组接口，把代理上的方法调用交给 `InvocationHandler`，可以围绕目标调用记录耗时、检查权限或做适当重试。它不是直接继承任意业务类；类代理属于另一种机制。需要处理 `equals/hashCode/toString`、异常传播与目标对象引用，尤其不能把所有失败都无限重试。[Proxy API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/reflect/Proxy.html)

## 四、策略、工厂、模板方法分别解决什么问题

以导出文件为例：策略把 CSV、JSON 的不同导出算法放到共同接口；工厂负责根据格式选择或组装策略；模板方法把“校验、读取、转换、写出、记录”的固定骨架放在父类，把变化步骤交给子类。它们可以协作，但不是每个 `if` 都值得增加一层继承。

```java
enum Format { CSV, JSON }
interface Exporter { byte[] export(List<String> rows); }

// 示例片段：注册表在构造阶段建立，发布后不再修改。
Map<Format, Exporter> exporters = new EnumMap<>(Format.class);
exporters.put(Format.CSV, new CsvExporter());
exporters.put(Format.JSON, new JsonExporter());
Exporter selected = Objects.requireNonNull(exporters.get(format), "不支持的格式");
```

这更准确地说是“基于注册表的简单工厂”，不要与 GoF 工厂方法模式混为一谈。启动时检查重复注册与缺失策略，通常比运行到一半才报空指针更清楚。模板方法适合流程稳定、少数步骤变化的场景；如果组合不断变化，用策略组合可能更灵活。上述模式是设计判断，不是框架自动提供的性能优化。

## 五、边界与常见误区

- `Collections.unmodifiableList` 可能是底层列表的只读视图，元素也可能可变；“不能通过此引用增删”不等于深度不可变。
- 不要背诵某个 JDK 的扩容倍数、树化阈值作为集合接口契约；这些实现细节要按版本核对。
- 反射与代理让代码更通用，也增加调用链和调试成本；日志应能辨认实际目标与根异常。
- 抽象围绕真实变化点建立；只出现一次的简单分支，保留直观写法可能更易维护。

## 六、一句话面试回答

> 应先按顺序、唯一性、访问方式和并发约束选集合，用泛型约束数据、用函数式接口表达行为，再把稳定流程与变化算法分开；反射和代理主要用于框架扩展，并保留可解释的调用与异常边界。

## 七、实操练习与预期成果（未执行）

1. 用 `List` 保存模拟订单，用 `Map` 建订单号索引，用 `Set` 检测重复；比较重复键、可变键和迭代时修改的行为，写明选型理由。
2. 实现 CSV/JSON 导出策略和 `EnumMap` 注册工厂，再提取可选的流程模板；故意漏注册一种格式，验证错误在何处暴露。
3. 给导出接口加一个 JDK 动态代理，记录耗时并透传目标异常；比较直接调用、代理调用和目标内部自调用的路径。
4. 交付源码、边界用例与一页设计说明，记录使用的 JDK 版本及实际输出；本笔记没有运行这些练习，也没有性能实测结论。

返回：[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]

资料核验日期：**2026-10-20**。API 链接采用 JDK 25；基础概念也适用于常见旧版本，但新增方法、模块反射限制与集合实现细节须按实际 JDK 核对。
