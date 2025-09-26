
![[Untitled diagram _ Mermaid Chart-2025-09-26-023138.png]]

那么就有老铁问了：为什么调度阶段 server 没有直接和 worker 进行交互？

在 gpustack 中调度和部署是两个不同的阶段：

### 调度，server 内部决策

scheduler 在调度阶段不和 worker 直接交互，而是基于数据库中已有的 worker 状态信息进行决策：

gpustack/scheduler/scheduler.py

```python
async with AsyncSession(self._engine) as session:
	workers = await Worker.all(session)
	if len(workers) == 0:
		state_message = "No available workers"

	model = await Model.one_by_id(session, instance.model_id)
	if model is None:
		state_message = "Model not found"

	model_instance = await ModelInstance.one_by_id(session, instance.id)
	if model_instance is None:
		logger.debug(
			f"Model instance(ID: {instance.id}) was deleted before scheduling due"
		)
		return

	candidate = None
	messages = []
	if workers and model:
		try:
			candidate, messages = await find_candidate(
				self._config, model, workers
			)
		except Exception as e:
			state_message = f"Failed to find candidate: {e}"
```

调度器通过 find_candidate() 函数选择合适的 Worker：

gpustack/scheduler/scheduler.py

```python
async def find_candidate(
    config: Config,
    model: Model,
    workers: List[Worker],
) -> Tuple[Optional[ModelInstanceScheduleCandidate], List[str]]:
    """
    Find a schedule candidate for the model instance.
    :param config: GPUStack configuration.
    :param model: Model to schedule.
    :param workers: List of workers to consider.
    :return: A tuple containing:
                - The schedule candidate.
                - A list of messages for the scheduling process.
    """
    filters = [
        GPUMatchingFilter(model),
        LabelMatchingFilter(model),
        StatusFilter(model),
    ]

    worker_filter_chain = WorkerFilterChain(filters)
    workers, filter_messages = await worker_filter_chain.filter(workers)
    messages = []
    if filter_messages:
        messages.append(str(ListMessageBuilder(filter_messages)) + "\n")

    try:
        if is_gguf_model(model):
            candidates_selector = GGUFResourceFitSelector(model, config.cache_dir)
        elif is_audio_model(model):
            candidates_selector = VoxBoxResourceFitSelector(
                config, model, config.cache_dir
            )
        elif model.backend == BackendEnum.ASCEND_MINDIE:
            candidates_selector = AscendMindIEResourceFitSelector(config, model)
        else:
            candidates_selector = VLLMResourceFitSelector(config, model)
    except Exception as e:
        return None, [f"Failed to initialize {model.backend} candidates selector: {e}"]

    candidates = await candidates_selector.select_candidates(workers)

    placement_scorer = PlacementScorer(model)
    candidates = await placement_scorer.score(candidates)

    candidate = pick_highest_score_candidate(candidates)

    if candidate is None and len(workers) > 0:
        resource_fit_messages = candidates_selector.get_messages() or [
            "No workers meet the resource requirements."
        ]
        messages.extend(resource_fit_messages)
    elif candidate and candidate.overcommit:
        messages.extend(candidates_selector.get_messages())

    return candidate, message
```

这个过程完全在 Server 端进行，使用的是数据库中存储的 Worker 状态信息，而不需要实时与 Worker 通信。

### 部署，通过事件驱动

真正的 Worker 交互发生在调度完成之后的部署阶段。当 Scheduler 讲 ModelInstance 分配给 Worker 并保存到数据库后；

Worker 端的 ServerManager 通过 Websocket 监听到这个变化：

gpustack/worker/serve_manager.py

```python
async def watch_model_instances(self):
	while True:
		try:
			logger.info("Started watching model instances.")
			await self._clientset.model_instances.awatch(
				callback=self._handle_model_instance_event
			)
		except asyncio.CancelledError:
			break
		except Exception as e:
			logger.error(f"Failed watching model instances: {e}")
			await asyncio.sleep(5)
```

然后 Worker 自动启动相应的推理服务：

![[Pasted image 20250926104604.png]]

### 为什么这样设计？

1. **异步解耦**：调度决策不需要等待 Worker 响应，提高了系统性能
2. **容错性**：即使某个 Worker 暂时不可达，调度仍可继续
3. **事件驱动**：通过数据库状态变化触发 Worker 行为，保证最终一致性

Worker 状态信息通过定期同步保持最新：

![[Pasted image 20250926105208.png]]

这确保了 Scheduler 在做调度决策时有相对准确的 Worker 资源信息。

```
sequenceDiagram
    participant Client
    
    box "Worker 节点"
        participant W as Worker
        participant WM as WorkerManager
        participant SM as ServeManager
        participant MFM as ModelFileManager
        participant CS as ClientSet
        participant Backend as "推理后端进程"
    end
    
    box "Server 节点"
        participant S as Server
        participant Scheduler
        participant Controllers as "Controllers"
        participant DB as Database
    end
    
    Note over W,S: 1. Worker 启动和注册阶段
    W->>WM: start_async()
    WM->>WM: _get_worker_uuid() & _get_worker_name()
    WM->>CS: register_with_server()
    CS->>S: POST /workers (创建) 或 PUT /workers/{id} (更新)
    S->>DB: 保存 worker 信息
    
    Note over W,S: 2. 定期状态同步 (每30秒)
    loop 每30秒
        W->>WM: sync_worker_status()
        WM->>WM: WorkerStatusCollector.collect()
        WM->>CS: workers.update(id, worker_status)
        CS->>S: PUT /workers/{id}
        S->>DB: 更新 worker 状态
    end
    
    Note over W,S: 3. 模型实例事件监听
    W->>SM: watch_model_instances()
    SM->>CS: model_instances.awatch()
    CS->>S: WebSocket /model_instances/watch
    
    loop 模型实例变化
        S-->>CS: Event{type: ADDED/MODIFIED/DELETED, data: ModelInstance}
        CS-->>SM: _handle_model_instance_event()
        
        alt 事件类型为 ADDED/MODIFIED
            SM->>SM: _start_serve_process()
            SM->>Backend: 启动推理后端进程
        else 事件类型为 DELETED
            SM->>Backend: _stop_serve_process()
        end
    end
    
    Note over W,S: 4. 模型文件下载管理
    W->>MFM: watch_model_files()
    MFM->>CS: model_files.awatch()
    CS->>S: WebSocket /model_files/watch
    
    loop 模型文件变化
        S-->>CS: Event{type, data: ModelFile}
        CS-->>MFM: _handle_model_file_event()
        MFM->>MFM: download_model()
    end
    
    Note over W,S: 5. 健康检查 (每3秒)
    loop 每3秒
        W->>SM: health_check_serving_instances()
        SM->>Backend: is_ready(backend, model_instance)
        alt 健康检查失败
            SM->>CS: model_instances.update()
            CS->>S: PUT /model_instances/{id}
            S->>DB: 更新实例状态
        end
    end
    
    Note over W,S: 6. 调度和请求路由
    S->>Scheduler: 模型部署请求
    Scheduler->>Scheduler: find_candidate() 选择合适的 worker
    Scheduler->>Controllers: 创建 ModelInstance
    Controllers->>DB: 保存 ModelInstance 分配给 worker
    
    Note over W,S: 7. 推理请求代理
    Client->>S: POST /v1/chat/completions
    S->>S: 根据模型路由到对应 worker
    S->>W: HTTP 代理请求到 worker:{port}
    W->>Backend: 转发到推理后端进程
    Backend-->>W: 流式响应
    W-->>S: 转发响应
    S-->>Client: 转发响应
```

