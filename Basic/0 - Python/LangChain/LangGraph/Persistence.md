

## Threads

A thread is a unique ID or thread identifier assigned to each checkpoint saved by a checkpointer. It contains the accumulated state of a sequence of runs. When a run is executed, the state of the underlying graph of the assistant will be persisted to the thread.

When invoking graph with a checkpointer, you must specify a thread_id as part of the configurable portion of the config:

```python
{"configurable": {"thread_id": "1"}}
```

A thread's current and historical state can be retrieved. To persist state, a thread must be created prior to executing a run. The LangGraph Platform API provides several endpoints for creating and managing threads and thread state. See the API reference for more details.

## Checkpoints



## Checkpointer libraries

在底层，检查点功能由符合 BaseCheckpointSaver 接口的检查点对象提供支持。LangGraph 提供了几种检查点实现，所有这些实现都通过独立的、可安装的库来完成：

`langgraph-checkpoint` 

`langgraph-checkpoint-sqlite`

`langgraph-chekpoint-postgres`

### Chekpointer interface

Each chekpointer conforms to BaseCheckpointSaver interface and implements the following methods:

`.put` Store a checkpoint with its configuration and metadata.

```
### Serializer

当检查点保存图状态的时候，需要序列化状态中的 channel values 。这个过程我们通过序列器对象来完成。`langgraph_checkpoint` 定义了实现序列器的协议，并提供了一个默认的实现 `JsonPlusSerializer` ，这个实现可以处理多种类型，including LangChain and LangGraph primitives, datetimes, enums and more.
#### Serialization with `pickle`

The default serializer `JsonPlusSerializer` , uses ormsgpack and JSON under the hood, which is not suitable for all types of objects.

If you want to fallback to pickle for objects not currently supported by our msgpack encoder (such as Pandas dataframes), you can use the `pickle_fallback` argument of the `JsonPlusSerializer`:
```

## Capabilities

### Human-in-the-loop

首先，检查点工具通过允许人类检查，中断和批准图步骤，促进了 human-in-the-loop 的能力。这些工作流程需要检查点工具，因为用户需要可以在任何时间点查看图的状态，并且在人类对状态进行任何更新后，图必须能够恢复执行。有关示例，请参阅操作指南。