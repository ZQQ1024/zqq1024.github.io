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

`v-bind`（缩写 `:`） 用于单向绑定数据到 DOM 元素的属性（如 class, style, href 等）。这意味着你可以动态设置 HTML 属性的值。

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

`v-on` （缩写`@`）是一个用于监听 DOM 事件并执行一些 JavaScript 代码的指令。这个指令非常有用，因为它可以让你在用户与网页交互时响应事件，如点击、输入、移动鼠标等。

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

`v-show` 是 Vue.js 中用于控制元素显示或隐藏的指令。它通过切换元素的 CSS display 属性来实现，而不是像 `v-if` 那样在 DOM 中添加或删除元素。这使得 `v-show` 非常适合于需要频繁切换显示状态的场景，因为它避免了频繁地操作 DOM 元素的代价。

```vue
<div id="app">
  <button @click="isVisible = !isVisible">Toggle Visibility</button>
  <p v-show="isVisible">You can see me now!</p>
</div>

new Vue({
  el: '#app',
  data() {
    return {
      isVisible: true
    };
  }
});
```

### v-pre

在 Vue.js 中，`v-pre` 是一个用来跳过当前元素和它的子元素的编译过程的指令。使用 `v-pre` 可以优化性能，特别是当你知道某些内容不需要 Vue 处理时，这个指令可以防止 Vue 解析或编译内部的表达式和指令。

```vue
<div id="app">
  <p v-pre>{{ This will not be compiled by Vue }}</p>
  <p>{{ message }}</p>
</div>

new Vue({
  el: '#app',
  data: {
    message: 'This will be compiled by Vue'
  }
});
```

在这个例子中：
- 第一个 <p> 元素使用了 v-pre，所以 {{ This will not be compiled by Vue }} 会原样显示。
- 第二个 <p> 元素没有使用 v-pre，因此 Vue 会解析 {{ message }} 并显示其对应的数据，即 `This will be compiled by Vue`。

### v-once

在 Vue.js 中，`v-once` 指令用于渲染静态内容。使用 v-once 的元素和组件只会被渲染一次，之后即使关联的数据发生变化，渲染结果也不会更新。这对于需要显示不会改变的数据的情况非常有用，可以提高性能，因为 Vue 不需要追踪这些数据的变化。

```vue
<div id="app">
  <!-- 静态内容使用 v-once -->
  <h1 v-once>Welcome to the site!</h1>
  
  <!-- 动态内容不使用 v-once -->
  <p>Current time: {{ currentTime }}</p>
</div>

new Vue({
  el: '#app',
  data: {
    currentTime: new Date().toLocaleTimeString()
  },
  mounted() {
    setInterval(() => {
      this.currentTime = new Date().toLocaleTimeString();
    }, 1000);
  }
});
```

在这个例子中：
- <h1> 元素使用 v-once，这意味着 "Welcome to the site!" 只会在页面加载时被渲染一次，之后即使组件重新渲染，这一行也不会更新。
- <p> 元素显示当前时间，每秒更新一次。这部分没有使用 v-once，因此它会随着 currentTime 数据的更新而动态更新。

## 组合式API

Vue 3 引入了一个新的编程范式称为 组合式 API（Composition API），这是一个可选的 API，旨在解决 Vue 2 中使用选项式 API（Options API）时遇到的一些限制，特别是在处理复杂组件时的可维护性问题。组合式 API 提供了更好的逻辑复用和代码组织方式。使用 JavaScript 函数来创建和管理状态、生命周期钩子和其他响应式功能，而不是将它们分散在 data、methods、computed 等选项中。在 Vue 3 的组合式 API 中，`setup()` 函数是一个新的组件选项，它充当组件使用组合式 API 的入口点，`setup()` 在组件被创建之前执行。在使用组合式 API 时，`setup()` 函数取代了 `data`、`computed`、`methods` 和大部分的 `lifecycle hooks` 等传统选项。

```vue
<template>
  <button @click="increment">{{ count }}</button>
</template>

<script>
import { ref, onMounted } from 'vue';

export default {
  setup() {
    const count = ref(0);

    function increment() {
      count.value++;
    }

    onMounted(() => {
      console.log('Component is mounted!');
    });

    return {
      count,
      increment
    };
  }
}
</script>
```

### 响应式数据

在 Vue 3 的组合式 API 中，`ref` 和 `reactive` 是两个核心的响应式 API，用于定义和管理响应式状态。它们使得状态的变更能自动触发视图的更新。响应式是指开发者只需关注数据的状态，而 Vue 框架负责确保视图与数据保持同步。

`ref` 用于定义一个响应式的引用，它可以包装一个基本类型或一个对象，但主要用于基本类型。使用 `ref` 包装的变量将被转化为一个对象，该对象有一个 `.value` 属性来获取或设置其内部的值。示例如下：

```vue
import { ref } from 'vue';

export default {
  setup() {
    const count = ref(0);

    function increment() {
      count.value++;
    }

    return { count, increment };
  }
}
```

