
## ToolMeta

我们定义了一个元类 `ToolMeta`，继承自 `ABCMeta`。元类负责**控制类的创建过程**，尤其在类体定义结束、类对象创建之前执行。

方法签名为：

```python
def __new__(mcs, name, base, attrs):
```

这个方法的4个参数都是由 **Python解释器在类创建时自动传入**的：

- `mcs` 是当前元类本身，类似于实例方法中的 `self`，表示的是 `ToolMeta` 这个元类对象。
    
- `name` 是正在创建的类的名字，字符串形式，比如 `"MyTool"`。
    
- `base` 是一个元组，包含了所有父类，例如你定义的类是 `class MyTool(BaseTool)`，那 `base` 就是 `(BaseTool,)`。
    
- `attrs` 是一个字典，包含了类体中定义的所有属性（包括方法、类变量、`__doc__` 等）。

---

接下来看元类逻辑部分的作用：

### 第一段初始化：

```python
is_toolkit, tool_desc = True, dict(
    name=name,
    description=Docstring(attrs.get('__doc__', '')).parse('google')[0].value
)
```

- `is_toolkit` 是一个布尔标记，用来判断这个类是否是一个“工具集”（toolkit，内部可能含有多个接口），初始设为 True。
    
- `tool_desc` 是这个工具类的描述信息字典，初始包含类名和从 docstring 中解析出来的描述信息。
    
- 使用了 `Docstring(...).parse('google')` 的方式解析 docstring，说明使用了某种标准格式（Google-style docstring）。
    

---

### 遍历类中的所有属性

```python
for key, value in attrs.items():
```

- 这里循环检查类中定义的所有方法和属性。
    
- 如果某个属性是可调用的（即函数或方法）并且拥有 `api_description` 属性，那么认为它是一个“API 方法”。

对于这些方法：

```python
api_desc = getattr(value, 'api_description')
```

拿到这个方法的描述信息。

- 如果这个方法的名字是 `run`，就认为这是主接口（非工具集），将它的参数列表、必填字段、描述信息等提取出来，填入 `tool_desc` 中，并标记为不是 toolkit。
    
- 如果不是 `run` 方法，就把这个方法描述加入到 `api_list` 里，认为是 toolkit 中的一个工具方法。

---

### 冲突检测

```python
if not is_toolkit and 'api_list' in tool_desc:
    raise KeyError('`run` and other tool APIs can not be implemented at the same time')
```

- 如果一个类同时定义了 `run` 方法（作为主接口）又定义了其他带 `api_description` 的方法，就会抛出错误。
    
- 因为不能既是主接口又是 toolkit，语义冲突。

---

### 第二次判断是否 toolkit

```python
if is_toolkit and 'api_list' not in tool_desc:
```

- 这是个 fallback 判断，如果没有明确列出 API 列表，也没有设置为非 toolkit，就重新尝试把 `run` 变成一个 API 方法。

下面这部分：

```python
if callable(attrs.get('run')):
    run_api = tool_api(attrs['run'])
```

- 如果 `run` 是一个方法，就用 `tool_api(...)` 包装它（说明 `tool_api` 是一个装饰器或函数，它会生成带有 `api_description` 属性的函数）。
    
- 把 API 描述提取出来，填到 `tool_desc` 中。
    
- 再把 `attrs['run']` 替换为 `run_api`，也就是说：用包装后的函数替代原始的 `run` 方法。

如果 `run` 方法不存在：

```python
tool_desc['parameters'], tool_desc['required'] = [], []
```

说明该工具没有参数输入。

### 最后两步

```python
attrs['_is_toolkit'] = is_toolkit
attrs['__tool_description__'] = tool_desc
```

- 在最终的类中添加两个属性：
    
    - `_is_toolkit`: 用于标记这个类到底是不是 toolkit。
        
    - `__tool_description__`: 存放提取出来的结构化工具描述信息，方便后续系统使用，比如自动生成文档、验证输入等。

---

### 调用父类的 **new**

```python
return super().__new__(mcs, name, base, attrs)
```

这一步是必要的，它实际**创建这个类对象**，必须返回一个类（type）对象。没有它，这个元类不会完成类的构造。

### 总结功能

这个元类的作用是：

- 自动识别和解析类中定义的 API 方法（基于是否有 `api_description`）。
    
- 支持两种风格的定义：一个主接口（`run` 方法）或者多个子接口（toolkit 风格）。
    
- 自动提取参数和描述信息生成一个 `tool_desc`。
    
- 强制不允许两种风格混用（否则报错）。
    
- 把结构化的工具说明保存在类属性中，供后续使用。

## BaseAction

```python
class BaseAction(metaclass=ToolMeta):
```

这表示 BaseAction 使用的是 ToolMeta 元类。也就是说，在 python 创建这个类时，会触发 `ToolMeta.__new__` 来构造这个类对象。这个元类会自动分析这个类的 docstring 和方法（如 run 或 @tool_api 装饰的），提取描述信息，保存在 `__tool_description__` 等属性中。

