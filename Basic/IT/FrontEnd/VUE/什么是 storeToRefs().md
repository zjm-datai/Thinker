
storeToRefs() 是 Pinia 提供的一个辅助函数，用于将 store 实例中的 state 和 getters 映射成一组 ref 对象，使得它们在解构之后仍然保持响应性，并且同时将无用的 actions 与非响应式字段过滤掉。

为什么手动解构 store 会丢失响应性？

下面这种写法不会触发组件更新：

```ts
const { base, detail } = patientStore
```

这是因为这样做只是取出当时的值，而不是引用一个响应式状态，一旦 store 中对应字段改变，组件不会自动重新渲染。Vue 的响应式机制不会追踪解构后的普通变量，导致潜在的 UI 不更新问题。

而 storeToRefs 会把每个字段变为一个 Ref，引用了原始响应式数据，在 store 更新时，该 ref 也会触发组件重新渲染，保持数据同步。

```ts
const patientStore = usePatientStore()
const { base, detail, scanCode, isLoading, errorMsg } = storeToRefs(patientStore)
```

- base.value 即时映射到 store.base 的当前值；
    
- 若 patientStore.base 更新，UI 中通过 base.value 显示的内容也会同步刷新；

