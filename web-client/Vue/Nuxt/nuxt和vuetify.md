**Nuxt 项目**和传统的**约定优于配置**的理念，大大简化了项目结构

### 🚀 **Nuxt 项目的入口文件差异：**

在 Vue 3 项目中，我们习惯有一个明确的 `main.ts`（`main.js`），通常这样写：

**Vue 3 项目 `main.ts`：**

```ts
import { createApp } from 'vue';
import App from './App.vue';
import router from './router';
import store from './store';

const app = createApp(App);
app.use(router);
app.use(store);
app.mount('#app');

```

**而在 Nuxt 项目** 中，没有`main.ts`，因为 Nuxt 已经帮你处理了大部分

- **应用创建**：N
- **路由初始化**：基于 `pages/` 目录生成路由。
- **插件注册**：`plugins/` 目录来扩展功能。
- **状态管理**：集成 Pinia（Nuxt 3）
- **中间件、布局、SEO 等功能**，全部由 Nuxt 自动处理



### 🌟 ***Nuxt 的等效入口机制**

虽然没有 `main.ts`，但你可以通过以下

1. **app.vue**
   这个是 Nuxt 应用的根组件，相当于 Vue 项目的 `App.vue`。你可以在这里进行全局布局、页面切换动画等逻辑。

2. **plugins 目录**
   如果你想在项目启动

   **示例：`plugins/my-plugin.ts`**

   ```ts
   export default defineNuxtPlugin((nuxtApp) => {
     console.log('Nuxt app is ready!');
   });
   ```

   Nuxt 会自动在启动时加载 `plugins` 目录

3. **nuxt.config.ts**
   类似`vue.config.js`，这里是 Nuxt 的全局配置文件。你

   ```ts
   // nuxt.config.ts
   export default defineNuxtConfig({
     modules: ['@pinia/nuxt'],
     plugins: ['~/plugins/my-plugin.ts'],
   });
   ```

4. **中间件和生命周期钩子**
   你还可以在页面级别使用 `onMounted` 等 Vue 生命周期钩子，或者 Nuxt 的专有钩子如 `onNuxtReady` 来控制项目的初始化逻辑。

在 **Nuxt 3** 项目中集成 **Vuetify** 非常直接，我们一步步来搞定！💪

### 🚀 **Nuxt 3 安装 Vuetify**

1️⃣ **安装 Vuetify 和相关依赖：**

```
bash复制编辑# 安装 Vuetify 及其 peer 依赖
npm install vuetify @mdi/font sass
```

2️⃣ **配置 Vuetify 插件：**

在 `plugins/` 目录下新建一个插件文件：

**`plugins/vuetify.ts`**

```ts
// plugins/vuetify.ts
import { createVuetify } from 'vuetify';
import * as components from 'vuetify/components';
import * as directives from 'vuetify/directives';
import 'vuetify/styles';

export default defineNuxtPlugin((nuxtApp) => {
  const vuetify = createVuetify({
    components,
    directives,
  });

  nuxtApp.vueApp.use(vuetify);
});
```

3️⃣ **注册插件：**

打开 `nuxt.config.ts` 并添加：

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  css: ['vuetify/styles'], // 引入全局样式
  build: {
    transpile: ['vuetify'], // 确保编译 Vuetify
  },
  plugins: ['~/plugins/vuetify.ts'], // 注册插件
});
```

4️⃣ **使用组件：**

现在你就可以在组件中直接使用 Vuetify 的组件啦！🌟

**示例：`pages/index.vue`**

```vue
<template>
  <v-container>
    <v-row>
      <v-col>
        <v-btn color="primary">Hello Vuetify</v-btn>
      </v-col>
    </v-row>
  </v-container>
</template>
```

5️⃣ **可选：定制主题**

如果你想定制主题，比如改颜色或者其他设置，可以在创建 Vuetify 实例时添加配置：

**`plugins/vuetify.ts`**

```ts
import { createVuetify } from 'vuetify';
import * as components from 'vuetify/components';
import * as directives from 'vuetify/directives';
import 'vuetify/styles';

