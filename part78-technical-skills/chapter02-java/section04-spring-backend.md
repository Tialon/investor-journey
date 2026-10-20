---
type: concept
topic: "Spring 后端基础：容器、代理与接口边界"
status: reference
created: '2026-10-20'
updated: '2026-10-20'
tags: [技术储备, Java, Spring, MyBatis-Plus, Security, 缓存]
related:
  - '[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]'
  - '[[part78-technical-skills/chapter02-java/section01-java-and-design|Java 基础与可扩展设计]]'
  - '[[part78-technical-skills/chapter02-java/section02-concurrency|Java 并发与异步任务]]'
  - '[[part78-technical-skills/chapter02-java/section05-http-and-async|HTTP 与异步并发模型]]'
---

# Spring 后端基础：容器、代理与接口边界

> 学习目标：能解释注解背后的对象管理和调用链，并把持久化、权限、缓存与接口契约落在明确的边界上。HTTP 完整流程及 Spring MVC/FastAPI 并发模型见 [[part78-technical-skills/chapter02-java/section05-http-and-async|HTTP 与异步并发模型]]。

## 一、核心机制：IoC、注解、AOP 与 SpEL

IoC 把依赖对象的创建和组装交给容器，DI 是其常见实现。构造器注入让必要依赖显式可见，便于替换与单元测试；容器还负责 Bean 生命周期、作用域与扩展。单例作用域表示容器内通常复用同一实例，不代表实例字段天然线程安全，Controller/Service 不宜用字段保存每个请求的可变状态。[Spring IoC](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)

注解是元数据，需要编译器、容器或其他处理器解释后才有行为；还要区分它的作用位置、保留策略和启用条件。`@Component` 扫描对象、`@Bean` 显式构造对象，`@Qualifier` 解决候选依赖选择，不能把“写了注解”等同“已经生效”。

