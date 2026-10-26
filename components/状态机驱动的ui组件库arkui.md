---
date: 2026-12-07
tags:
  - components
  - vue
  - react
  - arkui
  - svelte
  - solid
  - nuxt
  - nextjs
---
Ark UI 引入了 Zag.js 状态机、跨框架 API 一致性、复合组件设计等概念，是一个全新的架构范式，不仅是组件库，更是一种管理复杂交互逻辑的底层思维，因此，单开一个笔记用于记载。

虽然这套逻辑和视觉状态机非常类似，就是说VSM，视觉状态管理，WPF早就实现了很多，已经玩过了，但是架构层级和职责边界上有着根本区别。这点稍微留意。

我们知道WPF里VSM的核心是ControlTemplate结合使用的，比如Normal、MouseOver、Pressed、Focusd状态，表现为负责视觉表现层的切换，控件的内部逻辑触发状态改变的时候，VSM负责用Storyboard去播放不同动画或者改变属性。但是VSM是强绑定在WPF/XAML引擎内部的，对于复杂键盘导航、屏幕阅读器无障碍、焦点陷阱等行为逻辑，需要开发者在后台写cs代码处理。

zag.js的设计来源于XState，它不仅管理视觉，核心是管理行为和交互的契约。它管理的是一个纯粹逻辑状态机，定义这个状态下按下Escape会发生什么，按下ArrowDown焦点应该去哪等等。它本身没有DOM，也不关动画，只是导出一个核心状态对象，当前的状态以data-state、data-docus这种Data属性的形式喂给HTML DOM，剩下的交给CSS去渲染。

