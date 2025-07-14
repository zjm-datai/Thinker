
OAuth2.0 是一套由 IETF 发布，行业广泛采用的授权框架（Authorization Framework），其核心目标是让第三方应用在不直接获得用户的账号密码的前提下，安全地访问用户在其他服务（资源服务器）上受到保护的资源。

### 核心角色

1. 资源所有者：通常是用户，拥有对受保护资源（照片、文件、联系人等）的访问权。
2. 客户端（Client）：需要访问资源所有者受保护资源的应用：可以是网页应用、移动 App、服务器端程序等。
3. 授权服务器（Authorization Server）：负责验证资源所有者身份（Authentication）并发放访问令牌（Access Token）。常见如 Google、Facebook 的登录系统。
4. 资源服务器（Resource Server）：存放受保护资源的服务器，收到客户端带的令牌后决定是否授权访问。

### 基本流程（以授权码模式为例）

Authorization Code Flow 为例

1. 客户端引导用户去授权服务器登录并授权：客户端重定向用户到授权服务器的登录/授权页面，携带自己的 client_id ，回调地址 redirect uri，请求的权限范围 scopes 等参数。
2. 用户登录并同意授权：用户在授权服务器输入账号和密码（此时客户端并不知道用户名和密码），并同意客户端访问其权限。
3. 授权服务器返回授权码（Authorization Code）：成功之后，浏览器被重定向回到客户端的回调地址，URI 中带上一个一次性的 code。
4. 客户端用授权码换取令牌：后端服务器向授权服务器的 `/token` 接口 POST 请求，带上 `code`、`client_id`、`client_secret`（和回调地址）来换取访问令牌（Access Token）和可选的刷新令牌（Refresh Token）。
5. 客户端带令牌访问资源：客户端在后续请求资源服务器时，在 HTTP 头中加入

```
Authorization: Bearer <access_token>
```

资源服务器收到后验证令牌有效性，再决定是否返回受保护数据。

### 四种常见“授权模式”（Grant Types）

- **Authorization Code（授权码模式）**  
    最安全、最常用的浏览器→服务器端应用模式。适用于有后端的 Web 应用。

- **Implicit（简化模式）**  
    令牌直接发给浏览器前端，不经过后端。因为安全性较低，现在已不推荐新项目使用。

- **Resource Owner Password Credentials（密码模式）**  
    用户将用户名/密码直接给客户端，由客户端向授权服务器换取令牌。只建议在高信任环境（如自己开发的官方 App）中使用。

- **Client Credentials（客户端模式）**  
    用于服务器与服务器之间的通信，客户端本身就是资源所有者，没有用户参与，直接用 `client_id`/`client_secret` 拿令牌。

#### Resource Owner Password Credentials 密码模式

前端直接收集用户的用户名和密码，然后向授权服务器的 /token 端口发起请求，成功之后拿到 `{access_token: ak, refresh_token: rk}` 前端把 ak 存起来，每次 ajax 调用接口的时候带上 `Authorization: Bearer ak` 。

**缺点**：前端完全知道用户密码，安全风险高，只推荐在特别信任的 App（官方移动端）或内部工具使用。

### 令牌和作用域（Tokens & Scopes）

- **访问令牌（Access Token）**  
    一个短期有效的字符串，客户端用它来访问资源。可以是 JWT（自包含的）或不透明的随机串。

- **刷新令牌（Refresh Token）**  
    在访问令牌过期后，用刷新令牌向授权服务器再换取新的访问令牌，避免用户重新授权。

- **作用域（Scope）**  
    一组权限标识符（strings），客户端在请求令牌时声明，授权服务器会根据策略发放对应权限的令牌。例如：

```
scope = "read:user write:files"
```

OAuth 2.0 本身 **不规范加密传输**，要求在生产环境中 **必须** 通过 HTTPS 才能保证令牌和敏感数据不被窃取。

### Oauth 和 Cookie 

Cookie 只是 HTTP 协议层面的会话或存储手段，而 OAuth 2.0 把关注点放在“令牌如何发放和使用”上。