在这个例子中，count 是一个通过 `ref` 创建的响应式引用。在模板或其他响应式上下文中，可以直接使用 `count` 而不需要 `.value`。在 JavaScript 逻辑中访问或修改它时，需要使用 `count.value`。

`reactive` 用于创建一个响应式的对象。它深度地使对象的所有嵌套属性变为响应式的。这对于处理具有多个属性和嵌套对象的复杂状态非常有用。示例如下：
```vue
import { reactive } from 'vue';

export default {
  setup() {
    const state = reactive({
      count: 0,
      details: {
        text: 'Hello'
      }
    });

    function increment() {
      state.count++;
    }

    function updateText(newText) {
      state.details.text = newText;
    }

    return { state, increment, updateText };
  }
}
```

在这个示例中，`state` 是一个复杂对象，其中包含 `count` 和一个嵌套的对象 `details`。通过 `reactive`，整个 `state` 对象及其嵌套属性都变成了响应式，可以直接在模板或其他响应式函数中使用，而不需要 `.value`。

`computed` 是一个非常有用的特性，用于创建响应式的计算属性。计算属性基于响应式依赖进行自动计算，并且仅在相关依赖发生变化时重新计算，这使得它们非常高效。这种特性特别适用于需要根据现有数据派生新数据的场景。

```vue
import { computed, ref } from 'vue';

export default {
  setup() {
    const firstName = ref('John');
    const lastName = ref('Doe');
    const fullName = computed(() => `${firstName.value} ${lastName.value}`);

    return { firstName, lastName, fullName };
  }
}
```

在这个例子中，`fullName` 是一个计算属性，它依赖于两个响应式引用 `firstName` 和 `lastName`。任何对这些依赖的修改都会触发 `fullName` 的重新计算。

侦听器`watch` 允许你指定一个或多个数据源来监视，当这些数据源变化时，你可以执行一些特定的逻辑。示例如下：

```vue
import { ref, watch } from 'vue';

export default {
  setup() {
    const question = ref('');

    watch(question, (newValue, oldValue) => {
      console.log(`The question changed from ${oldValue} to ${newValue}!`);
      // 可以在这里执行更多操作
    });

    return { question };
  }
};
```

### 生命周期钩子

组合式 API 中的生命周期勾子是通过特定的函数来实现的，这些函数可以在 `setup()` 函数中直接使用。以下是 Vue 3 中可用的生命周期勾子函数：
- `onBeforeMount`：在组件挂载到 DOM 之前调用。
- `onMounted`：在组件挂载到 DOM 后调用。
- `onBeforeUpdate`：在组件更新之前调用。
- `onUpdated`：在组件更新后调用。
- `onBeforeUnmount`：在卸载组件之前调用。
- `onUnmounted`：在组件完全卸载后调用。
- `onErrorCaptured`：在捕获组件的一个错误时调用。
- `onRenderTracked`：在依赖追踪时调用（调试用途）。
- `onRenderTriggered`：在依赖触发重新渲染时调用（调试用途）。

示例：
```vue
import { onMounted, onBeforeUnmount, ref } from 'vue';

export default {
  setup() {
    const count = ref(0);

    onMounted(() => {
      console.log("Component is now mounted to the DOM!");
    });

    onBeforeUnmount(() => {
      console.log("Component is about to be unmounted from the DOM!");
    });

    const increment = () => {
      count.value++;
    };

    return { count, increment };
  }
}
```

### 组件和props

```vue
import { defineComponent, ref } from 'vue';

export default defineComponent({
  props: {
    title: String
  },
  setup(props) {
    const title = ref(props.title);
    return { title };
  }
});
```

在这个组件中：
- `props`：定义了一个名为 title 的 prop，类型为 String。这个 prop 可以从父组件传入。声明组件希望从其父组件接收什么样的数据，这种声明不仅明确了数据的类型，还可以指定数据的必要性、默认值以及自定义验证。
- 将 `props` 作为参数传递给 `setup()`，可以直接在 `setup()` 函数内部使用这些 `props`，在 `setup()` 中通过参数传入的 `props` 是响应式的。这意味着你可以在 `setup()` 内部使用它们来创建计算属性或者直接在模板中使用，它们会响应外部的变化。
- `defineComponent` 是一个帮助函数，用于定义 Vue 组件。尽管它不是创建组件的必需方法（你可以直接使用一个对象来定义组件）。

### provide/inject

在 Vue.js 中，`provide` 和 `inject` 是用于实现高级组件间通信的一对 API。这种机制允许一个祖先组件向所有其子孙组件提供数据，而无需通过所有层级的 `props` 逐层传递。`provide` 和 `inject` 特别适用于深层嵌套的组件，或者当你需要在一个组件树中多个位置访问同样的数据时。在 Vue 2 中，如果你 provide 一个普通对象而非响应式对象，后代组件接收到的数据将不是响应式的。在 Vue 3 中，推荐使用 `reactive` 或 `ref` 来确保数据的响应性。

```vue
import { provide, reactive } from 'vue';

export default {
  setup() {
    const state = reactive({
      user: 'John Doe',
      isAuthenticated: true
    });

    provide('userData', state);

    // 其余的 setup 逻辑
  }
};
```

