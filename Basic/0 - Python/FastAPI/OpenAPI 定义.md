### OpenAPI

这是一个开放的、语言无关的 API 描述规范（Specification），由 Linux 基金会（原 Swagger 社区）管理。它定义了一套标准格式（JSON 或 YAML），用来描述 RESTful API——包括路径（paths）、方法（operations）、请求参数、响应格式、安全方案等。

**核心作用**

- 统一规范，任何系统只要按这个格式描述，就能被各种工具识别和处理。
- 作为“合同”，前后端、工具和第三方都据此生成文档、客户端 SDK、测试用例等。

### Swagger

**原来与现在**

最初 “Swagger” 是这个规范的名字，并且伴随一套生态工具。后来规范改名为 OpenAPI（从 3.0 版开始），但“Swagger”依然被保留为工具品牌。

**Swagger 工具集**

- **Swagger Editor**：在线/本地编辑 OpenAPI 文档的可视化编辑器。
- **Swagger UI**：将 OpenAPI 文档渲染成交互式网页文档，用户可以在页面上试着调用 API。
- **Swagger Codegen**、**OpenAPI Generator**：从 OpenAPI 文档自动生成客户端或服务端代码框架。

**总结**：

- **OpenAPI** 是规范；
- **Swagger** 是一系列围绕这个规范的工具名称（UI、编辑器、生成器）。

### FastAPI

这是一个基于 python 的现代 web 框架。用于快速构建高性能的 API

- **与 OpenAPI/Swagger 的关系**

    - **自动生成 OpenAPI 文档**：你在 FastAPI 里定义路由、参数、模型（Pydantic），它在后台就把这些信息编译成一份完整的 OpenAPI 描述（JSON）。
    - **内置 Swagger UI**：启动应用后，访问 `/docs`（默认路径）就能看到 Swagger UI，直接基于那份 OpenAPI 文档渲染的交互式接口文档。
    - **内置 ReDoc**：另一个基于 OpenAPI 的文档 UI，地址默认 `/redoc`。
    - **自动校验**：请求进来前，FastAPI 会根据 OpenAPI 中定义的 schema（由你的 Pydantic 模型产生）自动校验参数类型、必填性，并给出标准化的错误响应。

- **优势**

    - 开箱即用，无需额外配置，就能同时获得标准化文档和参数校验。
    - 与 OpenAPI 生态高度兼容，能直接接入各种生成工具和第三方平台。

