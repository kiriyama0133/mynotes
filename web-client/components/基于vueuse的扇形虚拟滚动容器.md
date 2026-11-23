---
date: 2026-05-10
tags:
  - vue
  - components
---


在开发Nuxt Content博客的时候，我注意到这个扇形滚动容器实际上是很实用的，至少是数学层面上，对我这个数学白痴来说，过了这个路基本不可能再写出来，所以决定写下这个笔记来记录一下扇形容器和虚拟滚动容器。

一般虚拟滚动容器需要拆分为4个问题：

1. 总内容高度是多少
2. 当前滚动到了哪里
3. 哪些item进入了可视区域
4. 这些item应该放在什么位置


```
10000 个 item
每个 item 高度 50px

总高度：

10000 × 50 = 500000px

但实际上浏览器不需要创建10000个DOM
```

假设viewport高度为600px，那么理论上只需要600/ 15 = 20个，再加上一些上下缓冲，例如：buffer = 5，最后可能只会渲染22个item。

## 核心数学关系

我们假设itemHeight高度为50，scrollTop为2370，那么第一个进入viewport附近的item是：
`floor(scrollTop / itemHeight)` ，差不多就是47，如果viewport高度是600，那么可视的item数量就是12个。如果考虑上buffer，那么:

```
renderStart = max(0, 47 - 5)
            = 42

renderEnd = 47 + 12 + 5
          = 64
          
所以真正从数据里取

items[42 ... 64]

```

这样的话浏览器会认为容器就只有23 x 50这样高度。用户就无法滚到后面的item，虚拟滚动容器就会制造一个虚拟内容高度，结构类似于：

```html
<div class="viewport">

    <div class="content">
        <!-- 实际只放 23 个 item -->
    </div>

</div>
```

content设置height500000px都没有问题，这样scroll会认为10000x50是真实存在的，然后真正的item会通过 transform去移动到正确位置。

## Vuseuse提供的方法


vueuse这里提供了一个很好用的方法，这是我们组件的基底：`useVirtualList()`，这是vueuse提供的一套scroll position -> virtual items响应式计算逻辑。

```ts
const {
  list,
  containerProps,
  wrapperProps
} = useVirtualList(...)

VueUse 常见写法大概是：

const {
  list,
  containerProps,
  wrapperProps,
} = useVirtualList(items, {
  itemHeight: 50,
  overscan: 5,
})
```

list是当前应该渲染的数据，containerProps是外层滚动容器应该具备的属性，wrapperProps是内层虚拟容器应该具备的属性。


```vue
<div v-bind="containerProps">

  <div v-bind="wrapperProps">

    <!-- 实际只有几十个 item -->

  </div>

</div>
```

## 组件设计

我的虚拟滚动容器是一个扇形滚动容器，但是比较好地分离了扇形逻辑和虚拟滚动组件容器，所以组合起来使用还是比较方便地。


```ts
/** 页面滚动进度 0~1 */
const pageProgress = computed(() => {
  if (typeof document === 'undefined') return 0
  // viewportH 是响应式的，窗口缩放会触发重算；scrollHeight 每次现读，
  // 页面内容变高后（懒加载图片、字体回填）下一次滚动就会自动修正。
  const max = document.documentElement.scrollHeight - viewportH.value
  if (max <= 0) return 0
  return Math.min(1, Math.max(0, windowY.value / max))
})

/** 整程页面对应列表里的多少个条目（0 = 整份列表） */
const span = computed(() =>
  props.windowSpan > 0 ? props.windowSpan : Math.max(0, props.items.length - 1),
)
```

我们通过scrollHeight - 视口高度值就可以得到滚动进度，这个是个很常见地表达。windowSpan不是dom窗口宽度而是整个页面的滚动距离，对应列表中的多少个条目，假设items有100个，windowSpan是20个，那么对照起来就是这样：

```
页面位置             列表 progress

顶部       0%   →     0
          25%   →     5
          50%   →     10
          75%   →     15
底部      100%  →     20
```

通过span，我们就可以根据pageProgress的进度来**换算前位索引**：

```ts
/** 由页面进度直接换算出的「前位」索引 —— 绝对值，不是增量 */
const windowIndex = computed(() => pageProgress.value * span.value)
/**
 * 窗口模式下把轨道同步到前位：直接写绝对值（不是增量）。
 * useVirtualList 只认 container 的 scrollTop，同步过去它才取得到正确的窗口。
 * 依据 CSSWG css-overflow：overflow:hidden 的滚动容器「滚轮不滚、脚本可滚」。
 */
watchEffect(() => {
  if (props.driver !== 'window') return
  const el = scrollerRef.value
  if (!el) return
  const next = windowIndex.value * props.itemHeight
  if (Math.abs(el.scrollTop - next) > 0.5) el.scrollTop = next
})
```

