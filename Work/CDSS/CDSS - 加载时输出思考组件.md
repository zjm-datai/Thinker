---
tags: vue
---

[[Vue - 组件之间如何传递参数]]

### 背景和需求

在接入大模型或其他耗时的后端工作流接口时，单纯的转圈 `Spinner` 很难给用户足够的反馈 —— 后端可能在“思考”很久，尤其是模型推理、业务流程编排等场景。为了提升体验，我们希望：

- **实时输出**：把后端的「思考进度」或中间结果以流式形式渲染到页面，让用户看到“智能系统”在思考什么；

- **异步解耦**：在组件内部独立处理流式拉取，不影响外部页面数据的最终渲染；

- **优雅过渡**：当流式输出结束且真正的数据加载完成后，切换到主内容。

于是，就有了这个 `ThinkLoading` 组件。它接收两个核心参数：

1. **`activeLoading`**：外部请求是否仍在进行（Boolean），防止流式输出已经完成但数据尚未就绪就切换 UI；

2. **`loadingData`**：用于拉流的接口参数，传给后端启动流式推理。


以下分步骤解读组件实现。


### 组件结构概述

```vue
<template>
  <div class="think-loading-container">
    <!-- 顶部：思考状态栏 -->
    <div class="think-header">
      <template v-if="isThinking">
        <div class="spinner"></div>
        <div class="status">AI 正在思考<span class="dots"></span></div>
      </template>
    </div>
    <!-- 下方：滚动输出区 -->
    <div class="think-content" ref="contentRef">
      <pre class="text">
<code>{{ message }}</code><span v-if="isThinking" class="cursor"></span>
      </pre>
      <div ref="sentinel"></div>
    </div>
  </div>
</template>
```

### Props 和响应式状态

```js
const props = defineProps({
  loadingData:   { type: Object, required: true, default: () => ({}) },
  activeLoading: { type: Boolean, default: true }
})
```

内部状态

```js
const streaming  = ref(false)    // 当前是否处于流式拉取中
const message    = ref('')       // 已接收并渲染的文本内容
const sentinel   = ref<HTMLElement | null>(null)  // 滚动锚点
const isThinking = computed(() => props.activeLoading || streaming.value)
```

在 vue3 的组合式 API 中，computed 是一个很重要的函数，它的作用在于创建计算属性。计算属性是一种特殊的响应式数据，它的值是通过对其他响应式数据进行计算得来的，并且会依据依赖的变化自动更新。

在前端开发中，sentinal 滚动锚点是一种非常实用的技术手段，主要用于实现自动滚动功能。简单来说它就是一个隐藏的 HTML 元素（通常是一个空的 div），它被放置在内容的末尾位置。当新内容加载进来或者页面发生更新时，我们可以通过控制这个锚点元素的位置，来实现自动滚动到最新内容的效果。我们可以借助 JavaScript 来获取这个锚点元素的位置，然后使用 `scrollIntoView()` 方法让它滚动到可视区域内。

这里使用了 TypeScript 的泛型，表明 `sentinel` 要么是一个 HTML 元素，要么是 `null`（在元素还未挂载的时候就是 `null`）。初始值设为 `null`，这是因为在组件刚创建时，DOM 元素还没有被渲染出来。

### 自动滚动到底部

在 vue 中当模板中的 ref 属性和响应式数据中的 ref 变量名同名的时候，vue 会自动把 dom 引用赋值给对应的 ref 变量。

所以我们在前面的 template 部分有这样的代码：

```vue
<!-- 下方：滚动输出区 -->
<div class="think-content" ref="contentRef">
    <pre class="text">
<code>{{ message }}</code><span v-if="isThinking" class="cursor"></span>
    </pre>
    <div ref="sentinel"></div>
</div>
```

为了在输出增量到来时自动滚动：

```js
function scrollToBottom() {
  nextTick(() => sentinel.value?.scrollIntoView({ block: 'end' }))
}
```

用 `?.`（optional chaining）是为了保险，确保只有在 `sentinel.value` 不为 `null` 或 `undefined` 时才执行后面的调用，避免报错。

scrollIntoView 是浏览器原生的 DOM API ，属于所有 Element 也就是 HTMLElement 的一个方法。它的作用是将当前元素滚动到可见的区域。我们在这里传入了一个选项对象 `{ block: 'end' }` 意味着“尽量把元素的 **底部** 对齐到滚动容器的可视底部”，从而实现了「滚到底」的效果。

在调用

```js
element.scrollIntoView({ block: 'end' })
```

时，传入的选项对象 `{ block: 'end' }` 指的是 **垂直方向** 上的对齐方式：

- **block** 参数控制元素在滚动容器（或者视口）垂直方向上如何对齐，可选值有：
    
    - `"start"`：把元素的 **顶部** 对齐到滚动容器的顶部（默认行为）。
        
    - `"center"`：把元素的 **中点** 对齐到滚动容器的中点。
        
    - `"end"`：把元素的 **底部** 对齐到滚动容器的底部。
        
    - `"nearest"`：自动选择最小滚动距离的对齐方式。


