
**注意：** 以下指南适用于 Tailwind v4。如果你使用的是 Tailwind v3，请使用 `shadcn-vue@1.0.3`。

---

### 1. 创建项目

```bash
npm create vite@latest my-vue-app -- --template vue-ts
```

- **目的**：使用 Vite 快速生成一个基于 Vue 3 + TypeScript 的骨架项目。
    
- **生成文件**：包括 `package.json`、`vite.config.ts`、`src/` 目录（带基础示例文件）、TypeScript 配置文件等。
    
- **后续作用**：所有开发依赖、基本目录结构都已就绪，你可在此基础上安装 Tailwind、shadcn-vue 等。

---

### 2. 添加 Tailwind CSS

```bash
npm install tailwindcss @tailwindcss/vite
```

- **目的**：将 Tailwind CSS（实用类优先的 CSS 框架）以及它的 Vite 插件加入项目中。
    
- **依赖文件**：
    
    - `node_modules/tailwindcss`
        
    - `node_modules/@tailwindcss/vite`

- **后续作用**：Vite 会在构建时通过插件自动处理 Tailwind 的指令和按需生成样式。

将 `src/index.css` 全部替换为：

```css
@import "tailwindcss";
```

- **目的**：注入 Tailwind 的基础样式、预设和工具类。
    
- **作用**：这个入口文件会被 Vite 加载并由 Tailwind 进行解析，生成最终的 CSS。

同时在 `main.ts` 中添加 

```ts
import { createApp } from 'vue'
import './index.css'

import App from './App.vue'

createApp(App).mount('#app')
```

---

### 3. 编辑 `tsconfig.json`

```jsonc
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

- **目的**：让 TypeScript 编译器（及支持它的 IDE）认识 `@/` 别名。
    
- **涉及文件**：项目根目录下的 `tsconfig.json`。
    
- **修改作用**：
    
    - `baseUrl: "."`：将项目根目录设为模块解析的基准路径。
        
    - `paths`：将以 `@/` 开头的导入重定向到 `./src/`，避免使用相对路径如 `../../../components`。

---

### 4. 编辑 `tsconfig.app.json`

```jsonc
{
  "compilerOptions": {
    // ...
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
    // ...
  }
}
```

- **目的**：补充 IDE（例如 VSCode）对路径别名的支持，让跳转和提示正常工作。
    
- **涉及文件**：`tsconfig.app.json`（Vite 用于应用源码的配置）。
    
- **修改作用**：和上一步相同，确保在开发时编辑器不会报找不到模块的错误。

---

### 5. 安装 Node 类型定义

```bash
npm install -D @types/node
```

- **目的**：让 TypeScript 识别 Node.js 内置模块（如 `path`、`fs` 等）的类型声明。
    
- **后续作用**：在配置 `vite.config.ts` 时 `import path from 'node:path'` 不会报错。

---

### 6. 更新 `vite.config.ts`

```typescript
import path from 'node:path'
import tailwindcss from '@tailwindcss/vite'
import vue from '@vitejs/plugin-vue'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [vue(), tailwindcss()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

- **目的**：
    
    1. 注册 Vue 和 Tailwind 插件，让 Vite 在开发和构建时正确处理 `.vue` 文件和 Tailwind 指令。
        
    2. 在构建层面再次声明 `@` 别名，保证打包时模块解析一致。

- **涉及文件**：项目根目录下的 `vite.config.ts`。
    
- **修改作用**：
    
    - `plugins`：启用 Vite 官方的 Vue 插件与 Tailwind 插件。
        
    - `resolve.alias`：让 Vite 在运行时将所有 `@/...` 路径映射到 `./src/...`。

---

### 7. 运行 shadcn‑vue 初始化

```bash
npx shadcn-vue@latest init
```

- **目的**：使用官方 CLI 拉取并生成一份初始的 `components.json`（组件配置）文件。
    
- **过程**：根据交互选择（如基色 Neutral），CLI 会下载对应的组件模板并写入 `src/components/ui`。
    
- **输出文件**：
    
    - `components.json`：记录已启用的组件及其配置。
        
    - `src/components/ui/…`：各个 UI 组件的 `.vue` 文件和样式。

---

### 8. 添加具体组件

```bash
npx shadcn-vue@latest add button
```

- **目的**：基于 `components.json` 中的配置，下载或生成 `Button` 组件到项目里。
    
- **后续开发**：在代码中直接通过 `import { Button } from '@/components/ui/button'` 使用，无需手写样式或模板。

---

通过以上步骤，你就完成了：

1. 搭建了一个 Vite + Vue + TypeScript 的项目骨架。
    
2. 集成了 Tailwind CSS，并配置了项目和 IDE 对 `@/` 别名的支持。
    
3. 安装并配置了 `shadcn-vue`，可按需拉取和添加 UI 组件，开启低成本、高一致性的组件化开发。

这些配置既减少了手动调试的成本，也能让目录结构、导入路径更清晰，方便团队协作。