这里有两种驱动模式，一个是跟随window的进度，一个是像是普通的容器那样跟随鼠标滚轮滑动。这里主要看第二种，我们通过scrollTop的高度 / item的高度就可以得到个数，也就是索引。这里的观察者模式用于把scrollerRef的scrollTop强制同步到windowIndex所代表的位置。


```ts
/**
 * 容器 props。窗口模式下换成 overflow:hidden：
 * 它仍然是可以被脚本写 scrollTop 的滚动容器（"用户不可滚、脚本可滚"），
 * 所以页面滚动能独占这条轨道。
 *
 * 注意 overscroll-behavior 必须跟着一起放开：实测（Chrome）
 * 「overflow:hidden + overscroll-behavior:contain」会把滚轮整块吞掉 ——
 * 轨道不滚、页面也不滚，光标停在扇形上就彻底卡住。
 */
const scrollerProps = computed(() => {
  const { style, ...rest } = containerProps
  const base = (typeof style === 'object' && style !== null ? style : {}) as Record<string, unknown>
  const windowed = props.driver === 'window'
  return {
    ...rest,
    style: {
      ...base,
      overflowY: windowed ? 'hidden' : 'auto',
      overscrollBehavior: windowed ? 'auto' : 'contain',
    },
  }
})
/**
 * 轨道 scrollTop 的上限 = 总高 − 容器高，不补留白的话末尾
 * 「容器高 / itemHeight」条永远到不了前位。
 */
const endPadding = computed(() => {
  if (typeof props.endPadding === 'number') return Math.max(0, props.endPadding)
  if (props.endPadding === 'container') return boxH.value
  return props.driver === 'window' ? boxH.value : 0
})
```

同一个scroller再self模式下应该让用户滚动，window模式下不让用户滚动，和上一段代码结合起来看就知道这样设计的目的，window滚动以后，内部的scroller的位置也要同步过去并禁止用户直接滚动内部scroller。

```ts
function scrollToIndex(index: number, behavior: ScrollBehavior = 'auto') {
  if (behavior === 'auto') {
    scrollTo(index)
    return
  }
  const el = scrollerRef.value
  if (!el) return
  el.scrollTo({ top: index * props.itemHeight, behavior })
}
```

这端函数就是这个组件对外暴露的滚动到第几个item的封装，根据behavoir分为了两个模式，index是列表索引，behavoir是滚动方式，auto模式会走scrollTo，这个是通过useVirtualLsit得到的，这里可以直接复用它提供的滚动方法，这样我们就不用自己重复实现vueuse已经提供的index->scrollTo的转换了。那么非auto需要自己调用el.scrollTo，原生DOM提供了浏览器的平滑滚动的能力。

## 扇形组件

到刚才那一步，FanVirtualList的实现机制也就差不多了，接下来该看useFanArc.ts了，扇形容器十分依赖这个方法：

```ts
export function useFanArc(options: FanArcOptions) {
  /** 每条跨越的圆心角（弧度）—— 「固定角度」就来自这里 */
  const stepRad = computed(() => toValue(options.itemHeight) / toValue(options.radius))
  const stepDeg = computed(() => (stepRad.value * 180) / Math.PI)
  const maxAngleRad = computed(() => (toValue(options.maxAngle) * Math.PI) / 180)
```

这里面的三个属性主要是解决相邻item再圆弧上差的角度，扇形最多允许张开多少角度。stepRad是一个item占的弧度，这里可以用园弧长公式：s=rθ 所以：θ=rs。这里s = itemHeight，r=radius，所以可以得到stepRad。​stepDeg可以把弧度转换成角度，css中rotate使用的是deg，数学计算中sin/cos使用的正是弧度。

```
                 radius
                   │
                   ↓
itemHeight ───→ stepRad ───→ 每个 item 的角度间隔
                                  │
                                  ↓
                              angleOf(index)
                                  │
                                  ↓
                                theta
                                  │
                    ┌─────────────┴─────────────┐
                    ↓                           ↓
                 sin/cos                    maxAngleRad
                    ↓                           ↓
                  x / y                     visible
```

然后

```ts
function angleOf(index: number) {
  return (toValue(options.progress) - index) * stepRad.value
}
  function distanceOf(index: number) {
    const max = maxAngleRad.value
    if (max <= 0) return 1
    return Math.min(1, Math.abs(angleOf(index)) / max)
  }
```

这里实际上就是通过当前item距离前卫几个位置 x 每个位置对应的弧度，就可以得到角度了。distanceof计算某个item离前卫有多远，并把这个距离归一化成0~1。

到这里，还需要一个真正把角度换算成屏幕上x/y左边的方法：

