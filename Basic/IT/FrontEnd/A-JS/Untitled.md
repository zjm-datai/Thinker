
### 什么是 tsconfig.json ？

告诉  tsc “该编译哪些文件、用什么规则”。  

当 tsc、Vite、Webpack、ESLint 或 IDE TypeScript Server 启动时，会从项目根目录查找 `tsconfig.json`，读取里面的 **compilerOptions**（编译选项）和 **include / exclude / files**（要不要编译的文件）。

> tsc 指的是 **TypeScript 编译器（TypeScript Compiler）**，它是 TypeScript 语言的核心工具，负责将 TypeScript 代码转换为浏览器或 Node.js 可执行的 JavaScript 代码。
> 
> TypeScript 是 JS 的超集，增加了静态类型检查等特性，但是浏览器和 Node.js 无法直接运行 TS 代码。**tsc** 的主要作用就是将 `.ts`（TypeScript）文件编译为 `.js`（JavaScript）文件，同时进行类型检查，帮助开发者在运行前发现潜在错误。

#### tsconfig.app.json 和 tsconfig.node.json

这两个文件通常是 **脚手架或框架**（Angular CLI、NX、Vue CLI、Vite plugin Vue、create‑react‑app 等）在初始化项目时 **自动生成** 的，用来把 **浏览器端代码** 和 **Node 端脚本** 分开编译、各用各的类型环境。也可以手动加，原则完全一样。

在实际工程里，**一个 tsconfig 往往不够用**：

##### 为什么要进行拆分


### 运行时 vs 编译时

我们的浏览器 html 加载 `<script>` 

```html
// vite-vue-project/index.html
<script type="text/javascript" src="./test/uni.webview.1.5.6.js"></script>
```

uni.webview.1.5.6.js 执行 `window.uni = { postMessage()… }`

同时由于我们的 `tsconfig.app.json` 中：

```json
"include": ["src/**/*.ts", "src/**/*.tsx", "src/**/*.vue"],
```

没有将 `./test/uni.webview.1.5.6.js` 下的 js 文件纳入类型检查，所以是不会触发报错的。这个文件根本不参与编译。

那么就说到编译了。

我们的 `ChatPage.vue` 是参与编译的，所以当 tsc 编译器在源码里遇到 `window.uni` 却没找到类型声明，就会报错：  

```
Property 'uni' does not exist on type 'Window & typeof globalThis'
```

`.d.ts` 文件做的事就是：**告诉 TypeScript**  “放心，这东西到时候一定会存在，长这样：…” 

同时注意，这个文件仅供类型检查 & 智能提示，**不会** 被打进最终产物，也不会真正去创建变量。

#### 解决方案
