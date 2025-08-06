---
tags: python
---

在 python 中 `dict.copy` 是一个字典对象的内置方法，用于创建当前字典的 **浅拷贝** shallow copy 。

他将返回一个和原字典包含相同键值对的新字典：

```python
original = {"name": "Alice", "age": 30}
copied = original.copy()

print(copied)  # 输出: {'name': 'Alice', 'age': 30}
print(original is copied)  # 输出: False（新字典是独立对象）
print(original == copied)  # 输出: True（键值对完全相同）
```

### 为什么需要 copy 方法？

直接赋值并不会创建新字典，而是让新字典指向 **原字典的引用**。此时修改新字典会同时影响原字典，而 `copy()` 可以避免这种情况。

```python
# 直接赋值（引用传递）
original = {"name": "Alice"}
ref = original
ref["age"] = 30
print(original)  # 输出: {'name': 'Alice', 'age': 30}（原字典被修改）

# 使用 copy()（值拷贝）
original = {"name": "Alice"}
copied = original.copy()
copied["age"] = 30
print(original)  # 输出: {'name': 'Alice'}（原字典不受影响）
```

### 浅拷贝（shallow copy）的特性

`copy()` 方法创建的是 **浅拷贝**，这意味着：

- 对于字典中的 **不可变值（如字符串、数字、元组）**，会复制其值到新字典。
- 对于字典中的 **可变值（如列表、其他字典）**，只会复制其引用（即新字典和原字典中的可变值指向同一个对象）。

```python
original = {
    "name": "Alice",
    "hobbies": ["reading", "coding"]  # 可变值（列表）
}
copied = original.copy()

# 修改不可变值：不影响原字典
copied["name"] = "Bob"
print(original["name"])  # 输出: Alice（原字典不变）

# 修改可变值：原字典会受影响（因为共享引用）
copied["hobbies"].append("gaming")
print(original["hobbies"])  # 输出: ['reading', 'coding', 'gaming']（原字典被修改）
```

### 与深拷贝（deep copy）的区别

如果需要完全独立的拷贝（包括嵌套的可变对象），需要使用 `copy` 模块的 `deepcopy()` 方法：

```python
import copy

original = {"hobbies": ["reading", "coding"]}
shallow_copied = original.copy()  # 浅拷贝
deep_copied = copy.deepcopy(original)  # 深拷贝

# 浅拷贝：修改嵌套列表会影响原字典
shallow_copied["hobbies"].append("gaming")
print(original["hobbies"])  # 输出: ['reading', 'coding', 'gaming']

# 深拷贝：完全独立，不影响原字典
deep_copied["hobbies"].append("swimming")
print(original["hobbies"])  # 输出: ['reading', 'coding', 'gaming']（原字典不变）
```

### copy() 的其他等价方式

除了 `dict.copy()`，还可以通过以下方式创建字典的浅拷贝：

- 字典构造函数：`new_dict = dict(original_dict)`
- 字典解包：`new_dict = {**original_dict}`

**示例**：

```python
original = {"a": 1, "b": 2}
copy1 = original.copy()
copy2 = dict(original)
copy3 = {** original}

print(copy1 == copy2 == copy3)  # 输出: True（结果相同）
```