AOP 把日志、权限、事务等横切逻辑放入增强链。Spring 常用 JDK 接口代理或 CGLIB 类代理，调用者经过代理才触发对应增强；类代理也受 `final`、`private` 等可覆盖性约束。**目标对象里的 `this.someMethod()` 会绕过代理。**因此默认代理模式下，自调用不会新增被调方法声明的事务或缓存语义；若外层已经有事务，内部逻辑可能仍在该外层事务中，不能概括为“自调用后一切事务都没了”。优先通过职责拆分形成真实 Bean 边界，而不是到处取当前代理。[代理机制](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

SpEL 是表达式求值机制，可用于缓存键、条件等，例如 `key = "#id"`。它与配置占位符 `${...}` 的职责不同；表达式上下文决定可访问哪些属性与方法。不要把不受信任的任意输入直接当作具有完整能力的表达式执行。[SpEL](https://docs.spring.io/spring-framework/reference/core/expressions.html)

## 二、Boot 自动配置与 Starter

Starter 主要组织常用依赖，自动配置则根据类路径、属性和已存在 Bean 等条件提供默认装配。用户显式声明 Bean 后，带有相应缺失条件的默认配置可以退让；这不是“扫描所有第三方类并自动实例化”。诊断某配置为何生效，应查看条件评估报告和实际 Bean，而不只看依赖是否引入。[Boot 自动配置](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)

自定义 Starter 可以分离自动配置模块与便捷依赖模块，提供 `@ConfigurationProperties`、合理默认值、条件开关和配置元数据。当前自动配置文档使用 `@AutoConfiguration`，并通过 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 列出候选类；旧 Boot 的注册方式不能不加区分地照搬。生产项目先锁定 Boot、Framework、JDK 与第三方兼容矩阵。[开发自动配置](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html)

## 三、MVC REST、异常与响应契约

REST 接口设计以资源和操作语义为中心：请求 DTO 做输入校验，Service 处理业务规则，响应 DTO 表达对外契约，避免把数据库实体直接暴露为稳定 API。`@RestController` 的响应体通常经消息转换器序列化，具体转换器与内容协商决定输出形式。[Spring 响应体](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/responsebody.html)

`@RestControllerAdvice` 配合 `@ExceptionHandler` 可以集中映射参数错误、业务冲突与系统异常。可以设计统一响应结构，也可以采用 Spring 支持的 `ProblemDetail`；应同时保留正确 HTTP 状态、可读错误信息与追踪标识。把所有异常包装成 HTTP 200 会影响客户端重试、网关和监控；堆栈、SQL 与令牌不应直接进入对外错误响应。文件下载、流式响应、已提交的 SSE 连接也不能机械套 JSON 外壳。[异常处理](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html)、[错误响应](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)

## 四、事务与 MyBatis-Plus

本地事务围绕数据库连接等资源建立提交、回滚边界。代理模式下的 `@Transactional` 要经过代理，并选择正确事务管理器；异常被内部吞掉、异步切线程、传播配置不当都可能改变结果。通常默认回滚针对运行时异常与 `Error`，受检异常应结合版本、全局配置与显式规则核验；不能仅凭方法抛出了一个异常就断定一定回滚。数据库事务也不能直接撤销已经发生的第三方调用。[声明式事务](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

MyBatis-Plus 分页通过拦截器改写查询并生成所需分页/统计 SQL，不会自动把低效 SQL 变快。设置页大小上限、稳定且唯一的排序，并核对联表 count 与实际记录语义；深分页可评估基于游标的查询。**版本敏感：官方注明从 3.5.9 起分页拦截器相关解析依赖拆分，需按所用版本配置 `mybatis-plus-jsqlparser`。**插件顺序也可能影响 SQL，多个插件组合应检查最终 SQL。[分页插件](https://baomidou.com/plugins/pagination/)

动态表名插件在执行前按规则替换表名，可用于按时间分表，但不等于完整分片系统。表名由后端白名单与明确规则生成，不接受前端任意表名拼接；还需考虑跨表查询、建表、归档、事务和路由上下文清理。如果用 ThreadLocal 保存路由，须处理线程复用与异步传播。[动态表名插件](https://baomidou.com/plugins/dynamic-table-name/)

## 五、Spring Security、JWT 与 RBAC

Servlet 场景的 Spring Security 主要以过滤器链建立认证与授权流程：认证回答“是谁”，授权回答“能做什么”。RBAC 把用户关联到角色，再把角色关联到权限；URL 授权和方法授权可以协作，但还要检查资源归属，例如普通用户是否只能读取自己的订单。前端隐藏按钮不构成后端授权。[Security 架构](https://docs.spring.io/spring-security/reference/servlet/architecture.html)、[请求授权](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html)、[方法安全](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)

JWT 是令牌格式。常见签名 JWT 的载荷可被读取，签名主要用于校验完整性与签发可信度，不能当作数据加密。资源服务器应验证签名、允许的算法、有效期、签发者以及应用要求的受众，并明确 claims 如何映射为权限。短有效期、刷新令牌、密钥轮换和撤销策略需要共同设计；把角色写进长期有效令牌，可能使权限变更不能及时生效。[JWT 资源服务器](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)

401 与 403 分别对应认证问题和授权不足等语义，应在安全链的相应入口处理，不能假设所有过滤器异常都会进入 ControllerAdvice。使用 JWT 也不自动意味着可以关闭 CSRF；还要看凭证是否由浏览器通过 Cookie 等方式自动附带。[CSRF 说明](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html)

## 六、Spring Cache 与多级缓存

Spring Cache 是缓存抽象，不是某一种缓存存储。`@Cacheable` 在命中时复用结果，`@CachePut` 更新条目，`@CacheEvict` 驱逐条目；需要明确缓存名、键、空值、TTL、序列化与失效规则，默认代理模式同样有自调用边界。[Cache 抽象](https://docs.spring.io/spring-framework/reference/integration/cache.html)、[缓存注解](https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html)

典型多级方案是本地 Caffeine 作为 L1，Redis 作为共享 L2，数据库为事实来源：先读 L1，再读 L2，再查库并回填。引入 L1 后，每个实例可能持有不同旧值，需要用 TTL、版本或失效通知控制可接受陈旧时间；Redis 删除成功也不代表所有实例本地缓存已更新。多个 `CacheManager` 或 `CompositeCacheManager` 不能仅靠名称就自动形成这套逐级读取与回填协议；后者按顺序寻找能够提供指定缓存的管理器。[CompositeCacheManager API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/cache/support/CompositeCacheManager.html)

适合读多写少、允许短暂陈旧的数据；余额、权限等强约束数据必须先确定一致性要求。数据库提交后再失效通常比提交前失效更合理，但仍要处理并发回填、失效事件丢失与重试。缓存降低读成本，却增加了副本一致性问题。

## 七、一句话面试回答

> Spring 容器管理依赖，代理承载横切增强，Boot 用条件配置提供默认装配；后端设计要同时说明接口契约、事务和权限边界，以及缓存副本何时可能过期或失效。

## 八、实操练习与预期成果（未执行）

1. 设计一个小型只读目录接口，说明 Bean 注入、DTO 校验、异常映射和 HTTP 状态；另用条件报告解释某自动配置为何生效。
2. 对比外部调用与同类自调用的事务、缓存效果，增加“外层事务已存在”的一组用例，记录真实 SQL、提交/回滚和缓存命中，避免只凭注解判断。
3. 为分页设计唯一排序与页大小上限，对联表 count 做人工核对；为动态表名加允许列表，验证未知表名被拒绝。
4. 列出未登录、令牌过期、角色不足、资源不属于当前用户四类访问；设计 L1/L2 更新及失效事件丢失的时序图。
5. 预期交付 API 示例、权限矩阵、事务观察记录与缓存一致性说明。本次未启动应用、未调用真实身份或支付服务，也未执行这些练习。

返回：[[part78-technical-skills/chapter00-技术技能储备|技术技能储备]]

资料核验日期：**2026-10-20**。Spring 链接为官方滚动参考，不表示项目已升级到文档显示版本；Boot 自动配置注册、安全配置 DSL、事务方法可见性、缓存异步支持与 MyBatis-Plus 插件依赖均具有版本敏感性，落地时按项目锁定版本核验。
