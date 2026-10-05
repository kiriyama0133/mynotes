---
date: 2026-10-05
tags:
  - nuxt
  - vue
  - typescript
---
nuxt-content是一个以内容驱动的nuxt模块，本身是很适合制作我们的博客网站的，Nuxt Content 会读取你项目中的 `content/` 目录，解析 `.md`、`.yml`、`.csv` 或 `.json` 文件，并为你的应用程序创建一个强大的 [数据](https://nuxtjs.org.cn/modules/content#)层。此外，还支持通过 [MDC 语法](https://content.nuxtjs.org.cn/docs/files/markdown)在 Markdown 中使用 Vue 组件，这个在模板里有提到过一个counter计数器实现。

本次开发所使用的nuxt.config如下：


```ts
// https://nuxt.com/docs/api/configuration/nuxt-config
import tailwindcss from '@tailwindcss/vite'

export default defineNuxtConfig({
  css: ['~/assets/css/main.css', '~/assets/css/main.scss'],
  modules: ['@nuxt/content', '@pinia/nuxt', '@vueuse/nuxt', '@nuxt/eslint'],
  app: {
    head: {
      script: [
        {
          innerHTML:
            "(function(){try{var s=localStorage.getItem('blog-theme');var d=s?s==='dark':window.matchMedia('(prefers-color-scheme: dark)').matches;if(d)document.documentElement.classList.add('dark')}catch(e){}})()",
          tagPosition: 'head' as const
        }
      ]
    }
  },
  vite: {
    plugins: [tailwindcss()]
  },
  content: {
    build: {
      pathMeta: {
        // slugify 默认把中文整段删掉（\w 不含 CJK），补进白名单。
        slugifyOptions: {
          lower: true,
          remove: /[^\w\s$*_+~.()'"!\-:@\u4e00-\u9fff]+/g
        }
      }
    }
  },
  nitro: {
    preset: 'static',
    prerender: {
      crawlLinks: true,
      routes: ['/'],
      failOnError: false
    }
  },
  devtools: { enabled: true },
  compatibilityDate: '2024-04-03'
})
```

本次项目默认设置的toc配置深度为2，搜索深度也为2。因此只会找到 `## #` 这两个分节。我们默认打开了contentHeading: true，让nuxt content根据文章开头的H1和后续内容来自动生成title和description，这个可以为我们省去很多时间去写解析器。nuxt默认使用shiki来提供代码高亮，这个对于代码博客很有作用，我们还可以指定加载哪些语言。

## Slugfy

nuxt content的路径构建过程中使用了slugfy逻辑，我们配置的时候就能够发现相关配置。它可以把md文件路径通过content build环节 进行 path/slug 处理之后存入数据库，这样我们使用queryCollection就可以提出了。

Content 在构建时需要把**文件路径转换成规范化的 content path**。

## 构建相关思考

原先框架本身是偏向SSR，用户访问的时候会通过Nitro服务器进行预先的SSR行为，queryCollection行为会在nitro里执行，然后获取Content了之后再通过vue ssr返回html给用户，这个是本来的路径。用户每访问一次页面，都可能出发服务器端的渲染过程，这个cpu、内存资源消耗还是挺大的。

我们的服务器资源没那么强，所以最好选择SSG好一些。


```
content/
   │
   ▼
Content Build
   │
   ├── Markdown 解析
   ├── frontmatter
   ├── path
   ├── slug/path 规范化
   └── metadata
   │
   ▼
Content 数据库
```

差不多是这样

```ts
nitro: {
  preset: 'static',
  prerender: {
    crawlLinks: true,
    routes: ['/']
  }
}
```

这样配合pnpm generate，实际上构建服务器就会通过nuxt build使用content模块进行queryCollection先生成文档的index.html。构建阶段我们会建立应用和content数据并开始预渲染。crawler预渲染爬虫就会开始工作。

假设content下有一些md文档，经过了content的build以后就形成了对应的content documents，以及他们的Path。但是content db 里有这个文档不代表预渲染会渲染这个url。虽然slugfy已经通过文件路径生成了content path 和 metadata，但是没有预渲染环节。

当crawler渲染完了首页之后就会从html链接中发现文档链接，比如说/vue/reactive，这样就算是发现并进行了预渲染，这个也是很坑的地方之一，因为如果你写了分页查询的话，那么crawler就会找不到，也就不会预渲染和生成html，会导致导航失败。客户端可能是导航成功了，因为已经通过了slugfy得到了content document的path，结果静态部署目录里可能没有这个index.html，最后导致静态服务器找不到资源。

## 浏览器端的sqlite

这个是个很奇怪的一点，为什么content模块不去使用indexeddb，而却去使用wasm和嵌入式sqlite数据库，原先nuxt content v3的设计是客户端导航时，让queryCollection再浏览器本地执行sql查询，官方文档明确说，首次发生客户端content查询时，会从服务器下载构建好的database dump，并在浏览器里初始化本地化sqlite，之后查询在浏览器本地执行。在 Node 环境下，Nuxt Content 默认 SQLite adapter 可以使用 `better-sqlite3`；Node 22.5+ 也可以使用原生 `node:sqlite`

但是浏览器没有node，因此使用sqlite compiled to wasm这样在浏览器里使用sqlite。Nuxt Content 官方 API 本身就明确把 `queryCollection()` 描述成 SQL-backed query builder，并支持 `where`、`order`、`limit`、`skip` 等 SQL 查询语义。

第一次查询会出发获取wasm的sqlite二进制程序本体的请求，在获取到dump文件之后：

```
/__nuxt_content/{collection}/sql_dump.txt
```

会查询得到content来恢复页面，这个过程就不涉及payload了。

## _ payoad.json

一个_payload.json输出实际上可能是

```
/cpp/boost/编译python可调用的pyd-c模块/
│
├── index.html
└── _payload.json
```

index.html负责html，`_padload.js`  负责nuxt页面运行时需要恢复的payload/state。比如说这个payload里面已经有：

```
page-/cpp/boost/编译python可调用的pyd-c模块
       ↓
Content Document
       ↓
title
body
description
navigation
path
seo
...
```

本身就包含了文章数据，构建阶段的queryCollection查询出来的结果就会被nuxt序列化进入该页面的payload，而不是浏览器通过查询再sqlite里查询。用户访问的时候GET请求去静态服务器里找_payload.json。拿到已经序列化的page来恢复数据。

之前提到的sqlite wasm机制不一样，是另一套客户端content查询机制，主要解决客户端需要自己执行content查询时候，如何再浏览器里继续查询content数据。如果我去点击一个没有点击过的文档，抓到的就不是payload.json了，而是wasm，官方文档也明确说了：在 SSG/静态部署下，Nuxt Content 会把数据库带到浏览器，由 WASM SQLite 支持**客户端导航或客户端 action 中发生的 Content 查询**。第一次发生这类客户端 Content 查询时，会下载数据库 dump 并初始化浏览器本地 SQLite。

Nuxt Content 的 `queryCollection()` 在浏览器端确实会直接走 SQLite WASM；而且一旦这条查询代码真正被执行，`_payload.json` 不会自动阻止它。那为什么 Nuxt Content 默认还把这个能力放进来？

因为 Nuxt Content v3 的设计目标不是单纯：

> “Markdown → SSG HTML”

而是：

> **统一的 SQL Content 数据层，同时支持 server / serverless / static / client-side navigation。**

官方明确把 v3 定位成 SQL-backed storage，并且声称同一套系统支持开发、静态生成、server、serverless、edge 等部署模式，因此我们再推论一下，数据层甚至是可以做到云服务上的，例如Cloudfalre D1，本质上就是把 Content 的数据层放到了 Cloudflare 的数据库服务上。

至于SSR下的具体表现，我确实还没有尝试过，因此先不写了。