const customTheme = {
  dark: false,
  colors: {
    primary: '#4caf50',
    secondary: '#ff9800',
    accent: '#9c27b0',
  },
};

export default defineNuxtPlugin((nuxtApp) => {
  const vuetify = createVuetify({
    components,
    directives,
    theme: {
      defaultTheme: 'customTheme',
      themes: {
        customTheme,
      },
    },
  });

  nuxtApp.vueApp.use(vuetify);
});
```

### 🧑‍💻 **`nuxt.config.ts` 配置项总结**

| 配置项       | 描述                                                      |
| ------------ | --------------------------------------------------------- |
| `app`        | 配置应用的根级设置，如 `<head>` 信息等。                  |
| `css`        | 用于引入全局 CSS 和样式文件。                             |
| `plugins`    | 用于注册插件，可以在应用中全局使用的库或工具。            |
| `modules`    | 配置 Nuxt 模块，扩展应用功能。                            |
| `router`     | 配置路由相关选项，如自定义路由路径。                      |
| `build`      | 配置构建相关选项，如 Webpack、Babel 配置等。              |
| `env`        | 配置环境变量，用于不同环境下的变量设置。                  |
| `head`       | 配置 `<head>` 标签，主要用于标题、meta 标签、favicon 等。 |
| `middleware` | 配置路由中间件，用于执行路由切换前后的逻辑。              |
| `i18n`       | 配置多语言支持。                                          |

### 🧑‍💻 **`nuxt.config.ts` 配置项总结**

| 配置项       | 描述                                                      |
| ------------ | --------------------------------------------------------- |
| `app`        | 配置应用的根级设置，如 `<head>` 信息等。                  |
| `css`        | 用于引入全局 CSS 和样式文件。                             |
| `plugins`    | 用于注册插件，可以在应用中全局使用的库或工具。            |
| `modules`    | 配置 Nuxt 模块，扩展应用功能。                            |
| `router`     | 配置路由相关选项，如自定义路由路径。                      |
| `build`      | 配置构建相关选项，如 Webpack、Babel 配置等。              |
| `env`        | 配置环境变量，用于不同环境下的变量设置。                  |
| `head`       | 配置 `<head>` 标签，主要用于标题、meta 标签、favicon 等。 |
| `middleware` | 配置路由中间件，用于执行路由切换前后的逻辑。              |
| `i18n`       | 配置多语言支持。                                          |

`nuxt.config.ts` 是 **Nuxt.js** 应用的核心配置文件，**控制了应用的整体行为和功能。**

​	它提供了丰富的配置选项，允许你调整页面头部、引入插件、配置路由、构建选项、状态管理、环境变量等。

​	通过 `nuxt.config.ts`，你可以根据项目需求**自定义应用的各个方面**

runtimeConfig里面可以写运行时的一些全局变量。

```ts
// https://nuxt.com/docs/api/configuration/nuxt-config
export default defineNuxtConfig({
  compatibilityDate: '2024-11-01',
  devtools: { enabled: true },
  runtimeConfig:{
    count:1,
    public:{baseURL:'localhsot:8080'} //public表示这个配置会暴露给客户端，可以在前端中访问
  }
})

```

组件中写：

```vue
<template>
  <div>

  </div>
</template>
<script setup name="app">
const config = useRuntimeConfig()
console.log("1"+config)
console.log("2"+config.count) //count不是public的特性，因此浏览器里控制台得到的是
//undefined,意思是count只能在服务端期间获取到
//而public的既可以在服务端也可以在客户端
console.log("3"+config.public.baseURL)

</script>

