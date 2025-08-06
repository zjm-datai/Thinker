
### 什么是 css 弹性盒模型？

Flexbox 是弹性盒布局模块（Flexible Box Layout module）的缩写。  

Flexbox 是一种用于按行或列排列项目的布局方法。  

Flexbox 使得在不使用浮动或定位的情况下，更轻松地设计灵活的响应式布局结构。  

#### Flexbox 与网格布局（Grid）的比较  

CSS 弹性盒布局（Flexbox Layout）应用于一维布局，即按行或列布局。  

CSS 网格布局（Grid Layout）应用于二维布局，即按行和列布局。

### CSS Flexible Box Layout Module

Before the Flexible Box Layout module, there were four layout modes:

- Block, for sections in a webpage
- Inline, for text
- Table, for two-dimensional table data
- Positioned, for explicit position of an element

CSS flexbox is supported in all modern browsers.

### CSS 弹性盒模型组件

一个 flexbox 总是由以下部分组成：

- 一个 flex container - the parent (container) `<div>` element
- **Flex Items** - the items inside the container `<div>` 

### A Flex Container with Three Flex Items

```html
<div class="flex-container">  
  <div>1</div>  
  <div>2</div>  
  <div>3</div>  
</div>
```

