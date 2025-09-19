
太好了，这里确实留了个“口子”：`emit_signal()` 里只做了 `enable_vad = True`，并写了 TODO。给你一个**可直接落地的信号机制设计**，尽量贴合你现有的线程 + 队列架构（不用引入 asyncio 也能跑）。

# 核心思路

- 新增一个**SignalBus（信号总线）**：独立于数据分发的“轻量事件系统”，专门传递控制类事件（BEGIN/INTERRUPT/END/VAD_ON/OFF/参数更新/取消等）。
    
- 采用**发布-订阅**模型：handler（或其它组件）按信号类型进行订阅；SignalBus 负责匹配与投递。
    
- **优先级 + 超时 +（可选）ACK**：例如 `INTERRUPT` 要最高优先级；支持“需要 ACK”时等待订阅者确认（可选）。
    
- 每个订阅者收到信号的方式依旧是**队列**（保持你现有模式）：即“每个 handler 一个信号队列”，SignalBus 往里放信号。
    

---

## 1) 信号模型（在你现有 `ChatSignal` 基础上补字段）

```python
from dataclasses import dataclass, field
from enum import IntEnum
import time, uuid
from typing import Any, Callable, Dict, List, Optional, Set, Tuple

class SignalPriority(IntEnum):
    HIGH = 0     # 比数值小优先
    NORMAL = 1
    LOW = 2

@dataclass
class SignalEnvelope:
    signal: ChatSignal                    # 复用你现有的 ChatSignal
    id: str = field(default_factory=lambda: uuid.uuid4().hex)
    ts: float = field(default_factory=time.monotonic)
    priority: SignalPriority = SignalPriority.NORMAL
    target: Optional[Set[str]] = None     # 目标 handler 名集合；None 表示广播
    payload: Optional[Dict[str, Any]] = None
    need_ack: bool = False
    timeout_sec: float = 0.0              # 需要 ACK 才有意义
```

> 不想改 `ChatSignal` 结构的话，就像上面那样**用信封**包起来，额外信息都在 `SignalEnvelope` 里。

---

## 2) SignalBus：发布-订阅与分发线程

```python
import queue
from dataclasses import dataclass

@dataclass
class SignalSubscriber:
    owner: str
    types: Set[ChatSignalType]            # 订阅的信号类型集合
    filt: Optional[Callable[[SignalEnvelope], bool]] = None
    queue: queue.Queue = None             # 订阅者的“信号队列”

class SignalBus:
    def __init__(self):
        self._active = False
        self._pq = queue.PriorityQueue()  # (priority, ts, seq, SignalEnvelope)
        self._seq = 0
        self._subs: List[SignalSubscriber] = []
        self._ack_waiters: Dict[str, threading.Event] = {}
        self._thread: Optional[threading.Thread] = None
        self._lock = threading.Lock()

    def start(self):
        if self._active: return
        self._active = True
        self._thread = threading.Thread(target=self._run, daemon=True)
        self._thread.start()

    def stop(self):
        self._active = False
        if self._thread:
            self._thread.join()
            self._thread = None
        with self._lock:
            self._subs.clear()
            self._ack_waiters.clear()

    def subscribe(self, sub: SignalSubscriber):
        with self._lock:
            self._subs.append(sub)

    def unsubscribe_owner(self, owner: str):
        with self._lock:
            self._subs = [s for s in self._subs if s.owner != owner]

    def emit(self, env: SignalEnvelope):
        with self._lock:
            self._seq += 1
            item = (env.priority, env.ts, self._seq, env)
            if env.need_ack:
                self._ack_waiters[env.id] = threading.Event()
        self._pq.put_nowait(item)
        return env.id

    def ack(self, signal_id: str, owner: str):
        with self._lock:
            ev = self._ack_waiters.get(signal_id)
        if ev: ev.set()

    def wait_ack(self, signal_id: str, timeout: float) -> bool:
        with self._lock:
            ev = self._ack_waiters.get(signal_id)
        if not ev: return True
        ok = ev.wait(timeout)
        with self._lock:
            self._ack_waiters.pop(signal_id, None)
        return ok

    def _run(self):
        while self._active:
            try:
                _, _, _, env = self._pq.get(timeout=0.05)
            except queue.Empty:
                continue
            # 分发
            targets = []
            with self._lock:
                subs = list(self._subs)
            for s in subs:
                if env.signal.type not in s.types: 
                    continue
                if env.target and s.owner not in env.target:
                    continue
                if s.filt and not s.filt(env):
                    continue
                try:
                    s.queue.put_nowait(env)
                    targets.append(s.owner)
                except queue.Full:
                    # 你也可以记录告警或丢弃策略
                    pass
            # 这里不强制 ACK；由上层决定是否 wait_ack
```

---

## 3) ChatSession 集成点

- 在 `ChatSession.__init__` 里创建并启动：
    

```python
self.signal_bus = SignalBus()
self.signal_bus.start()
```

- 修改 `emit_signal()`：把临时逻辑替换为发布到总线（并**可选**等待 ACK）
    

