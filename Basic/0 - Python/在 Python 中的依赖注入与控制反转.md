
[DependencyInjector](https://python-dependency-injector.ets-labs.org/introduction/index.html)

依赖注入（Dependency Injection）这一设计模式最初是在 Java 等静态类型语言中流行起来的。它是一种帮助实现 **控制反转（Inversion of Control, IoC）** 的原则。在静态类型语言中，使用依赖注入框架可以显著提高程序的灵活性。不过，要在这类语言中实现一个完善的依赖注入框架，通常非常复杂，并且耗时较长。

而 Python 是一门动态类型的解释型语言，自带了许多灵活性。有些开发者认为，依赖注入在 Python 中并不像在 Java 中那样“刚需”。不少 Python 程序员表示，借助 Python 自身的语言特性，实现依赖注入其实很简单，甚至无需引入专门的框架。

不过，依赖注入在 Python 中依然有其价值。这篇文章就将介绍在 Python 中应用依赖注入的好处，并通过代码示例展示如何实现这一模式。文中会演示如何使用 Dependency Injector 框架，它提供了：

- **容器（Container）**
    
- **工厂（Factory）**
    
- **单例（Singleton）**
    
- **配置提供器（Configuration Provider）**

此外，还将展示如何使用 **providers 的重写功能**（overriding feature），以便在不同的环境中对项目进行测试或重新配置。这种方式相比“猴子补丁”（monkey patching）更加安全、清晰，也更易维护。

总结来说，这篇内容将向你展示：哪怕是在灵活的 Python 中，依赖注入依然是提升架构可维护性和测试友好性的有力工具——而使用 Dependency Injector 框架，可以让这件事变得更优雅。

## 什么是依赖注入（Dependency Injection）？

我们来看看什么是依赖注入。

依赖注入是一种设计原则，它的主要目的是：**降低耦合度，提升内聚性**。

### 什么是耦合（Coupling）和内聚（Cohesion）？

耦合和内聚描述的是系统中各个组件之间“绑定”的紧密程度。

- **高耦合（High Coupling）**  
    就像用强力胶或者焊接把组件粘在一起，一旦组装好了，就很难再拆开或修改。
    
- **高内聚（High Cohesion）**  
    更像是用螺丝将组件连接在一起，拆装方便，重组容易。高内聚通常意味着模块职责单一，独立性强。

**高内聚，低耦合** 是良好软件设计的基本目标。它让代码更易于测试、维护和扩展。

### 那依赖注入是怎么实现低耦合、高内聚的呢？

核心理念是：**对象不再自己创建它所依赖的东西，而是由外部把它们传进来**。

#### 传统方式（高耦合）

```python
import os

class ApiClient:
    def __init__(self) -> None:
        self.api_key = os.getenv("API_KEY")  # <-- 依赖直接读取环境变量
        self.timeout = int(os.getenv("TIMEOUT"))  # <-- 依赖

class Service:
    def __init__(self) -> None:
        self.api_client = ApiClient()  # <-- 直接创建依赖

def main() -> None:
    service = Service()  # <-- 又创建了依赖
    ...
```

上面这个写法中，所有依赖都**硬编码**在类的内部，改动起来很不方便，也很难测试。

使用依赖注入（低耦合、高内聚）

```python
import os

class ApiClient:
    def __init__(self, api_key: str, timeout: int) -> None:
        self.api_key = api_key  # <-- 外部注入依赖
        self.timeout = timeout

class Service:
    def __init__(self, api_client: ApiClient) -> None:
        self.api_client = api_client  # <-- 外部注入依赖

def main(service: Service) -> None:  # <-- 外部注入依赖
    ...
```

执行时通过装配（composition）来组装：

```python
if __name__ == "__main__":
    main(
        service=Service(
            api_client=ApiClient(
                api_key=os.getenv("API_KEY"),
                timeout=int(os.getenv("TIMEOUT")),
            ),
        ),
    )
```

### 这样做有什么好处？

- **`ApiClient`** 不再关心配置从哪来 —— 配置可以来自环境变量、配置文件、数据库等；
    
- **`Service`** 不再创建 `ApiClient`，你可以轻松替换为 mock 或 stub 进行测试；
    
- **`main()`** 函数也不绑定 `Service`，更易于扩展和测试。

这就是**灵活性**的来源。

### 但灵活性是有代价的

你现在需要手动“拼装”这些对象：

```python
main(
    service=Service(
        api_client=ApiClient(
            api_key=os.getenv("API_KEY"),
            timeout=int(os.getenv("TIMEOUT")),
        ),
    ),
)
```

如果项目变大了，装配逻辑会散落在多个地方，代码会变得冗长难维护。

这时就轮到 **Dependency Injector** 框架登场了！

它能帮你优雅地管理这些依赖关系，把对象的装配和注入过程自动化，让结构清晰、配置灵活、测试方便，而不必靠手写装配或“猴子补丁”。

接下来我们会看看如何使用这个框架。

## 依赖注入器（Dependency Injector）是做什么的？

当你使用依赖注入模式时，**对象本身不再负责“组装”它所依赖的其他对象**。这个“组装”的责任由 **Dependency Injector 框架**来承担。

### Dependency Injector 能做什么？

它的作用是：**帮助你组装（assemble）和注入（inject）依赖**。

框架提供了一个 **容器（Container）** 和各种 **提供器（Providers）**，用来帮助你自动构建对象。当你需要某个对象时，只需要在函数参数中用一个 `Provide` 标记默认值，框架就会自动组装并注入依赖对象。

来看个例子：

```python
from dependency_injector import containers, providers
from dependency_injector.wiring import Provide, inject

class Container(containers.DeclarativeContainer):

    config = providers.Configuration()

    api_client = providers.Singleton(
        ApiClient,
        api_key=config.api_key,
        timeout=config.timeout,
    )

    service = providers.Factory(
        Service,
        api_client=api_client,
    )

@inject
def main(service: Service = Provide[Container.service]) -> None:
    ...
```

在程序启动时：

```python
if __name__ == "__main__":
    container = Container()
    container.config.api_key.from_env("API_KEY", required=True)
    container.config.timeout.from_env("TIMEOUT", as_=int, default=5)
    container.wire(modules=[__name__])

    main()  # <-- 框架自动注入 service 的依赖

    # 覆盖依赖（用于测试）
    with container.api_client.override(mock.Mock()):
        main()  # <-- 注入的是 mock，而非真实 ApiClient
```

### 发生了什么？

- 当你调用 `main()` 函数时，**框架自动帮你组装并注入** `Service` 实例；
    
- 在测试时，你可以通过 `container.api_client.override()` 方法，把真实依赖换成一个 mock 对象；
    
- 调用 `main()` 时，mock 自动被注入，无需手动修改业务代码；
    
- 你甚至可以替换任何 provider，把真实 API 客户端换成开发环境下的 stub。

### 为什么要用 Dependency Injector？

- **所有依赖的组装逻辑集中在容器中**，项目结构清晰；
    
- **注入是显式的**，依赖清单一目了然；
    
- 更易于重构、测试、配置环境（如开发、测试、线上）。

### Monkey-patching 与依赖注入

在 Python 中你可以随时 monkey-patch 几乎任何东西。但这并不是好主意：

- monkey-patch 很脆弱，因为你 patch 的是实现细节；
    
- 一旦实现改了，你的 patch 就失效了；
    
- monkey-patch 太“脏”，除了测试，不应该用于配置或部署逻辑。


相比之下，**依赖注入更稳健**：你 patch 的是“接口”而非“实现”。更符合软件设计原则，也更容易维护。

---

## 总结：依赖注入的三大优势

1. **灵活性**  
    各组件之间解耦，可以轻松组合、替换，甚至运行时动态注入。
    
2. **可测试性**  
    测试更容易，可以方便地用 mock 替代真实依赖（如数据库、API 等）。
    
3. **清晰与可维护性**  
    依赖是显式定义的，组件结构透明。正如《Python 之禅》所说：“**显式优于隐式**”。

### 那么，在 Python 项目中是否值得使用依赖注入？

这取决于你的项目类型：

- **脚本/小工具**：可能不需要，结构简单，注入带来的收益不明显；
    
- **中大型应用程序**：依赖注入的优势将非常明显，尤其是在复杂项目中。

### 是否值得使用框架？

在 Python 中，手动实现依赖注入不算难，但仍然需要组织代码逻辑。使用框架的好处是：

- 已经实现好，不用重复造轮子；
    
- 跨平台、跨版本测试过，稳定可靠；
    
- 文档齐全，有社区支持；
    
- 其他开发者熟悉上手快；
    
- 可以专注业务逻辑，不必分心处理依赖细节。

### 最后一点建议：

> **试试看吧！**

依赖注入最初可能让人感觉“反直觉” —— 当我们需要某个对象时，本能是自己去创建，而不是“声明我需要它”。但这其实是一次小小的“设计投资”，**短期可能多写几行代码，长期将获得更清晰、更灵活、更可测试的系统结构**。

建议你试用两周，足够形成自己的感受。如果不喜欢，也不会有什么损失。

### 切记：**常识优先**

依赖注入是一种好方法，但不是万能药。用得太多会暴露过多实现细节。**经验来自实践和积累**，记住目标是让代码更清晰、更可靠，而不是追求技术本身。




















