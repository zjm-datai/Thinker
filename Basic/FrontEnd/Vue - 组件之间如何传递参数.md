在 Vue 里，**“组件之间传递参数”** 可以分成四大类场景：

|场景|典型方向|关键 API／机制|何时适用|
|---|---|---|---|
|父 → 子|单向数据流|`props`|绝大多数日常情况|
|子 → 父|事件回传|`emit()`（Vue 3） / `$emit`（Vue 2）|子组件需要把结果、用户操作、状态变化告诉父组件|
|深层级 / 祖孙|跨层级注入|`provide` / `inject`|祖辈向多层后代共享同一份数据或函数|
|兄弟 & 跨模块|全局状态 or 事件|**Pinia / Vuex**、`mitt`、`ref/reactive` 单例、URL 参数等|组件并无父子关系，或数据需要在页面间持久|

下面按常用程度分别举例。

---

## 1. 父传子：`props`

```vue
<!-- Parent.vue -->
<script setup>
import Child from './Child.vue'
const user = { id: 1, name: 'Alice' }
</script>

<template>
  <Child :user="user" :theme="'dark'" />
</template>
```

```vue
<!-- Child.vue -->
<script setup>
defineProps<{
  user: { id: number; name: string }
  theme: string
}>()
</script>
```

要点

- 传对象/数组时是 **引用**，子组件修改会影响父组件；若想只读，建议在子组件里拷贝或使用 `readonly()`.
    
- 可用 `withDefaults(defineProps(), { theme: 'light' })` 设默认值。
    

---

## 2. 子传父：`emit`

```vue
<!-- Child.vue -->
<script setup lang="ts">
const emit = defineEmits<{ (e: 'submit', payload: { name: string }): void }>()
function handleClick() {
  emit('submit', { name: 'Bob' })
}
</script>

<template><button @click="handleClick">Submit</button></template>
```

```vue
<!-- Parent.vue -->
<Child @submit="onSubmit" />
<script setup>
function onSubmit(payload) {
  console.log(payload) // { name: 'Bob' }
}
</script>
```

---

## 3. 跨层级：`provide / inject`

```vue
// GrandParent.vue
<script setup>
import { ref, provide } from 'vue'
const locale = ref('zh-CN')
provide('locale', locale)
</script>
```

```vue
// DeepChild.vue
<script setup>
import { inject } from 'vue'
const locale = inject('locale', 'en-US')  // 第二个参数是默认值
</script>
```

- 适合同一页面内**树状**共享数据（如主题、i18n、表单校验依赖）。
    
- 注意：`inject` 得到的是 **响应式引用**（若祖先用 `ref`/`reactive`）。
    

---

## 4. 兄弟组件或跨页面

### 4-1 Pinia（推荐）

```ts
// stores/counter.ts
import { defineStore } from 'pinia'
export const useCounter = defineStore('counter', {
  state: () => ({ n: 0 }),
  actions: { inc() { this.n++ } }
})
```

```vue
<!-- AnyComponent.vue -->
<script setup>
const counter = useCounter()
</script>
<template>{{ counter.n }} <button @click="counter.inc">+</button></template>
```

- Pinia 支持持久化插件、TypeScript 完整推导，几乎取代 Vuex。
    

### 4-2 事件总线

```ts
// eventBus.ts
import mitt from 'mitt'
export const bus = mitt()
```

```vue
// A.vue
bus.emit('refresh', data)
```

```vue
// B.vue
bus.on('refresh', handler)
```

- 轻量、无依赖，但只能存活在当前标签页，且监听与销毁要管理好。
    

### 4-3 共享单例 `reactive`

```ts
// sharedState.ts
import { reactive } from 'vue'
export const appState = reactive({ token: '' })
```

- 组件 A、B 导入同一个对象即可同步响应式。
    

---

## 5. 其他技巧

|技巧|用途|
|---|---|
|**插槽 (slots)**|向子组件“传递”模板片段。可结合作用域插槽返回子组件内部数据。|
|**URL 查询参数 / 路由 params**|页面级别通信（如 `/article/42?preview=1`）。|
|**`v-model` 在组件上**|语法糖，等价于 `modelValue` prop + `update:modelValue` emit。|
|**`expose()`**|子组件主动暴露内部方法给父组件通过 `ref` 调用。|

---

### 总结记忆口诀

> **“上 props，下 emit，跨层 provide/inject，远亲 Pinia（或事件）。”**

按照数据流向、组件层级与持久化需求选择最合适的方法即可。祝编码愉快！