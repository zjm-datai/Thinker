
**如果你希望每次实例都生成一个新值（即使该值类型不可变），使用 `default_factory` 是更专业、更可维护的选择**。

### 为什么需要？

在 Python 中，使用可变对象（如 list 或 dict）作为函数或类默认参数时，往往会导致多个实例共享同一个默认对象。例如：

```python
def fn(x=[]):
    x.append(1)
    return x

print(fn())  # [1]
print(fn())  # [1, 1] —— 意外的共享状态
```

为了解决这个常见的坑，Python 的 `dataclasses` 提供了 `default_factory`，强制每次生成一个新的默认对象。Pydantic 作为更先进的数据验证工具，也采纳了该机制，以保护开发者免受这种 bug 的影响。

