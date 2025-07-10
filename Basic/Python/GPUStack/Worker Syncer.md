---
tags: gpustack
---

## 一、业务流程回顾与挑战

1. **定时健康检查**
    
    - 每隔固定间隔（默认 15 秒），对所有注册的 Worker 发起一次 `/healthz` 请求。

2. **在线性与可达性双重阈值**
    
    - 若某次请求在可达性超时（默认 10 秒）内无应答，则将 Worker 标记为 **unreachable**。
        
    - 若累计离线时间超过更长的离线阈值（默认 100 秒），则将其状态降级为 **NOT_READY**。

3. **最小化写库开销**
    
    - 只有当 `unreachable` 标志、`state` 枚举或 `state_message` 真实变化时，才持久化更新。

4. **容错与自恢复**
    
    - 整轮检查包裹在最外层 `try/except` 中，单次故障不会让整个服务宕机。

尽管整体流程合理，以下技术改进可让该同步器在高并发、分布式场景下更具鲁棒性、可观察性和可维护性。

---

## 二、原始实现的技术细节

### 1. 周期性调度与异常保护

```python
async def start(self):
    while True:
        await asyncio.sleep(self._interval)
        try:
            await self._sync_workers_connectivity()
        except Exception as e:
            logger.error(f"Failed to sync workers: {e}")
```

- **调度间隔**：通过 `asyncio.sleep(self._interval)` 以非阻塞方式定时发起健康检查。

- **容错机制**：最外层 `try/except` 捕获整轮同步的任何异常，避免单次错误导致服务挂掉。

一次同步轮次里我们只打开了一个 AsyncSession，然后把这个同一个 session 对象，传给了所有并发检查任务（`_check_worker_connectivity`）。这正是“多协程共用同一个事务句柄”会触发 `InterfaceError: another operation is in progress` 、或更糟糕的脏读/死锁等并发问题的经典场景。


### 2. 并发拉取与批量检查

```python
async with AsyncSession(self._engine) as session:
    workers = await Worker.all(session)
    tasks = [ self._check_worker_connectivity(w, session) for w in workers ]
    results = await asyncio.gather(*tasks)
```

- **单会话复用**：为一整个检查轮次创建一个 `AsyncSession`。

- **并发执行**：通过 `asyncio.gather` 并行地对所有 Worker 发起检查请求，等待最慢的那一个完成。


### 3. 单 Worker 的检查与状态计算

```python
async def _check_worker_connectivity(self, worker, session):
    orig_unreach = worker.unreachable
    orig_state   = worker.state
    orig_msg     = worker.state_message

    unreachable = not await self.is_worker_reachable(worker)
    worker = await Worker.one_by_id(session, worker.id)  # 重载以防“会话陈旧”
    worker.unreachable = unreachable
    worker.compute_state(self._worker_offline_timeout)

    if (orig_unreach != worker.unreachable
        or orig_state   != worker.state
        or orig_msg     != worker.state_message):
        return worker
    return None
```

- **重载逻辑**：再次通过 `one_by_id` 拉取最新行，避免前后数据不一致。

- **状态驱动**：`compute_state(timeout)` 根据 `unreachable` 及“距上次在线”的时间阈值决定 `READY`、`UNREACHABLE` 或 `NOT_READY`。


### 4. 最终写库与日志

```python
for worker in should_update_workers:
    await WorkerService(session).update(worker)
```

- **差异化更新**：仅对状态确实发生变化的 Worker 执行写库，降低数据库压力。

- **调试日志**：输出本轮被标记为各状态的 Worker 名单，辅助线上排查。

---

## 三、存在的风险与改进方向

