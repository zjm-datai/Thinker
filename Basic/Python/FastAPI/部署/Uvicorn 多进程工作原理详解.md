
当使用 `uvicorn main:app --workers 4` 启动时，Uvicorn 会通过 Python 的多进程 `spawn` 模式创建 4 个独立的 worker 进程，每个进程各自绑定同一监听端口（启用 SO_REUSEPORT），由操作系统内核在接收新连接时自动将 TCP 连接轮询或基于 CPU 哈希地分配给各个进程。每个 worker 启动自己的异步事件循环（默认使用性能更优的 `uvloop`）并加载 HTTP 解析器（如 `httptools`），以高并发地接收、解析、封装为 ASGI scope，然后交给 FastAPI/Starlette 应用协程处理。各 worker 进程在独立的内存空间中运行，拥有各自的 GIL，操作系统调度负责分配 CPU 时间片；对于阻塞或 CPU 密集型任务，Uvicorn 可以通过内部线程池（由 `--limit-concurrency` 控制）进行脱载，确保整个服务的吞吐与稳定性。

## 1. Worker 进程启动与端口绑定

### 1.1 Spawn 模式创建多进程

Uvicorn 的 `--workers` 参数让它在启动时使用 Python 的 `multiprocessing.spawn` 模式来生成多个子进程，每个进程都会重新初始化 ASGI 服务器并独立运行，从而绕过 GIL 限制，实现多核并行处理

[uvicorn.org](https://www.uvicorn.org/deployment/?utm_source=chatgpt.com "Deployment - Uvicorn")

### 1.2 SO_REUSEPORT 端口共享

在 UNIX-like 系统上，Uvicorn 为监听 socket 设置 `SO_REUSEPORT`，允许所有 worker 进程对同一端口并发 `bind`，无需主进程转发连接；这样每个进程都能直接接受客户端连接 

[stackoverflow.com](https://stackoverflow.com/questions/79528455/is-there-any-way-to-serve-a-django-application-using-preload-asgi-and-so-reuse?utm_source=chatgpt.com "Is there any way to serve a Django application using preload, ASGI ...")

## 2. 操作系统层面的请求分发

### 2.1 内核负载均衡

当新 TCP 连接到达时，启用了 `SO_REUSEPORT` 的监听 socket 会在所有 worker 进程的队列中轮询或基于 CPU 哈希地分配连接，实现请求在进程间的自动负载均衡，无需用户手动管理

[reddit.com](https://www.reddit.com/r/learnpython/comments/11vzc9n/eli5_how_gunicornhypercornuvicorn_multiple/?utm_source=chatgpt.com "ELI5 How Gunicorn/Hypercorn/Uvicorn multiple workers work - Reddit")

## 3. Worker 内部的事件循环与 ASGI 调度

### 3.1 事件循环与协议解析器

每个 worker 进程启动后，会初始化异步事件循环（默认选用 `uvloop`，如已安装则自动采用）以及高性能的 HTTP 解析器 `httptools`，负责底层 I/O 及协议细节

[uvicorn.org](https://www.uvicorn.org/?utm_source=chatgpt.com "Uvicorn"), [uvicorn.org](https://www.uvicorn.org/deployment/?utm_source=chatgpt.com "Deployment - Uvicorn")

### 3.2 接受连接与 ASGI Scope 构建

当操作系统将连接派发到某个 worker，进程的事件循环立即执行 `accept()`，并将原始 socket 封装成 ASGI 连接 scope，包含路径、请求头、客户端地址等元信息，后续由框架统一处理

[stackoverflow.com](https://stackoverflow.com/questions/72897199/uvicorn-backlog-vs-limit-concurrency?utm_source=chatgpt.com "Uvicorn backlog vs limit-concurrency - asgi - Stack Overflow")

### 3.3 ASGI 应用协程执行

封装好的 scope 会传递给 ASGI 应用的入口协程（如 FastAPI 的 `__call__`），通过 `receive()` 拉取请求体、通过 `send()` 推送响应数据。这一过程完全异步调度，可在单个事件循环内并发处理多个请求协程

[uvicorn.org](https://www.uvicorn.org/server-behavior/?utm_source=chatgpt.com "Server Behavior - Uvicorn")

## 4. 每个请求的资源分配与并发控制

### 4.1 进程隔离与 GIL

每个 worker 在独立的 OS 进程中运行，拥有各自的内存空间与 Python 解释器实例，也拥有独立的 GIL，因此可获得真并行的 CPU 使用，不同请求互不干扰

[uvicorn.org](https://www.uvicorn.org/deployment/?utm_source=chatgpt.com "Deployment - Uvicorn")

### 4.2 协程并发调度

在同一 worker 中，多条请求协程由 asyncio 事件循环协作式切换——当某协程遇到 I/O 操作（如网络、磁盘）时会让出控制权，切换执行其他就绪协程，从而最大化吞吐

[stackoverflow.com](https://stackoverflow.com/questions/72897199/uvicorn-backlog-vs-limit-concurrency?utm_source=chatgpt.com "Uvicorn backlog vs limit-concurrency - asgi - Stack Overflow")

### 4.3 线程池脱载与限流

对于同步阻塞或 CPU 密集型任务，Uvicorn 可利用内置线程池执行，避免阻塞事件循环；可通过 `--limit-concurrency` 限制同时入线程池的任务数，超出时自动返回 503，保障内存与吞吐的可预测性

[stackoverflow.com](https://stackoverflow.com/questions/72897199/uvicorn-backlog-vs-limit-concurrency?utm_source=chatgpt.com "Uvicorn backlog vs limit-concurrency - asgi - Stack Overflow")

### 4.4 OS 调度与资源调配

操作系统内核负责在多进程中分配 CPU 时间片和内存资源，依据进程优先级与运行负载动态调度，使整体负载均衡且防止单一 worker 饱和

[reddit.com](https://www.reddit.com/r/learnpython/comments/11vzc9n/eli5_how_gunicornhypercornuvicorn_multiple/?utm_source=chatgpt.com "ELI5 How Gunicorn/Hypercorn/Uvicorn multiple workers work - Reddit")