```

![image-20250223054448331](image-20250223054448331.png)

## SSR

​	在控制台里可以看得到数据是由客户端还是由服务端渲染出来的，具体就看它有没有ssr这个前缀

​	nuxt里有一个nitro的引擎，当我们通过链接发起请求，那么nitro就回去渲染页面，并返回给页面。

​	客户端渲染则是浏览器的引擎负责渲染。当服务器内部遇到渲染错误的时候，会爆500这个错误（比如alert的渲染）。

## 静态资源

按照一般的约定，我们要把静态资源放在文件夹/assets里

我们可以在assets里添加新的文件夹比如css，之后在里面添加样式，如base.scss

![image-20250223054801893](image-20250223054801893.png)

​	使用scss来定义样式，我们需要使用sass这个工具， 使用pnpm i -D sass来添加到工作环境。

​	在 `pnpm` 中，`-D` 参数是 `--save-dev` 的简写，表示将依赖项安装为 **开发依赖**。开发依赖是指**只在开发过程中需要的依赖**，而**不是在生产环境中运行时需要的依赖。**

然后回到nuxt.config.ts中去引用样式。

```ts
// https://nuxt.com/docs/api/configuration/nuxt-config
export default defineNuxtConfig({
  compatibilityDate: '2024-11-01',
  devtools: { enabled: true },
  css:['@/assets/css/base.scss'],
  runtimeConfig:{
    count:1,
    public:{baseURL:'localhsot:8080'}

  }
})

```

base.scss可以写入用选择器自定义控件的样式。比如下面这个。

h1{

  color: red;

}

![image-20250223055317380](image-20250223055317380.png)

我们一般会在预处理器里定义一个变量 比如：

```scss
$mycolor: green;
```

如果要使用这种变量，就需要回到nuxt.config.ts去配置vite的配置项：

```ts
// https://nuxt.com/docs/api/configuration/nuxt-config
export default defineNuxtConfig({
  compatibilityDate: '2024-11-01',
  devtools: { enabled: true },
  vite:{
    css:{
      preprocessorOptions:{
        scss:{
          additionalData:`@use "~/assets/css/base.scss" as *;`
            //在 SCSS 中，@use 语句用于引入另一个 SCSS 文件，并通过模块化的方式将其中的内容作为一个命名空间导入。as * 的作用是将被引入文件中的所有内容（如变量、混合器、函数等）直接注入到当前作用域，而不是通过命名空间来访问它们。
        }
      }
    }
  },
  runtimeConfig:{
    count:1,
    public:{baseURL:'localhsot:8080'}

  }
})

```

​	注意原先的css没了，原先的是直接引用，会和下面这个冲突，因此建议直接用刚才说的这个方式。

最后在vue里可以灵活的使用$去引用预处理的变量。

```vue
<template>
  <div>
  <h1>这里是样式测试</h1>
  <p>11111</p>
  </div>
</template>
<script setup name="app">
const config = useRuntimeConfig()
console.log("1"+config)
console.log("2"+config.count) //count不是public的特性，因此浏览器里控制台得到的是
//undefined
console.log("3"+config.public.baseURL)

</script>

<style scoped lang="scss">
 p{
  color: $mycolor;
 }
</style>

```

这个引用体现在<style>标签里。

## 使用elementUI

使用方式依然是在nuxt.config.ts里配置相关的模板。

​	当引入了moudles之后就会自动配置，**之后我们需要去配置全局样式**，如果在引入moudles之后依然没有什么作用。

```vue
<template>
  <div>
  <h1>这里是样式测试</h1>
  <p>11111</p>
  <el-button type="primary">1111</el-button>
  </div>
</template>
```

然后使用标签引入就可以直接使用了。

## useRuntimeConfig

 **`useRuntimeConfig` 的工作原理：**

​	`useRuntimeConfig` 允许你访问在 `nuxt.config.ts` 或 `nuxt.config.js` 中定义的 `runtimeConfig` 对象。

### **如何使用 `useRuntimeConfig`：**

在组件、页面或插件中，可以通过 `useRuntimeConfig` 来访问这些配置。

#### 在组件中使用：

```vue
<script setup>
const config = useRuntimeConfig()

// 访问公共配置
console.log(config.public.baseURL) // 输出: http://localhost:8080

// 访问私有配置，只能在服务端访问
if (process.server) {   //这也是判断是否在服务端
    
  console.log(config.private.apiKey) // 输出: your-private-api-key
}