1. **会话并发复用的隐患**  
    SQLAlchemy 明确指出——每个 asyncio 任务都应使用独立的 `AsyncSession`，否则会因为多协程共用同一事务对象而触发 `InterfaceError: another operation is in progress` 等竞态问题 ([SQLAlchemy 文档](https://docs.sqlalchemy.org/en/latest/orm/session_basics.html?utm_source=chatgpt.com "Session Basics — SQLAlchemy 2.0 Documentation"))。

[[AsyncSession]] 

2. **未显式事务边界与提交**  
    原实现依赖 `WorkerService.update` 隐式提交，外部调用方难以把握事务何时完成，增加调试难度。

3. **竞态覆盖的可能性**  
    “在同一 Session 重载最新行” 有助减少卡住的会话，但无法防止几乎同时的两轮检查对同一行做出冲突更新。业内更可靠的做法是引入**乐观锁**（`version_id`）保证并发写入时的原子性 ([SQLAlchemy 文档](https://docs.sqlalchemy.org/en/latest/orm/versioning.html?utm_source=chatgpt.com "Configuring a Version Counter — SQLAlchemy 2.0 Documentation"))。

4. **可测试性不足**  
    直接在 `compute_state` 内部调用系统时钟不利于单元测试。应将时间获取抽象为可注入的函数，模拟各种超时场景。

5. **可观测性与精细化监控缺失**  
    建议为 HTTP 超时、写库失败等关键环节埋点 Prometheus 指标，配合告警触发，实现真正的在线故障检测与容量规划。

---

## 四、改进后实现要点

### 4.1 “一任务一 Session” + 短事务

```python
from sqlalchemy.ext.asyncio import async_sessionmaker

class WorkerSyncer:
    def __init__(..., now_fn: Callable[[], datetime] = datetime.utcnow):
        self._session_maker = async_sessionmaker(self._engine, expire_on_commit=False)
        self._now_fn = now_fn

    async def _check_and_update(self, worker_id: int):
        async with self._session_maker() as session:
            async with session.begin():
                worker = await Worker.one_by_id(session, worker_id)
                orig = (worker.unreachable, worker.state)
                reachable = await self.is_worker_reachable(worker)
                worker.unreachable = not reachable
                worker.compute_state(self._worker_offline_timeout, now_fn=self._now_fn)
                if (worker.unreachable, worker.state) != orig:
                    session.add(worker)  # 乐观锁检测 version_id
```

- **独立短事务**：每个 Worker 检查及更新都在自己的事务上下文内完成，确保隔离性。
    
- **显式提交/回滚**：`async with session.begin()` 自动管理事务。
 

### 4.2 乐观锁保证并发安全

在模型中增加版本列：

```python
class Worker(SQLModel, table=True):
    __tablename__ = "worker"
    # …
    version_id: int = Field(sa_column=Column(Integer, nullable=False), default=1)
    __mapper_args__ = {"version_id_col": version_id}
```

- 并发更新时，如果版本不匹配，SQLAlchemy 会抛出 `StaleDataError`，可捕获重试或记录异常。


### 4.3 时间源与可测试性

```python
def compute_state(self, offline_timeout: int, now_fn: Callable[[], datetime]):
    elapsed = (now_fn() - self.last_seen).total_seconds()
    # 根据 elapsed 与 unreachable 决定 state
```

- 测试时传入固定 `now_fn=lambda: fixed_datetime`，精准验证超时边界。


### 4.4 精细化异常处理与指标埋点

```python
try:
    reachable = await self.is_worker_reachable(worker)
except TimeoutError:
    reachable = False
    metrics.worker_ping_timeout.inc()
# …
metrics.worker_sync_round_failures.inc()  # 在 _run_one_round 最外层统计
```

- 分层捕获 HTTP、数据库异常；配合 Prometheus 指标，便于量化和告警。

---

## 五、结语

通过上述改进，`WorkerSyncer` 达到了：

- **并发安全**：严格遵循 “AsyncSession per task” 模式和乐观锁，消除共享会话与竞态更新隐患 ([SQLAlchemy 文档](https://docs.sqlalchemy.org/en/latest/orm/session_basics.html?utm_source=chatgpt.com "Session Basics — SQLAlchemy 2.0 Documentation"), [SQLAlchemy 文档](https://docs.sqlalchemy.org/en/latest/orm/versioning.html?utm_source=chatgpt.com "Configuring a Version Counter — SQLAlchemy 2.0 Documentation"))。

- **事务清晰**：显式短事务边界，令持久化逻辑一目了然。

- **高可测试性**：时间函数注入与模块化设计，可覆盖各种超时与状态转换场景。

- **深度可观测性**：细粒度日志与指标，为生产环境的稳定性与容量规划提供数据支撑。

- **扩展友好**：新增状态、告警、通知插件等，无需改动核心框架。


这样，当集群规模从几十扩展到上千台 Worker 时，仍能保持高可靠性、易维护和可扩展的状态同步能力。

## 六、代码附录

原本的完整代码

```python
import asyncio
import logging
from sqlmodel.ext.asyncio.session import AsyncSession
from gpustack.schemas.workers import Worker, WorkerStateEnum
from gpustack.server.db import get_engine
from gpustack.server.services import WorkerService
from gpustack.utils.network import is_url_reachable

logger = logging.getLogger(__name__)

class WorkerSyncer:
    """
    WorkerSyncer syncs worker status periodically.
    """

    def __init__(
        self, interval=15, worker_offline_timeout=100, worker_unreachable_timeout=10
    ):

        self._engine = get_engine()
        self._interval = interval
        self._worker_offline_timeout = worker_offline_timeout
        self._worker_unreachable_timeout = worker_unreachable_timeout

    async def start(self):
        while True:
            await asyncio.sleep(self._interval)
            try:
                await self._sync_workers_connectivity()
            except Exception as e:
                logger.error(f"Failed to sync workers: {e}")

    async def _sync_workers_connectivity(self):
        """
        Mark offline workers to not_ready state.
        """

        async with AsyncSession(self._engine) as session:
            workers = await Worker.all(session)
            if not workers:
                return

            tasks = [
                self._check_worker_connectivity(worker, session) for worker in workers
            ]

            results = await asyncio.gather(*tasks)

            should_update_workers = []
            state_to_worker_name = {
                WorkerStateEnum.NOT_READY: [],
                WorkerStateEnum.UNREACHABLE: [],
                WorkerStateEnum.READY: [],
            }

            for worker in results:
                if worker:
                    should_update_workers.append(worker)
                    state_to_worker_name[worker.state].append(worker.name)

            for worker in should_update_workers:
                await WorkerService(session).update(worker)

            for state, worker_names in state_to_worker_name.items():
                if worker_names:
                    logger.debug(f"Marked worker {', '.join(worker_names)} as {state}")

    async def _check_worker_connectivity(self, worker: Worker, session: AsyncSession):
        original_worker_unreachable = worker.unreachable
        original_worker_state = worker.state
        original_worker_state_message = worker.state_message

        unreachable = not await self.is_worker_reachable(worker)
        worker = await Worker.one_by_id(session, worker.id)
        worker.unreachable = unreachable
        worker.compute_state(self._worker_offline_timeout)

        if (
            original_worker_unreachable != worker.unreachable
            or original_worker_state != worker.state
            or original_worker_state_message != worker.state_message
        ):
            return worker
        return None

    async def is_worker_reachable(
        self,
        worker: Worker,
    ) -> bool:
        healthz_url = f"http://{worker.ip}:{worker.port}/healthz"
        reachable = await is_url_reachable(
            healthz_url,
            self._worker_unreachable_timeout,
        )
        return reachable
```

