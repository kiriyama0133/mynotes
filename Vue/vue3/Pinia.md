​	**Pinia** 是 Vue 3 官方推荐的状态管理库，它是 **Vuex** 的继任者，专门为 Vue 3 设计。Pinia 提供了一个更加简洁、直观、功能强大的方式来管理和**共享应用中的状态（数据）**。

​	在 Vue 3 中，Pinia 通过使用 **Composition API** 提供了更灵活的状态管理方式，相比 Vuex，它有一些显著的优势，例如更好的类型推导（支持 TypeScript）、更简洁的 API，以及与 Vue 3 的新特性（如 Vue Router、Vue Devtools 等）更好地兼容。

### Pinia 的特点：

1. **简洁易用**： Pinia 提供了更简洁的 API，使得状态管理更加直观，减少了冗余的代码。例如，获取状态、修改状态以及定义 getters 和 actions 都变得非常简单。
2. **TypeScript 支持**： Pinia 完美支持 TypeScript，提供了**类型推导和检查功能**。你可以在开发过程中得到**更好的类型提示和代码补全**。
3. **支持模块化**： 你可以**创建多个 store（类似于 Vuex 的模块）**，以便**组织和管理应用程序的状态**，使得大项目的状态管理更加清晰和易于维护。
4. **与 Vue 3 配合得更好**： Pinia 完全兼容 Vue 3 的 Composition API，充分利用了 Vue 3 的响应式系统，确保状态管理更加高效和灵活。

### 使用 Pinia 的示例

#### 安装：

```
npm install pinia
```

#### 创建 Store：

首先，我们创建一个 store 来管理应用的状态。例如，定义一个 `counterStore` 来管理计数器状态。

```
// stores/counterStore.ts
import { defineStore } from 'pinia';

export const useCounterStore = defineStore('counter', {
  state: () => {
    return {
      count: 0,
    };
  },
  getters: {
    // 计算属性
    doubleCount: (state) => state.count * 2,
  },
  actions: {
    // 修改状态的方法
    increment() {
      this.count++;
    },
    decrement() {
      this.count--;
    },
  },
});
```

#### 使用 Store：

在组件中，我们可以通过 `useCounterStore` 来访问和修改 store 的状态。

```
<template>
  <div>
    <p>Count: {{ counter.count }}</p>
    <p>Double Count: {{ counter.doubleCount }}</p>
    <button @click="counter.increment">Increment</button>
    <button @click="counter.decrement">Decrement</button>
  </div>
</template>

<script setup>
import { useCounterStore } from './stores/counterStore';

const counter = useCounterStore();
</script>
```

#### 配置 Pinia：

通常，Pinia 需要在应用的根组件中进行配置。

```
// main.ts
import { createApp } from 'vue';
import { createPinia } from 'pinia';
import App from './App.vue';

const app = createApp(App);
app.use(createPinia());  // 将 Pinia 添加到 Vue 应用中
app.mount('#app');
```

### 什么是状态管理？

**状态管理** 是指管理应用程序中各种数据的流程和方法。在现代前端开发中，尤其是在构建复杂应用时，状态管理显得尤为重要。状态通常指的是应用的变量数据，它们可以影响 UI 渲染和用户交互，**也就是在组件之间可以共享重要的数据**。

- **全局状态管理**：通常指的是将跨多个组件或页面需要共享的数据放到一个中心化的管理位置，比如 Vuex、Pinia、Redux 等工具。这样，你可以在任何需要的地方访问这些共享的状态。
- **局部状态管理**：则是指在某个组件内部管理该组件的局部数据。Vue 的 `data` 和 React 的 `useState` 就是局部状态管理的典型例子。

### 状态管理的意义：

1. **数据共享**： 状态管理使得不同组件可以方便地共享数据，避免了组件之间通过 props 传递数据的繁琐过程，尤其是当多个嵌套组件需要同一数据时，中心化的状态管理就显得非常有用。
2. **组件解耦**： 状态管理可以将数据的管理从组件逻辑中分离出来，使得组件更加关注视图的渲染，而不是处理数据和逻辑。这样，代码的结构和维护性会更好。
3. **可预测性**： 状态管理库（如 Pinia）往往提供了明确的规则和约定，保证了数据的流动性和一致性。当所有的数据操作都通过 store 来进行时，整个应用的状态变得更容易追踪和调试。
4. **便于维护**： 随着项目规模的扩大，应用的状态可能变得复杂且难以维护。通过中心化的状态管理，我们可以清晰地看到哪些数据是全局共享的，哪些是局部状态。

