
https://dify.ai/blog/dify-plugin-system-design-and-implementation

## 用户需求分析

在设计插件系统之前，我们面临着几个关键挑战：

1. 代码耦合度高：向 Dify 添加新模型和工具很繁琐，且会引入过多依赖，导致工具和模型的版本管理问题。
2. 用户需求不完整：一些需求，如集成即时通讯服务，需要在 Dify 之外额外封装一层服务。
3. 自定义模块固定：例如，Dify 的 PDF 解析器表现不佳，像 RAG 这样的自定义模块也不容易调整。  

为解决这些问题，我们决定实施一个统一框架，将 Dify 的工具和模型解耦，使其能够独立安装并按需选择。与 RAG 相关的功能，如文档解析器和 OCR，也被插件化，以满足不同用例需求。对于无法在 Dify 内完全闭环的场景，插件系统提供开放接口与外部系统集成，比如支持即时通讯平台的外向 Webhook。  

这些产品需求看似简单，但工程实现颇具挑战。在最初设计阶段不到一周的时间里，我们就遇到了一系列问题：  

- 多工作区设计：Dify 采用多工作区设计，这意味着不能简单地通过将 Python 源代码挂载到工具 / 模型目录来实现功能。这还会导致依赖冲突。  

- 插件环境一致性：我们希望插件在不同环境下表现一致。虽然 Docker 可以解决这个问题，但为每个插件分配单独的 Docker 容器会显著增加部署复杂性。  

- 高并发云服务负载：Dify 的 SaaS 服务拥有数十万用户。如果每 10 个用户就有一个自定义插件，Dify 将面临数万个插件的运行时负载，给云服务成本带来巨大压力。  

- 插件开发与调试：作为开发者，每次修改代码都需要重新打包并重新安装插件，且日志必须发送到 Dify 的后端，这严重影响开发体验。

- 插件长期运行：例如，是否应该允许插件长期运行一个 HTTP 服务器来监听即时通讯平台的 Webhook 事件？

## 解决方案

### 调试体验

Before thinking about how to implement the plugin system, we first considered how to optimize the debugging experience, especially for developers. An ideal debugging experience should meet the following two requirements:

1. **What You See Is What You Get**: After modifying the code, no installation is needed; it should take effect directly in Dify.
    
2. **Local Debugging**: The plugin’s code should run locally so that we can debug it more easily with breakpoints.

We referenced well-established debuggers like GDB and used a separation of debugger and runtime: the debugger waits for the runtime to initiate a connection. Once the connection is established, the local plugin can create a long connection with Dify, and Dify treats it as an installed plugin, marking it for debugging. User requests are forwarded via this long connection to the local plugin, and the plugin’s response is sent back to Dify, achieving a smooth debugging experience.

However, this design faced an issue: long connections are stateful, but Dify’s current services are stateless. In a Kubernetes cluster, load balancing routes requests to different Dify pods. For example, Plugin 1 connects to Dify 1, and Plugin 2 connects to Dify 2. When a user requests Plugin 1, the request might be routed to Dify 2, making Plugin 1 inaccessible. To resolve this, we needed to implement a traffic forwarding mechanism.

![[Pasted image 20250815142149.png]]

解释：

先我们忘记掉现在我们知道的架构这样，如果说要是实现我们说的插件，我们首先想的是一个 dify 对应一个 plugin 的集合。那么就会存在上面的问题。

本地插件会和 Dify 建立一条 长连接（通常是 WebSocket/TCP）用于把 “用户请求 → 本地代码” 打通。但是这条长连接 只存在于某一台 Dify 实例（某个 Pod）上，比如 Dify-1。

注意到 Kubernetes 的默认负载均衡是无状态的，用户后续发起调用 “用这个处于调试态的插件” 时，请求会被 LB 随机分到 Dify-1、Dify-2、Dify-3…中的任意一个。  

如果这次被分到 Dify-2，而 长连接其实挂在 Dify-1，Dify-2 自己并没有那条连接能够把请求转发给你本地的插件进程，于是 “找不到插件 / 插件不可达”。

这就是作者说的“需要流量转发机制”的原因，既然长连接终止在某个具体实例上，就必须有一种办法把落在 任意实例 上的业务请求，精确路由 到 持有该长连接的实例，再由它通过长连接把请求送到你的本地插件，拿到结果后再回传。

可以落地的几种解决思路（本质都是“把无状态请求路由到有状态连接”）：

1. 会话粘性（Sticky Session）  

让 “同一调试会话/同一插件” 的后续请求都落到最初建立长连接的那台 Dify 实例上。

- 优点：实现简单（Ingress/网关支持 Cookie/IP hash 等）。        
- 缺点：伸缩/故障切换不友好；多插件多连接时粘性规则复杂。

2. 集中式“连接代理/调度器”（Broker/Gateway）  

 把所有调试长连接都 统一接入 一个专职的网关/守护进程（可以是 plugin-daemon 的“调试通道”角色），业务请求无论落到哪个 Dify 实例，都先查 连接注册表（plugin_id → connection_owner），然后 转发到网关，由网关把请求通过对应的长连接发给本地插件。

- 优点：最清晰、对应用层最透明；实例无状态可保持。
- 缺点：网关本身需要高可用与水平扩展。

3. 实例间点对点转发（Pod-to-Pod forwarding）  

每个 Dify 实例都维护“谁持有哪条连接”的分布式注册表（如基于 Redis/Consul）。请求打到 Dify-2 时，Dify-2 发现 plugin-1 的连接在 Dify-1，于是把请求内部转发给 Dify-1，再由 Dify-1 走长连接到本地插件。
   
