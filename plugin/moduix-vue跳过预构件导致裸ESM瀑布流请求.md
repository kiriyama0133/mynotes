---
date: 2026-11-01
tags:
  - moduix
  - plugin
  - esm
---
在11/01/2026的开发中我发现了一件少见的事情，我在导航到一个页面的时候发送了将近2100多个请求，这个很可能是某个模块没有被预构建导致的，因此我后来在检查的时候发现是这个@moduix/vue跳过了预构建导致@lucide/vue裸esm瀑布流请求。

## 根因

这就是个经典的vite依赖预构建失效导致的esm请求瀑布，vite在启动本地开发服务器的时候会做一个叫做依赖预构建的操作，核心目的是把cmj转成esm，很多老的库是cjs写的，浏览器不支持；合并碎片化文件，这个是主要，一个庞大的第三方库（比如 `lodash` 或 `lucide`）可能有几千个零碎的 JS 文件。如果不预构建，浏览器按原生 ESM 规范逐个去请求这些文件，**HTTP 请求数会直接爆炸**。Vite 会用 `esbuild` 把这几千个文件提前打包成**一个或几个**巨大的 `.js` 文件，浏览器只需请求 1 次即可。

我在 `nuxt.config.ts` 里配置了 `build.transpile: ['@moduix/vue']`。 在 Nuxt 中，`transpile` 告诉服务端/构建端：“这个包没有被编译好，不要直接扔给构建器，请手动处理它”。

这引发了一个vite的机制，vite遇到transpile里的包的时候，默认会排除在依赖预构建之外，结果就是`@moduix/vue` 以及**它内部依赖的所有包**（尤其是 `@lucide/vue`），都不会被打包，而是以**原始的、裸的 ESM 模块**形式被加载。

第二步是@lucide/vue的原因，这个图标设计的非常激进，为了实现完美的tree-shaking，入口文件有成百上千个导出，如果预构建生效，这1900多个export会被瞬间合并成一个，但是因为预构建被跳过了，因此浏览器看到这1900多个Import会直接发送这么多个`/@fs/` HTTP 请求，去挨个拉取那 1900 个小 `.mjs` 文件。这就是导致 2100+ 请求和 20s 卡顿的元凶。

日志里的 `No sources are declared in this source map`。  
当 `@lucide/vue` 被原封不动地加载时，它底部的 `//# sourceMappingURL=...` 被浏览器读取。但由于它在打包发布时配置有问题，这个内联的 Base64 映射表里 `"sources": []` 是空的。

## 解决办法

将 @lucide/vue 别名映射至项目已安装的 lucide-vue-next（两者同源且导出完全一致），复用同一个预构建产物，避免加载两套完整的 Lucide 图标库。

显式声明 Vite 依赖预构建（vite.optimizeDeps.include）：
将 lucide-vue-next、@moduix/vue > clsx、以及 @moduix/vue 内部引用的 @ark-ui/vue/factory、@ark-ui/vue/splitter、@ark-ui/vue/tooltip、@ark-ui/vue/utils 等组件显式加入optimizeDeps.include。

已验证 @moduix/vue 内部的图标引用现已直接指向单文件预构建产物 node_modules/.cache/vite/client/deps/lucide-vue-next.js，消除了 1900+ 个散碎图标请求与 Source Map 报错。


```ts
    vite: {
    plugins: [
      tailwindcss()
    ],
    resolve: {
      alias: {
        '@lucide/vue': 'lucide-vue-next'
      }
    },
    optimizeDeps: {
      include: [
        'lucide-vue-next',
        '@moduix/vue > clsx',
        '@ark-ui/vue/factory',
        '@ark-ui/vue/splitter',
        '@ark-ui/vue/tooltip',
        '@ark-ui/vue/utils',
        '@ark-ui/vue/avatar',
        '@ark-ui/vue/clipboard',
        '@ark-ui/vue/collapsible',
        '@ark-ui/vue/dialog',
        '@ark-ui/vue/field',
        '@ark-ui/vue/fieldset',
        '@ark-ui/vue/file-upload',
        '@ark-ui/vue/segment-group',
        '@ark-ui/vue/select',
        '@ark-ui/vue/steps',
        '@ark-ui/vue/switch',
        '@aurouscia/vue-china-area-select'
      ]
    },
    ssr: {
      noExternal: [
        '@moduix/vue'
      ]
    }
  },
```

