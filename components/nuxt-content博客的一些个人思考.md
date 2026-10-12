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

index.html负责html，`_padload.js`  负责nuxt页面运行时需要恢复的payload/state。可以知道的是，这些内容都是SSG部署的时候在构建期完成的，每个预渲染路由由旁写出_payload.json。

<svg viewBox="0 0 680 368" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>_payload.json 从构建期生成到客户端消费的链路</title>
<desc>构建期每个预渲染路由旁写出 _payload.json；运行期点击链接时 router.beforeResolve 调用 loadPayload 拉取，命中则写入 nuxtApp.static.data，404 则回退到客户端真查询。</desc>
<defs>
<marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<style>
.bx{stroke-width:.5}
.tt{font:500 14px var(--font-sans)}
.sb{font:400 12px var(--font-sans)}
</style>
<rect class="bx" x="40" y="40" width="600" height="44" rx="12" fill="var(--color-background-secondary)" stroke="var(--color-border-tertiary)"/>
<text class="tt" x="340" y="62" text-anchor="middle" dominant-baseline="central" fill="var(--color-text-primary)">构建期 · nuxi generate：每个预渲染路由旁写出 _payload.json</text>
<path d="M340 84 L340 144" fill="none" stroke="var(--color-text-tertiary)" stroke-width="1.5" marker-end="url(#arrow)"/>
<text class="sb" x="352" y="116" dominant-baseline="central" fill="var(--color-text-tertiary)">与 index.html 同目录</text>
<rect class="bx" x="40" y="144" width="600" height="56" rx="12" fill="#0C447C" stroke="#85B7EB"/>
<text class="tt" x="340" y="166" text-anchor="middle" dominant-baseline="central" fill="#E6F1FB">运行期 · 客户端导航（点站内链接或悬停预取）</text>
<text class="sb" x="340" y="184" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4">router.beforeResolve → loadPayload(path) → fetch &lt;route&gt;/_payload.json?_b=buildId</text>
<path d="M340 200 L340 232 M185 232 L495 232" fill="none" stroke="var(--color-text-tertiary)" stroke-width="1.5"/>
<path d="M185 232 L185 264" fill="none" stroke="var(--color-text-tertiary)" stroke-width="1.5" marker-end="url(#arrow)"/>
<path d="M495 232 L495 264" fill="none" stroke="var(--color-text-tertiary)" stroke-width="1.5" marker-end="url(#arrow)"/>
<text class="sb" x="178" y="250" text-anchor="end" dominant-baseline="central" fill="var(--color-text-tertiary)">200</text>
<text class="sb" x="502" y="250" dominant-baseline="central" fill="var(--color-text-tertiary)">404</text>
<rect class="bx" x="40" y="264" width="290" height="64" rx="12" fill="#0C447C" stroke="#85B7EB"/>
<text class="tt" x="185" y="284" text-anchor="middle" dominant-baseline="central" fill="#E6F1FB">命中</text>
<text class="sb" x="185" y="303" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4">写入 nuxtApp.static.data</text>
<text class="sb" x="185" y="319" text-anchor="middle" dominant-baseline="central" fill="#B5D4F4">useAsyncData 取缓存，零查询</text>
<rect class="bx" x="350" y="264" width="290" height="64" rx="12" fill="#791F1F" stroke="#F09595"/>
<text class="tt" x="495" y="284" text-anchor="middle" dominant-baseline="central" fill="#FCEBEB">缺失或过期</text>
<text class="sb" x="495" y="303" text-anchor="middle" dominant-baseline="central" fill="#F7C1C1">回退到客户端真查询</text>
<text class="sb" x="495" y="319" text-anchor="middle" dominant-baseline="central" fill="#F7C1C1">拉 WASM sqlite，慢且易失配</text>
</svg>
我们来看看Nuxt4.5.2中NuxtPayload类型的原样：

