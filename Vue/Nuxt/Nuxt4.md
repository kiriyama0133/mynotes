
## useHead

useHead是一个核心的组合式函数，用来程序化管理Html文档中的head部分，如果你想修改网页的标题、关键词、描述，或者引入外部的 CSS/JS 文件，`useHead` 就是你的首选工具。

他可以做到**管理元数据**，**注入外部资源**，也可以**传入响应式数据**，当你的业务数据发生变化时，**页面的标题或描述也会随之自动更新**，而无需手动操作 DOM。

它和SSR是兼容的，nxut在服务端渲染阶段就处理好了这些头信息，当浏览器接收到Html的时候，`<head>` 里的内容就已经是完整的了。


我们可以通过useHead来设置`title`、`meta` 标签（如描述、关键词、Open Graph 标签等）。这对于搜索引擎优化（SEO）至关重要，因为爬虫主要通过这些信息来了解你的网页内容。

很多的图标库或者UI框架式是都要使用sciprt和link的，一个是引入js，一个是引入css，因此我们可以在useHead里完成。


### SEO优化

nuxt推荐在大多数的SEO场景下使用useSeoMeta，它是useHead的快捷方式，专门为meta标签设定的，有很棒的ts自动补全，能防止写错og:title、twitter:card等属性。

虽然 `useHead` 也可以通过 `meta` 数组来配置 SEO，但 `useSeoMeta` 专门为 SEO 场景做了优化，提供了更强大的类型提示和更简洁的语法。

用法如下：

```ts
useSeoMeta({ title: 'Kiriyama | 视觉开发与架构', ogTitle: 'Kiriyama 的个人技术空间', description: '专注于 .NET、C++ 图形学及现代前端技术的深度探索。', ogDescription: '在这里展示我的技术栈：C#, Skia, Nuxt4 以及更多艺术感与技术结合的项目。', ogImage: 'https://example.com/og-image.png', twitterCard: 'summary_large_image', })
```

title是给浏览器和搜索引擎看的，而ogTitle是给社交媒体和分享卡片使用的

title的标签就是我们熟悉的title标签，但是ogTitle则是：
```html
<meta property="og:title" content="...">
```

显示的位置是: 微信、飞书、Discord、Facebook 等分享链接时的卡片，核心目的是点击率。

## 生命周期

### 水合

同构渲染一般叫做通用渲染，指的是代码可以同时运行在服务端和客户端的能力，同构渲染的工作模式为两个阶段，第一阶段是服务端渲染阶段，也可以叫做脱水。

当你第一次访问页面的时候，服务器执行vue代码，就爱那个数据填充进入组件，生成完整的html字符串给浏览器，用户立即就可以看到内容，SEO友好（搜索引擎爬虫能抓到完整的html）。
>[!tip] 注意
>第一阶段生成的是静态html，组件树转换为纯字符串，服务器会将抓取到的数据序列化成一个 JSON 对象（通常挂载在 `window.__NUXT__` 下），随 HTML 一起发给浏览器。

第二阶段是**客户端激活，水合阶段**，浏览器解析完了html后，开始加载**nuxt的客户端脚本，vue接管控制器**，在浏览器中再次运行一次代码。

水合的过程中vue不会重新创建DOM，而是会去扫描现有的HTML，将虚拟DOM（VNDOE）和真实的DOM节点挂钩。绑定事件监听器，把 `v-on:click` 之类的交互逻辑关联到 DOM 元素上。

**客户端代码不会再次发送网络请求去拿首页数据**，而是**直接读取服务器传过来的 JSON 对象**（刚才说的“零件列表”）。

这个阶段很容易遇到水合错误，如果**第一阶段生成的东西**和**第二阶段扫描到的东西**对不上，Vue 就会报警告甚至崩溃。

典型的场景就比如服务器渲染时间，或者`<p>`标签里面塞div导致浏览器自动修复的时候发现DOM不一样；如果你在 `setup` 顶层直接用了 `window` 或 `localStorage`。服务器端没有这些对象，渲染会失败；客户端有，导致两端结果不一致。