- 优点：不新增外部组件。
- 缺点：实现复杂、链路更长、跨实例依赖更多。

4. 把长连接统一下沉到 plugin-daemon  

让长连接不终止在 Dify API 实例，而是终止在一个（或一组）有状态的 plugin-daemon 上；Dify API 的所有请求都调用 daemon，daemon 再把请求通过长连接发给本地插件。

- 优点：把 “状态” 集中在本就负责插件生命周期/调试的组件里。
- 缺点：daemon 本身要做 HA/扩容（如增加多副本 + 一致性注册表）。

不管采用哪种方案，关键要素都是这三个：

- 连接注册表：能从 plugin_id / debug_session_id → connection endpoint 做 O(1) 查找。
- 可达的转发通道：任意 Dify 实例都能把请求路由到持有连接的节点（或统一网关）。
- 断线重连与失活清理：长连接掉线要及时摘除注册表；重连后要原子地更新持有者。

### Endpoint Plugin

We studied many existing IM and office collaboration software solutions and, based on current requirements, clarified the key issue to solve: how to make Dify receive webhook requests from these platforms and let plugins handle these HTTP requests. For example, using Dify’s App to process user messages.

To solve this, we designed a mechanism to generate random URLs and integrated these URLs with platforms like Discord. This approach avoids the need for each plugin to run a server long-term since Dify takes on the responsibility of forwarding HTTP requests. Dify uses the generated URL to receive webhook requests from platforms, and the plugin processes the forwarded requests.

After solving how to receive messages, the next challenge is how to process them. For example, if I’m developing a Discord bot and want Dify’s chatflow to reply to user messages, the code might look like this:

```python
class Webhook:
    def _invoke(self, r: Request) -> Response:
        message = r.json()['message']
        response = invoke_app(app_id, message)
        return Response(response)
```

In this plugin, I need to invoke Dify’s app to handle the request, enabling the bot functionality. This introduces a key concept in Dify v1.0: **Reverse Call**.

假设我们要做一个 Discord Bot 插件，它的工作方式是：

1. Discord 有人发消息，Discord 会通过 webhook 把这个消息 post 到我们提供的一个 URL 。
2. 插件拿到消息，处理一下（比如调用 Dify 的 App 做 NLP 回复），再返回给 Discord。

但是通常 Discord 要求 **给它一个公网可以访问的 URL** 来接收它的消息。

但我们的插件 **通常也不会单开一个 HTTP 服务**（常驻监听端口、处理 TLS、开放公网入口等很麻烦，而且插件可能跑在本地、serverless 或 debug 模式下，根本没公网 IP）。

这就变成了：**“消息能到达 Dify，但插件本身没有公网入口。”**

#### Dify 的解决方法：由 Dify 当“门面”

- **生成一个随机 URL**（例如 https://dify.com/hooks/abc123 ），专门指向插件。
- 把这个 URL 配到 Discord 的 webhook 里，让 Discord 把消息发给这个 URL。
- **Dify API** 收到这个 HTTP 请求后，会根据 URL 里的标识（abc123）找到对应的插件，并**转发**给它的运行环境（runtime）：
    
    - 如果插件是 Local runtime → 把请求通过 STDIN 传给本地进程。
    - 如果是 Debug runtime → 通过长连接发给本地调试插件。
    - 如果是 Serverless runtime → 触发云函数执行插件代码。

这就是一种 “流量转发” 机制，只不过：

- **Debug 场景** 转发的是 **内部有状态连接**（从某个 Dify pod 转发到持有长连接的 pod）
- **Endpoint Plugin 场景** 转发的是 **外部 HTTP webhook 流量**（从 Dify 的公网入口转发到插件运行时）

在 Endpoint Plugin 的情况下，转发机制解决的是：

> “外部平台来找我（Dify），我再去找你（插件）”。

### Reverse Call

Reverse call is a critical concept in Dify’s plugin system, allowing plugins to call internal Dify services. For instance, plugins can call authenticated models, tools, or Dify’s apps. Reverse calls play a crucial role in several scenarios:

- **LlamaIndex Implementation**: LlamaIndex implements various agentic RAG strategies to summarize a retrieved list using LLMs. In Dify, it is used as a tool where users only need to configure model parameters and input lists, and the tool can be installed or uninstalled.
    
- **Models as Tools**: Previously, OCR, ASR, and TTS models could only be used as standalone models. Now, they can be used as tools, such as using Gemini as an OCR tool to simplify the operation process.
    
- **OpenAI-Compatible API**: Through the Endpoint plugin, Dify provides OpenAI-compatible formats, allowing plugins to call Dify’s app and return responses in a unified format, supporting different models like Claude or Gemini.
    
- **Agents as Plugins**: Reverse tool calling enables Agent pluginization, automating parameter reception, operation execution, result delivery, and custom Agent strategy implementation.

### Implementation Details

The design of the plugin runtime was the first challenge we needed to solve. Ultimately, should the plugin runtime be a Docker container, a process, a virtual machine, or a serverless runtime? After evaluating Dify’s user base, we decided to implement four completely different runtimes:

- **Local Deployment**: Aimed at small teams and individual developers, with relatively low deployment demands, focusing on high availability without requiring large-scale use.
    
- **SaaS Service**: Designed for hundreds of thousands of users, Dify needs to consider high user load.
    
- **Enterprise Version**: Similar to SaaS, but the enterprise version requires higher controllability, privacy protection, and private deployment.
    
- **Remote Debugging**: Supports debugging mode and must consider providing runtime support for it.