<svg viewBox="0 0 680 410" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
<title>_payload.json 的顶层字段构成</title>
<desc>payload 顶层含 data、state、config、path、prerenderedAt、_errors、once 等字段，其中 data 承载 useAsyncData 与 useFetch 的返回值。</desc>
<style>
.bx{stroke-width:.5}
.hd{font:500 14px ui-sans-serif,system-ui,sans-serif}
.nm{font:500 12px ui-monospace,SFMono-Regular,Consolas,monospace}
.ds{font:400 12px ui-sans-serif,system-ui,sans-serif}
</style>
<rect class="bx" x="40" y="40" width="600" height="350" rx="20" fill="#FFFFFF" stroke="#D3D1C7"/>
<text class="hd" x="64" y="68" dominant-baseline="central" fill="#2C2C2A">_payload.json 顶层字段</text>
<rect class="bx" x="64" y="84" width="552" height="34" rx="8" fill="#E6F1FB" stroke="#85B7EB"/>
<text class="nm" x="80" y="101" dominant-baseline="central" fill="#0C447C">data</text>
<text class="ds" x="210" y="101" dominant-baseline="central" fill="#185FA5">这一页的数据：useAsyncData / useFetch 的返回值</text>
<rect class="bx" x="64" y="126" width="552" height="34" rx="8" fill="#F4F3F0" stroke="#DCDAD3"/>
<text class="nm" x="80" y="143" dominant-baseline="central" fill="#2C2C2A">state</text>
<text class="ds" x="210" y="143" dominant-baseline="central" fill="#5F5E5A">useState 的值，键名带 $s 前缀</text>
<rect class="bx" x="64" y="168" width="552" height="34" rx="8" fill="#F4F3F0" stroke="#DCDAD3"/>
<text class="nm" x="80" y="185" dominant-baseline="central" fill="#2C2C2A">config</text>
<text class="ds" x="210" y="185" dominant-baseline="central" fill="#5F5E5A">runtimeConfig 的 public 部分</text>
<rect class="bx" x="64" y="210" width="552" height="34" rx="8" fill="#F4F3F0" stroke="#DCDAD3"/>
<text class="nm" x="80" y="227" dominant-baseline="central" fill="#2C2C2A">path</text>
<text class="ds" x="210" y="227" dominant-baseline="central" fill="#5F5E5A">这份 payload 属于哪个路由</text>
<rect class="bx" x="64" y="252" width="552" height="34" rx="8" fill="#F4F3F0" stroke="#DCDAD3"/>
<text class="nm" x="80" y="269" dominant-baseline="central" fill="#2C2C2A">prerenderedAt</text>
<text class="ds" x="210" y="269" dominant-baseline="central" fill="#5F5E5A">预渲染时间戳，搭档 serverRendered</text>
<rect class="bx" x="64" y="294" width="552" height="34" rx="8" fill="#F4F3F0" stroke="#DCDAD3"/>
<text class="nm" x="80" y="311" dominant-baseline="central" fill="#2C2C2A">_errors</text>
<text class="ds" x="210" y="311" dominant-baseline="central" fill="#5F5E5A">每个 asyncData key 对应的错误对象</text>
<rect class="bx" x="64" y="336" width="552" height="34" rx="8" fill="#F4F3F0" stroke="#DCDAD3"/>
<text class="nm" x="80" y="353" dominant-baseline="central" fill="#2C2C2A">once</text>
<text class="ds" x="210" y="353" dominant-baseline="central" fill="#5F5E5A">callOnce 用过的 key，防止重复执行</text>
</svg>

本身就包含了文章数据，构建阶段的queryCollection查询出来的结果就会被nuxt序列化进入该页面的payload，而不是浏览器通过查询再sqlite里查询。用户访问的时候GET请求去静态服务器里找_payload.json。拿到已经序列化的page来恢复数据。

之前提到的sqlite wasm机制不一样，是另一套客户端content查询机制，主要解决客户端需要自己执行content查询时候，如何再浏览器里继续查询content数据。如果我去点击一个没有点击过的文档，抓到的就不是payload.json了，而是wasm，官方文档也明确说了：在 SSG/静态部署下，Nuxt Content 会把数据库带到浏览器，由 WASM SQLite 支持**客户端导航或客户端 action 中发生的 Content 查询**。第一次发生这类客户端 Content 查询时，会下载数据库 dump 并初始化浏览器本地 SQLite。

Nuxt Content 的 `queryCollection()` 在浏览器端确实会直接走 SQLite WASM；而且一旦这条查询代码真正被执行，`_payload.json` 不会自动阻止它。那为什么 Nuxt Content 默认还把这个能力放进来？

