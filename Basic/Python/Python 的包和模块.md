models 是一个文件夹，当我们做：

```python
import models
```

的时候，会发生什么？

首先 python 会去搜索 models 包。它会在 sys.path 所列出的目录中查找名为 models 的文件夹。sys.path 是什么？

接着，Python 会判断 `models` 文件夹是否为一个包。这就要求该文件夹里必须存在 `__init__.py` 文件（在 Python 3.3 及之后的版本中，这个文件不是必需的，但为了保证代码的兼容性，最好还是加上）。

之后，Python 会执行包的初始化操作。若 `models` 文件夹中有 `__init__.py` 文件，Python 会执行这个文件里的代码，并且把其中定义的对象添加到 `models` 命名空间中。

最后，Python 会创建模块对象。完成上述步骤后，Python 会创建一个代表 `models` 包的模块对象，并且将其添加到 `sys.modules` 缓存里，这样后续再次导入时就能直接使用了。

```
project/
----main.py
----models/
	----__init__.py
	----user.py
	----post.py
```

若 `models/__init__.py` 文件内容如下：

```python
from .user import User
from .post import Post

__all__ = ["User", "Post"]
```

在 `main.py` 中执行 `import models` 之后，就能够这样使用：

```python
import models

user = models.User()
post = models.Post()
```

---

命名空间是 python 中存储变量名和对象映射关系的容器。简单来说，就是一个字典，键是变量名，值是对应的对象。

全局命名空间 global namespace

每个模块 .py 都有自己的全局命名空间
通过 import 导入的对象会被添加到当前模块的全局命名空间中。

包命名空间 package namespace

当我们导入一个包 import models 的时候，python 会被这个包创建一个命名空间。
包的命名空间内容来自 `__init__.py` 文件中定义的所有变量，函数和类等。

示例说明

```python
# models/__init__.py
from .user import User
from .post import Post

VERSION = "1.0"

def init_database():
	pass
```

当执行 import models 之后：

models 包的命名空间就包含了：User Post VERSION init_database

这样我们就可以通过 models.xxx 来访问这些对象了：

```python
import models

user = models.User # 可以访问从 user.py 中导入的 User 类
```

---

什么是 sys.modules ？

`sys.modules` 是 Python 解释器内部的一个 **全局缓存**，用于存储所有已导入的模块对象。它本质上是一个字典，键是模块名，值是对应的模块对象。

作用

- **避免重复导入**：当你多次导入同一个模块时，Python 会直接从 `sys.modules` 中获取已存在的模块对象，而不是重新执行模块代码。
- **提高性能**：减少重复加载模块的开销。

```python
import sys
import models

print("models" in sys.modules)

models_alias = sys.modules["models"]
user = models_alias.User()
```

