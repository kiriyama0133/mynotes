---
date: 2026-10-15
tags:
  - vue
  - emit
excalidraw-plugin: parsed
---

emit用于触发事件，当它在子组件里被调用的时候，就可以通知父组件，这样父组件监听到了事件就可以更新自己的状态

## 代码切入

举例来说，我这里有个dialog组件，这个dialog组件被包含在了一个父组件里，并且使用了emit来通知父组件，父组件使用了v-model来传递modelValue。


```ts
<template>
  <Teleport to="body">
    <div v-if="isOpen" class="overlay" @click="closeDialog">
      <div class="dialog" @click.stop>
        <VueUiSparkStackbar :dataset="store.chartDataset" :config="sparkConfig" />
        <UpdateHeatmap />
      </div>
    </div>
  </Teleport>
</template>

<script setup lang="ts">
import { VueUiSparkStackbar } from 'vue-data-ui'
import 'vue-data-ui/style.css'

const store = useContentStore()
const { t } = useI18n()

const props = defineProps<{
  modelValue: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: boolean]
}>()

const isOpen = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val)
})

// 父组件
<ClientOnly><Dialog v-model="showDialog" /></ClientOnly>
<button
  type="button"
  class="btn chart-btn"
  aria-haspopup="dialog"
  :aria-label="t('home.chartButtonAria')"
  @click="showDialog = true"
>
  <span class="chart-btn__icon" :style="chartIconStyle" aria-hidden="true" />
  <span class="chart-btn__label">{{ t('home.chartButton') }}</span>
</button>
```

我们知道，v-model是一个语法糖，展开为一个prop和一个事件监听。父组件这里showDialog通过v-model传入进去，v-model展开大致如下：

```vue
<Child
  :modelValue="show"
  @update:modelValue="show = $event"
/>
```

子组件**只能读** `props.modelValue`，不能直接赋值。直接写 `props.modelValue = true` 会报错。子组件不能直接改 props，只能**通知父组件**“我想改成 true”

```ts
const emit = defineEmits<{
  'update:modelValue': [value: boolean]
}>()

emit('update:modelValue', true)
```

我们可以使用一个computed来定义一个isOpen，这样v-if获取isOpen的时候就会触发延迟计算，触发get和set方法，这样就可以决定是访问props.modelValue还是触发set的emit方法了。我们点击了遮罩overlay之后就会调用closeDialog方法，然后closeDialog就会给isOpen.value赋值，vue当然知道isOpen是一个computed ref ，所以赋值的时候会调用set，这时候就是触发emit事件，通知父组件，父组件的@update:modelValue就会被触发，showDialog会变成false，然后重要的来了！之后父组件会重新传递值进去，get会被调用，computed又会调用props.modelValue，dialog组件关闭。

## 编译器产物

有没有想过v-model在编译后产物里长什么样子。

```vue
<script setup>
import { ref } from 'vue'
const showDialog = ref(false)
</script>

<template>
  <Dialog v-model="showDialog" />
</template>
```

编译为render方法之后大致如下：

```js
import { ref, createVNode, toDisplayString } from 'vue'
import Dialog from './Dialog.vue'

export default {
  setup() {
    const showDialog = ref(false)

    return () => {
      return createVNode(Dialog, {
        modelValue: showDialog.value,
        'onUpdate:modelValue': ($event) => (showDialog.value = $event)
      })
    }
  }
}
```

如果是多个v-model的情况就是：

```vue
<UserForm v-model="user" v-model:email="email" 
```

```js
h(UserForm, {
  modelValue: user.value,
  'onUpdate:modelValue': ($event) => (user.value = $event),
  email: email.value,
  'onUpdate:email': ($event) => (email.value = $event)
})
```


带修饰符版本：

```vue
<Dialog v-model.trim="name" />
```


```js
h(Dialog, {
  modelValue: name.value,
  'onUpdate:modelValue': ($event) => {
    name.value = $event.trim()   // .trim 修饰符
  }
})
```

**修饰符的处理**：

|修饰符|编译结果|
|---|---|
|`.trim`|`$event.trim()`|
|`.number`|`Number($event)`|
|`.lazy`|用 `onChange` 代替 `onInput`（原生元素）|

我们刚才的组件可能会展开为：

```vue
<script setup lang="ts">
const props = defineProps<{ modelValue: boolean }>()
const emit = defineEmits<{ 'update:modelValue': [value: boolean] }>()

const isOpen = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val),
})

const closeDialog = () => {
  isOpen.value = false
}
</script>

<template>
  <Teleport to="body">
    <div v-if="isOpen" class="overlay" @click="closeDialog">
      <div class="dialog" @click.stop>
        <slot />
      </div>
    </div>
  </Teleport>
</template>
```

```js
import { defineComponent, computed, createVNode, Teleport, withCtx } from 'vue'

const Dialog = defineComponent({
  name: 'Dialog',
  props: {
    modelValue: { type: Boolean, required: true }
  },
  emits: ['update:modelValue'],
  setup(props, { emit, slots }) {
    const isOpen = computed({
      get: () => props.modelValue,
      set: (val) => emit('update:modelValue', val)
    })

    const closeDialog = () => {
      isOpen.value = false
    }

    return () => {
      return isOpen.value
        ? createVNode(Teleport, { to: 'body' }, {
            default: withCtx(() => [
              createVNode('div', {
                class: 'overlay',
                onClick: closeDialog
              }, [
                createVNode('div', {
                  class: 'dialog',
                  onClick: (e) => e.stopPropagation()
                }, slots.default?.())
              ])
            ])
          })
        : null
    }
  }
})

export default Dialog
```