因为 Nuxt Content v3 的设计目标不是单纯：

> “Markdown → SSG HTML”

而是：

> **统一的 SQL Content 数据层，同时支持 server / serverless / static / client-side navigation。**

官方明确把 v3 定位成 SQL-backed storage，并且声称同一套系统支持开发、静态生成、server、serverless、edge 等部署模式，因此我们再推论一下，数据层甚至是可以做到云服务上的，例如Cloudfalre D1，本质上就是把 Content 的数据层放到了 Cloudflare 的数据库服务上。

至于SSR下的具体表现，我确实还没有尝试过，因此先不写了。

## mermaid策略

原先我并不能很好的处理Mermaid，原先会直接变成代码块渲染出来，为了解决这个，我采用了覆写组件，使用ProsePre组件，当language === 'mermaid' 的时候，交给新的MermaidBlock.vue，其余语言走默认渲染，默认实现就是一行 

```html
<pre :class><slot /></pre>
```

高亮在解析阶段已经完成，所以覆写零成本。这两个组件大致为：

```vue
<!--
  ProsePre 覆写（@nuxtjs/mdc 的默认实现就是 <pre :class><slot /></pre>，
  高亮在 AST 阶段已经完成，slot 里就是高亮后的节点）：
  - ```mermaid 代码块 → MermaidBlock，客户端渲染成 SVG
  - 其余语言 → 和默认实现完全一致
-->
<script setup lang="ts">
defineProps<{
  code?: string
  language?: string
  filename?: string
  highlights?: number[]
  meta?: string
  class?: string
}>()
</script>

<template>
  <MermaidBlock v-if="language === 'mermaid'" :code="code ?? ''" />
  <pre v-else :class="$props.class"><slot /></pre>
</template>

```

```vue
<!--
  MermaidBlock —— 把 ```mermaid 代码块渲染成 SVG。
-->
<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import { useTheme } from '~/composables/useTheme'

const props = defineProps<{ code: string }>()

const { isDark } = useTheme()
const el = ref<HTMLElement | null>(null)
const failed = ref(false)

type Mermaid = typeof import('mermaid').default
let mermaid: Mermaid | null = null
let seq = 0

async function render() {
  const target = el.value
  if (!target || !props.code.trim()) return
  try {
    mermaid ??= (await import('mermaid')).default
    // 主题跟着全站明暗走；每次 initialize 都生效于下一次 render
    mermaid.initialize({
      startOnLoad: false,
      securityLevel: 'strict',
      theme: isDark.value ? 'dark' : 'default'
    })
    // render 要求全局唯一 id（会往 body 塞同名隐藏元素）
    const { svg } = await mermaid.render(`mermaid-${++seq}`, props.code)
    target.innerHTML = svg
    failed.value = false
  } catch (error) {
    // 语法错误等：别把整页弄崩，回退显示源码
    failed.value = true
    console.warn('[mermaid] 渲染失败', error)
  }
}

onMounted(render)
// 切换明暗后重画（mermaid 的主题是渲染期定死的，不是 CSS 变量）
watch(isDark, render)
</script>

<template>
  <pre v-if="failed" class="mermaid-fallback"><code>{{ code }}</code></pre>
  <div v-else ref="el" class="mermaid" />
</template>

```

你只要给组件取名叫做ProsePre，那么这个组件是自动引用的，不需要手动导入，或者注册它。Nuxt Content和Nuxt UI的机制是自动映射，而不是自动导入，你在markdown文件里写下一个代码块，mdc解析器会把他解析为一个pre标签，然后nuxt content会自动将这个pre标签替换为对应的ProsePre组件，这个替换过程发生在底层，因此你在项目源码中不会看到 `import ProsePre from ...` 或显式的组件调用。

上面的接受的props就是一般我们使用的属性，这样可以根据语言来决定渲染方式，其他语言可以退回到pre结构。code是代码字符串，language是代码块语言比如js、mermaid。filename是文件名，如果有[filename]语法。highlights是高亮行号，meta是代码块元信息，class是默认类名。

> 高亮在 AST 阶段已经完成，slot 里就是高亮后的节点，`@nuxtjs/mdc` 在解析阶段就会把代码高亮成 `<span>` 节点，`ProsePre` 的 `slot` 就是这些高亮后的内容。

