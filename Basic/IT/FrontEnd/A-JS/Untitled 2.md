
setTimeout 是 js 中用于延迟执行代码的定时器函数，属于浏览器/nodejs 提供的全局 API，常用于实现延迟操作，定时任务。

### 基本语法

```js
const timerId = setTimeout(callback, delay[, arg1, arg2, ...]);
```

- **参数说明**：
    - `callback`：延迟后要执行的函数（必选）
    - `delay`：延迟时间（毫秒，ms），默认值为 0（可选）
    - `arg1, arg2...`：传递给 `callback` 的参数（可选，ES5 后支持）

- **返回值**：`timerId`（定时器标识，用于后续取消定时器）

### 基本用法

```javascript
// 1. 基础用法：延迟 1 秒后执行
const timer = setTimeout(() => {
  console.log("1秒后执行");
}, 1000);

// 2. 带参数的回调
setTimeout((name, age) => {
  console.log(`我是${name}，今年${age}岁`);
}, 2000, "张三", 20); // 2秒后输出：我是张三，今年20岁
```

### 取消定时器

如果需要在延迟时间到达前取消定时器，可以使用 `clearTimeout` 

```javascript
const timer = setTimeout(() => {
  console.log("这段代码不会执行");
}, 1000);

// 立即取消定时器
clearTimeout(timer);
```

### 核心特性和注意事项

#### delay 时间的近似性

注意： `delay` 是 “最小延迟时间”，而非 “精确延迟时间”。因为 JavaScript 是单线程的，若主线程被阻塞（如执行耗时操作），定时器会等待主线程空闲后才执行。

```javascript
// 示例：实际延迟可能超过 1000ms
console.log("开始");
setTimeout(() => console.log("定时器执行"), 1000);
// 阻塞主线程 2 秒
const start = Date.now();
while (Date.now() - start < 2000) {}
console.log("结束阻塞");
// 输出顺序：开始 → 结束阻塞 → 定时器执行（总延迟约 2000ms）
```

#### this 的指向问题

定时器回调函数中的 `this` 默认指向全局对象（浏览器中是 window，nodejs 中是 global），而非定义回收的上下文：

```javascript
const obj = {
  name: "测试",
  start() {
    setTimeout(function() {
      console.log(this.name); // 输出：undefined（this 指向 window）
    }, 100);
  }
};
obj.start();
```

解决方案：使用箭头函数（继承外层的 this）或 bind

```javascript
// 箭头函数（推荐）
setTimeout(() => {
  console.log(this.name); // 输出：测试（this 指向 obj）
}, 100);
```

#### 延迟时间的最小值

- HTML5 规范规定，`delay` 最小为 4ms（当嵌套层级超过 5 层时），目的是防止恶意代码阻塞浏览器。

- 若 `delay` 为 0 或负数，会被视为 0，但仍需等待主线程空闲（并非立即执行）。