```ts
  function positionOf(index: number): FanArcPosition {
    const theta = angleOf(index)
    const r = toValue(options.radius)
    // 圆心在「锚点向右 r」处：theta = 0 时条目正好落在锚点，向两端张开并轻微右凸
    const x = toValue(options.anchorX) + r * (1 - Math.cos(theta))
    const y = toValue(options.anchorY) - r * Math.sin(theta)
    const thetaDeg = (theta * 180) / Math.PI
    const orientation = toValue(options.orientation)
    const rot =
      orientation === 'upright' ? 0 : orientation === 'radial' ? thetaDeg - 90 : thetaDeg
    return {
      x,
      y,
      rot,
      theta,
      thetaDeg,
      visible: Math.abs(theta) <= maxAngleRad.value,
      distance: distanceOf(index),
    }
  }
```

拿到item角度之后取半径，这里的r是圆的半径，anchorX和anchorY不是圆心：

```
假设
anchorX = 100
anchorY = 300
radius = 200

那么圆心就在：
(100 + 200, 300)
= (300, 300)

                    圆心
                     ●
                     │
                     │ 200
                     │
前位锚点              │
   ●─────────────────●
 (100,300)          (300,300)
```


然后计算得到x和y，transformOf方法会将x和y取出来，这里是x和y第一次被使用：

```ts
function transformOf(index: number) {
  const { x, y, rot } = positionOf(index)

  return `translate3d(${x.toFixed(2)}px, ${y.toFixed(2)}px, 0) translate(-50%, -50%) rotate(${rot.toFixed(2)}deg)`
}

function itemStyle(index: number) {
  const pos = positionOf(index)
  if (!pos.visible) return { display: 'none' }
  return {
    transform: transformOf(index),
    opacity: String(1 - toValue(options.fade) * pos.distance),
    zIndex: String(1000 - Math.round(Math.abs(pos.thetaDeg))),
  }
}
像
x = 126.79
y = 200
rot = 30
最后生成的css是
transform:
  translate3d(126.79px, 200px, 0)
  translate(-50%, -50%)
  rotate(30deg);
```

itemStyle又调用transformOf，const pos...这里又拿了一次x/y，后面的transform: transforOf(index)又拿了一次，所以实际上同一个index 的positionOf会算两次，两次的结果应该是一样的，因为输入没有在中间改变。

```vue
<template>
  <div ref="root" class="fan-list" :data-driver="driver">
    <div v-bind="scrollerProps" class="fan-list__scroller">
      <!-- 视觉层：sticky 钉在滚动视口上，条目用 transform 摆到弧上 -->
      <div class="fan-list__stage" :style="{ height: `${boxH}px` }">
        <div
          v-for="entry in list"
          :key="entry.index"
          class="fan-list__item"
          :style="arc.itemStyle(entry.index)"
        >
          <slot
            :item="entry.data"
            :index="entry.index"
            :active="Math.abs(progress - entry.index) < 0.5"
            :distance="arc.distanceOf(entry.index)"
          />
        </div>
      </div>
      <!-- 占位层：只负责撑出滚动高度（vueuse 自己维护 marginTop / height） -->
      <div v-bind="wrapperProps" />
      <!--
        末尾留白：让最后的条目也能滚到前位（见脚本里 endPadding 的注释）。
      -->
      <div
        v-if="endPadding > 0"
        class="fan-list__tail"
        :style="{ height: `${endPadding}px` }"
      />
    </div>
  </div>
</template>

```

到这里我们就已经实现了一个扇形的容器，而且是一个带虚拟滚动的扇形容器。END。
食用方法如下：

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useContentStore } from '~/stores/content'
import FanVirtualList from '~/components/FanVirtualList.vue'
import RightRail from '~/components/RightRail.vue'
import { canonicalPath } from '~/utils/path'
const store = useContentStore()
const route = useRoute()
const items = computed(() => store.summaries)
const currentPath = computed(() => canonicalPath(route.path))

function isActive(path: string) {
  return currentPath.value === path
}
</script>

<template>
  <RightRail>
    <FanVirtualList
      class="fan"
      :items="items"
      driver="window"
      :item-height="56"
      :radius="520"
      :max-angle="40"
      :fade="0.85"
      orientation="tangent"
      :anchor-x="0.62"
      :anchor-y="0.5"
    >
      <template #default="{ item, active }">
        <NuxtLink
          :to="item.path"
          class="doc-link hidden truncate md:block"
          :class="{ 'doc-link--front': active, 'doc-link--active': isActive(item.path) }"
        >
          {{ item.title }}
        </NuxtLink>
      </template>
    </FanVirtualList>
  </RightRail>
</template>

<style lang="scss" scoped>
.fan {
  height: min(100vh, 34rem);
}
.doc-link {
  width: 8rem;
  padding: 0.45rem 0.75rem;
  border: 1px solid transparent;
  border-radius: 0.5rem;
  font-size: 0.8125rem;
  line-height: 1.15rem;
  text-decoration: none;
  box-shadow: 0 0 0 0 transparent;
}
.doc-link--front {
  box-shadow: 0 6px 18px -8px rgb(0 0 0 / 0.35);
}
.doc-link--active {
  font-weight: 600;
}
</style>
```