并且这里mermiadblock组件还设置了回退机制，如果失败了，就会回退为普通代码块。

## 代码高亮

「shiki」这个名字确实来自日语——**式**（しき），就是「样式、形式」的意思。作者 Pine Wu 把 VSCode 编辑器里那套语法高亮引擎抽出来做成了独立库，名字取的就是「样式」；现在主要由 Anthony Fu 维护，Nuxt Content 的 mdc 底层用的就是它。

**shiki 不是编译器，它不解析代码**。整条链路是这样的：

<svg width="100%" viewBox="0 0 680 402" role="img" xmlns="http://www.w3.org/2000/svg">
  <title>Shiki 的语法高亮流水线</title>
  <desc>源代码依次经过 TextMate 正则分词、scope 标注、主题配色映射，最后输出带 CSS 变量的 HTML 结构</desc>
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>

  <rect x="40" y="40" width="600" height="54" rx="12" fill="#2F2F42" stroke="#6E6E93" stroke-width="0.5"/>
  <text x="64" y="61" font-family="var(--font-sans, sans-serif)" font-size="14" font-weight="500" fill="#E8E8F4" dominant-baseline="central">① 输入：一段代码 + 一个语言名</text>
  <text x="64" y="80" font-family="var(--font-sans, sans-serif)" font-size="12" fill="#A9A9C2" dominant-baseline="central">```cpp … 就是一串纯文本，没有任何结构信息</text>

  <line x1="340" y1="94" x2="340" y2="132" stroke="#8E8EA8" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>

  <rect x="40" y="136" width="600" height="54" rx="12" fill="#3C3489" stroke="#7F77DD" stroke-width="0.5"/>
  <text x="64" y="157" font-family="var(--font-sans, sans-serif)" font-size="14" font-weight="500" fill="#CECBF6" dominant-baseline="central">② TextMate 语法分词：正则状态机</text>
  <text x="64" y="176" font-family="var(--font-sans, sans-serif)" font-size="12" fill="#AFA9EC" dominant-baseline="central">逐字符推进，切出 token（字符区间），给每个 token 贴 scope 名</text>

  <line x1="340" y1="190" x2="340" y2="228" stroke="#8E8EA8" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>

  <rect x="40" y="232" width="600" height="54" rx="12" fill="#3C3489" stroke="#7F77DD" stroke-width="0.5"/>
  <text x="64" y="253" font-family="var(--font-sans, sans-serif)" font-size="14" font-weight="500" fill="#CECBF6" dominant-baseline="central">③ 主题映射：scope 名 → 颜色</text>
  <text x="64" y="272" font-family="var(--font-sans, sans-serif)" font-size="12" fill="#AFA9EC" dominant-baseline="central">查 VSCode 主题的 tokenColors，同一个 token 同时算出两套色值</text>

  <line x1="340" y1="286" x2="340" y2="324" stroke="#8E8EA8" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>

  <rect x="40" y="328" width="600" height="54" rx="12" fill="#0F4C3E" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="64" y="349" font-family="var(--font-sans, sans-serif)" font-size="14" font-weight="500" fill="#9FE1CB" dominant-baseline="central">④ 输出 HTML：嵌套 span + CSS 变量</text>
  <text x="64" y="368" font-family="var(--font-sans, sans-serif)" font-size="12" fill="#5DCAA5" dominant-baseline="central">构建期算完写进 HTML，浏览器运行时不做任何高亮计算</text>
</svg>

关键是第 ② 步：**它靠的是 VSCode 那套 TextMate 语法**——一个从 `.tmLanguage.json` 加载进来的正则状态机。它从头到尾扫一遍字符串，用一堆正则去匹配「这里像关键字」「这里像字符串」，匹配上就切出一个 token 并贴个标签（scope）。比如笔记里的 `const int x = 42;` 会被切成：

- `const` → `storage.type.cpp`
- `int` → `storage.type.primitive.cpp`
- `x` → `variable.other.cpp`
- `42` → `constant.numeric.cpp`

这也是为什么 VSCode 里 `// int` 这种注释里的关键字不会上色（注释规则优先级更高），但把变量命名成 `int` 就可能被误判——没有 AST、没有类型检查，纯正则博弈。好处是快、对残缺代码也宽容，而且**任何语言只要有一份语法文件就能支持**，这也是它语言覆盖那么广的原因。