## 使用

现在先引入pinia创建pinia再安装。

```ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue' //根组件
//引入路由器
import router from './router'
//创建应用并使用路由器
const app = createApp(App)
const pinia = createPinia()
app.use(router)
app.use(pinia)
app.mount('#app') //挂载到页面上

```

这样就可以使用了。

就可以在开发工具里看见pinia了。

![image-20250222150130242](./assets/image-20250222150130242.png)

### 存储和读取数据的方式

一般情况我们会在/src文件夹里添加一个/store的文件夹，然后依照组件的名称去定义.ts的文件，在里面编写存储数据的方式。   state是真正存储数据的地方

![image-20250222151025993](./assets/image-20250222151025993.png)

### 修改数据

#### ￥patch方法

$patch:



![image-20250222152549936](./assets/image-20250222152549936.png)

批量改变的方式。

#### action方式

​	actions里面是放置的动作方法，用于响应组件中的动作。



![image-20250222152903004](./assets/image-20250222152903004.png)

在.ts里修改和编写。

​	我有一个疑问，hooks可以把一个组件的数据的处理和方法集成在一起，pinia的写法又可以把数据的存储和处理放在一起，那么在实际的开发，我要怎么权衡这两种方法。

你的问题很有意义，实际上，`Hooks` 和 `Pinia` 主要适用于不同的场景，理解它们各自的优势，可以帮助你在开发中作出合理的选择。

### 1. **组件状态 vs 全局状态管理**

- **Hooks**：主要用于**管理组件内部的状态和逻辑**。当你只需要在一个组件内部处理一些数据，并且**这个数据不会被其他组件共享时**，使用 `useState`, `useEffect` 等 Hook 来管理局部状态是很简洁且高效的。它的好处是让每个组件保持独立，管理逻辑清晰，不需要考虑全局状态的复杂性。
- **Pinia**：用于全局状态管理。它通常在应用的各个部分都需要访问和更新相同的数据时使用。例如，用户登录状态、购物车内容等需要在多个页面或者组件之间共享的数据，Pinia 提供了集中式管理和一致性。你将数据的存储和方法都封装在 `store` 中，多个组件可以通过 `Pinia` 来访问和更新这些状态。

### 2. **何时使用 Hook，何时使用 Pinia**

- 使用 Hooks：
  - 适合局部状态：当一个状态只在一个组件内部使用时，不需要依赖于其他组件，使用 Hooks 是最简单和直接的选择。例如，一个表单输入框的状态，或者一个展开/折叠菜单的状态。
  - 简单逻辑处理：如果一个组件的逻辑简单且不需要跨组件共享数据，直接在组件内部用 Hooks 来处理是非常高效的。
- 使用 Pinia：
  - 适合**全局共享状态**：当**多个组件需要访问或修改同一份数据时，使用 Pinia 会让数据管理更加集中和清晰**。例如，用户认证信息、购物车数据等，在不同页面之间共享时就很适合使用 Pinia。
  - 跨页面状态管理：如果你需要在多个页面之间传递数据，Pinia 的 `store` 可以作为一个单一的数据源，方便进行全局数据管理，避免了通过 props 或 context 进行复杂的传递。

### 3. **代码结构和维护性**

- **Hooks** 提供了一个非常直观的方式来将组件的状态与逻辑封装在一起，这使得组件更加独立、易于理解和维护。特别是对于小型应用和较少共享状态的场景，使用 Hooks 会更轻便。
- **Pinia** 会让你将数据和处理逻辑集中到 store 中，可以帮助你避免在多个组件之间传递复杂的状态，提升维护性。对于中大型项目，Pinia 的结构化管理方式可以显著降低代码的复杂度，尤其是当你的应用需要管理多个全局状态时，Pinia 能够提供更加清晰的代码结构。

### sotreToRefs

![image-20250222153515839](./assets/image-20250222153515839.png)

​	当模板的读取数据的方式过于繁杂的时候，就像上面的h2标签一样，可以使用这个方法。

![image-20250222153619571](./assets/image-20250222153619571.png)

​	先把sum,school,address等数据从store中拿出来后，还不是响应式数据，因此，需要用toRefs去转化为双向绑定的响应式数据。	

