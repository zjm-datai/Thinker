
在 Vue 中，id 和 ref 是两个不同的概念，用途和工作方式有明显的区别：

### id 属性

- 是原生 HTML 元素的属性，用于标识 DOM 元素
- 在整个页面中保持唯一

通过 document.getElementById() 获取 DOM 元素，可用于 css 选择器定位元素

### ref 属性

- 是 Vue 提供的特殊属性，用于在组件中获取 DOM 元素或子组件实例
- 只在当前 Vue 组件实例范围内有效

通过 this.$refs.refName 在 vue 组件中访问 DOM 元素

- 访问子组件实例，调用子组件的方法或访问其数据
- 避免直接操作 DOM 时使用 getElementById 等原生方法

---

当需要与原生 DOM API 交互或关联 label 时，使用 id