```python
def emit_signal(self, signal: ChatSignal, *, 
                priority: SignalPriority = SignalPriority.NORMAL,
                target: Optional[List[str]] = None,
                payload: Optional[Dict] = None,
                need_ack: bool = False, timeout: float = 0.0):
    env = SignalEnvelope(
        signal=signal, priority=priority,
        target=set(target) if target else None,
        payload=payload, need_ack=need_ack, timeout_sec=timeout
    )
    sid = self.signal_bus.emit(env)
    if need_ack and timeout > 0:
        ok = self.signal_bus.wait_ack(sid, timeout)
        if not ok:
            logger.warning(f"Signal {signal.type} ack timeout")
    # 兼容旧语义（例如 END → VAD enable）
    if signal.source_type == ChatSignalSourceType.CLIENT and signal.type == ChatSignalType.END:
        self.session_context.shared_states.enable_vad = True
```

- 在 `prepare_handler()` 之后，**给每个 handler 也配一个“信号队列”并订阅**（放进它的 context，便于在 `handle()` 或内部线程里读）：
    

```python
handler_env.context.signal_queue = queue.Queue()
sub = SignalSubscriber(
    owner=handler_info.name,
    types=handler_env.handler.get_subscribed_signals(),  # 新增接口，或写死一组
    queue=handler_env.context.signal_queue
)
self.signal_bus.subscribe(sub)
```

> 如果不想给 `HandlerBase` 增口，可以约定：**所有 handler 默认订阅 `INTERRUPT`**，其它按需在 handler 的 `create_context` 里注册。

---

## 4) handler 如何“感知信号”与 ACK？

- 方式 A（轮询非阻塞，贴合你现状）：
    

```python
try:
    env = handler_env.context.signal_queue.get_nowait()
    if env.signal.type == ChatSignalType.INTERRUPT:
        handler.cancel_current_job()  # 你自定义
        # （可选）发送 ACK
        handler_env.context.session.signal_bus.ack(env.id, handler_env.handler_info.name)
except queue.Empty:
    pass
```

把这段检查，放到 `handler_pumper` 取到 `input_data` 前后，或 handler 内部的工作循环中。

- 方式 B（独立线程/回调）：  
    让 handler 起个“信号监听线程”，专门阻塞 `get()`，收到就打断当前处理。
    

---

## 5) 优先级 & 典型信号约定

建议约定（可扩展）：

- `INTERRUPT`：`HIGH`，用于打断 ASR/LLM/TTS（目标可为空→广播；或点名 target 列表）。
    
- `BEGIN`：`NORMAL`，开始一轮响应（你在 RTC 的 `chat` 分支里正发这个）。
    
- `END`：`NORMAL`，一轮交互结束（END→VAD on 可以保持兼容）。
    
- `PARAM_UPDATE`：`LOW`，例如动态调整 TTS 语速、VAD 阈值等（携带 `payload`）。
    
- `HEARTBEAT`/`ERROR`：按需定义。
    

---

## 6) 与你现有 RTC 桥接

- 在 `RtcStream.set_channel()` 的 DataChannel 回调里，你已有：
    
    - `stop_chat` → `INTERRUPT`
        
    - `chat` → `BEGIN`
        

把这些调用从 `client_session_delegate.emit_signal(...)` 替换/同时转发到 `ChatSession.emit_signal(...)`（如果你在更上层管理 session/bus），或在 `ClientSessionDelegate.emit_signal` 里**转发**到 `ChatSession.signal_bus.emit(...)`，保证一处对外、一处对内。

---

## 7) 一个端到端小例子（打断 TTS）

- 前端点了“stop_chat” → `INTERRUPT`（HIGH, broadcast, need_ack=True, timeout=0.5）
    
- SignalBus 把信号投递到订阅了 `INTERRUPT` 的 `H_TTS` / `H_LLM` / `H_ASR` 的 signal_queue
    
- `H_TTS` 收到后立刻停止拉取数据/关闭音频推流，并 `ack(signal_id)`
    
- `ChatSession.emit_signal(..., need_ack=True, timeout=0.5)` 返回前就能知道是否**全部**关键组件都响应了；若超时，记日志并做补救（比如强制清空输出队列）
    

---

## 8) 为什么用“信号总线”而不是复用数据分发？

- **语义不同**：信号是控制面（control plane），数据是承载面（data plane）。
    
- **时效性**：需要优先级（例如 `INTERRUPT` 抢占），独立队列才能确保低延迟。
    
- **解耦**：不污染 `ChatData` 的语义与流控；handler 的输入数据队列与信号队列分离，避免被大数据量“淹没”。
    

---

## 9) 最小落地清单

1. 新增 `SignalEnvelope / SignalPriority / SignalBus` 三个类。
    
2. `ChatSession.__init__` 里创建 `self.signal_bus.start()`。
    
3. 扩展 `prepare_handler()`：为每个 handler 注册 `SignalSubscriber`（并把 `signal_queue` 放进 context）。
    
4. 改造 `emit_signal()`：走 `SignalBus.emit()`（可选 ACK）。
    
5. 在 `handler_pumper` 或 handler 内部循环里**非阻塞**检查 `signal_queue` 并处理打断等。
    

做到这 5 步，你的“信号机制”就具备：**订阅、优先级、广播/定向、ACK**，而且与现有线程/队列模型完全兼容。需要的话我可以按你项目的类名/路径把补丁代码再精简成可直接拷贝的版本。