**`toRefs`**

- **作用**：`toRefs` 将一个 `reactive` 对象中的**每个属性转换为响应式**的 `ref`。
- **适用场景**：当你需要将 `reactive` 对象的**属性单独暴露**为 `ref`，并且希望**这些属性仍然保持响应式时使用**。

### getters配置项

![image-20250222154248008](./assets/image-20250222154248008.png)

![image-20250222154301313](./assets/image-20250222154301313.png)

​	`getters` 是一个用于从 store 中获取状态的计算属性，它们类似于 Vuex 中的 `getters`。通过 `getters`，你可以在 **store 中计算并返回一些派生状态**，而这些状态的**值是基于 store 中已有的状态来计算**的。`getters` 使得你可以在**多个组件中共享计算结果**，而不需要重复计算。

![image-20250222154444019](./assets/image-20250222154444019.png)

一样可以取出来

### 订阅$subscribe 监视vuex里数据的变化

![image-20250222154733555](./assets/image-20250222154733555.png)

可以看到在数据变化后：

![image-20250222154749466](./assets/image-20250222154749466.png)

看得到事件(events)和关键的state。有些像**watch**

![image-20250222154920561](./assets/image-20250222154920561.png)

可以用这种方式保存在浏览器本地，可以实现刷新不丢失。

然后可以在sotre里读取本地存取的数值。

![image-20250222155048352](./assets/image-20250222155048352.png)

只是这个不一定取得出来，可能值为null 可以在前面加个断言

![image-20250222155226466](./assets/image-20250222155226466.png)

### store组合式

![image-20250222155523132](./assets/image-20250222155523132.png)

记得把函数和属性交出去。

​	如果你导出的是一个变量（常量、对象、数组等），你不需要使用 `return`，因为 `export` 只是让这个变量可用于其他模块。

​	如果你导出的是一个**函数**，那么是否需要 `return` 取决于函数的类型。

- **普通函数**：如果函数需要返回一个值，你应该使用 `return` 来返回数据。
- **没有返回值的函数**：如果函数只是执行某些操作（如修改全局状态或触发副作用），则不需要 `return`。

## 高级用法

### getters

**Getter 就是 Pinia 里的“计算属性）”**。它的作用是**基于 state 派生出新的数据**，并且**自动缓存**，只有依赖的 state 变化时才会重新计算。

state里面存储的是原始数据，但是展示的时候需要格式化，我们可以用getter中的方式进行格式化：


```ts
getters: {
  // 不用 getter：每次都要写
  // 组件里：`${person.age}岁` 到处重复

  // 用 getter：统一格式化
  formattedPeople: (state) => {
    return state.peopleList.map(p => ({
      ...p,
      displayName: `${p.name} (${p.age}岁)`,
      statusLabel: p.status === 'active' ? '✅ 活跃' : '❌ 非活跃',
    }))
  },
}
```

可以根据条件来动态展示数据的不同子集：

```ts
getters: {
  // 根据关键词搜索
  searchedPeople: (state) => {
    if (!state.searchKeyword) return state.peopleList
    return state.peopleList.filter(p =>
      p.name.includes(state.searchKeyword)
    )
  },

  // 排序
  sortedByAge: (state) => {
    return [...state.peopleList].sort((a, b) => a.age - b.age)
  },

  // 组合：过滤 + 排序
  filteredAndSorted: (state) => {
    let result = state.peopleList
    // 先过滤
    if (state.searchKeyword) {
      result = result.filter(p => p.name.includes(state.searchKeyword))
    }
    // 再排序
    return [...result].sort((a, b) => a.age - b.age)
  },
}
```

一些统计数据的场景可以在这里很方便的实现。

 带参数的 Getter（动态查询）

```typescript
getters: {
  // 返回函数，接收参数
  getPersonById: (state) => (id: number) => {
    return state.peopleList.find(p => p.id === id)
  },
  getPeopleByAgeRange: (state) => (min: number, max: number) => {
    return state.peopleList.filter(p => p.age >= min && p.age <= max)
  },
}
```

以及可以用于跨store调用的数据的场景，用这种方式可以继续压缩业务代码，防止在vue模板文件中减少代码量。


```ts
// store/people.ts
import { useAuthStore } from './auth'

getters: {
  // 访问其他 Store
  currentUserPeople: (state) => {
    const authStore = useAuthStore()
    return state.peopleList.filter(p => p.userId === authStore.userId)
  },
}
```