![](https://ark-ui.com/images/architecture_dark.svg)

ArkUI通过数据属性来提供语义标签，比如 `data-scope="accordion"` 表示这是手风琴组件，`data-part="item"` 表示这是其中的列表项。这就让样式层可以在不依赖具体的 HTML 结构（比如 `div > ul > li`）的情况下精准定位。

 **`data-state`**：这是状态机直接映射到 DOM 上的结果。Zag.js 内部状态机算出当前是 `open` 还是 `closed`，就会直接反映为 `data-state="open"`。

## 组件组合

它解决了传统组件库一个非常恶心的痛点：**组件库自带的 HTML 标签太死板，但如果你自己写标签，又会丢失组件库内置的交互逻辑。**

asChild是一个很好用的属性，如果你想用一个弹窗触发器，你必须要用库提供的标签：

```vue
<Popover.Trigger>打开</Popover.Trigger>
<!-- 渲染出来可能是 <button>打开</button> -->
```

但如果你想要一个带有特殊路由跳转（如 `<RouterLink>`）或者复杂样式的自定义按钮，这套标签就不够用了。这时候可以用asChild：

```vue
<Popover.Trigger asChild>
  <MyCustomButton>打开</MyCustomButton>
</Popover.Trigger>
```

加上 `asChild` 后，`Popover.Trigger` 自己**不会渲染任何 DOM 元素**。它会把它内部包含的所有交互逻辑（`onClick` 事件、`aria-expanded` 等无障碍属性、键盘焦点管理）作为 Props，**全部注入到你提供的 `<MyCustomButton>` 上**,这不仅避免了多余的 DOM 嵌套，还让你百分百掌握了 DOM 结构和样式控制权。

官方文档里还提到了ark.span、ark.div，这些本身就是普通的标签，但是它们支持了asChild属性，当他嵌套a标签时候，ark会把自身属性直接合并给a，最终渲染出纯粹的a标签，这样就打破了组件库强加的标签限制，让你能在任何标签上复刻Ark UI的强大交互和可访问支持。

## 组件状态的控制

之前我们聊zag.js的时候，知道它管理复杂交互逻辑，但是作为开发者，当组件状态发生变化的时候，我们怎么去读取它，或者怎么从外部干预它。ArkUI提供了三种方案来应对不同业务的场景。

一种是使用Rendser Props，v-slot或者子函数，直接获取当前状态，适用于你只需要在当前组件内部，根据状态条件渲染一小块UI。官方文档里解析了，在 `<Avatar.Root>` 内部，用 `<Avatar.Context v-slot="avatar">` 拿到 `loaded` 状态。如果图片加载完了显示 "PA"，没加载完显示 "Loading"。

如下：

```vue
<script setup lang="ts">
import { Avatar } from '@ark-ui/vue/avatar'
import styles from './avatar.module.css'
</script>

<template>
  <Avatar.Root :class="styles.Root">
    <Avatar.Context v-slot="avatar">
      <Avatar.Fallback :class="styles.Fallback">
        {{ avatar.loaded ? 'PA' : 'Loading' }}
      </Avatar.Fallback>
    </Avatar.Context>
    <Avatar.Image :class="styles.Image" src="https://i.pravatar.cc/300?u=a" alt="avatar" />
  </Avatar.Root>
</template>
```

这像是一个局部的“监控探头”。缺点是会在 DOM 外层包一层 Context 组件，稍微有些嵌套感。

另一种是ARKUI导入useDialogContext或者useMenuContext等Hooks，在任意深度的子组件里调用，如果你想抽离出一个自定义的子组件，并且想要让他有改变父组件状态的能力，就可以使用这。**文档示例解析**：你写了一个 `<CustomCloseButton />`，把它放在 `<Dialog.Content>` 里面。在这个自定义按钮里，调用 `useDialogContext()` 拿到 `dialog.setOpen(false)`。点击就能关闭弹窗。

```jsx
import { Dialog, useDialogContext } from '@ark-ui/react/dialog'

function CustomCloseButton() {
  const dialog = useDialogContext()
  return <button onClick={() => dialog.setOpen(false)}>Close ({dialog.open ? 'open' : 'closed'})</button>
}

export const Demo = () => (
  <Dialog.Root>
    <Dialog.Trigger>Open</Dialog.Trigger>
    <Dialog.Content>
      <CustomCloseButton />
    </Dialog.Content>
  </Dialog.Root>
)
```

这是最推荐的做法。它彻底消灭了繁琐的 Render Props 嵌套（即不需要再写 `v-slot`），代码更干净。只要在 Context 树内，任何后代组件都能调用它。

还有一种模式，用 `useAccordion()`（或 `useDialog()`）在组件树**外部**初始化状态，然后用 `<Accordion.RootProvider>` 替换掉原本的 `<Accordion.Root>`，将刚刚的状态实例传进去。如果你需要从外部比如pinia中读取，甚至直接修改这个组件的状态，就可以使用：

```vue
<script setup lang="ts">
import { Accordion, useAccordion } from '@ark-ui/vue/accordion'
import { ChevronDownIcon } from 'lucide-vue-next'
import styles from './accordion.module.css'

const items = [
  {
    value: 'ark-ui',
    title: 'What is Ark UI?',
    content: 'A headless component library for building accessible web apps.',
  },
  {
    value: 'getting-started',
    title: 'How to get started?',
    content: 'Install the package and import the components you need.',
  },
  {
    value: 'maintainers',
    title: 'Who maintains this project?',
    content: 'Ark UI is maintained by the Chakra UI team.',
  },
]

const accordion = useAccordion({
  multiple: true,
  defaultValue: ['ark-ui'],
})
</script>

<template>
  <div class="stack">
    <output>Value: {{ JSON.stringify(accordion.value) }}</output>

    <Accordion.RootProvider :class="styles.Root" :value="accordion">
      <Accordion.Item v-for="item in items" :key="item.value" :class="styles.Item" :value="item.value">
        <Accordion.ItemTrigger :class="styles.ItemTrigger">
          {{ item.title }}
          <Accordion.ItemIndicator :class="styles.ItemIndicator">
            <ChevronDownIcon />
          </Accordion.ItemIndicator>
        </Accordion.ItemTrigger>
        <Accordion.ItemContent :class="styles.ItemContent">
          <div :class="styles.ItemBody">{{ item.content }}</div>
        </Accordion.ItemContent>
      </Accordion.Item>
    </Accordion.RootProvider>
  </div>
</template>
```

## 表单部分

表单一直是前端开发中最繁琐的部分：既有复杂的交互校验，又面临极其严苛的无障碍（A11y）要求，还要和第三方的状态管理库（如 React Hook Form）做集成。

在传统 UI 中，如果一个表单项被禁用了（`disabled`）或校验失败了（`invalid`），你需要把 `disabled` 或 `invalid` 属性传递给内部的 `Label`、`Input`、`HelperText` 甚至 `ErrorText`。

Ark UI 通过 `Field.Root` 解决了这个问题。只要你把任何 Ark 组件（哪怕是复杂的 `NumberInput`）放在 `<Field.Root disabled>` 里，**内部的组件会自动通过 Context 继承 `disabled`、`invalid`、`required` 和 `readOnly` 状态**。你不需要手动下发 props，这极大地减少了样板代码。

>在 Ark UI 中，各种表单组件可以通过上下文自动与 `Field` 集成。当这些组件被嵌套在 `Field.Root` 内部时，它们会自动继承 `disabled` 、 `invalid` 、 `required` 和 `readOnly` 的状态。


```ts
import { Field } from '@ark-ui/react/field'
import { NumberInput } from '@ark-ui/react/number-input'

const Demo = () => (
  <Field.Root disabled>
    <NumberInput.Root /> {/* NumberInput will be disabled */}
  </Field.Root>
)
```

在构建可访问的表单时，需要确保这些表单具有正确的标签和描述：

`Field.Label` ：用于为输入框提供一个可访问的标签。`Field.HelperText` ：用于提供关于输入的额外说明。这些组件通过 `aria-describedby` 属性自动与输入元素关联起来。

>最佳实践：确保标签是可见的（而不是仅作为占位符使用），这样屏幕阅读器能够正常读取这些标签内容。

```ts
import { Field } from '@ark-ui/react/field'

const Demo = () => (
  <form>
    <Field.Root>
      <Field.Label>Username</Field.Label>
      <Field.Input placeholder="Enter your username" />
      <Field.HelperText>This will be your public display name.</Field.HelperText>
    </Field.Root>
  </form>
)
```

当输入内容无效时，可以使用 `Field.ErrorText` 组件来显示错误信息，并将 `invalid` 属性传递给 `Field.Root` 组件。

>最佳实践：务必提供清晰、具体的错误信息，使得这些信息易于理解和修复。

```ts
import { Field } from '@ark-ui/react/field'

const Demo = () => (
  <form>
    <Field.Root invalid>
      <Field.Label>Username</Field.Label>
      <Field.Input placeholder="Enter your username" />
      <Field.ErrorText>Username is required.</Field.ErrorText>
    </Field.Root>
  </form>
)
```

如果某个字段是必填项，可以将 `required` 属性传递给 `Field.Root` 组件。或者，也可以使用 `Field.RequiredIndicator` 组件来标识该字段为必填项。

>最佳实践：不要仅通过颜色来指示所需的状态。

```ts
import { Field } from '@ark-ui/react/field'

export const Demo = () => (
  <form>
    <Field.Root required>
      <Field.Label>
        Username
        <Field.RequiredIndicator>(required)</Field.RequiredIndicator>
      </Field.Label>
      <Field.Input placeholder="Enter your username" />
      <Field.ErrorText>Username is required.</Field.ErrorText>
    </Field.Root>
  </form>
)
```

如果某个字段是可选的，可以在 `Field.RequiredIndicator` 组件上使用 `fallback` 属性来表示。

```html
<Field.RequiredIndicator fallback="Optional">(required)</Field.RequiredIndicator>
```

Field 支持原生 HTML 表单控件，包括 `input` 、 `textarea` 和 `select` ：

```jsx
import { Field } from '@ark-ui/react/field'

const Demo = () => (
  <form>
    {/* Input */}
    <Field.Root>
      <Field.Label>Email</Field.Label>
      <Field.Input type="email" placeholder="you@example.com" />
    </Field.Root>

    {/* Textarea */}
    <Field.Root>
      <Field.Label>Bio</Field.Label>
      <Field.Textarea placeholder="Tell us about yourself" />
    </Field.Root>

    {/* Select */}
    <Field.Root>
      <Field.Label>Country</Field.Label>
      <Field.Select>
        <option value="us">United States</option>
        <option value="uk">United Kingdom</option>
        <option value="de">Germany</option>
      </Field.Select>
    </Field.Root>
  </form>
)
```

当在表单或组件中有一系列字段，且这些字段包含多个 `input` 元素时，可以使用 `Fieldset` 组件将它们组合在一起。常见的使用场景包括：复选框组、单选框组、输入框与选择框的组合使用等。

```jsx
import { Fieldset } from '@ark-ui/react/fieldset'
import { Checkbox } from '@ark-ui/react/checkbox'
import { CheckIcon } from 'lucide-react'

const items = [
  { label: 'React', value: 'react' },
  { label: 'Solid', value: 'solid' },
  { label: 'Vue', value: 'vue' },
  { label: 'Svelte', value: 'svelte' },
]

const Demo = () => (
  <Fieldset.Root>
    <Fieldset.Legend>Frameworks</Fieldset.Legend>
    <Checkbox.Group name="framework">
      {items.map((item) => (
        <Checkbox.Root value={item.value} key={item.value} />
      ))}
    </Checkbox.Group>
    <Fieldset.HelperText>Choose your preferred frameworks</Fieldset.HelperText>
  </Fieldset.Root>
)
```

