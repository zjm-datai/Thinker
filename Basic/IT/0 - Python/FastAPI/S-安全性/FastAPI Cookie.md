FastAPI 默认使用 OAuth2PasswordBearer 从请求头中的 `Authorization: Bearer` 拿 token 

但是如果要 **从 Cookie 中获取 token，而不是请求头**，可以自己定义一个从 Cookie 中读取的依赖项。

```python
from fastapi import Depends, Cookie, HTTPException, status

async def get_token_from_cookie(token: str = Cookie(None)):
    if not token:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token not found in cookie"
        )
    return token

@app.get("/users/me")
async def read_users_me(token: str = Depends(get_token_from_cookie)):
    return {"token": token}
```

### 发送 Cookie 给浏览器

设置 token 到 Cookie（一般在登录接口）

```python
from fastapi.responses import JSONResponse

@app.post("/login")
async def login():
    token = "example-token"  # 正常情况你会用 JWT encode 出来
    response = JSONResponse(content={"message": "Login successful"})
    response.set_cookie(key="token", value=token, httponly=True)
    return response
```

- 设置 `httponly=True` 可以防止 JavaScript 访问 token，增强安全性。

- Cookie 的名字你可以自定义，比如叫 `access_token`、`token` 等。

- 如果你用的是 SPA 前端，请确保 `withCredentials` 开启，服务器允许跨域携带 cookie。

## HTTP 中的 Cookie

在 HTTP 中，**Cookie 的传输是通过请求头（Headers）完成的**，分为：

### 服务器设置 Cookie 响应头

当客户端（比如浏览器）第一次登录成功后，**服务器通过响应头告诉客户端设置 Cookie**：

```
HTTP/1.1 200 OK
Set-Cookie: token=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
```

这个响应头的含义

- `token=abc123`：设置名为 `token` 的 Cookie，值是 `abc123`
    
- `Path=/`：Cookie 对所有路径生效
    
- `HttpOnly`：禁止 JavaScript 访问，增强安全性
    
- `Secure`：只有 HTTPS 才会发送此 Cookie
    
- `SameSite=Lax`：阻止跨站请求伪造（CSRF）

### 客户端发送 Cookie

当浏览器之后再次访问相同的域名时，它会 **自动将 Cookie 附带到请求头中发给服务器**：

```
GET /users/me HTTP/1.1
Host: example.com
Cookie: token=abc123
```

这个 `Cookie` 请求头是浏览器自动加上的（前提是域名、路径匹配，`SameSite` 和 `Secure` 设置不阻止）。

### 注意

非浏览器客户端（如 Postman、Python requests）默认**不会自动管理 Cookie**，要手动加：

```
Cookie: token=abc123
```

- Cookie 会自动发送给相同域名下的接口，前提是设置路径、域、SameSite等不冲突

> **Same-origin** 要求：
> - 协议（http/https） 
> - 域名
> - 端口  
> 这三者都相同。


- 跨域请求如果要带 Cookie，要设置：


    - 前端：`withCredentials: true`
        
    - 后端：CORS 响应头加上 `Access-Control-Allow-Credentials: true`

```js
fetch("https://api.example.com/user", {
  credentials: "include"  // 告诉浏览器带上 cookie
})
```

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,  # ✅ 必须有这个
    allow_methods=["*"],
    allow_headers=["*"],
)
```

在浏览器中，跨域请求默认是 **不带 cookie 的**，除非你显式要求带上（如上）。但即使你设置了 `credentials: "include"`，如果后端没有明确说“我允许你带 cookie”，**浏览器也会拒绝响应！**