### pinia插件

Pinia 插件是一个普通函数，通过 `pinia.use()` 注册，可以为**核心的状态管理添加全局能力**，比如在 store 上挂载公共属性、封装可复用的逻辑、添加加密存储等，对大量 store 统一进行增强[](https://pinia.vuejs.org/core-concepts/plugins)[](https://cloud.tencent.com.cn/developer/article/2528008)

```ts
export function myPiniaPlugin(context) {
  context.pinia // 用 `createPinia()` 创建的 pinia。
  context.app // 用 `createApp()` 创建的当前应用(仅 Vue 3)。
  context.store // 该插件想扩展的 store
  context.options // 定义传给 `defineStore()` 的 store 的可选对象。
  // ...
}
```

比如说，我们可以使用 **Pinia 插件** 配合 **VueUse** 的 `useDebounceFn`，给任意 action 自动加上防抖，避免重复触发，这里我们需要使用ts的模块扩展语法来告诉ts这个模块/类型是存在的，即使我找不到它的定义”。

```ts
declare module 'pinia' {
  export interface DefineStoreOptionsBase<S, Store> {
    debounce?: Record<string, number>
  }
}
```

现在在 `defineStore` 中可以使用 `debounce` 选项，TypeScript 不会报错，并且有类型提示。

我们去识别store中是否有debounce然后去实现将普通方法通过vueuse的防抖注册为防抖办法，用于我们自己的store中方法：


```ts
/* eslint-disable @typescript-eslint/no-unused-vars */
// plugins/pinia-debounce.ts
import type { PiniaPluginContext, Pinia } from 'pinia'
import { useDebounceFn } from '@vueuse/core'
declare module 'pinia' {
  export interface DefineStoreOptionsBase<S, Store> {
    debounce?: Record<string, number>
  }
}
export function createDebouncePlugin() {
  return (context: PiniaPluginContext) => {
    const { options, store } = context
    if (!options.debounce) return
    const debouncedActions: Record<string, unknown> = {}
    Object.entries(options.debounce).forEach(([actionsName, delay]) => {
      const originalAction = store[actionsName]
      if (typeof originalAction !== 'function') return
      debouncedActions[actionsName] = useDebounceFn(
        originalAction.bind(store),
        delay
      )
    })
    return debouncedActions
  }
}
  
export default defineNuxtPlugin((nuxtApp) => {
  const pinia = nuxtApp.$pinia as Pinia
  pinia.use(createDebouncePlugin())
})
```


### 持久化

 Pinia 持久化，核心原理非常清晰：通过 `store.$subscribe` 监听状态变化并存入本地存储，再在 store 初始化时从存储中读取数据，通过 `store.$patch` 恢复状态[](https://blog.csdn.net/weixin_62936589/article/details/143031003)[](https://cloud.tencent.cn/developer/article/2567715)[](https://juejin.cn/post/7212435153573609529)。目前社区最主流、也最推荐的做法，就是使用 `pinia-plugin-persistedstate` 这个官方生态插件。

可以从指令安装：

```
pnpm add pinia-plugin-persistedstate
```

此插件与 `pinia>=2.0.0` 兼容， 请确保在继续之前 [已安装 Pinia](https://pinia.vuejs.org/getting-started.html) 。 `pinia-plugin-persistedstate` 具有许多功能，使 Pinia store 的持久化变得轻松且可配置：

- 一个类似于 [`vuex-persistedstate`](https://github.com/robinvdvleuten/vuex-persistedstate)的 API。
- Per-store 配置.
- 自定义存储和自定义数据序列化程序。
- Pre/post persistence/hydration hooks.
- 每个store有多个配置。

这个包可以导出一个模块，能够更好的和nuxt集成和开箱即用的ssr支持，

将插件添加到你的 pinia 实例中：

```ts
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)
```

在声明store时，请将新`persist`选项设置为 `true`。

```ts
import { defineStore } from 'pinia'

export const useStore = defineStore('main', {
  state: () => {
    return {
      someState: 'hello pinia',
    }
  },
  persist: true,
})
```

整个 store 现在将使用 [默认的持久性设置](https://prazdevs.github.io/pinia-plugin-persistedstate/guide/config.html)进行保存。这个插件提前将localStorage作为存储，store.$id作为默认存储键，使用JSON.stringfy作为序列化器和反序列化器，与destr进行通信。

与 Nuxt 模块相比，默认设置有一些不同，因为它能提供更适合 SSR 体验的配置。如需了解更多信息，请参考 Nuxt 的使用文档。你可以将一个对象传递给商店的 `persist` 属性，以配置持久化功能。

你可以通过使用自动导入的 `piniaPluginPersistedstate` 变量中提供的**存储选项**，来配置想要使用的**存储类型**。

比如使用cookies作为存储方式：

```ts
import { defineStore } from 'pinia'

export const useStore = defineStore('main', {
  state: () => {
    return {
      someState: 'hello pinia',
    }
  },
  persist: {
    storage: piniaPluginPersistedstate.cookies(),
  },
})
```
`persistedState.cookies` 方法接受一个对象参数，用于配置 cookies，支持以下选项（这些选项继承自 Nuxt 的 `useCookie` 方法）：

- [`domain`](https://nuxt.com/docs/api/composables/use-cookie#domain)
- [`encode`](https://nuxt.com/docs/api/composables/use-cookie#encode)/[`decode`](https://nuxt.com/docs/api/composables/use-cookie#decode)
- [`expires`](https://nuxt.com/docs/api/composables/use-cookie#maxage-expires)
- [`httpOnly`](https://nuxt.com/docs/api/composables/use-cookie#httponly)
- [`maxAge`](https://nuxt.com/docs/api/composables/use-cookie#maxage-expires)
- [`partitioned`](https://nuxt.com/docs/api/composables/use-cookie#partitioned)
- [`path`](https://nuxt.com/docs/api/composables/use-cookie#path)
- [`sameSite`](https://nuxt.com/docs/api/composables/use-cookie#samesite)
- [`secure`](https://nuxt.com/docs/api/composables/use-cookie#secure)

>[!warning] 在保存包含大量数据的存储时需要注意，因为 cookie 的大小限制为 4098 字节。有关 cookie 存储的更多信息，请参阅 MDN 文档。


 `localStorage`：


```ts
import { defineStore } from 'pinia'

export const useStore = defineStore('main', {
  state: () => {
    return {
      someState: 'hello pinia',
    }
  },
  persist: {
    storage: piniaPluginPersistedstate.localStorage(),
  },
})
```

 `sessionStorage`

```ts
import { defineStore } from 'pinia'

export const useStore = defineStore('main', {
  state: () => {
    return {
      someState: 'hello pinia',
    }
  },
  persist: {
    storage: piniaPluginPersistedstate.sessionStorage(),
  },
})
```

> `sessionStorage` is client side only.  
`sessionStorage`仅适用于客户端侧的数据存储。

这和localStorage都是浏览器提供的web storage api ，用于在客户端存储键值对数据。它们的**核心区别在于数据的“有效期”和“作用域”**。

localStorage是永久有效期，是否手动清除，不然就不会消失，同源（协议+域名+端口）所有标签页窗口共享，sessionStorage似乎临时的同源或者一个ifname内共享，在浏览器重启之后会丢失，可以用于表单临时数据、本次会话的状态和一次性验证码。


#### 存储选项

该模块接受在 `nuxt.config.ts` 中定义的某些选项，这些选项属于 `piniaPluginPersistedstate` 键下的子键：

- [`cookieOptions`](https://prazdevs.github.io/pinia-plugin-persistedstate/frameworks/nuxt.html#cookies) 
    `cookieOptions` （除了 `decode` 和 `encode` ，因为函数不支持这些编号）
- `debug`
- [`key`](https://prazdevs.github.io/pinia-plugin-persistedstate/frameworks/nuxt.html#global-key)
- `storage`

存储选项仅**接受预配置存储的字符串值**（ `'cookies'` 、 `'localStorage'` 、 `'sessionStorage'` ）。这是由于 Nuxt 传递模块选项到运行时的方式。

```ts
export default defineNuxtConfig({
  modules: [
    '@pinia/nuxt',
    'pinia-plugin-persistedstate/nuxt'
  ],
  piniaPluginPersistedstate: {
    storage: 'cookies',
    cookieOptions: {
      sameSite: 'lax',
    },
    debug: true,
  },
})
```

#### 全局秘钥

我们可以提供一个模板字符串，用于**标记全局使用的键前缀/后缀**。所提供的键必须包含令牌 `%id` ，该**令牌将被相应存储体的 ID 替代**。

```ts
export default defineNuxtConfig({
  modules: [
    '@pinia/nuxt',
    'pinia-plugin-persistedstate/nuxt'
  ],
  piniaPluginPersistedstate: {
    key: 'prefix_%id_postfix',
  },
})
```

任何以 `my-store` 作为持久化键的store（无论是用户提供的还是从store ID 推断出来的），都会被保存在 `prefix_my-store_postfix` 这个键下。在nuxt的集成中，global 是一个**全局的前缀/命名空间**，用来**区分不同应用或环境的存储数据**。

为所有持久化的 Store 状态**添加一个全局前缀**，避免**多个应用**共享同一个 `localStorage` 时发生**键名冲突**。Global Key 只在客户端存储生效，服务端不影响

example:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@pinia/nuxt',
    'pinia-plugin-persistedstate/nuxt'
  ],
  piniaPersistedstate: {
    key: 'my-app-v1',        // ✅ Global Key
    storage: 'localStorage'  // 默认
  }
})


// 没有 global key
localStorage.getItem('userStore')   // { name: 'Alice' }
// 有 global key
localStorage.getItem('my-app-v1:userStore')   // { name: 'Alice' }
```


虽然在中间件内部访问store是可行的，但从 `cookies` 持久化/存储数据时可能会出现问题，并导致 `[nuxt] A composable that requires access to the Nuxt instance was called outside of a plugin, Nuxt hook, Nuxt middleware, or Vue setup function.` 错误。这是因为 Pinia 存储在 **Nuxt 实例之前就已经被实例化**，而 `pinia-plugin-persistedstate` 则需要 Nuxt 实例才能访问 `useCookie` 。

为了克服这个限制，你可以采取以下方法：

在 Nuxt 实例可用之后，务必访问该商店。需要注意的是，要考虑到中间件的使用问题。

在这种特定情况下，可以直接使用 `useCookie` 可组合对象，而不是使用存储系统。

### 更多高级用法

#### 每个store配置多个数据持久化

在某些特定情况下，您可能需要将来自单个存储的数据持久化到不同的存储。 ` `persist` ` 选项也接受一个配置数组。

```ts
import { defineStore } from 'pinia'

defineStore('store', {
  state: () => ({
    toLocal: '',
    toSession: '',
    toNowhere: '',
  }),
  persist: [
    {
      pick: ['toLocal'],
      storage: localStorage,
    },
    {
      pick: ['toSession'],
      storage: sessionStorage,
    },
  ],
})
```

在这个示例中， `toLocal` 的值会被保留在 `localStorage` 中，而 `toSession` 的值则会被保留在 `sessionStorage` 中。 `toNowhere` 的值则不会被保留。

>[!warning] 在未指定 `paths` 选项，或者两个持久化配置中目标路径相同的情况下，请务必小心操作。这可能导致数据不一致的问题。在重新 hydration 过程中，持久化操作将按照声明顺序进行执行。

#### 水合

如果你需要手动触发从存储中读取数据的过程，每个存储接口现在都提供了一个相关的方法。默认情况下，调用此方法还会触发 `beforeHydrate` 和 `afterHydrate` 这两个钩子函数。如果你不想触发这些钩子，可以指定不调用这些方法。


```ts
import { defineStore } from 'pinia'

const useStore = defineStore('store', {
  state: () => ({
    someData: 'Hello Pinia'
  })
})
```

比如说这个store， 你可以调用$hydrate

```ts
const store = useStore()
store.$hydrate({ runHooks: false })
```

这将从存储中获取数据，并将其应用到当前状态。在上述示例中，钩子不会被触发如果需要手动触发持久化到存储，现在每个存储都暴露了一个。

如果需要手动触发持久化到存储，现在每个存储都暴露了一个 ` `$persist` ` 方法。

```ts
import { defineStore } from 'pinia'

const useStore = defineStore('store', {
  state: () => ({
    someData: 'Hello Pinia'
  })
})
```

你可以调用$persist

```ts
const store = useStore()
store.$persist()
```

`pinia-plugin-persistedstate` 对 SSR（服务端渲染）的支持，核心机制是“客户端优先”，即插件会避免在服务端执行存储读写，防止因访问 `localStorage` 导致应用报错，同时确保客户端激活（Hydration）时数据一致。

