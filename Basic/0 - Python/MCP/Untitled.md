## 概述

模型上下文协议允许应用程序以标准化方式为大模型提供上下文，将提供上下文的关注点和实际大模型交互分离开来。我们这里学习的 python sdk 实现了完整的 MCP 规范，使得以下操作变得简单：

- 创建可以连接到任何 MCP 服务器的 MCP 客户端
- 创建公开资源，提示和工具的 MCP 服务器
- 使用标准传输方式，比如标准输入输出（stdio）、服务器发送事件（SSE）和可流式传输的 HTTP
- 处理所有的 MCP 协议消息和生命周期

## 安装

### Adding MCP to your python project

我们建议使用 uv 来管理您的 Python 项目。

如果您还没有创建一个由 uv 管理的项目，请创建一个：

```bash
uv init mcp-server-demo
cd mcp-server-demo
```

然后将 MCP 添加到我们的项目依赖中：

```bash
uv add "mcp[cli]"
```

### 运行独立的MCP开发工具

To run the mcp command with uv:

```bash
uv run mcp
```

## 快速入门

让我们创建一个简单的 MCP 服务器，它提供一个计算器工具和一些数据：

```python

```