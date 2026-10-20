---
type: concept
topic: HTTP流程与并发模型
aliases: [Spring MVC与FastAPI, 异步阻塞陷阱]
status: reference
created: "2026-10-20"
updated: "2026-10-20"
tags: [技术储备, HTTP, Spring, FastAPI, 并发]
related: "[[part78-technical-skills/chapter02-java/section02-concurrency]], [[part78-technical-skills/chapter01-ai/section05-spring-ai-and-tools]]"
---

# HTTP 请求如何经过 Spring MVC 与 FastAPI

> 本页是原理与练习材料；练习尚未执行。版本核验日期：2026-10-20。框架版本、ASGI/Servlet服务器、worker数、线程池及HTTP客户端配置会改变实际行为，不能只凭方法名推断并发能力。

## 核心解释：先分清协议、框架和执行资源

HTTP定义方法、状态码、请求头与表示等语义；TCP提供可靠、有序的字节流，不保留应用消息边界。因此从socket读到一次数据不一定等于一个完整请求，连接超时也不代表业务一定未完成。HTTP/1.1、HTTP/2通常使用TCP，HTTP/3使用QUIC，不能把“HTTP就是TCP”当成完整结论。[HTTP语义](https://www.rfc-editor.org/rfc/rfc9110.html)、[TCP规范](https://www.rfc-editor.org/rfc/rfc9293.html)

普通REST请求适合一次请求返回一个结果。SSE在HTTP响应中持续发送`text/event-stream`事件，适合模型文本、进度和通知；浏览器EventSource主要负责服务端到客户端的文本事件及重连。WebSocket适合需要双方持续主动发送消息的交互，但要自行设计心跳、重连、消息确认和业务去重。[SSE规范](https://html.spec.whatwg.org/multipage/server-sent-events.html)、[WebSocket规范](https://www.rfc-editor.org/rfc/rfc6455.html)

## 两条典型HTTP处理路径

```text
Spring MVC（常见Servlet部署）
连接/HTTP解析 → Servlet Filter（可含Security） → DispatcherServlet
→ HandlerMapping/HandlerAdapter → 参数转换与校验 → Controller/Service
→ 返回值转换或异常处理 → HTTP响应

FastAPI（常见Uvicorn/ASGI部署）
连接/HTTP解析 → ASGI scope/receive/send → Middleware → 路由匹配
→ 依赖解析/参数校验 → path operation → 响应序列化 → ASGI send
```

Spring MVC的前端控制器把映射、执行和返回值处理交给对应组件；拦截器不等于Servlet Filter，也不覆盖容器之前的网络阶段。FastAPI基于Starlette/ASGI，请求校验与业务执行也不是同一层。两者都可以做依赖注入、校验、异常转换和流式响应，框架选择主要取决于生态、团队能力、调用库和可观测性需求。[DispatcherServlet](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html)

## 阻塞与异步究竟发生在哪里

| 写法/部署 | 等待外部I/O时通常占用什么 | 关键约束 |
|---|---|---|
| MVC普通控制器 + 阻塞JDBC/HTTP库 | 处理该调用的工作线程 | 池大小、连接池和超时要协调 |
| MVC返回`Callable`/`DeferredResult` | 可释放初始Servlet线程；任务仍需要执行资源 | 有异步派发、超时和上下文传播，不能变出数据库连接 |
| WebFlux + 非阻塞客户端 | 事件循环通过回调/信号继续处理 | 在事件循环中阻塞仍会拖慢其他请求 |
| FastAPI `async def` + 可等待I/O | `await`挂起时让出事件循环 | 调用链必须真正支持异步 |
| FastAPI框架调用的普通`def`路由/依赖 | 框架线程池中的线程 | 线程池有容量，耗尽后会排队 |
| 从协程直接调用普通helper | 仍在调用它的线程执行 | helper不会因使用`def`而自动进入线程池 |

Spring还有Servlet异步处理、WebFlux及依版本和配置启用的虚拟线程方案。虚拟线程可降低部分阻塞等待的线程成本，但CPU、数据库连接、下游配额和共享锁仍有限。MVC异步不意味着所有读写都改成非阻塞；必须观察实际调用栈。[MVC异步机制](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html)

FastAPI把它负责调用的同步路由/同步依赖交给线程池；普通工具函数按Python调用规则执行。`async def`只是协程函数声明，直接调用`requests.get()`、同步数据库驱动、`time.sleep()`仍会阻塞当前事件循环线程。把函数声明成异步，甚至在外层再加一个`await`，都不能把内部同步调用自动变成非阻塞。[FastAPI异步说明](https://fastapi.tiangolo.com/async/)

以下仅演示调度差异，未在本次工作中启动服务或压测：

```python
import asyncio
import time
from fastapi import FastAPI

app = FastAPI()

def blocking_helper():
    time.sleep(1)   # 模拟同步I/O等待
    return {"ok": True}

@app.get("/blocks-loop")
async def blocks_loop():
    return blocking_helper()  # 普通helper不会自动卸载

@app.get("/framework-pool")
def framework_pool():
    return blocking_helper()  # 此路由由框架在线程池中调用

@app.get("/explicit-offload")
async def explicit_offload():
    return await asyncio.to_thread(blocking_helper)
```

`asyncio.to_thread`适合卸载同步I/O，本身使用的执行资源不等同于Starlette/AnyIO线程池；不要假定修改一个池就控制了全部工作。线程卸载需配合有界并发、连接池及库的线程安全要求。取消协程不保证已经运行的线程函数停止。纯Python CPU密集任务一般应考虑进程池或独立作业；会释放GIL的原生库、不同解释器构建另当别论。[Python线程卸载](https://docs.python.org/3/library/asyncio-task.html#asyncio.to_thread)、[Starlette线程池](https://starlette.dev/threadpool/)

## 适用场景与常见误区

- 同步数据库和传统Java生态占主导时，MVC是自然选择；Python模型推理周边、异步HTTP扇出较多时，FastAPI可减轻集成成本。二者都需要依据实际延迟、吞吐和运维能力选择，不能断言某框架总是更快。
- 并发是同时推进多个任务，CPU并行是同时执行计算。一个事件循环可以推进很多等待中的请求，不代表一个worker可以同时执行无限CPU计算；多worker也会增加模型与连接池的总内存。
- 100个等待请求不应该直接对应100个无限下游调用。限流、排队、超时和拒绝策略共同决定故障是否扩散；仅增加线程数可能把数据库拖垮。
- SSE响应一旦发送响应头，后续异常通常无法改成普通JSON错误码，需设计错误事件及终止事件。代理缓冲、空闲超时、心跳和重连游标都影响体验。浏览器断开、服务端取消订阅和远端模型终止是三层不同动作；SSE断开不保证远端停止生成或计费，需按供应商取消协议与用量账单核对。

## 一句话面试回答

“MVC常见部署以Servlet线程处理请求，FastAPI的异步路由以事件循环协作等待I/O；两者都有线程与异步方案，关键是调用链是否阻塞、阻塞发生在哪个执行器，以及下游资源是否有界。”

## 实操练习（未执行）

1. 在独立练习目录运行上述三个路由，另加一个立即返回的健康检查；单worker下并发发送等待请求，同时记录健康检查的p50/p95、总耗时、线程数与事件循环延迟。预期现象只是待验证假设：直接阻塞事件循环会连带影响其他协程；线程池版也会在资源饱和时排队。
2. 再比较异步HTTP客户端、有界线程卸载与CPU循环；固定请求数、超时、连接池及并发上限，每次只改一个条件。不要把模拟sleep的结果外推为生产模型推理性能。
3. 画一张MVC普通请求与异步返回的线程/派发时间线，标出Security上下文、日志traceId、事务连接的边界；为SSE模拟客户端中断，分别检查浏览器、应用和模型供应商的状态。

验收成果：调度时间线、可重复的命令和环境版本、延迟原始记录、池容量与超时表，以及“本次数据能说明什么/不能说明什么”。没有这些记录，不把示例写成已验证结论。

**返回：** [[part78-technical-skills/chapter00-技术技能储备|技术技能储备]] | [[part00-index/part00-主页|主页]]