// 访问共享的 count 配置
console.log(config.count) // 输出: 1
</script>
```

​	判断当前是否在客户端，你可以使用 `process.client`，它是 Nuxt 3 内置的全局变量。与 `process.server` 类似，`process.client` 可以帮助你区分代码是否在 **客户端** 环境中执行。

## 路由

​	和vue不一样，nuxt已经对路由vue.router进行了封装，可以直接使用和完成导航。

```bash
 WARN  [nuxt] Your project has pages but the <NuxtPage /> component has not been used. You might be using the <RouterView /> component instead, which will not work correctly in Nuxt. You can set pages: false in nuxt.config if you do not wish to use the Nuxt vue-router integration.
```

注意一下如果在写完了pages后直接进行跳转页面就会有这个报错。

​	这个警告意味着你正在使用 Nuxt 项目时，系统发现你的**项目中定义了页面**（`pages` 目录下有 Vue 文件），但是你没有在模板中使用 `<NuxtPage />` 组件，而是使用了 `<RouterView />` 组件。Nuxt 依赖于 `<NuxtPage />` 来正确渲染路由页面，而 `<RouterView />` 是 Vue Router 的组件，并不能与 Nuxt 的自动化路由系统兼容。

​	也就是没有写路由的入口，如果在/pages里 创建一个index.vue的组件，那么它就会称为 / 的入口 ，当你进入localhsot:3000的时候，就会显示这个页面。

查看这个图片

![image-20250223181646375](image-20250223181646375.png)

这里的router-view就是路由的入口，当路由发生变化的时候，这个地方就会变化

只是我们一般用的是NuxtPage标签。它可以支持命名路由和可选路由。

### 导航守卫和中间件

中间件必须定义为middleware的文件夹里

假设在文件夹里定义一个文件叫做test.js

```js
export default defineNuxtRouteMiddleware((to, from) => {
    console.log(to.path, from)
    console.log("拦截到了，上面是输出")
})
```

### 输出示例：

假设用户从 `/home` 页面跳转到 `/profile` 页面，则控制台的输出会是：

```bash
Target path: /profile
Current path: /home
```

### 使用场景：

- **调试路由跳转**：你可以用它来调试路由跳转，查看哪些路径在跳转过程中被访问。
- **路由跳转条件**：你也可以根据 `to` 和 `from` 判断是否允许某次跳转。例如，如果用户未登录且试图访问某些页面，可以取消跳转。

定义好test.js后在页面里引用

```vue
<template>
    <div>
        <h2>首页</h2>
    </div>
</template>

<script setup name="index" lang="ts">
definePageMeta({
    middleware:["test"]
})
</script>
```

然后试着去请求页面：

![image-20250223185626938](image-20250223185626938.png)

在 Nuxt 项目中，页面加载时会经历一系列的生命周期和事件，理解这些事件的顺序对于掌握 Nuxt 的内部工作原理非常重要。以下是从请求到页面渲染过程中，前后发生的一些关键步骤。

### 全局中间件的定义:

结尾以global.js为结尾定义。

![image-20250223191415333](image-20250223191415333.png)

这种写法就不需要向上面那样在vue组件中引入了。

### 导航守卫

​	导航守卫就是靠全局中间件去定义的。一些页面可能需要携带一定的token去进入，否则就是去重定向。

​	那么token的存储是需要依靠localStore去实现的，里面还有很多的细节，千万要小心服务端侧的代码执行和客户端侧的代码执行，比如ElMessage.error()等方法无法在服务端段执行，而在test.global.js这种全局中间件都是默认是服务端侧的代码执行。

![image-20250223193805377](image-20250223193805377.png)

​	这就很麻烦。因为ElMessage.error()是一个客户端组件，无法在服务端运行，这时候可以使用query去传递参数。

![image-20250223200224833](image-20250223200224833.png)

携带参数后就去返回的页面去获取参数

我们直接在跳转到的页面这样引入，顺便**判断在浏览器段执行的代码**。

```vue
<template>
    <div>
        <h2>首页</h2>
    </div>
