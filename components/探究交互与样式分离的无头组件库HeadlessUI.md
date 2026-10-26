---
date: 2026-12-07
tags:
  - headlessui
  - components
  - css
  - vue
  - react
---
样式与交互分离的思想是现代前端架构演进的一次解耦运动，我们之前使用的primevue、element ui 、ant desigen等等其实都是成品组件，它们把样式和交互死死绑在了一起。当我们需要修改部分样式的时候，就不得不使用!import来实现。

就算是primevue也只有一个:deep的这种方案，这可以让我们穿透作用域去修改子组件包括primevue渲染出的dom的样式，这可以实现深度定制，但是一些组件比如toast 、dialog、dropdown等组件通过appendTo属性将dom渲染到body末尾的时候，这些元素已经不在vue组件的dom子树内了，:depp就无法触及了。还有就是primevue的:pt，它通过 `:pt="{ icon: { class: 'mr-2' } }"` 这样的配置，让你直接将 Tailwind 类名或内联样式“穿透”到组件的内部特定部位（如图标）。**它能完美解决 `:deep()` 对“传送门”组件无效的问题**。

但是即便是这样，也是无法完全实现交互和样式分离的，只是说这样是一个这种方案，能够让开发者深度定制样式。

所以，我们来谈谈一个完全符合样式与交互分离的组件库headless ui。这个是tailwindcss官方强推的组件库，很建议学习。现在是2026年，tailwind在前端的占量已经越来越大了，bootstrap存量大，但是在新项目里使用的占比已经很小了，可能是因为ai agent的原因，也是变相地推动了使用tailwnd的项目的数量持续上升，所以这个场景下我们学习和tailwind配套的headlessui还是一个比较切实际的方案。

## headlessui本质

headlessui的实现本质是一个交互逻辑和视觉表现彻底分离的行为引擎，本身不渲染任何html和css样式，而是提供了一套机制来负责处理组件的状态、交互和无障碍性。

他用的是复合组件模式，通过上下文的传递来管理状态，最典型的就是使用 `<Tab.Group>` 包裹 `<Tab.List>` 和 `<Tab.Panels>` 的结构，父组件 `Tab.Group` 维护当前激活的 Tab 索引，子组件通过 Context 消费这个状态，从而实现点击 `Tab` 切换对应 `Tab.Panel` 的基础逻辑。

你看官方的tabs的code展示：

```vue
<template>
  <div class="w-full max-w-md px-2 py-16 sm:px-0">
    <TabGroup>
      <TabList class="flex space-x-1 rounded-xl bg-blue-900/20 p-1">
        <Tab
          v-for="category in Object.keys(categories)"
          as="template"
          :key="category"
          v-slot="{ selected }"
        >
          <button
            :class="[
              'w-full rounded-lg py-2.5 text-sm font-medium leading-5',
              'ring-white/60 ring-offset-2 ring-offset-blue-400 focus:outline-none focus:ring-2',
              selected
                ? 'bg-white text-blue-700 shadow'
                : 'text-blue-100 hover:bg-white/[0.12] hover:text-white',
            ]"
          >
            {{ category }}
          </button>
        </Tab>
      </TabList>

      <TabPanels class="mt-2">
        <TabPanel
          v-for="(posts, idx) in Object.values(categories)"
          :key="idx"
          :class="[
            'rounded-xl bg-white p-3',
            'ring-white/60 ring-offset-2 ring-offset-blue-400 focus:outline-none focus:ring-2',
          ]"
        >
          <ul>
            <li
              v-for="post in posts"
              :key="post.id"
              class="relative rounded-md p-3 hover:bg-gray-100"
            >
              <h3 class="text-sm font-medium leading-5">
                {{ post.title }}
              </h3>

              <ul
                class="mt-1 flex space-x-1 text-xs font-normal leading-4 text-gray-500"
              >
                <li>{{ post.date }}</li>
                <li>&middot;</li>
                <li>{{ post.commentCount }} comments</li>
                <li>&middot;</li>
                <li>{{ post.shareCount }} shares</li>
              </ul>

              <a
                href="#"
                :class="[
                  'absolute inset-0 rounded-md',
                  'ring-blue-400 focus:z-10 focus:outline-none focus:ring-2',
                ]"
              />
            </li>
          </ul>
        </TabPanel>
      </TabPanels>
    </TabGroup>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { TabGroup, TabList, Tab, TabPanels, TabPanel } from '@headlessui/vue'

const categories = ref({
  Recent: [
    {
      id: 1,
      title: 'Does drinking coffee make you smarter?',
      date: '5h ago',
      commentCount: 5,
      shareCount: 2,
    },
    {
      id: 2,
      title: "So you've bought coffee... now what?",
      date: '2h ago',
      commentCount: 3,
      shareCount: 2,
    },
  ],
  Popular: [
    {
      id: 1,
      title: 'Is tech making coffee better or worse?',
      date: 'Jan 7',
      commentCount: 29,
      shareCount: 16,
    },
    {
      id: 2,
      title: 'The most innovative things happening in coffee',
      date: 'Mar 19',
      commentCount: 24,
      shareCount: 12,
    },
  ],
  Trending: [
    {
      id: 1,
      title: 'Ask Me Anything: 10 answers to your questions about coffee',
      date: '2d ago',
      commentCount: 9,
      shareCount: 5,
    },
    {
      id: 2,
      title: "The worst advice we've ever heard about coffee",
      date: '4d ago',
      commentCount: 1,
      shareCount: 2,
    },
  ],
})
</script>
```