在这个例子中，我们创建了一个响应式的 `state` 对象，并通过 `provide` 函数提供了这个对象。我们使用的键是 `userData`。

```vue
import { inject } from 'vue';

export default {
  setup() {
    const userData = inject('userData');

    return {
      userData
    };
  }
};
```

在这个示例中，子组件使用 `inject` 函数通过之前祖先组件 `provide` 的键 `userData` 来接收数据。

## Vue Router

Vue Router 是 Vue.js 的官方路由管理器。Vue Router 允许你通过不同的 URL 访问不同的内容而不需要重新加载页面，提供了基于 Vue.js 组件的路由系统，是单页应用（Single Page Application, SPA）的一个核心特性。

{{< hint info >}}
在传统的多页应用（MPA）中，每当用户请求一个新页面（比如点击链接），浏览器会向服务器发送一个完整的页面请求，服务器响应这个请求并发送整个新页面的 HTML 回浏览器，这导致整个页面重新加载。这个过程往往伴随着明显的闪烁和延迟。

相比之下，单页应用（SPA）通过前端路由管理器如 Vue Router 控制 URL 的变化，并且不会引发真正的页面重新加载。工作流程如下：
1. 初始化加载：当你首次访问 SPA 时，应用加载必要的 HTML、CSS 和 JavaScript。这是唯一的一次完整加载。
2. 路由管理：在用户与应用交互过程中（比如点击链接或者按钮导航到不同部分），**Vue Router 会拦截浏览器**的 URL 变化并决定不发送真正的新请求给服务器。
3. 视图更新：Vue Router 根据当前 URL 的变化动态加载或展示相应的组件，而无需重新加载整个页面。它利用了浏览器的 History API 来管理浏览历史记录、前进和后退的行为。
4. 数据处理：如果新的视图需要新的数据，前端应用会向服务器发送 API 请求，获取必要的数据，并更新展示在当前视图的内容。这一切都在不重新加载整个页面的情况下进行。
{{< /hint >}}

```vue
import { createRouter, createWebHistory } from 'vue-router';
import Home from './components/Home.vue';
import About from './components/About.vue';

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: Home },
    { path: '/about', component: About }
  ]
});

export default router;
```

## 组件注册

在 Vue.js 中，组件可以通过两种主要方式进行注册：全局注册和局部注册。

全局注册组件的方式在 Vue.js 应用开发中非常有用，尤其是当某个组件需要在多个地方使用时，例如通用的 UI 控件如按钮、输入框、导航条等。

创建一个简单的 `BaseButton` 组件`BaseButton.vue`
```vue
<template>
  <button class="base-button">{{ buttonText }}</button>
</template>

<script>
export default {
  name: 'BaseButton',
  props: {
    buttonText: String
  }
}
</script>

<style>
.base-button {
  padding: 10px 20px;
  background-color: #007BFF;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}
</style>
```

应用的入口文件（通常是 main.js）进行全局注册。

`main.js`
```vue
import { createApp } from 'vue'
import App from './App.vue'
import BaseButton from './components/BaseButton.vue'

const app = createApp(App)

// 全局注册 BaseButton 组件
app.component('BaseButton', BaseButton)

app.mount('#app')
```

我们使用 `app.component` 方法注册了 `BaseButton`，这意味着 `BaseButton` 可以在任何其他组件的模板中直接使用，而无需再次导入和注册。

现在，`BaseButton` 已经全局注册，你可以在应用中的任何其他组件中直接使用它，无需额外的导入或局部注册。

`Home.vue`
```vue
<template>
  <div>
    <h1>Welcome to the Home Page</h1>
    <BaseButton buttonText="Click Me!"></BaseButton>
  </div>
</template>

<script>
export default {
  name: 'Home'
}
</script>
```

局部注册组件的方法是在需要使用组件的特定 Vue 文件中导入并注册。这种方式在 Vue 应用中非常常见，特别适合那些仅在少数几个地方使用的组件。

假设我们有一个 `SpecialButton` 组件，这个组件可能只在某个特定视图 `UserProfile` 中使用

`SpecialButton.vue`:
```vue
<template>
  <button class="special-button">{{ buttonText }}</button>
</template>

<script>
export default {
  name: 'SpecialButton',
  props: {
    buttonText: String
  }
}
</script>

<style>
.special-button {
  padding: 10px 20px;
  background-color: #28a745;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}
</style>
```

`UserProfile.vue`
```vue
<template>
  <div>
    <h1>User Profile</h1>
    <!-- 使用局部注册的 SpecialButton 组件 -->
    <SpecialButton buttonText="Edit Profile"></SpecialButton>
  </div>
</template>

<script>
import SpecialButton from './SpecialButton.vue'

export default {
  name: 'UserProfile',
  components: {
    SpecialButton
  }
}
</script>
```

`SpecialButton` 组件是在 `UserProfile` 组件的 components 选项中注册的。这意味着 `SpecialButton` 只能在 `UserProfile` 组件的模板中使用，像使用任何其他标签一样。
