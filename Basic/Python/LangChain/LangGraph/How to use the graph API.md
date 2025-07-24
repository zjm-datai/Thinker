
This guide demonstrates the basics of langgraph's graph api. It walks through state, as well as composing common graph structures such as sequnces, branches, and loops. It also covers Langgraph's control features, including the send api for map-reduce workflows and command api for combining state updates with loops across nodes.

## Setup

```
pip install langgraph
```

## Define and update state

Here we show how to define and update state in langgraph. We will demonstrate:

1. How to use state to define a graph's schema
2. How to use reducers to control how state updates are processed

### Define state

State in langgraph can be a  `TypedDict` , `Pydantic` model, or dataclass. Below we will use `TypedDict`. See this section for detail on using Pydantic.

By default, graphs will have the same input and output schema, and the state determines that schema. See this section for how to define distinct input and output schemas.

Let us consider a simple example using messages. This represents a versatile formulation of state for many LLM applications. See our concepts page for more detail.

```python
from langchain_core.messages import AnyMessage
from typing_extensions import TypedDict

class State(TypedDict):
	messages: list[AnyMessage]
	extra_field: int
```

This state tracks a list of message objects, as well as an extra integer field.

### Update state

Let us build an example graph with a single node. Our node is just a python function that reads our graph's state and makes updates to it. The first argument to this function will always be the state:

```python
from langchain_core.messages import AIMessage

def node(state: State):
	messages = state["messages"]
	new_message = AIMessage("Hello!")
	return {
		"messages": messages + [new_messages],
		"extra_field": 10
	}
```

This node simply appends a message to our message list, and populates an extra field.

>[!important]
>Nodes should return updates to the state directly, instead of mutating the state.

### 使用 reducer 处理状态更新

状态中的每个键都可以有自己独立的 reducer 函数，用于控制如何应用来自节点的更新。如果没有明确指定 reducer 函数，则默认所有对该键的更新都会直接覆盖它。

对于 TypedDict 状态模式，我们可以通过用 reducer 函数注释状态的相应字段来定义 reducer。

在之前的示例中，我们的节点通过向状态中的 “messages” 键追加消息来更新该键。下面，我们为这个键添加一个 reducer，使得更新会自动追加：

```python
from typing_extensions import Annotated

def add(left, right):
    """也可以从 `operator` 内置模块中导入 `add`。"""
    return left + right

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add]
    extra_field: int
```

现在我们的节点可以简化：

```python
def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}
```

```python
from langgraph.graph import START

graph = StateGraph(State).add_node(node).add_edge(START, "node").compile()

result = graph.invoke({"messages": [HumanMessage("Hi")]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

### MessagesState

在实际应用中，更新消息列表时还需要考虑其他因素：
- 我们可能希望更新状态中已有的消息。
- 我们可能希望接受消息格式的简写形式，例如 OpenAI 格式。

LangGraph 包含一个内置的 reducer `add_messages`，可以处理这些情况：

API 参考：add_messages

```python
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    extra_field: int

def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}

graph = StateGraph(State).add_node(node).set_entry_point("node").compile()
```

```python
input_message = {"role": "user", "content": "Hi"}

