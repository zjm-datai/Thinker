[原文链接](https://goshippo.com/blog/why-is-my-fastapi-throughput-so-low)

翻译如下：

## 翻译

我们的 webhook 处理程序非常简单：接收请求后立即将其发送到 SQS（Amazon Simple Queue Service，亚马逊简单队列服务，一种分布式消息队列服务，用于解耦和削峰填谷）进行后台处理，以尽快响应调用方。

```python
@router.post("/invoice")
async def invoice_handler(
    request: Request,
    producer: Producer = Depends(Producer),  # FastAPI 的依赖注入机制
) -> JSONResponse:
    body = await request.body()              # 从请求中异步读取原始字节
    body_str = body.decode("utf-8")
    try:
        Invoice.model_validate_json(body_str)  # Pydantic 模型校验
        producer.send_message(body_str)        # 同步地将消息发送到 SQS
    except ValidationError as e:
        ...
    except Exception as e:
        ...
    return JSONResponse(status_code=200, content={...})
```

我们使用 [Vegeta](https://github.com/tsenart/vegeta)（一个高性能 HTTP 压力测试工具）来做负载测试：

```
vegeta attack -rate 500 -duration 5s
```

|Success Rate|Fastest Response|Throughput|
|--:|--:|--:|
|59%|932 ms|42 RPS|
在对现代框架期望很高的情况下，「最快响应近 1 s、平均仅 42 RPS（每秒请求数）」远低于预期。

### 异步和线程

仔细看上面那行 `producer.send_message(body_str)`：它是**同步/阻塞**调用，却被放在了 `async def` 路由里。

- **异步路由函数（async def）** 在单一的事件循环（Event Loop）中运行，任何阻塞调用都会“卡住”整个循环，导致无法同时处理其他请求。

- 相反，**同步路由函数（def）** 会被 Starlette 自动交给后台的线程池（Thread Pool）去执行，不会阻塞主事件循环。

>[!cite]
>Async handlers in FastAPI run on a single event loop on the main thread, sync handlers run on a thread pool.

为了不阻塞，我们把发送消息的同步调用移入线程池：

```python
from fastapi.concurrency import run_in_threadpool

@router.post("/invoice")
async def invoice_handler(
    request: Request,
    producer: Producer = Depends(Producer),
) -> JSONResponse:
    body = await request.body()
    body_str = body.decode("utf-8")
    try:
        Invoice.model_validate_json(body_str)
        # 将同步调用切换到线程池执行
        await run_in_threadpool(producer.send_message, body_str)
    except ValidationError as e:
        ...
    except Exception as e:
        ...
    return JSONResponse(status_code=200, content={...})
```

- **run_in_threadpool**（底层基于 AnyIO）会把给定函数放在线程池执行，并在 await 时调度回事件循环。

- FastAPI/Starlette 默认的线程池大小是 40 个线程，如果大量并发的同步请求排队，线程耗尽时仍会阻塞。

我们把线程池容量调至 100：

```python
from anyio import to_thread

DEFAULT_FASTAPI_THREAD_POOL_WORKERS = 100

async def lifespan(app: FastAPI):
    workers = settings.get_int("FASTAPI_THREAD_POOL_WORKERS", DEFAULT_FASTAPI_THREAD_POOL_WORKERS)
    to_thread.current_default_thread_limiter().total_tokens = workers
    ...
```

再次跑相同的测试

| Success Rate | Fastest Response | Throughput |
| -----------: | ---------------: | ---------: |
|          78% |           894 ms |     55 RPS |

经过这一改进，“成功率”略升、“吞吐量”提升，但「最快响应仍接近 900 ms」。

### 性能刨析

为了找出到底耗时在哪儿，我们用了 [PyInstrument](https://github.com/joerick/pyinstrument)（一个轻量级的 Python 性能分析器），分别对 **纯同步** 和 **异步 + run_in_threadpool** 两种 handler 进行剖析。

sync

```
0.464 Runner.run  asyncio/runners.py:86
├─ 0.427 coro  starlette/middleware/base.py:65
│ 	[45 frames hidden]  starlette, fastapi, anyio, asyncio
│    	0.158 run_sync_in_worker_thread  anyio/_backends/_asyncio.py:834
│    	├─ 0.128 [await]  anyio/_backends/_asyncio.py
│    	0.105 run_endpoint_function  fastapi/routing.py:182
│    	└─ 0.105 invoice_handler  billing/routers/webhooks/invoice.py:19
│             	[6 frames hidden]  loguru, core, <built-in>
│    	0.055 ProfilingMiddleware.__call__  starlette/middleware/base.py:24
│    	└─ 0.053 ProfilingMiddleware.dispatch  
└─ 0.006 RequestResponseCycle.run_asgi  uvicorn/protocols/http/httptools_impl.py:417
  	[15 frames hidden]  uvicorn, fastapi, starlette, opentele...
```

async

```
0.531 Runner.run  asyncio/runners.py:86
├─ 0.480 coro  starlette/middleware/base.py:65
│ 	[45 frames hidden]  starlette, fastapi, anyio, asyncio
│    	0.180 run_sync_in_worker_thread  anyio/_backends/_asyncio.py:834
│    	├─ 0.158 [await]  anyio/_backends/_asyncio.py
│    	0.124 run_endpoint_function  fastapi/routing.py:182
│    	└─ 0.124 invoice_handler  billing/routers/webhooks/invoice.py:19
│       	├─ 0.082 run_in_threadpool  starlette/concurrency.py:35
│             	[6 frames hidden]  loguru, core, <built-in>
│    	0.046 ProfilingMiddleware.__call__  starlette/middleware/base.py:24
│    	└─ 0.045 ProfilingMiddleware.dispatch  billing/main.py:227
│       	└─ 0.043 call_next  starlette/middleware/base.py:31
│             	[9 frames hidden]  starlette, anyio, asyncio
```

我们发现异步版本总体耗时约 530 ms，但只有 ~200 ms 在实际 handler 里，其余 ~300 ms 分散在中间件、依赖注入等环节。

进一步展开堆栈（stack trace）后发现：在处理依赖注入 `Producer = Depends(Producer)` 时，SQS 客户端在实例化阶段会**同步**地向 AWS 验证队列是否存在，每次请求都重新创建客户端并发出网络调用，导致再次阻塞主循环。

Profile with SQS request

```
1.240 Runner.run  asyncio/runners.py:86
├─ 1.215 coro  starlette/middleware/base.py:65
│  ├─ 1.014 ExceptionMiddleware.__call__  starlette/middleware/exceptions.py:53
│  │  └─ 1.014 AsyncExitStackMiddleware.__call__  fastapi/middleware/asyncexitstack.py:12
│  │ 	└─ 1.014 APIRouter.__call__  starlette/routing.py:697
│  │    	├─ 1.012 APIRoute.handle  starlette/routing.py:265
│  │    	│  └─ 1.012 app  starlette/routing.py:63
│  │    	│ 	├─ 0.824 app  fastapi/routing.py:217
│  │    	│ 	│  ├─ 0.487 solve_dependencies  fastapi/dependencies/utils.py:508
│  │    	│ 	│  │  ├─ 0.486 solve_dependencies  fastapi/dependencies/utils.py:508
│  │    	│ 	│  │  │  └─ 0.486 get_queue  billing/sqs/invoice.py:21
│  │    	│ 	│  │  │ 	├─ 0.478 SqsSimpleQueue.__init__  aws/sqs/queue.py:136
│  │    	│ 	│  │  │ 	│  ├─ 0.335 _get_queue_url_create_if_necessary  aws/sqs/queue.py:160
│  │    	│ 	│  │  │ 	│  │  └─ 0.335 SQS._api_call  botocore/client.py:544
│  │    	│ 	│  │  │ 	│  │ 	└─ 0.335 SQS._make_api_call  botocore/client.py:925
│  │    	│ 	│  │  │ 	│  │    	├─ 0.325 SQS._make_request  botocore/client.py:1013
│  │    	│ 	│  │  │ 	│  │    	│  └─ 0.325 Endpoint.make_request
botocore/endpoint.py:113
```

### 单例模式复用客户端

我们根本没必要每次请求时都新建一个 SQS 客户端。改用 **单例模式（Singleton）** 来缓存并复用同一个客户端：

```python
# billing/sqs/invoice.py

class Producer:
    _client = None

    def __new__(cls, *args, **kwargs):
        if cls._client is None:
            cls._client = boto3.client("sqs", ...)
            # 这里做一次 _get_queue_url_create_if_necessary 调用
        return super().__new__(cls)

    def send_message(self, body_str: str):
        return self._client.send_message(QueueUrl=..., MessageBody=body_str)
```

剖析结果：**整体耗时降到 ~212 ms**，几乎全都集中在真正的 `send_message` 调用上。

再次压测：

| Success Rate | Fastest Response | Throughput |
| -----------: | ---------------: | ---------: |
|          99% |           256 ms |    150 RPS |

### 重构：让 Handler 同步更美观

虽然 `await run_in_threadpool(...)` 有效，但代码不够「简洁」。我们把读取请求体的异步逻辑抽成依赖，然后整个 handler 改为同步 `def`，由 Starlette 自动走线程池：

```python
DEFAULT_FASTAPI_THREAD_POOL_WORKERS = 200

async def get_body(request: Request) -> str:
    body = await request.body()
    return body.decode("utf-8")

@router.post("/invoice")
def invoice_handler(
    body_str: str = Depends(get_body),    # 先异步拿到 body 再传进来
    producer: Producer = Depends(Producer),
) -> JSONResponse:
    try:
        Invoice.model_validate_json(body_str)
        producer.send_message(body_str)    # 同步调用，线程池中执行
    except ValidationError as e:
        ...
    except Exception as e:
        ...
    return JSONResponse(status_code=200, content={...})
```


压测结果：

```
vegeta attack -rate 500 -duration 5s
```

| **Sucess Rate** | **Fastest Response** | **Throughput** |
| --------------- | -------------------- | -------------- |
| 100%            | 437ms                | 195 RPS        |

These numbers look reasonably good for a single worker process running on a single Kubernetes pod. Then we decided to run a final test aiming at 200 RPS for a couple of minutes to see if it sustains the RPS or if it was just an effect of test duration.

```
vegeta attack -rate 200 -duration 4m (sustainable 200 RPS)
```

| Success Rate | Fastest Response | Throughput |
| -----------: | ---------------: | ---------: |
|         100% |           187 ms |    198 RPS |

### 总结

总而言之，我们并没有将网络服务器的速度提升 5 倍，但解决了一些使其速度慢 5 倍的问题。

通过了解 FastAPI 如何处理同步/异步请求、增加同步请求的工作线程池大小、从异步请求和依赖项解析中移除阻塞调用以及避免对工作线程池进行显式调用，我们成功将每秒请求数（RPS）从 40 提升并稳定到 200。

此外，性能分析再次证明是一个有用的工具，能够发现隐藏的性能问题，比如我们案例中的同步依赖项解析问题。 