- **四大核心授权模式**（Authorization Code、Implicit、Password Credentials、Client Credentials）都不依赖 Cookie；它们关心的是：客户端怎样换取 Access Token（以及可选的 Refresh Token）。
    
- **Cookie** 常见的用途有两种：
    
    1. **授权服务器端的用户会话管理**：用户在登录授权服务器后，后台通常用 Cookie 来维护登录态（Session），这样用户在重定向流程中不必每次都重新输入用户名/密码。
        
    2. **客户端存储令牌**：拿到 Access Token 以后，有的前端会把它存到 HTTP-only Secure Cookie 里，以利用浏览器自动带 Cookie 的特性——但这完全是“存储/传输策略”，并不是 OAuth 2.0 定义的授权模式之一。


如果你在问“有没有一种 OAuth 2.0 自身提供、专门用来发放或管理 Cookie 的授权模式？”那答案是：**没有**。OAuth 2.0 的所有授权流程都围绕 Token，而不是 Cookie；Cookie 只是配合浏览器和服务端做会话或安全防护（如 CSRF）时的辅助设施。

### 存放 token 的方式

在前端或客户端存储 Access Token（`ak`）和 Refresh Token（`rk`）时，主要有以下几种方案，各有安全与便利上的权衡：

| 存储方式                                                         | 优点                                            | 缺点                                                  | 适用场景                 |
| ------------------------------------------------------------ | --------------------------------------------- | --------------------------------------------------- | -------------------- |
| **Memory（内存）**                                               | 最安全：关闭页面或进程即清空，不易被持久化攻击利用                     | 刷新页面或重启应用即丢失，不适合需要“记住我”场景                           | 对安全要求极高的短时会话         |
| **Session Storage**                                          | 页面会话内持久（刷新后有效，关闭标签页即清空），简单易用                  | 易受 XSS 攻击：JS 可直接读取                                  | 对持久性需求低、风险可控的 Web 应用 |
| **Local Storage**                                            | 持久化（可跨会话），简单易用                                | 易受 XSS 攻击：JS 可直接读取，难以做 CSRF 防护                      | 桌面 Web 应用；风险可控时      |
| **HTTP-only Secure Cookie**                                  | JS 无法访问（防 XSS），可自动随同请求发送，支持 `SameSite` 防 CSRF | 需要额外配置过期/刷新策略，默认会附带所有同域请求，需配合 `SameSite`/CSRF Token | 高安全要求的 Web 应用；需要跨子域时 |
| **Encrypted Storage（加密存储）**如 Keychain（iOS）、Keystore（Android） | 平台级安全保障，不易被其它 App 或脚本读取                       | 实现复杂度高，与平台绑定，跨平台迁移成本大                               | 移动 App/桌面原生应用        |

#### 选型建议

- **单页应用（SPA）**
    
    - 推荐：**HTTP-only Secure Cookie + SameSite=Strict**
        
        - 令牌由后端设置 `Set-Cookie`（`HttpOnly; Secure; SameSite=Strict; Path=/api`），前端 JS 无法读取，浏览器自动带上，后端从 Cookie 中提取并验证。

        - 刷新机制：在 Cookie 中存放的 `rk` 也可通过类似方式存储，或在前端定时调用刷新端点，由服务器再设置新 Cookie。

- **传统多页 Web 应用**

    - 推荐同上，或将 `ak` 放在 Cookie，`rk` 存在后端 Session 中，前端无需管理。

- **移动/桌面原生 App**
    
    - 推荐：**平台安全存储（Keychain/Keystore）**
        
        - 利用系统级加密存储，减少被读取风险。
            
        - 应用启动后从安全存储加载令牌，发起请求前附加到 Authorization 头。

- **低敏场景或内部工具**
    
    - 可考虑 Session/Local Storage，开发更简单，但须严格做好 XSS 审计与 CSP 策略。

### 实操

我们现在举一个真实的案例来看看具体是怎么操作的：

[[Dify 中是如何使用 LocalStorage 存储 token 的]]

