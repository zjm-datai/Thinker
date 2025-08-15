
### 先说结论

vllm server 从目前 0.9.x 的版本来看不能给 Uvicorn 传入 `--workers` 之类的参数。vLLM 的 OpenAI 兼容 API 服务器，自己起 Uvicorn(FastAPI)，默认就是一个 API 进程和一个引擎进程的架构。

如果我们想要增加前端并发能力，通常做法是 **起多个 vLLM 副本** 放到负载均衡后面，或者按 GPU 维度做并行（TP/PP），而不是把 Uvicorn workers 撑多。大多数场景下，**瓶颈在 GPU 推理**，不是 Uvicorn。

---

实际上我们的问题已经被提到了，不止我们发现了这样的问题

[Scale the API server across multiple CPUs](https://github.com/vllm-project/vllm/issues/12705) 

### 不建议直接加 Uvicorn workers

Uvicorn 多 worker = **多进程**。每个进程如果要服务同一个模型，就会涉及 **模型/引擎复用问题**；简单粗暴地多开会导致 **重复加载/显存冲突** 或多个前端进程竞争同一个后端引擎，官方并未把 “多 Uvicorn worker” 作为推荐路径。

社区也在提 “能否让 API 层横向扩 CPU”的诉求（还在讨论/排队），比如上面那篇文章。

一些实践文章也特别提醒：给 Uvicorn 加 workers 往往意味着 **每个 worker 会各自“拥有”一套模型/客户端**，对于大模型（尤其是单卡）几乎不可行/不经济。

[如何使用 vLLM 和 FastAPI 在云 GPU 上部署 Phi-2](https://www.runpod.io/articles/guides/serving-phi-2-cloud-gpu-vllm-fastapi)

我们摘录一下原文：

#### 单进程问题

默认情况下，fastapi 配合 uvicorn 启动是单进程运行的，但是 llm.generate 会调用 GPU 并且阻塞这个进程，直到任务完成。

如果用户的请求要生成很长的结果，那么这个进程会被一直占用。

#### 异步并发的局限

虽然 FastAPI 通过 Uvicorn/Starlette 本身是支持 asyncio 并发的，但是如果 python 代码被计算密集型任务比如 GPU 调用占用，事件循环就没法去处理其他请求，直到当前任务结束。

#### 多进程解决方案（有问题）

你可以用多进程运行，例如：

```
uvicorn app:app --workers 2 --host 0.0.0.0 --port 8000
```

这样会启动多个进程，但 **每个进程都会加载一份模型**。

- 对于大模型，这意味着会在同一块 GPU 上放两份模型，占用巨大显存。
    
- 比如 Phi-2 在 40GB 显存上可能刚好放下两份，但依然浪费且没有意义。  

所以在单块 GPU 上，多进程加载多个大模型是不可取的。

#### vllm 内置并发

如果用 vLLM 自带的 API 服务器模式，它会自动处理并发和批处理（batching）。

- 但在你用 Python 代码直接调用 llm.generate 时，需要自己保证并发安全：
    
    - 如果多个请求同时调用 llm.generate，可能会出错，或者内部会排队。
        
    - vLLM 的 LLM 类可能 **不是线程安全** 的。

#### 避免并发冲突的办法
    
- 用一个 **asyncio.Lock** 把 generate 包起来，让同一时刻只有一个任务访问 GPU 推理。

- 或者用一个专门的 **异步队列任务**，所有生成请求都放进去顺序处理。

#### 最简单的替代方案

告诉用户尽量保持请求短一些（减少一次生成的时间），这样单进程也能应付得过来。