所以当你用 `{ block: 'end' }` 时，就是告诉浏览器：

> “请滚动，让这个元素的 **底部** 恰好出现在滚动区域（或者视口）的底部。”

配合我们把锚点元素（`sentinel`）放在内容末尾，就能实现「每次新增文本后滚到底」的效果。

如果你想同时控制水平方向，还可以加上 `inline` 选项，比如

```js
element.scrollIntoView({
  block: 'end',     // 垂直对齐到底部
  inline: 'nearest' // 水平方向尽量不滚动
})
```

但在大多数垂直滚动场景下，只设置 `block` 就足够了。

#### nextTick

在 Vue 中，`nextTick` 是一个核心的异步更新队列相关的 API。当你修改 Vue 组件中的响应式数据时，DOM 更新并不是立即执行的，而是会被推入一个队列中，等到下一个 "tick"（即所有数据变化都处理完毕后）再统一更新 DOM，这样可以避免频繁的 DOM 操作导致性能问题。

`nextTick` 的作用就是在 DOM 更新完成后执行回调函数。在我们的代码中：

```js
function scrollToBottom() {
  nextTick(() => sentinel.value?.scrollIntoView({ block: 'end' }))
}
```

这里使用 `nextTick` 的原因是：如果你在修改数据后立即尝试访问 DOM 元素（比如滚动到某个元素），此时 DOM 可能还没有更新，从而导致操作无效。通过 `nextTick` 包裹 `scrollIntoView` 方法，可以确保在 DOM 更新完成后再执行滚动操作，从而保证滚动到正确的位置。

#### 何时需要使用 nextTick 

- 当你需要在数据变化后访问更新后的 DOM 时
- 当你需要在组件生命周期钩子中执行 DOM 操作时
- 当你需要在修改数据后立即执行某些依赖于新 DOM 状态的操作时

### 核心：流式拉取

首先我们封装了一个 fetch 请求，实际上也就是自己多写了一些东西罢了。

```js
import store from '../store/index'

const getBaseURL = () => {
    return store.state.api;
};

export async function difyRequest({
    path, body=null, method='POST', headers={}, stream=false
}) {
    // const url = getBaseURL() + path
    const url = "http://211.90.240.240:30010" + path
    const isFormData = body instanceof FormData
    const fetchHeaders = isFormData
        ? headers
        : { 'Content-Type': 'application/json', ...headers }
    const options = {
        method, headers: fetchHeaders
    }
    if (body) options.body = isFormData ? body : JSON.stringify(body)
    const res = await fetch(url, options)
    if (!res.ok) throw new Error(`请求失败，状态：${res.status}`)

    return stream ? res : res.json()
}
```

然后我们再写这个流式拉取函数：

```js
async function streamAnswer() {
	message.value = ""
    streaming.value = true

    try {
        const res = await difyRequest({
            path: '/v1/chat-messages',
            body: {
                query: props.loadingData,
                inputs: {},
                response_mode: 'streaming',
                user: 'cdss',
                conversation_id: ''
            },
            headers: {
                'Content-Type': 'application/json',
                'Authorization': 'Bearer ...'
            },
            stream: true
        })

        const reader = res.body!.getReader()
        const decoder = new TextDecoder('utf-8')
        let buffer = ''

        while(true) {
            const { value, done } = await reader.read()
            if(done) break

            buffer += decoder.decode(value, {stream: true})

            const lines = buffer.split('\n')
            buffer = lines.pop()!

            for(const line of lines){
                if(!line.startWith('data: ')) continue
                const payload = line.slice(6)
                if(payload === '[DONE]') continue

				let obj: any
				try{
					obj = JSON.parse(payload)
				} catch{
					continue
				}

				if(obj.event === 'message' && obj.answer){
					message.value += obj.answer
					scrollToBottom()
				}
            }
        }
    } catch(err){
	    message.value = '抱歉，接口出现异常'
	    scrollToBottom()
    } finally{
	    streaming.value = false
    }
}
```

reader 是浏览器 Fetch API 中 ReadableStream 的一个默认读取器（ReadableStreamDefaultReader）实例。

1. res.body 当我们调用 fetch 去调用一些流式返回的接口的时候，返回的 Response 对象的 body 属性就是一个符合 Stream API 的可读流 `ReadableStream<Uint8Array>`我们可以把它想象成一个从服务器不断“输送”二进制数据块（`Uint8Array`）的管道。

2. getReader() ，ReadableStream 本身只是一个管道的抽象，要从里面拿数据，需要先获取一个读取器。调用 `res.body.getReader()` 就是获得了一个 **`ReadableStreamDefaultReader`**。

3. **`reader.read()`** ，拿到 `reader` 后，你通过 `await reader.read()` 来“拉”管道中的下一个数据块。每次 `read()` 都返回一个对象 `{ value: Uint8Array, done: boolean }`：

    - `value` 是本次读到的字节数据；
    - `done` 是布尔，表示流是否已经结束（`true` 时没有更多数据了）

这样我们就简单实现了一个消费 sse 接口的逻辑。

### 监听外部变化