result = graph.invoke({"messages": [input_message]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

这是一种适用于涉及聊天模型的应用程序的通用状态表示。LangGraph 包含一个预构建的 `MessagesState` 以方便使用，因此我们可以这样写：

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    extra_field: int
```

### 定义输入和输出模式

默认情况下，StateGraph 使用单一模式运行，所有节点都需要使用该模式进行通信。不过，也可以为图定义不同的输入和输出模式。

当指定了不同的模式时，节点之间的通信仍会使用内部模式。输入模式确保提供的输入符合预期结构，而输出模式则会根据定义的输出模式过滤内部数据，只返回相关信息。

下面我们将了解如何定义不同的输入和输出模式。

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 定义输入的模式
class InputState(TypedDict):
    question: str

# 定义输出的模式
class OutputState(TypedDict):
    answer: str

# 定义整体模式，结合输入和输出
class OverallState(InputState, OutputState):
    pass

# 定义处理输入并生成答案的节点
def answer_node(state: InputState):
    # 示例答案和一个额外的键
    return {"answer": "bye", "question": state["question"]}

# 构建指定了输入和输出模式的图
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)  # 添加答案节点
builder.add_edge(START, "answer_node")  # 定义起始边
builder.add_edge("answer_node", END)  # 定义结束边
graph = builder.compile()  # 编译图

# 使用输入调用图并打印结果
print(graph.invoke({"question": "hi"}))
```

```
{'answer': 'bye'}
```

注意，`invoke` 的输出仅包含输出模式中定义的字段。

### 在节点间传递私有状态

在某些情况下，你可能希望节点之间交换一些对中间逻辑至关重要，但无需纳入图的主模式的信息。这些私有数据与图的整体输入/输出无关，仅应在特定节点之间共享。

下面，我们将创建一个包含三个节点（node_1、node_2 和 node_3）的顺序图示例。其中，私有数据在最开始的两个步骤（node_1 和 node_2）之间传递，而第三个步骤（node_3）只能访问公共的整体状态。

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 图的整体状态（这是在节点间共享的公共状态）
class OverallState(TypedDict):
    a: str

# node_1 的输出包含不属于整体状态的私有数据
class Node1Output(TypedDict):
    private_data: str

# 私有数据仅在 node_1 和 node_2 之间共享
def node_1(state: OverallState) -> Node1Output:
    output = {"private_data": "由 node_1 设置"}
    print(f"进入节点 `node_1`：\n\t输入：{state}。\n\t返回：{output}")
    return output

# node_2 的输入仅获取 node_1 执行后可用的私有数据
class Node2Input(TypedDict):
    private_data: str

def node_2(state: Node2Input) -> OverallState:
    output = {"a": "由 node_2 设置"}
    print(f"进入节点 `node_2`：\n\t输入：{state}。\n\t返回：{output}")
    return output

# node_3 只能访问整体状态（无法访问 node_1 的私有数据）
def node_3(state: OverallState) -> OverallState:
    output = {"a": "由 node_3 设置"}
    print(f"进入节点 `node_3`：\n\t输入：{state}。\n\t返回：{output}")
    return output

# 按顺序连接节点
# node_2 接收来自 node_1 的私有数据，而 node_3 看不到该私有数据
builder = StateGraph(OverallState).add_sequence([node_1, node_2, node_3])
builder.add_edge(START, "node_1")
graph = builder.compile()

# 用初始状态调用图
response = graph.invoke(
    {
        "a": "在开始时设置",
    }
)

print()
print(f"图调用的输出：{response}")
```

```
进入节点 `node_1`：
    输入：{'a': '在开始时设置'}。
    返回：{'private_data': '由 node_1 设置'}
进入节点 `node_2`：
    输入：{'private_data': '由 node_1 设置'}。
    返回：{'a': '由 node_2 设置'}
进入节点 `node_3`：
    输入：{'a': '由 node_2 设置'}。
    返回：{'a': '由 node_3 设置'}

图调用的输出：{'a': '由 node_3 设置'}
```

### 将 Pydantic 模型用于图状态

`StateGraph` 在初始化时接受一个 `state_schema` 参数，用于指定图中节点可访问和更新的状态“形状”。

在我们的示例中，通常使用 Python 原生的 `TypedDict` 或 `dataclass` 作为 `state_schema`，但 `state_schema` 可以是任何类型。

这里，我们将展示如何使用 Pydantic 的 `BaseModel` 作为 `state_schema`，以对输入进行运行时验证。

**已知限制**
- 目前，图的输出不会是 Pydantic 模型的实例。
- 运行时验证仅发生在节点的输入上，而非输出上。
- Pydantic 的验证错误跟踪不会显示错误出现在哪个节点。
- Pydantic 的递归验证可能较慢。对于性能敏感的应用，你可能需要考虑使用 `dataclass` 替代。

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
from pydantic import BaseModel

# 图的整体状态（这是在节点间共享的公共状态）
class OverallState(BaseModel):
    a: str

def node(state: OverallState):
    return {"a": "goodbye"}

# 构建状态图
builder = StateGraph(OverallState)
builder.add_node(node)  # node 是第一个节点
builder.add_edge(START, "node")  # 从 node 开始图
builder.add_edge("node", END)  # 在 node 之后结束图
graph = builder.compile()

# 用有效的输入测试图
graph.invoke({"a": "hello"})
```

**用无效输入调用图**

```python
try:
    graph.invoke({"a": 123})  # 应该是字符串
except Exception as e:
    print("由于 `a` 是整数而非字符串，引发了异常。")
    print(e)
```

```
由于 `a` 是整数而非字符串，引发了异常。
1 validation error for OverallState
a
  Input should be a valid string [type=string_type, input_value=123, input_type=int]
    For further information visit https://errors.pydantic.dev/2.9/v/string_type
```