## 数据获取

数据获取是最核心的一环，如果你用传统的axios放在OnMounted里的话，爬虫得到是一个空壳，SEO是没用的，因此Nuxt会提供四个核心工具：useFetch，useAsyncData、$fetch和useLasyFetch四个工具。

useFetch 是SSR友好的封装，可以自动去重，用于获取初始页面的核心数据，useAsyncData是更加灵活的异步数据处理，**需要自定义请求逻辑或者复杂转换逻辑场景**。

它们两个都是服务端获取+客户端复用payload。 $fetch是纯http请求工具，用户交互触发的请求，可以用于点击、滚动加载等，仅限于客户端执行。

useLazyFetch工具的核心特征是useFetch但是表现为lazy模式，主要用于非首屏关键数据，不阻塞路由导航。

那么我们需要给BaseUrl配置代理，比如我这里的apifox mock接口的地址是：http://127.0.0.1:4523/m1/6982732-6700215-default
api是:/api/user。

那么我们可以在nuaxt.config.ts中去配置nitro。
nitro有两个配置一个是devProxy，一个是routeRules，前者只在csr下生效，后者是ssr、csr上都生效。

![[Pasted image 20260424235322.png]]

### pick

很多时候后端接口返回的对象很大，包含着create_time、update_time、author_email等，但是前端只需要title、content等，这时候可以使用pick来选择保留进入html的字段，但是pick只能选第一层。
![[Pasted image 20260425131608.png]]

### transform

对于大部分的api数据都是嵌套的格式，推荐使用transform来处理数据：
![[Pasted image 20260425131614.png]]


### refresh 
useFetch的第一个参数是它的key，如果你在同一个页面措辞调用同一个接口，由于key相同，数据可能不会更新。


```ts
const {data, refresh} = await useFetch("/api/article")
// 点击刷新按钮调用refresh即可重新获取一次
```


### useAsyncData

如果接口需要复杂逻辑，比如需要判断参数，或者想把它放在一个函数里面，可以使用这个：


```ts
async function getData(){
const {data, pending, error} = await useFetch<BaseRespones<ArticleType>>("/api/article/detail")
}
```


### useLazyFetch

这是useFetch的一个变体，它的逻辑是先跳页面，再加载数据，通常情况下，Nuxt的useFetch会阻塞路由跳转，也就是说：如果你在setup里面使用了await useFetch的话，用户点击链接，**浏览器就会卡在当前页，直到新页面的数据请求完成后再进行跳转**。