</template>

<script setup name="index" lang="ts">
const route = useRoute()
if (import.meta.client) { // 只在客户端执行
    const { code, message } = route.query
    if (code && message) {
        ElMessage.error(`${code} ${message}`)
    }
}
definePageMeta({
    // middleware:["test"]
})
</script>
```

另一种是使用生命周期的钩子比如 onmounted这个HooK,`onMounted` 只会在**组件的 DOM 元素被挂载到页面上后执行**，而挂载是在客户端进行的，所以它也适合用于判断是否在客户端。

![image-20250223200633096](image-20250223200633096.png)

​	试问为什么要加上if(route.query.code){}。因为我们正常访问login页面的时候这段代码也会执行，用的是onMounted去判断，因此要判断有无code,也就是去前往list网页的时候通过query携带的参数code，若没有，则正常进入就是了。

`localStorage` 和 **cookie** 都是浏览器提供的客户端存储机制，但它们有一些不同之处：

### 1. **存储方式**：

- **`localStorage`**：用于存储数据的**简单键值对数据。数据存储在浏览器的本地存储中，不会随着页面刷新或浏览器关闭而清除**，除非手动清除。
- **Cookies**：用于存储小量数据，可以在请求头中自动随每次 HTTP 请求发送到服务器，**适用于身份验证、会话管理等**。

### 2. **存储大小**：

- **`localStorage`**：每个域名下最多可以存储约 **5MB** 的数据（不同浏览器可能略有不同）。
- **Cookies**：每个 cookie 的大小限制通常为 **4KB**，而且通常最多允许设置 **20-50** 个 cookie。

### 3. **持久性**：

- **`localStorage`**：数据不会过期，除非显式删除（例如通过 JavaScript 清除）。
- **Cookies**：数据有有效期设置，可以是会话级别的（即浏览器关闭时删除）或者长期有效（通过设置 `expires` 或 `max-age`）。

### 4. **发送机制**：

- **`localStorage`**：数据存储在浏览器中，不会自动发送给服务器。它只适用于客户端 JavaScript 获取和存储数据。
- **Cookies**：会随着每次 HTTP 请求自动发送到服务器，适合用于保存身份认证信息等。

### 5. **跨域访问**：

- **`localStorage`**：只能在**相同的源（协议、域名、端口）下访问**。
- **Cookies**：同样**有相同的域名访问限制**，但是可以通过设置 `Domain` 属性来共享跨子域名的 Cookie。

### 总结：

- **`localStorage`** 和 **cookie** 并不是同一个东西。`localStorage` 是一个用于在浏览器端存储数据的 API，而 **cookie** 主要用于在浏览器与服务器之间传递信息。
- 它们有不同的用途和行为，`localStorage` 更适合用于存储大量数据，**cookie** 则适合用于在客户端与服务器之间交换少量信息（如会话标识）。

### 1. **路由请求和中间件触发的顺序**

​	当用户访问某个页面时，浏览器向 Nuxt 应用发出路由请求，Nuxt 会依次执行以下步骤：

#### 1.1 **请求开始：**

- **浏览器向服务器发出请求**，例如用户访问 `/profile` 页面，浏览器会发出请求到服务器。

#### 1.2 **Nuxt 中间件执行：**

- **全局中间件** 和 **页面中间件** 会**首先执行**（也就是上面的middleware会开始执行）。这些**中间件用于处理一些页面加载前的逻辑，例如权限检查、数据获取、重定向**等。

  - **全局中间件**：在 `nuxt.config.ts` 中配置的全局中间件会首先被调用。它们**对所有路由生效**。
  - **页面级中间件**：你可以在每个页面的 `<script setup>` 中通过 `definePageMeta` 来配置中间件，它们在对应页面加载之前被调用。

  **例如**：

  ```js
  // middleware/auth.js
  export default defineNuxtRouteMiddleware((to, from) => {
      console.log('Before page load', to.path)
      // 权限检查
      if (!isAuthenticated) {
          return navigateTo('/login') // 如果用户未认证，跳转到登录页面
      }
  })
  ```

  这些中间件会执行如下操作：

  - **验证用户身份**：如果是**需要登录才能访问的页面**，检查用户是否已认证。
  - **数据加载**：有些中间件也会**负责在页面加载前异步请求数据**。
  - **重定向**：如果中间件决定**用户无法访问某个页面**，可以通过 `navigateTo('/login')` 实现页面重定向。

#### 1.3 **服务器端渲染（SSR）或客户端渲染（CSR）**

- **服务器端渲染**：在 SSR 模式下，Nuxt 会在服务器上渲染页面内容，生成 HTML 并发送给浏览器。`useFetch`、`useAsyncData` 等函数通常在服务器端运行，用于数据加载。
- **客户端渲染**：在 CSR 模式下，Nuxt 会在浏览器中进行渲染，动态生成页面内容。通过路由跳转，页面数据会在客户端获取。

#### 1.4 **App.vue 的加载和渲染：**

- 在页面中间件处理完后，Nuxt 会加载 `App.vue` 文件。`App.vue` 是 Nuxt 的根组件，所有页面内容都会渲染在这个组件内。

  **`App.vue` 大致结构**：

  ```
  vue复制编辑<template>
    <div>
      <NuxtLayout />
      <NuxtPage /> <!-- 页面内容 -->
    </div>
  </template>
  
  <script setup>
  // 可在此执行一些初始化逻辑
  </script>
  ```

- **`NuxtLayout`**：这个组件负责渲染布局，布局通常包含页面的头部、侧边栏、底部等固定部分。

- **`NuxtPage`**：这个组件负责渲染具体的页面内容，它会根据当前路由加载对应的页面组件。

### 2. **客户端加载的详细过程：**

在客户端渲染时，页面加载过程会略有不同。

#### 2.1 **页面路由跳转：**

当你在浏览器中点击链接，触发路由跳转时，Nuxt 会执行如下操作：

1. **执行中间件**：
   - 如果页面有配置中间件（全局或局部），中间件会在路由跳转前执行。
   - 页面级中间件会在特定页面渲染之前执行。
2. **动态加载页面组件**：
   - Nuxt 会根据路由配置，异步加载相应的页面组件，通常是通过 `pages` 目录下的 `.vue` 文件进行自动路由映射。
   - 页面组件会传递给 `NuxtPage` 来渲染。
3. **数据获取**：
   - 在页面加载前，Nuxt 会触发 `useFetch` 或 `useAsyncData` 来获取数据，确保页面渲染时能够展示最新的数据。
   - 这些数据会在 `beforeMount` 或 `mounted` 生命周期钩子之前加载完成。
4. **渲染页面**：
   - 页面内容渲染到 `NuxtPage` 组件内，最终显示在浏览器中。

### 3. **总结：页面加载生命周期**

1. **浏览器发起请求**，访问特定页面。
2. **执行全局中间件**，检查访问权限、数据加载等。
3. **执行页面级中间件**，特定页面的逻辑（如权限验证）。
4. **服务器端渲染（SSR）** 或 **客户端渲染（CSR）**：根据渲染模式决定如何加载页面内容。
5. **App.vue 加载**：Nuxt 渲染应用的根组件，展示页面布局和内容。
6. **数据加载**：通过 `useFetch` 或 `useAsyncData` 获取异步数据。
7. **渲染页面组件**，并在浏览器中显示。

### 4. **App.vue 和 中间件的关系**

- **App.vue** 是应用的根组件，负责渲染全局布局和页面内容。它的作用是作为一个容器，里面包含了 `NuxtLayout` 和 `NuxtPage`，这些是用来呈现路由视图的。
- **中间件** 则是在页面渲染前或者路由跳转时执行的逻辑，通常用来处理如权限验证、重定向、异步数据获取等。

比如这里，可以利用return navigateTo去进行重定向：

```js
export default defineNuxtRouteMiddleware((to, from) => {
    console.log(to.path, from)
    console.log("拦截到了，上面是输出")
    return navigateTo("/list")
    //重定向到list
})
```



### 5. **运行顺序（客户端的详细流程）：**

1. 浏览器发起路由请求，Nuxt 开始处理。
2. 执行**全局中间件**（`middleware/` 下的配置）。
3. 执行**页面级中间件**（在页面组件内定义的）。
4. 如果是客户端渲染，执行客户端的**数据加载**（`useFetch`、`useAsyncData`等）。
5. 渲染页面的**组件**，更新页面显示。
6. 页面渲染完成后，浏览器显示页面，用户看到最终内容。

这种结构使得 Nuxt 在处理路由跳转和页面渲染时，能够有效地管理数据和用户权限等，确保页面加载的流畅性与可控性。

## SEO优化



**SEO优化**（Search Engine Optimization，搜索引擎优化）是指通过一系列的技术手段、内容策略以及用户体验的改进，提高网站在搜索引擎（如Google、Bing等）中的排名，从而增加网站的自然流量。SEO的目标是让你的网站更容易被搜索引擎抓取、理解和排名，从而让更多的潜在用户能够通过搜索引擎找到你的网站。

### SEO优化的主要方面

1. **站内优化（On-Page SEO）**：
   - **关键词研究**：根据用户搜索习惯选择合适的关键词，确保网站内容包含这些关键词。
   - **标题标签和Meta描述**：为每个页面设置优化的标题标签（<title>）和Meta描述（meta description），使它们既有吸引力又包含目标关键词。
   - **URL优化**：URL结构简洁、包含关键词且易于理解。
   - **内容优化**：确保页面内容有价值、原创、富有信息性，并且合理地使用关键词。
   - **图片优化**：图片需要有描述性的文件名和Alt属性，且优化图片大小以提高加载速度。
   - **内部链接**：通过适当的内部链接结构，帮助搜索引擎更好地抓取和理解网站内容。
2. **站外优化（Off-Page SEO）**：
   - **外部链接（Backlinks）**：通过获得其他高质量网站的链接来增加网站的权威性和可信度。外部链接的质量和数量对排名影响较大。
   - **社交媒体信号**：社交媒体的分享和互动可以间接影响搜索排名。
   - **品牌建设**：提高品牌在网络上的知名度和信誉，增强网站的影响力。
3. **技术优化（Technical SEO）**：
   - **网站速度**：确保网站加载速度快，避免长时间等待，这对用户体验和搜索引擎排名都非常重要。
   - **移动优化**：随着移动设备的普及，搜索引擎更倾向于优先考虑对移动设备友好的网站。
   - **网站结构和导航**：清晰的网站结构和导航，确保搜索引擎能够高效地抓取网站内容。
   - **XML站点地图**：创建并提交站点地图，帮助搜索引擎快速发现并抓取新内容。
   - **Robots.txt文件**：通过设置robots.txt文件，控制哪些页面可以被搜索引擎抓取，哪些页面应当被忽略。
4. **内容优化**：
   - **更新内容**：保持网站内容的新鲜度，定期更新，以符合用户和搜索引擎对新信息的需求。
   - **长尾关键词**：除了热门关键词，还可以针对较长的、细分的搜索词进行优化。这些关键词的竞争通常较小，但也能带来流量。

### SEO优化的重要性

- **提高可见性**：良好的SEO优化能**帮助你的网站在搜索引擎结果页面（SERP）中排名靠前，从而吸引更多的用户访问**。
- **增加流量**：通过搜索引擎获得自然流量，而不需要依赖广告或其他营销手段。
- **增强用户体验**：SEO优化**不仅能提升网站排名，还能改善用户体验，提升网站的加载速度、易用性和内容质量**。
- **提升转化率**：排名靠前的网站**更可能吸引到高质量的访客，进而提高转化率（如购买、注册等）**。

## composables文件夹

在 Nuxt 3 中，`composables` 文件夹是用来存放 **Composition API** 相关代码的地方，主要用于存放可复用的功能逻辑，通常以 **函数** 的形式进行定义。这样的设计使得你可以在多个组件中重用代码，从而提高代码的可维护性和可复用性。

### 1. **什么是 Composables？**

**Composables** 是使用 Vue 3 的 Composition API 来封装可复用的逻辑函数。这些函数可以包含 reactive 状态、生命周期钩子、计算属性、方法等，且可以在组件中被引用。

### 2. **Composables 的用途**

- **逻辑复用**：将某些逻辑封装在一个函数中，可以在多个组件中共享和复用。
- **提高可维护性**：避免将业务逻辑混杂在组件的模板或脚本部分，使得代码更加清晰易维护。
- **按需引入**：可以按需在组件中导入使用，不会造成不必要的加载。

### 3. **在 `composables` 文件夹中如何定义函数**

​	你可以在 `composables` 文件夹中定义很多功能函数（例如 API 请求、状态管理、逻辑处理等），然后在组件中通过导入的方式使用。

### 4. **举个例子：**

​	假设你有一个需求，需要在多个页面中**使用相同的逻辑去获取 API 数据**，你可以在 `composables` 文件夹中**创建一个函数来进行封装**。

​	但是nuxt只会扫面一层目录去全局导入和使用，如果有多个文件夹，就需要导入，而不会被自动注册。

## fetch方法

​	`$fetch` 是 Nuxt 3 提供的一个用来发起 HTTP 请求的函数，作为一个更简单且灵活的方式替代传统的 `axios` 或 `fetch` API。它内部封装了很多常见的功能，如自动处理请求拦截、错误处理和响应处理等。

​	在 Nuxt 3 中，你可以通过 `$fetch` 方法来发送 API 请求。

**GET 请求**：
使用 `$fetch` 方法发起 GET 请求时，只需要传递请求的 URL。

```js
// Example of using $fetch in a composable or component
const data = await $fetch('https://api.example.com/data')
console.log(data)

