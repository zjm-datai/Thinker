

要实现这种效果我们需要可以 ping 通 gpustack 的后端 ip ，然后可以拿到 gpustack 的 token ，这样我们就可以通过这个 token 在本地使用源码启动一个前端，然后将后端地址配置进去，将受信任的 token 配置进去，我们这个本地的前端就可以连接上其他人给我们的 gpustack 的后端了，就可以进行管理了。

 gpustack 的前端是使用 umi 框架进行构建的，所以我们只需要知道 umi 框架是在哪里设置前端对后端请求配置就好了。在这里我们介绍一些基本的关于 umi 的基础知识：

## UMI 配置基础

### 通过环境变量注入后端地址

Umi 会自动把以 `UMI_APP_` 或我们在 `defineConfig.define({ define: {...} })` 指定 `process.env.*` 注入进我们的前端代码中。具体而言就是：

当我们在项目根目录下面放置了 `.env.development`、`.env.production` 这样的文件，Umi 在启动或构建时会用 [dotenv](https://github.com/motdotla/dotenv) 把它们读进来。

自动注入带 `UMI_APP_` 前缀的变量出于安全和可维护的考虑，Umi 默认只会把那些以 `UMI_APP_` 开头的环境变量（例如 `UMI_APP_BASE_URL`、`UMI_APP_TOKEN` 等）“放到”前端代码里，也就是说，在你的业务代码里可以直接写：

```js
console.log(process.env.UMI_APP_BASE_URL);
```

但是要注意的是浏览器中并不存在真正的 `process.env` 对象，Umi（底层借助 Webpack 的 DefinePlugin）会在打包的时候把所有 `process.env.UMI_APP_XXX` 直接替换成对应的字符串字面量。例如，若你有：

```js
UMI_APP_API=http://localhost:7001
```

则在源码中我们可以读到：

```js
fetch(process.env.UMI_APP_API + '/users')
```

在构建后就会变成：

```js
fetch("http://localhost:7001" + '/users')
```

如果你有一些不想或不能以 `UMI_APP_` 开头的环境变量（比如 `GPUSTACK_BACKEND`、`GPUSTACK_TOKEN`），也可以在 `defineConfig` 里手动指定：

```ts
export default defineConfig({
  define: {
    'process.env.GPUSTACK_BACKEND': JSON.stringify(process.env.GPUSTACK_BACKEND),
    'process.env.GPUSTACK_TOKEN'  : JSON.stringify(process.env.GPUSTACK_TOKEN),
  },
  // … 其他配置
});
```

这样打包时就同样会把 `process.env.GPUSTACK_BACKEND`、`process.env.GPUSTACK_TOKEN` 替换成你设置的值。

在 Umi 的 `defineConfig` 里，我们可以通过一系列字段来控制开发／构建过程中的行为，下面把你当前配置里出现的主要字段一一说明：

| 字段                    | 用途说明                                                                                                                                                                                   |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **proxy**             | **开发环境下的 HTTP 代理**，把本地 `fetch('/api/…')` 的调用转发到真正的后端地址，解决跨域并且不用在代码里写全量域名。常见写法：`proxy: { '/api': { target: 'http://localhost:7001', changeOrigin: true, pathRewrite: {'^/api': ''} } }` |
| **history**           | **路由模式**，决定前端 URL 的展示和内部跳转方式：- `browser`（默认）：干净的 `/users/123`，需要后端做 rewrite fallback- `hash`：`/#/users/123`，无需后端额外配置                                                                   |
| **analyze**           | **打包体积可视化**，集成 `webpack-bundle-analyzer`：`analyzerMode: 'server'` 会在本地起一个可视化页面（如 `localhost:8888`）查看各模块体积占比，方便查体积大头。                                                                   |
| **mfsu**              | **模块联邦加速（Module Federation Speed Up）**，Umi 内置的依赖预构建方案，能大幅缩短首次启动/重新编译时间。`exclude` 数组里填你不想预编译的包。                                                                                         |
| **base**              | **应用部署根路径**，当你的前端不部署在 `/` 而在子路径下（如 GitHub Pages 的 `/<repo>/`）时，用它来统一前缀所有路由和静态资源 URL。也可以通过命令行 `npm run build --base=/foo/` 临时覆盖。                                                        |
| **jsMinifierOptions** | **JS 压缩器选项**（这里配给 `terser`），可以控制打包时是否去掉 `console.*`、`debugger` 等。                                                                                                                      |
| **scripts**           | **额外插入到 `index.html` 的 `<script>` 标签**，可以用来给一些全局脚本打时间戳、或插入外部 SDK。                                                                                                                      |
| **chainWebpack**      | **直接改写底层 Webpack 配置** 的钩子，接收一个 `config`，你可以用它来：- 改 filename/chunkFilename- 插入插件（如 `CompressionWebpackPlugin` 生成 `.gz`）- 调整 loader、plugin 的选项                                           |
| **favicons**          | **页面标题栏的小图标**（favicon），传入数组，可以同时指定多种尺寸或不同格式。                                                                                                                                           |
| **jsMinifier**        | 指定用哪种工具压缩 JS，常见 `terser`（Umi 默认）或 `swc`。                                                                                                                                               |
| **cssMinifier**       | 指定用哪种工具压缩 CSS，常见 `cssnano`（Umi 默认）或 `parcelCSS`。                                                                                                                                       |
| **presets**           | Umi 的预设集合（Preset），一次性打开一堆约定好的插件和配置。像你这儿用了 `umi-presets-pro`，会带进一系列「后台管理常用」的特性。                                                                                                         |
| **clickToComponent**  | Umi Pro 能力：在运行中页面上点击某个组件自动跳到源码里对应位置，提速开发。默认空对象表示启用。                                                                                                                                    |
| **antd**              | Ant Design 插件配置，你可以配置：- `style: 'css'                                                                                                                                                  |
| **hash**              | 给静态资源文件名加 content hash，方便长期缓存。                                                                                                                                                         |
| **access**            | Umi 内置的「访问权限」插件入口，配合约定式路由可在路由层做权限校验（例如只有某角色能访问某页面）。                                                                                                                                    |
| **model**             | Umi Max 的数据流（基于 `@umijs/plugin-model`），可在页面／组件里按需写 `useModel` 做状态管理。空对象表示开启。                                                                                                           |
| **initialState**      | Umi Max 的「应用初始状态」插件，常用来在每次刷新时先拉用户信息、权限等，同步到全局。                                                                                                                                         |
| **request**           | Umi 内置 `umi-request` 的全局配置入口，会自动把你在 `src/requestConfig.ts` 导出的 `requestConfig` 合并进来。                                                                                                   |
| **keepalive**         | 基于 `@umijs/plugin-keep-alive` 的页面缓存插件，用来在不同路由间切换时保留组件状态（scroll、表单等）。                                                                                                                   |
| **locale**            | **国际化（i18n）配置**：- `default`：默认语言- `antd`：是否让 antd 组件也跟着切- `baseNavigator`：是否根据浏览器语言自动切换- `useLocalStorage`：记住用户上次选择                                                                    |
| **layout**            | 基于 `@umijs/plugin-layout` 的整体框架布局，`false` 表示你自己手动在 `routes` 里写布局。或传对象自定义侧边栏、头部等。                                                                                                       |
| **routes**            | **路由表**，指定了每个页面的路径、组件、嵌套关系、meta 信息等。可以内联写，也可以像你项目里 `import routes from './routes'` 统一维护。                                                                                               |
| **npmClient**         | Umi 启动脚本安装依赖时用哪个包管理器，常见 `npm`、`yarn`、`pnpm`。                                                                                                                                           |
#### PROXY 字段

在这里我们主要关注 proxy 这个字段，`proxy` 字段就是在开发模式（`umi dev`）下，告诉 Umi 内置的开发服务器（基于 `webpack-dev-server` + `http-proxy-middleware`）：

1. **哪些请求要被拦截**
    
2. **把它们转发到哪个后端**
    
3. **转发时要做哪些改写**

##### 基本写法

```ts
export default defineConfig({
  proxy: {
    // 当请求以 /api 开头时
    '/api': {
      target: 'http://localhost:7001',  // 转发到这个后端
      changeOrigin: true,               // 修改 Host header 到 target
      secure: false,                    // 如果是 https 的后端，是否验证证书
      pathRewrite: { '^/api': '' },     // 去掉前缀，/api/users -> /users
      logLevel: 'debug',                // 控制台打印代理调试信息
    },
    // 你可以配置多个
    '/socket.io': {
      target: 'ws://localhost:7002',
      ws: true,                         // 支持 websocket
    },
  },
});
```

- **key** (`'/api'`)：可以写单个路径、路径数组 `['/api','/auth']`，或用函数动态过滤
- **target**：后端服务地址（协议 + 主机 + 端口）
- **changeOrigin**：将请求头的 `Host` 改为 `target`；大部分跨域场景都要 `true`
- **secure**：如果后端是自签名证书，设 `false` 可以不校验证书
- **pathRewrite**：用正则把前端路径改为后端实际路径
- **ws**：是否代理 WebSocket
- **其他可选**：`headers`、`onProxyReq`、`onProxyRes`、`cookieDomainRewrite`……

##### 结合环境变量动态注入

在项目里常见的做法是，把后端地址写到环境变量里，然后在 `config/proxy.ts` 写一个工厂函数：

```ts
// config/proxy.ts
export default function makeProxy(proxyHost: string = 'http://localhost:7001') {
  return {
    '/api': {
      target: proxyHost,
      changeOrigin: true,
      pathRewrite: { '^/api': '' },
    },
    // … 如果有别的端点也一起写
  };
}
```

```ts
// config/config.ts
import { defineConfig } from '@umijs/max';
import makeProxy from './proxy';

export default defineConfig({
  proxy: {
    ...makeProxy(process.env.PROXY_HOST),
  },
  // … 其他配置
});
```

**本地开发**

```bash
# macOS/Linux
export PROXY_HOST=http://10.0.0.123:7001
umi dev
```

**CI 或多环境**

在 `package.json` 的脚本里用 `cross-env`：

```json
"scripts": {
  "dev": "cross-env PROXY_HOST=http://localhost:7001 umi dev",
  "qa":  "cross-env PROXY_HOST=https://qa.backend.com umi dev",
  "prod": "cross-env PROXY_HOST=https://prod.backend.com umi dev"
}
```

这样，不同环境下一条命令就能接入不同后端。

##### proxy 只在开发时生效

- **`umi dev`**：开发服务器启动时，根据你的 `proxy` 配置拦截接口请求并转发
- **`umi build`**：打包时不会包含 proxy，前端代码里写 `fetch('/api/xxx')` 还是会请求相对路径。上线后你需要让生产环境的 Nginx/Apache/CDN 做同样的转发，或者在业务里用 `process.env` 注入全量后端地址。

在 Umi 里 **`proxy` 只在 `umi dev` 时生效**，打包（`umi build`）之后，开发服务器的代理中间件就不在了，那么这个时候如果我们在代码中写：

```ts
fetch('/api/users')
```

- **开发时**：`umi dev` 的 proxy 会把它转发到 `PROXY_HOST`，比如 `http://localhost:7001/users`
    
- **打包后**：

    - 浏览器直接向 **当前页面所在的域名** 发起请求：`GET https://your-site.com/api/users`
        
    - **不会**再走你在 dev 时配置的 `PROXY_HOST`
        
    - 如果你的静态服务器（比如 Nginx）没有做 `/api` → 后端的反向代理，就会 404


### requestConfig 

**触发时机**：在浏览器里、业务代码调用

```ts
import { request } from '@umijs/max';
await request('/api/users');
```

这条语句在真正发 HTTP 去后端之前，会依次跑上面 `requestInterceptors`；拿到响应后再走 `responseInterceptors`；如果抛出或出网络错误，就交给 `errorConfig` 里去处理。

### 两者如何协同

1. **开发阶段**
    
    - 业务里写 `request('/api/foo')`
        
    - **requestConfig** 拦截到加上 token header，然后 URL 是 `/api/foo`
        
    - 浏览器发 `/api/foo` → **proxy** 拦 → 转给 `process.env.PROXY_HOST` 指定的后端

2. **生产阶段（或无 proxy）**
    
    - 业务里写 `request('/api/foo')`
        
    - **requestConfig** 可以在拦截器里改成 `options.baseURL = 'https://prod.backend.com'`，最终真正请求 `https://prod.backend.com/api/foo`
        
    - **proxy** 不再生效，所有请求都靠前端拼好的绝对地址，或由线上反代接力