useLazyFetch则可以避免这种阻塞，useLazyFetch相当于执行了useFetch(url, {lazy: true}，它可以进行非阻塞跳转，路由会立即切换，组件立即挂载。前台异步：数据再后台抓取，抓取过程中页面已经展示出来了。响应式：返回的pending属性会从true变为fasle，可以根据这个展示加载动画Loading。

比如你有一个页面数据很大，很复杂的表，接口可能要2s，这时候可以用这个，页面秒开，并显示转圈/骨架屏。

const {pending, data: post} = useLazyFetch('/api/post')


### 获取原始响应

如果你需要获取原始响应的数据，比如判断响应的http状态码、http响应头。有两种做法，一种是onResponse属性，一种是$fetch.rAw


```ts
const {data, error} = await useFetch('/api/current/user', {
	onResponse ({response}){
		const customHeader = reseponse.headers.get('x-custom-header')
		console.log(customHeader)
	}
})
```

然后是$fetch.raw

```ts
const fetchUser = async()=>{
try{
const res = await $fetch.raw("/api/current/user")
const user = res._data
const serverTime = res.headers.get('data')
}}
```


## 状态管理

### useState

useState是nuxt 为SSR环境设计的响应式状态管理器，它能够在服务器和浏览器之间共享状态，可以确保每个用户的请求都有独立的状态，并且能解决水合不匹配。


```ts
// 服务端
const config = useState('app-config', () => ({
  theme: 'dark',
  language: 'zh-CN'
}))
// 服务端渲染时：config 的值被序列化到 payload
```


```html
<!-- 生成的 HTML 中包含 -->
<script>
  window.__NUXT__ = {
    data: {
      'app-config': { theme: 'dark', language: 'zh-CN' }
    }
  }
</script>
```

```ts
// 客户端水合时
const config = useState('app-config')  // 自动从 window.__NUXT__ 读取
// ✅ 值和服务端完全一致，无需重新请求或计算
```

如果使用ref的话，会导致全局/组件级别的跨请求共享，用户A会看到用户B的数据。

### pinia

`@pinia/nuxt` 模块为 Pinia 提供了开箱即用的 SSR 支持。它可以自动处理状态在服务端和客户端之间的序列化与“注水”（hydration），让你在 Nuxt 应用中能无缝地使用 Pinia 进行状态管理[](https://deepwiki.com/vuejs/pinia/4.2-nuxt-integration)。

它做了几件关键的事情，但你安装并且配置了@pinia/nuxt模块之后，他就会自动在应用启动的时候创建 Pinia 实例，并将其与 Nuxt 的运行时环境连接起来[](https://deepwiki.com/vuejs/pinia/4.2-nuxt-integration)。

在服务端，他会捕获pinia在组件渲染过程中产生的所有状态变更，并自动将这些状态序列化到 Nuxt 的 payload (`nuxtApp.payload.pinia`) 中[](https://deepwiki.com/vuejs/pinia/4.2-nuxt-integration)。

在客户端，应用启动时，模块会读取 payload 中的状态，并直接将其作为 Pinia 的初始状态进行恢复。这样就**保证了服务端和客户端状态的完全一致，从而避免了水合不匹配的问题**[](https://deepwiki.com/vuejs/pinia/4.2-nuxt-integration)。

模块会配置好自动导入功能，你可以在组件或者Composables中直接使用 `defineStore()`、`storeToRefs()` 等函数，无需手动导入[](https://deepwiki.com/vuejs/pinia/4.2-nuxt-integration)。

如果你的 Store 中确实需要存储这类无法被json序列化的数据，可以使用 Pinia 提供的 `skipHydrate()` 辅助函数来标记它们，告诉框架：“这个数据不需要被传递和激活”[](https://blog.csdn.net/gitblog_01060/article/details/151535419)。

```ts
import { skipHydrate } from 'pinia'

export const useMyStore = defineStore('my-store', () => {
  // 这个复杂实例不会被序列化，客户端会使用服务端计算后的新实例
  const complexInstance = skipHydrate(new SomeComplexClass())
  
  return { complexInstance }
})
```

**生产配置建议**：Pinia官方与Nuxt团队的集成中，**强烈建议保持 `renderJsonPayloads` 为默认开启状态**。关闭它可能会导致与Pinia SSR 相关的序列化警告[](https://blog.gitcode.com/38dd8771bae7d48517395c904c5d3947.html)。


**背后的机制**：

```ts
// Nuxt 内部简化逻辑
const requestScopedState = new Map()  // 每个请求一个独立 Map
export const useState = (key, init) => {
  // 当前请求的独立存储
  if (!requestScopedState.has(key)) {
    requestScopedState.set(key, init ? init() : null)
  }
  return requestScopedState.get(key)
}
// 请求结束 → 清空 requestScopedState
```

但不要在useState中存储不可序列化的数据。

```ts
// ❌ 错误：函数无法被序列化
const myFn = useState('fn', () => () => console.log('hi'))

// ❌ 错误：Symbol 无法被序列化
const sym = useState('sym', () => Symbol())

// ✅ 正确：基础类型、对象、数组
const config = useState('config', () => ({ theme: 'dark' }))
```

并且当ref在顶层，比如说utils/[...].ts的时候会发生泄漏，但是如果是在setup里面的话，就很不会发生泄漏。泄漏的是“进程级共享内存”，不是“请求级实例内存”。

setup里面的ref属于组件实例，而ssr每个请求都会创建一套新的应用/组件实例，因此数据就会跟着当前的组件实例走，本次的请求渲染完就结束，不会被下一个请求复用。

- 模块顶层：`const x = ref(...)`（文件一加载就创建）
    - 跟着 Node 进程走
    - 后续所有请求都可能读写同一个 `x`

在 Nuxt SSR 里，每个请求都会重新跑页面 `setup`（服务端那次），所以 setup 内状态天然是 request-scoped。  
而模块顶层代码通常只在进程启动/首次 import 时执行一次，因此是 process-scoped，才会串用户。


## HTTP处理文件分片

这是HTTP协议中`multipart/form-data`格式的核心机制。当浏览器上传多个文件（或一个文件加其他表单字段）时，会**自动**使用这种格式进行打包。

假设你提交了一个包含两个文件（`avatar.jpg` 和 `resume.pdf`）和一个用户名（`name`）的表单，浏览器生成的请求体大致长成这样：


```http
POST /upload HTTP/1.1
Host: example.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Length: [自动计算的总长度]
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="name"
张三
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="avatar"; filename="avatar.jpg"
Content-Type: image/jpeg
[这里是一大段二进制的图片数据]
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="resume"; filename="resume.pdf"
Content-Type: application/pdf
[这里是一大段二进制的PDF数据]
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```
`boundary` 是整个过程的核心。它本质上是一个**随机生成的、足够复杂的唯一字符串**（如上面的 `----WebKitFormBoundary7MA4YWxkTrZu0gW`）。

它的工作就是充当分割线，他首先在Content-Type头部被声明，用于告诉服务器这个字符串来分割不同数据块。

无论是普通文本还是文件数据，开始之前都会加上-- 和 boundary，表明一个新的字段开始。

结束的时候所有字段最后都会加上 -- + boundary + -- 来表示整个请求体到此结束。

服务器收到请求的时候会先读取Content-Type头部，**解析出boundary的值，然后按照这个规则去切片和解析请求体，从而还原出你上传的表单字段和文件。**

这里需要澄清一个很常见的概念混淆，“浏览器自动处理为分片”**，和我们通常提到的**“文件分片上传”**，是两个完全不同的东西，只是名字里都有“分片”：

|特性|**`multipart/form-data` 格式**|**大文件分片上传**|
|---|---|---|
|**触发方式**|**浏览器自动**，只要你在 `<form>` 或 `FormData` 中设置了 `type="file"`|**需要手动编码**实现，浏览器不会自动做|
|**目的**|将**多个不同**的文件/字段打包在一个请求中发送|将**一个巨大**的文件切成小块，分多次请求发送|
|**行为**|所有部分（分片）在**同一个HTTP请求**里|每个分片是一个**独立的HTTP请求**|
|**分片大小**|没有固定大小，就是文件的完整数据|由你控制（通常1MB-10MB）|

当我们需要监听上传进度的时候，可以用原生`XMLHttpRequest` 或者 `axios` 的 `onUploadProgress` 事件，结合 `FormData`：

```ts
// 在 Vue 组件或 composable 中
const uploadFiles = async (files) => {
  const formData = new FormData()
  files.forEach(file => {
    formData.append('files', file) // 浏览器会自动处理 boundary
  })
  // 使用 axios 并监听上传进度
  const { data } = await axios.post('/api/upload', formData, {
    headers: {
      'Content-Type': 'multipart/form-data' // 注意：不设置 boundary，axios 会帮你生成
    },
    onUploadProgress: (progressEvent) => {
      const percentCompleted = Math.round((progressEvent.loaded * 100) / progressEvent.total)
      console.log(`上传进度: ${percentCompleted}%`)
      // 更新你的响应式状态
    }
  })
}
```

## Cookie扩展工具

useCookie是nuxt提供的SSR友好类型的Cookie管理工具，他可以在浏览器和服务器之间同步Cookie，就算用户刷新页面或者关闭浏览器，数据依然是存在的，这个是SSR安全的，**浏览器端渲染的时候**，他可以自动去读请求头里的Cookie，在客户端又可以通过Js去读写。

我们会在存储用户的token和存储用户的个性化配置的时候使用。

