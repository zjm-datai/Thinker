
在 python 入参中 * 是一个特殊的分隔符号，在它之后的所有参数都必须作为关键字参数传入。

---

inspect 是 python 标准库中的一个模块，主要用于检查和分析 python 对象的信息，尤其是函数、类、方法等可调用对象的结构和属性。

inspect.iscoroutinefunction(func) 的作用是判断传入的 func 是否是一个异步协程函数（即用 async def 定义的函数）

---

对于不带参数的装饰器 @tool_api

等价于：

```python
def my_func():
	...

my_func = tool_api(my_func) # 直接调用 tool_api，参数是 my_func
```

这种情况下，tool_api 可以直接处理 func，生成新的 wrapper 并返回

对于带参数的装饰器 @tool_api(explode_return=True)

此时等价于：

```python
def my_func():
	...
# 第一步：先调用 tool_api(explode_return=True)，得到一个“临时装饰器”
temp_decorator = tool_api(explode_return=True)
# 第二步：用这个临时装饰器处理 my_func
my_func = temp_decorator(my_func)
```

这里的 temp_decorator 必须是一个**能接收 func 作为参数的函数**，而 decorate 正是这个 “临时装饰器”。

---

@wraps(func) 是 python functools 模块中的一个装饰器，它的主要作用是保留被装饰函数（或方法）的元信息。

```python
@wraps(func)
def wrapper(self, *args, **kwargs):
	return func(self, *args, **kwargs)
```

它的具体作用在于：当 wrapper 函数装饰 func 的时候，@wraps(func) 会将 func 的元信息（如函数名 `__name__` 、文档字符串 `__doc__` 、参数列表 `__annotations__` 等）复制到 wrapper 函数中去。

如果不使用 @wraps(func)，装饰后的函数 wrapper 会掩盖原函数 func 的身份信息，例如：

`print(wrapper.__name__)` 会输出 wrapper 而不是原函数名

原函数的文档字符串也会丢失

---