<svg width="100%" viewBox="0 0 680 344" role="img" xmlns="http://www.w3.org/2000/svg">
  <title>Shiki 双主题：一份 HTML 携带两套颜色</title>
  <desc>同一个 token 同时输出浅色与暗色两个 CSS 变量，切换主题时只改一个 class，颜色由 CSS 规则翻</desc>
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>

  <text x="40" y="34" font-family="var(--font-sans, sans-serif)" font-size="12" fill="#9A9AB4" dominant-baseline="central">github-light 的色值</text>
  <rect x="40" y="44" width="600" height="44" rx="8" fill="#FFFFFF" stroke="#D0D0D8" stroke-width="0.5"/>
  <text x="64" y="67" font-family="var(--font-mono, monospace)" font-size="13" dominant-baseline="central">
    <tspan x="64" fill="#D73A49">const</tspan>
    <tspan x="108" fill="#005CC5">int</tspan>
    <tspan x="140" fill="#24292E">x = 42;</tspan>
  </text>

  <text x="40" y="102" font-family="var(--font-sans, sans-serif)" font-size="12" fill="#9A9AB4" dominant-baseline="central">github-dark 的色值</text>
  <rect x="40" y="112" width="600" height="44" rx="8" fill="#0D1117" stroke="#3A3A4A" stroke-width="0.5"/>
  <text x="64" y="135" font-family="var(--font-mono, monospace)" font-size="13" dominant-baseline="central">
    <tspan x="64" fill="#F97583">const</tspan>
    <tspan x="108" fill="#79B8FF">int</tspan>
    <tspan x="140" fill="#E1E4E8">x = 42;</tspan>
  </text>

  <line x1="340" y1="156" x2="340" y2="180" stroke="#8E8EA8" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>

  <rect x="40" y="184" width="600" height="74" rx="12" fill="#23233A" stroke="#6E6E93" stroke-width="0.5"/>
  <text x="64" y="208" font-family="var(--font-mono, monospace)" font-size="12" fill="#9FE1CB" dominant-baseline="central">&lt;span class="sJ8bj"&gt;const&lt;/span&gt;</text>
  <text x="64" y="234" font-family="var(--font-mono, monospace)" font-size="12" fill="#A9A9C2" dominant-baseline="central">.sJ8bj { --shiki-default: #D73A49; --shiki-dark: #F97583 }</text>

  <line x1="340" y1="258" x2="340" y2="282" stroke="#8E8EA8" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>

  <rect x="40" y="286" width="600" height="58" rx="12" fill="#3C3489" stroke="#7F77DD" stroke-width="0.5"/>
  <text x="64" y="307" font-family="var(--font-mono, monospace)" font-size="12" fill="#CECBF6" dominant-baseline="central">html.dark .shiki span { color: var(--shiki-dark) }</text>
  <text x="64" y="328" font-family="var(--font-sans, sans-serif)" font-size="12" fill="#AFA9EC" dominant-baseline="central">切主题只是换掉 html 上的一个 class，颜色由 CSS 翻，不重新高亮</text>
</svg>

第 ③ 步还有个细节值得单说：**同一个 token 会同时算出两套色值**，写进 CSS 变量，而不是生成两份 HTML。所以切明暗时不需要重新高亮——只是 `html` 上的 class 变了，CSS 变量跟着翻。这也是为什么当初把开关定成 `<html>.dark` 这个类名：mdc 生成的规则硬编码了 `html.dark .shiki span`，两边必须对齐。

shiki 最早用 Oniguruma（一个 C 写的正则引擎，编译成 WASM 跑在浏览器里），但 mdc 现在默认换成了 JS 原生正则引擎。

在我们这个项目里，shiki **不在浏览器里跑**。Nuxt Content 构建时解析 md，mdc 就在那一刻调用 shiki 把代码块变成 HTML，直接写进页面和 payload。所以你线上页面打开时代码已经是彩色的，客户端零成本——这也是为什么当初配错 `highlight: false` 就"静默无高亮"：构建期悄悄跳过了，运行期没有任何补救机制。（mdc 另外留了条 `/api/_mdc/highlight` 兜底路由，只在客户端动态高亮时用得上，我们这种 SSG 场景用不到。）




