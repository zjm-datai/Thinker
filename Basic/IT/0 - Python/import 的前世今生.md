
早期 import 的缺陷

只有两行代码的模块系统，你能指望它有多好呢？

首先遇到的问题就是命名的冲突。。。

随着包模块的爆发出现了严重的问题

1994 年的重构之战

废除 import PEP 0

最大争议

反对声如潮，最后大比分提案失败

包目录解决了命名冲突的问题，但是却带来了新的地狱 -- 相对导入

1996 年的 python1.5 允许

```
from .foo import bar
```

这种写法，初衷是为了让子包自给自足。然而在早期的实现中把相对路径写入了编译期常量：

>编译期常量指的是在程序编译阶段就已经确定的值，且在运行期间无法被修改的量。它的关键特征是：值在代码编译的时候就被固化，不会随着程序运行而发生变化。

```c
/* import.c */  
static char *relative_base = NULL; /* 线程不安全 */
```

多线程下，两个包同时触发相对导入，relative_base 会被覆盖成 [[野指针]]，解释器直接段错误。

举例：

```python
# 在 package_a/subpackage/module_y.py 中
from .module_x import some_function  # 相对导入
```

```python
# 在 package_b/subpackage/module_y.py 中
from .module_x import another_function  # 另一个相对导入
```

1. 线程 1 开始执行：将 relative_base 设置为 package_a/subpackage
2. 就在线程 1 还没完成导入的时候，线程 2 抢占了 CPU
3. 线程 2 将 relative_base 覆盖为 package_b/subpackage
4. 线程 1 重新获得 CPU 时间，继续使用已经被修改的 relative_base
5. 实际导入错误
6. 更严重的是，可能导致指针指向无效内存区域，直接引发解释器崩溃（段错误）

这个问题的根源在于 relative_base 是一个全局变量，没有线程隔离机制。当多个线程同时进行相对导入时，它们会互相覆盖这个基础路径，导致导入的模块完全错误或内存访问异常。


