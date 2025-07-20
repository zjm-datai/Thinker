### 一句话总结

- **文件** 是存放代码/数据的载体；
    
- **模块** 是“导入后运行、带名字空间的代码单元”；
    
- **包** 是“可以装模块的模块”，表现为一个文件夹（有时需要 `__init__.py`）。

### 文件

文件，是操作系统中的实际文件。python 只会把特定扩展名的文件当作 “代码来源” 来看待。（`.py`、`.pyc`、`.so`/`.pyd`）。

**要点：** 文件只是承载代码或数据的容器，本身并不等同于“模块”或“包”。

### 模块 Module

模块导入后在内存中运行、拥有自己名字空间（变量、函数、类等）的 “代码单元”。

**怎么来的？**

- `import foo`：Python 找到 `foo.py`（或 `foo.so`、内置模块等），执行一遍，生成模块对象。
- 模块对象会放入 `sys.modules`，下次再导入就直接复用，不会重复运行。

**关键属性：**

- `module.__name__`：模块的名字（`foo` 或 `package.bar`）
- `module.__file__`：源文件路径（内置模块可能没有）
- `module.__dict__`：模块的所有变量和函数都存在这里

```python
# demo.py
print("Executing demo top-level")
value = 42
```

```python
import demo      # 打印: Executing demo top-level
import demo      # 不再打印
```

```python
import demo, sys, types

print(demo.__name__)        # demo
print(demo.__file__)        # /path/to/demo.py
print(isinstance(demo, types.ModuleType))  # True
print(sys.modules["demo"] is demo)         # True
```

内置模块没有文件路径

```python
import sys
print(sys.__file__)   # 可能报 AttributeError 或为 None
```

### 包 Package

- **是什么？** 能装多个模块（或子包）的 “文件夹”，它本身也是一个模块。

- **传统包 (含 `__init__.py`)**
    
    - 一个目录下有 `__init__.py`，Python 把它当“包模块”先运行一次。
        
    - 其他 `.py` 文件放这里，就变成了 `package.module`、`package.subpackage.module`。

- **命名空间包 (PEP 420)**
    
    - 目录里 **没有** `__init__.py`，但如果多个地方都有同名这类文件夹，Python 会把它们“拼在一起” 当一个包。

- **关键属性：**

    - `package.__path__`：告诉 Python 还可以在哪些目录里找它的子模块

#### 绝对导入的前提：顶级包根在 sys.path

```
project/
  .venv/                # 虚拟环境目录（可执行、site-packages 等）
  app/
    __init__.py
    main.py
    core/
      __init__.py
      engine.py
      util.py
```

启动解释器时，**当前工作目录**（这里是 `project/`）被放进 `sys.path` 的首元素（空字符串表示当前目录）。于是当你写：

```python
import app.core.util
```

Python 会在 `sys.path` 里看到当前目录，找到子目录 `app/`，里面有 `__init__.py`，确认为顶级包 `app`，再逐层解析到 `core.util`。  

虚拟环境提供的是 **隔离的解释器 + site-packages 路径**，并不会阻止把项目根目录加入 `sys.path`。所以绝对导入成功是正常的。

#### 相对导入的前提：模块有正确的 `__package__`

当一个模块是通过包路径被导入（或用 `python -m app.main` 运行）时，解释器为它设置：

- `__name__ = "app.main"`
    
- `__package__ = "app"`（若是 `main.py`）或：在包内的其他模块例如 `engine.py` 导入后 `__name__ = "app.core.engine"`，`__package__ = "app.core"`。

于是 `engine.py` 中的：

```python
from . import util                # 解释成从 app.core 导入 util
from .. import main               # 解释成从 app 导入 main
```

能被正确解析。解析规则是 **基于包名层级**，不再重新 “猜测” 文件系统；Python 只需要知道当前模块属于哪个包（`__package__`），与虚拟环境是否存在无关。

#### 什么时候相对导入会失灵？

若你直接（错误地）以文件路径运行包内脚本：

```bash
$ python app/core/engine.py
```

此时该脚本成为 `__main__`，默认 `__package__` 为空（或 `None`），句子 `from . import util` 没有“当前包”参照，会触发：

```
ImportError: attempted relative import with no known parent package
```

### 示例

```
project/
  ├── utils.py           # 文件，也是模块 utils
  ├── native.so          # C 扩展文件，也是模块 native
  └── web/               # 包，因为有 __init__.py
      ├── __init__.py    # 包模块 web
      ├── router.py      # 模块 web.router
      └── api/           # 子包
          └── __init__.py
```

```
import utils              # 加载 utils.py，生成 utils 模块
import web.router         # 先运行 web/__init__.py，再加载 router.py，生成 web.router 模块
```