```

**POST 请求**：
通过 `$fetch` 发送 POST 请求时，你可以传递一个对象来指定请求的选项。

```js
const data = await $fetch('https://api.example.com/data', {
  method: 'POST',
  body: {
    key1: 'value1',
    key2: 'value2'
  }
})
console.log(data)
```

### 配置选项：

你可以通过 `$fetch` 传递配置对象来控制请求的其他方面，比如**请求头、请求体、超时等**。

- `method`: HTTP 方法（如 `GET`, `POST`, `PUT`, `DELETE` 等）。
- `headers`: 请求头，作为对象传递。
- `body`: 请求体，用于 `POST` 或其他请求。
- `params`: URL 参数，等效于查询字符串。
- `timeout`: 请求超时时间，单位为毫秒。

## layout布局

可以解放app.vue 把样式布局扔在这个/layouts文件夹里定义的vue文件里。

 <NuxtPage></NuxtPage>这种导航标签要变成slot插槽

而在app.vue里则使用<nuxt-page>标签，意思就是default.vue里的样式。

请注意，如果app.vue这个**父组件**里使用<nuxt-layout>标签去**包裹**上面的<nuxt-page>标签，那么会强制使用default.vue的样式。

![image-20250223211908547](image-20250223211908547.png)

这种方法可以设置一个独立的页面而不受default的影响。

## nuxt的自定义hooks

​	.js的hooks放在/plugins文件夹里，并且：使用 `defineNuxtPlugin` 包裹。在 Nuxt 3 中，推荐将插件包裹在 `defineNuxtPlugin` 中，这样可以让 Nuxt 更好地识别和管理插件，并为未来可能的增强功能提供支持。

比如：

```js
// plugins/useAxios.js
import axios from 'axios'

export default defineNuxtPlugin(nuxtApp => {
  // 你可以在这里进行插件的初始化逻辑，例如配置 axios 实例等
  const api_get = async (url) => {
    try {
      const response = await axios.get(url)
      return response.data
    } catch (error) {
      console.error('请求错误:', error)
      throw error
    }
  }

  // 在 Nuxt 应用中注入 axios
  nuxtApp.provide('api_get', api_get)
})

```

