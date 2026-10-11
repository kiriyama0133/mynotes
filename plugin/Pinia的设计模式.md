---
date: 2026-10-11
tags:
  - plugin
  - pinia
---
pinia的核心原理是使用vue的核心响应式系统包装一个状态对象，用闭包管理store实例，这个和composable的思想是一致的，用插件机制扩展能力

## 单例模式的实现

Pinia是通过单例模式实现，无论在应用的哪个组件调用useStore，拿到的都是同一个store实例，状态共享、响应式同步，它的单例实现比较巧妙，核心是用全局变量缓存已经创建的store实例，而不是靠javascript的类单例或者模块单例。

Pinia会在内部维护一个全局的Map，比如`pinia._s`用来缓存所有已经创建了的store，当调用useStore的时候，会通过id去`pinia._s`里查找，如果有就返回，没有就创建，这样无论调用多少次都用于返回一个实例。

下面是一个简化后的逻辑：

```ts
// Pinia 内部简化逻辑
function createPinia() {
  const pinia = {
    _s: new Map(),  // 全局缓存：store id → store 实例

    install(app) {
      // 把 pinia 挂到 app 上，供全局访问
      app.config.globalProperties.$pinia = pinia
    },
  }
  return pinia
}

function useStore(id) {
  const pinia = getCurrentPinia()  // 获取当前应用的 pinia 实例

  // 1. 先查缓存
  if (pinia._s.has(id)) {
    return pinia._s.get(id)
  }

  // 2. 没有就创建
  const store = createStore(id)
  pinia._s.set(id, store)  // 存入缓存
  return store
}
```

每个store定义的时候都有一个唯一的id，比如defineStore('counter', ...)里面的 coutner，这个Id就是缓存的key，同id的store永远只有一个实例，缓存挂在pinia实例上，而不是模块级的变量。这样就算一个页面里有多个Vue应用，每个应用也可以有独立的Pinia实例，互相不干扰。

## ssr支持

pinia是支持ssr的，服务端渲染的时候，每个请求都会创建独立的pinia实例，避免不同请求之间的状态串扰。什么叫做pinia每个请求都会创建独立的pinia实例，ssr下，你什么时候又支持这个代码会在服务端执行？

因为ssr下，同一个vue应用的代码会在两个地方执行，首次请求是服务端，执行vue组件代码生成html字符串，首次响应之后客户端接管服务器生成的html，继续交互，后续导航也是在客户端执行。

我们知道，在一个vue应用，我们会

我们在vue组件的setup位置里执行的代码比如下面：

```vue
<script setup>
import { useCounterStore } from '@/stores/counter'

const counter = useCounterStore()  // ← 这行在服务端会执行

// 服务端会执行：获取数据
await counter.fetchData()          // ← 这行在服务端会执行

// 客户端才会执行：操作 DOM
onMounted(() => {
  console.log('只在客户端执行')    // ← 服务端不执行
})
</script>

<template>
  <div>{{ counter.count }}</div>   <!-- 服务端会渲染成 HTML -->
</template>
```

里面的useCounterStore在服务端，可能就表现为：

```js
app.get('*', async (req, res) => {
  const app = createSSRApp(App)       // 1. 创建 Vue 应用
  const pinia = createPinia()          // 2. 每个请求新建 Pinia 实例
  app.use(pinia)                       // 3. 把 Pinia 挂到应用上

  const html = await renderToString(app)  // 4. 执行组件 setup()，这里会调用 useCounterStore()
  res.send(html)
})
```

setup里的pinia的代码可能就会在服务端里创建这个实例，就会执行setup，创建和填充store，然后收集了所有的store状态之后，把状态序列化后注入html，返回给浏览器。

如果我们手写server.js，那么注入的是自己控制，如果在nuxt中的话，我们不需要手动序列化和注入，内部会有nitro和payload机制自动完成，具体是nuxt会在服务端执行页面组件之后，pinia的store状态会被nuxt的payload系统收集，我们只需要在nuxt.config.ts里面配置pinia模块就行。注入的位置可能是_prelaod.json或者内联__NUXT__脚本。

Nuxt 把状态合并进了它自己的 payload 系统，不会单独暴露一个 `__PINIA_STATE__`。你可能会在 HTML 里看到类似：

```
<script type="application/json" id="__NUXT_DATA__">[...]</script>
```

## pinia的持久化

Pinia 官方没有内置持久化，但社区有成熟的插件 **`pinia-plugin-persistedstate`**，它把 store 状态自动同步到 `localStorage` 或 `sessionStorage`。


```ts
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)
app.use(pinia)

// store里启用
export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0, name: 'Alice' }),
  persist: true,  // 整个 store 都持久化
})

// 精细控制
export const useUserStore = defineStore('user', {
  state: () => ({
    token: '',
    profile: {},
    tempData: {},
  }),
  persist: {
    key: 'my-user',              // 存储的 key，默认是 store id
    storage: sessionStorage,     // 存储方式，默认 localStorage
    pick: ['token', 'profile'],  // 只持久化这两个字段
    // 或者用 omit 排除
    // omit: ['tempData'],
    serializer: {                // 自定义序列化
      serialize: JSON.stringify,
      deserialize: JSON.parse,
    },
  },
})
```

