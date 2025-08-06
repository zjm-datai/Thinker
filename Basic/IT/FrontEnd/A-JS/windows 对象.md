
在 JS 中，windows 对象是浏览器环境下的全局对象。它有如下的特点和作用：

### 全局作用域

在浏览器中编写的全局变量和函数，其实都是 windows 对象的属性和方法。比如：

```js
var globalVar = 10; 
console.log(window.globalVar); // 输出：10
```

### 浏览器控制

借助这个对象可以对浏览器窗口进行控制，像调整窗口大小，导航等操作：

```js
window.resizeTo(800, 600); // 调整窗口大小 
window.location.href = "https://example.com"; // 跳转到新页面
```