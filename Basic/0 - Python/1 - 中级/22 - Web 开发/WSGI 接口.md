
WSGI 接口定义非常简单，它只要求 web 开发者实现一个函数，就可以响应 HTTP 请求。

```python
def application(environ, start_response):
	start_response('200 OK', [('Content-Type', 'text/html')])
	return [b'<h1>Hello, web!</h1>']
```

我们可以模拟一个 miniflask 

```python
from wsgiref.simple_server import make_server

class MiniFlask:
    def __init__(self):
        self.routes = {}

    # 用作装饰器：@app.route('/path')
    def route(self, path):
        def decorator(func):
            self.routes[path] = func
            return func
        return decorator

    # 真正的 WSGI 应用
    def wsgi_app(self, environ, start_response):
        path = environ.get("PATH_INFO", "/")
        view = self.routes.get(path)
        if view is None:
            start_response("404 Not Found", [("Content-Type", "text/plain; charset=utf-8")])
            return [b"Not Found"]

        try:
            body = view()          # 调用视图函数
            if isinstance(body, str):
                body = body.encode("utf-8")
            start_response("200 OK", [("Content-Type", "text/html; charset=utf-8")])
            return [body]
        except Exception as exc:
            start_response("500 Internal Server Error", [("Content-Type", "text/plain; charset=utf-8")])
            return [f"Error: {exc}".encode("utf-8")]

    # 让实例本身可调用 ⇒ 适配 WSGI 服务器
    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)

# ------------------ 使用示例 ------------------
app = MiniFlask()

@app.route("/")
def index():
    return "<h1>MiniFlask says hello!</h1>"

@app.route("/hello")
def hello():
    return "你好，WSGI!"

if __name__ == "__main__":
    print("Serving on http://127.0.0.1:5000")
    make_server("127.0.0.1", 5000, app).serve_forever()
```