在 Vue 3 的 Composition API 中，`watch` 是用来“观察”一个或多个响应式 **源**（sources），并在它们发生变化时执行 **回调** 的 API。结合你这段代码：

```ts
watch(
  () => props.loadingData,                      
  // 观察的源
  (newVal, oldVal) => Object.keys(newVal).length 
  // 回调：当源变更时被调用
                       && streamAnswer(),
  { immediate: true }                           
  // 选项：初始挂载时也触发一次
)
```

```js
v => Object.keys(v).length && streamAnswer()
```

等同于

```js
v => {
  if (Object.keys(v).length > 0) {
    streamAnswer()
  }
}
```

也就是 **只有当 `props.loadingData` 是一个非空对象**（至少有一个字段）时，才调用 `streamAnswer()`。

JavaScript 的 `&&` 会先计算左边表达式：

- 如果左边是「假值」（`0`、`''`、`null`、`undefined`、`false`），就直接返回左边，不执行右边；

- 如果左边是真值（非零数字、非空字符串、对象等），就继续计算并返回右边的结果。

所以当 `Object.keys(v).length` ≥ 1 时，`streamAnswer()` 会被调用；否则完全跳过。

#### 什么是 Getter 和 Setter

在普通 JavaScript 对象里，你可以用 `get` 和 `set` 关键字，定义访问某个属性时会自动执行的函数：

```js
const obj = {
  _val: 10,
  get value() {
    console.log('读取 value');
    return this._val
  },
  set value(v) {
    console.log('写入 value:', v);
    this._val = v
  }
}

console.log(obj.value)   // 触发 getter，打印 "读取 value"，然后输出 10
obj.value = 42           // 触发 setter，打印 "写入 value: 42"
```

- **Getter**（`get value()`）在你访问 `obj.value` 时跑；
- **Setter**（`set value(v)`）在你给 `obj.value = ...` 赋值时跑。

它们让你能在背后插入“拦截”逻辑。

#### Vue Composition API 中的 `ref` 也有 Getter/Setter

```js
const count = ref(0)
```

- Vue 会给你返回一个对象 `{ value: 0 }`，但它用 `Object.defineProperty` 或 Proxy，在内部给 `.value` 加上了 **getter** 和 **setter**：

    - **读取** `count.value` 时，会触发依赖收集（让模板和 watcher 知道它依赖了这个值）；

    - **写入** `count.value = newVal` 时，会触发更新（让相关组件或 watcher 重新执行）。

你在模板里写 `{{ count }}` 或在 `setup()` 返回 `{ count }`，Vue 会把 `.value` 自动解包给模板，你无需手动 `.value`。

#### vue watch 方法签名解读

```ts
watch(
  source,
  callback,
  options?
)
```

##### source

**`source`**

- 可以是一个 **getter 函数**（`() => someRef.value`、`() => props.xxx`、`() => state.obj` 等），

- 也可以是一个 **响应式引用**（`Ref`）、一个 **响应式对象**（`reactive`）、甚至一个数组或元组，支持多源同时监听。

---

##### callback

`callback`

```ts
(newValue, oldValue, onCleanup) => { /* ... */ }
```

- **`newValue`**：最新的源的值。

- **`oldValue`**：变化前的值。

- **`onCleanup`**：可选，用来注册一个清理函数，当下次触发回调或 watcher 被停止时，会先执行清理，常用于异步请求或订阅的取消。

在这个回调函数的 {...} 里面，我们可以放置**当源变化时**要执行的逻辑，包括用到 `newValue`、`oldValue`，以及通过 `onCleanup` 注册 “下次变化前或组件卸载时” 要做的清理工作。

像这里就是：

```ts
watch(
  () => props.loadingData,
  (newVal, oldVal) => {
    // 当新的 loadingData 是非空对象时，触发 streamAnswer
    if (Object.keys(newVal).length > 0) {
      streamAnswer()
    }
  },
  { immediate: true }
)
```

```ts
watch(
  () => props.count,
  (newCount, oldCount) => {
    console.log(`count 从 ${oldCount} 变为了 ${newCount}`)

    if (newCount > oldCount) {
      // 递增时做 A
    } else {
      // 递减时做 B
    }
  }
)
```

Tips：回调函数的参数 **名字** 本身没有任何强制要求，只要它们在正确的位置，Vue 就会把对应的值传进来。

---

##### options

**`options`**（可选）

- **`immediate: boolean`**
    
    - `false`（默认）：只有当源第一次发生 **变化** 时才调用一次回调；
        
    - `true`：在 watcher 创建后 **立刻** 用当前值调用一次回调。

- **`deep: boolean`**
    
    - `false`（默认）：只对 **引用** 的变化敏感（`Ref` 或对象指针改变）；
        
    - `true`：对对象内部任意深层属性的改变都能触发。
        
- **`flush: 'pre' | 'post' | 'sync'`**
    
    - 决定回调是在组件更新周期的哪个阶段执行：
        
        - `'pre'`：在渲染前；
            
        - `'post'`（默认）：在渲染并更新 DOM 之后；
            
        - `'sync'`：同步立即执行（会阻塞后续渲染）。