TapGroup是最外层，负责维护哪个Tab被选中，TabList和Tab负责渲染按钮，TabPanels和TabPanel负责渲染对应的内容区域，它们之间通过Vue的provide/inject共享状态，你不需要手动编写任何v-model来绑定高亮逻辑。

里面使用了 **`as="template"`**：让 Headless UI：“不参与渲染任何多余的 `<div>` 或 `<button>` 标签，只需要提供逻辑”。然后它把渲染控制权交还给开发者（我们在里面写了原生的 `<button>`）。

`v-slot="{ selected }"`：这就是 **Render Props**。Headless UI 在底层算出了当前这个 Tab 到底有没有被选中（`selected` 是 true 还是 false），把它暴露给你。它只提供状态，至于选中了是变蓝还是加粗，它完全不管。下面就可以看到selected的参与，随后我们用tailwind来接管样式：

```ts
:class="[
  'w-full rounded-lg py-2.5 ...',
  selected ? 'bg-white text-blue-700 shadow' : 'text-blue-100 hover:bg-white/[0.12]'
]"
```

Headless UI 没有给你提供任何 `.tab-active` 或 `.tab-inactive` 这种自带的 CSS 类名。所有的视觉效果，完全是由你通过 Tailwind 驱动的 `selected` 布尔值拼接出来的。

这段代码使用了 `v-slot="{ selected }"`，这是 Headless UI **v1 版本**的经典写法（Render Props）。如果是v2版本的话，官方写法会比较简洁，我们会使用data属性来完成，类似下面：

```html
<Tab v-for="category in Object.keys(categories)" :key="category"
     class="w-full rounded-lg py-2.5 ... data-[selected]:bg-white data-[selected]:text-blue-700 data-[selected]:shadow">
  {{ category }}
</Tab>
```

现在最尴尬的事情是headlessui目前生态不对称，截止到2026/10/26，react版本已经迭代到了2.2版本，但是vue版本现在依旧停留在了1.7，这种版本差异是有明确的技术和规划原因的。由 **Tailwind Labs 团队的资源优先级**决定的，核心团队的精力投入到了tailwindcss v4的研发上，headlessui的作者adam wathan表示团队年内没有明确的vue v2开发路线图，因为安排已经满了.....

此外，React v2 底层依赖了 `@react-aria/focus` 和 `@react-aria/interactions` 这些来自 Adobe 的库来处理焦点和交互。在 Vue 生态中，**可能还没有完全对等的替代方案**，这也构成了技术迁移上的一个实际障碍。

## v2的更新

React v2 是一次重大的架构重写，带来了许多 v1.7（Vue版）不具备的现代化特性：[](https://illegalscreed.cn/zh/frontend-framework/components/headless-ui/)

- **从 Render Props 转向 Data 属性**：这是最核心的体验提升。v2 不再需要 `v-slot` 来获取状态，而是直接在 DOM 上暴露 `data-open`、`data-selected` 等属性，样式完全由 CSS（如 `data-[selected]:`）驱动，**不会触发不必要的组件重渲染**，性能更优。
    
- **内置定位引擎**：v2 集成了 `anchor` 属性（底层基于 Floating UI），用于处理下拉菜单、弹出层的自动翻转和边界碰撞检测，无需再手动计算位置。[](https://illegalscreed.cn/zh/frontend-framework/components/headless-ui/)
    
- **简化的过渡动画**：v2.1 引入了全新的 `transition` 属性，结合 `data-closed`、`data-enter`、`data-leave` 等阶段属性，只需添加几个 CSS 类就能实现复杂的进出场动画。[](https://www.tailwindcss.cn/blog/2024-06-21-headless-ui-v2-1)
    
- **更丰富的表单组件**：v2 新增了 `Checkbox`、`Input`、`Select`、`Textarea` 等一系列原生表单组件，并自动处理 `aria-describedby`、`id` 等无障碍关联，扩展了组件库的适用范围。

## 其余类似的库

例如Ark UI，支持vue，基于状态机，功能比较强大，或者Primevue的unstyled模式，它们的设计理念和headlessui v2比较接近。

AruUI核心思路和headless UI类似，但是依赖的基础库更加重，也更加灵活和强大。它基于zag.js状态机，arkui的交互由zag.js驱动，这是框架无关的有限状态机库，所以组件的所有状态比如打开/关闭、选中/未选中和状态抓换都由一个独立的、可预测的状态机管理，而不是散落在组件内部，这带来了极高的测试和逻辑上的严谨。

并且得益于状态机的抽象层，arkui可以给react、vue、solid、svelte提供完全相同的api和交互行为，如果开发团队涉及到了多个框架，那么Ark UI是可以极大降级维护成本的。生产级别的组件有45+个，功能覆盖面很广，包括headlessUI所有没有的data picker、color picker、carousel等复杂组件，ark ui 本身是chakra ui v3的底层基础，定位非常清晰。

primevue unstyled模式是让原本功能齐全的primevue组件卸载css样式，如果你安装的时候通过unstyled: true来全局开启，或者可以给单个组件添加unstyled属性来局部启用。pass through属性是unstyled模式下的关键，组件本身由于不包含像是，但是可以通过:pt来向组件内部传递特定DOM节点，比如root label icon传递class style或者事件处理器。

