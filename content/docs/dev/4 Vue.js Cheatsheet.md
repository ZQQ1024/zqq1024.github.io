---
title: "Vue.js Cheatsheet"
weight: 4
bookToc: true
---

## 项目创建
```bash
npm install -g @vue/cli
vue create my-project
```

## 创建挂载实例
```js
import { createApp } from 'vue';
import App from './App.vue'; // App.vue 根组件，它是整个 Vue 应用的主要入口点

const app = createApp(App);
app.mount('#app');
```

## vue文件格式

Vue.js 的单文件组件（Single File Component，简称 SFC）是 Vue 项目中常用的一种文件类型，通常以 .vue 扩展名结束。这种文件类型将 HTML、JavaScript 和 CSS 封装在一个文件中，每个部分对应一个标签：

`<template>`、`<script>` 和 `<style>`

`<template>` 标签 这部分包含 Vue 组件的 HTML 模板。它定义了组件的结构和呈现的 HTML。在这个部分中，你可以使用 Vue 的模板语法，如插值、指令（如 v-if, v-for 等）、事件处理等

`<script>` 标签 这部分用于编写 JavaScript 代码，通常包含组件的逻辑、数据、方法等。在 `<script>` 标签中，你可以定义组件的属性、生命周期钩子、事件等。

`<style>` 标签 这部分包含组件的 CSS 样式。默认情况下，这里的样式是全局的，可以通过添加 scoped 属性使得样式只应用于当前组件。

```vue
<template>
  <div class="message-box">
    <p>{{ message }}</p>
    <button @click="reverseMessage">Reverse Message</button>
  </div>
</template>

<script>
export default {
  name: 'MessageBox',
  data() {
    return {
      message: 'Hello, Vue!'
    };
  },
  methods: {
    reverseMessage() {
      this.message = this.message.split('').reverse().join('');
    }
  }
}
</script>

<style scoped>
.message-box {
  padding: 20px;
  border: 1px solid #ccc;
  margin-top: 20px;
}
</style>
```

这个组件会显示一条消息和一个按钮，点击按钮会反转消息中的文本

## 模版指令
在 Vue.js 中，模板指令是一些特殊的属性，用于在模板中添加反应性行为到 DOM 元素上。这些指令以 `v-` 前缀开始，表明它们是 Vue 的特殊属性。使用这些指令可以简化 DOM 操作和数据绑定的过程，当数据变化时，视图会自动更新，无需任何 DOM 操作

声明式渲染
```js
<p>{{ message }}</p>
```
上面的代码自动将 message 数据与文本内容绑定，当 message 改变时，文本内容也会自动更新

### v-bind

`v-bind` 用于单向绑定数据到 DOM 元素的属性（如 class, style, href 等）。这意味着你可以动态设置 HTML 属性的值。

- 单向数据流：数据从 Vue 实例流向模板，如果数据变化，绑定的属性将自动更新，但如果属性变化（如用户交互导致的变化），数据不会自动更新 Vue 实例中的数据。
- 用途：主要用于动态更新 HTML 属性。

```js
<img v-bind:src="imageSrc">
<!-- 或者使用缩写 -->
<img :src="imageSrc">
```

在这个例子中，`imageSrc` 变量的值会被设置为 `<img>` 标签的 `src` 属性，当 `imageSrc` 变化时，图片的来源也会相应更新。

### v-model

`v-model` 用于在表单元素（如 `input`, `textarea`, `select` 等）上创建双向数据绑定。这意味着它不仅可以将 Vue 实例的数据绑定到表单元素，还可以将表单元素的变化实时同步回数据。

- 双向数据流：数据可以从 Vue 实例流向模板，并且用户在表单元素上的输入会自动更新 Vue 实例中的数据。
- 用途：主要用于表单数据的双向绑定，简化表单元素与数据状态同步的处理。

```js
<input v-model="username">
```

在这个例子中，`username` 数据属性与 `<input>` 元素的值双向绑定。用户在输入框中的任何输入都会即时反映到 `username` 数据属性中，同时如果 `username` 的值被程序修改，输入框的内容也会更新。

### v-for

`v-for` 用于基于一个数组渲染一个元素列表。这个指令可以在模板中遍历数组或对象的每个项，并为每个项生成新的 DOM 元素。

```vue
data() {
  return {
    items: [
      { id: 1, text: 'Apple' },
      { id: 2, text: 'Banana' },
      { id: 3, text: 'Cherry' }
    ]
  };
}

<ul>
  <li v-for="item in items" :key="item.id">
    {{ item.text }}
  </li>
</ul>
```

```vue
data() {
  return {
    userProfile: {
      name: 'Alice',
      age: 25,
      location: 'New York'
    }
  };
}

<ul>
  <li v-for="(value, key, index) in userProfile" :key="key"> // index 从 0 开始计数，顺序是 value, key 与 Object.entries() 一致
    {{ index }} - {{ key }}: {{ value }}
  </li>
</ul>
```

### v-if/v-else-if/v-else

`v-if`, `v-else-if`, 和 `v-else` 是用于条件渲染的指令。这些指令允许你根据不同的条件动态地渲染元素。这种条件渲染不仅仅隐藏或显示元素，它实际上会在满足条件时添加元素到 DOM 中，不满足时则从 DOM 中移除元素。
```vue
new Vue({
  el: '#app',
  data: {
    type: 'B'
  }
});

<div id="app">
  <p v-if="type === 'A'">Type is A</p>
  <p v-else-if="type === 'B'">Type is B</p>
  <p v-else-if="type === 'C'">Type is C</p>
  <p v-else>Type is neither A, B, nor C</p>
</div>
```

在这个实例中，因为 type 初始化为 'B'，所以页面上会显示 "Type is B"。由于 `v-if` 指令会频繁地添加和删除元素，如果涉及到频繁变化的条件，使用 `v-show` 可能更高效。`v-show` 只是简单地切换元素的 CSS `display` 属性，而不是在 DOM 中添加和移除元素。当你需要频繁切换某个元素的显示状态时，考虑使用 `v-show`。

### v-on

`v-on` 是一个用于监听 DOM 事件并执行一些 JavaScript 代码的指令。这个指令非常有用，因为它可以让你在用户与网页交互时响应事件，如点击、输入、移动鼠标等。

```vue
<button v-on:click="doSomething">
  Click me
</button>

new Vue({
  el: '#app',
  methods: {
    doSomething() {
      alert('Button clicked!');
    }
  }
});
```

在这个例子中，每当按钮被点击，doSomething 方法就会被调用，弹出一个警告框。

### v-show

### v-pre

### v-cloak

### v-once

