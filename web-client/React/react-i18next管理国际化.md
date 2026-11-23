---
date: 2026-11-23
tags:
  - web-client
  - react
  - i18n
  - react-i18n
---
不同的框架都有自己的i18n模块实现，为了更好的学习，这里我们可以草草对比一下@nuxtjs/i18n模块，看看两个模块在使用体感和设计上的区别

`react-i18next` 和 `@nuxtjs/i18n` 都解决国际化问题，但它们处于不同的技术生态。前者用于给react应用提供国际化能力，或者则是给nuxt提供完整国际化，包括了翻译、语言路由、seo、语言资源加载等。

然后我们来个核心api对照看看，如果要实现一个简单的标题和一个语言切换按钮的话怎么做。首先，对于react-i18next：

```tsx
const { t, i18n } =
  useTranslation()

return (
  <>
    <h1>{t('home.title')}</h1>

    <button
      onClick={() =>
        i18n.changeLanguage('zh')
      }
    >
      中文
    </button>
  </>
)
```

然后nuxtjs/i18n：

```vue
<script setup>
const { t, setLocale } =
  useI18n()
</script>

<template>
  <h1>{{ t('home.title') }}</h1>

  <button
    @click="setLocale('zh')"
  >
    中文
  </button>
</template>
```

这在两者的体感上，近乎是一致的，都是使用t方法来获得翻译文本，只有切换语言和获取当前语言的差别而已。

|功能|React|Nuxt|
|---|---|---|
|获取翻译函数|`useTranslation()`|`useI18n()`|
|获取翻译文本|`t('home.title')`|`t('home.title')`|
|切换语言|`i18n.changeLanguage()`|`setLocale()`|
|获取当前语言|`i18n.language`|`locale`|
|获取可用语言|自己维护或读取配置|`locales`|
|翻译资源|i18next resources|Vue I18n messages|

注意：Nuxt 中 `useI18n()` 返回的 `t` 来自 Vue I18n，而不是 i18next。两者 API 相似，但底层实现不同。

在翻译资源的组织方式上，两者都偏向于使用json对象来组织翻译文本，假设项目里使用了中文和英文两种语言，对于React来说：

```
src/
└── locales/
    ├── en/
    │   └── translation.json
    └── zh/
        └── translation.json
```
`zh/translation.json`：

```json
{
  "home": {
    "title": "首页",
    "description": "管理你的任务"
  }
}
```

初始化 i18next 时注册资源：

```ts
import i18n from 'i18next'
import { initReactI18next } from 'react-i18next'

i18n
  .use(initReactI18next)
  .init({
    resources: {
      en: {
        translation: en
      },
      zh: {
        translation: zh
      }
    },
    lng: 'zh',
    fallbackLng: 'en',
    interpolation: {
      escapeValue: false
    }
  })
```

对于nuxt应用来说也是差不多的：

```
i18n/
└── locales/
    ├── en.json
    └── zh.json
```

`nuxt.config.ts`：

```ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/i18n'],

  i18n: {
    defaultLocale: 'zh',
    locales: [
      {
        code: 'zh',
        name: '简体中文',
        file: 'zh.json'
      },
      {
        code: 'en',
        name: 'English',
        file: 'en.json'
      }
    ]
  }
})
```

Nuxt i18n 会根据配置加载语言资源。它还支持按需加载语言文件，而不是把所有语言都放进初始资源中。

到这里其实两者的功能都差不多，到了这一步之后开始出现明显差异，nuxt i18n不止管理语言翻译，还管理了URL，这个是两者最值得注意的区别，React 的 `react-i18next` 本身不负责路由。

切换语言后，页面 URL 可以完全不变，只改变界面文本，而 Nuxt i18n 可以将语言和路由关联：

```
/       → 中文首页
/en     → 英文首页
/about  → 中文关于页面
/en/about → 英文关于页面
```

它提供了：

- `useLocalePath()`：生成对应语言的路由。
- `useSwitchLocalePath()`：生成其他语言的当前页面路由。
- `setLocale()`：切换语言。
- `localePath()`：解析本地化路由。

这些能力对于需要 SEO、多语言网站非常重要，但对于 Electron 桌面应用，URL 语言前缀通常不是核心需求。

i18n负责UI层功能，但是实际上会影响到用户配置、数据展示、路由、日期时间、货币、权限提示、业务规则，甚至数据库设计。

我们建议一个概念，i18n不等于翻译，国际化本身真正目的是让软件适应不同语言、地区和文化环境的设计过程，本地化（Localization，L10n）则是针对具体语言和地区提供实际适配。

日期、时间和数字、货币等等内容，很多都容易被忽略，比如说数据库存储了：

```json
{
  "createdAt": "2026-11-23T07:30:00Z",
  "price": 1250.5,
  "distance": 1.25
}
```

不同地区的用户可能希望看到不同格式。

**同一份数据，不同地区的展示**

|数据|中国|美国|日本|
|---|---|---|---|
|日期|2026/11/23|11/23/2026|2026/11/23|
|数字|1,250.50|1,250.50|1,250.50|
|货币示例|¥1,250.50|$1,250.50|￥1,250|
|小数|1.25|1.25|1.25|
这里有了好几个不同概念，Locale涉及地区文化的格式，中国zh-CN，英文en-US，日本ja-JP。Locale和Timezone不能混为一谈，React中可以直接使用浏览器提供的Intl API：

```ts
const locale = 'zh-CN'

const dateFormatter = new Intl.DateTimeFormat(locale, {
  dateStyle: 'medium',
  timeStyle: 'short',
  timeZone: 'Asia/Shanghai'
})

const numberFormatter = new Intl.NumberFormat(locale, {
  maximumFractionDigits: 2
})

dateFormatter.format(new Date())
numberFormatter.format(1250.5)
```

Nuxt 中同样可以使用 `Intl`，也可以使用 Vue I18n 的日期、数字格式化能力：

```ts
const { d, n } = useI18n()

d(new Date(), 'short')
n(1250.5, 'decimal')
```

Nuxt 可以在配置中集中定义日期和数字格式。数据库中存储标准化的数据，例如 UTC 时间戳、数值和 ISO 货币代码，而不是直接存储格式化后的字符串。

