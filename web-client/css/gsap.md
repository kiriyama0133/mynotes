---
date: 2026-09-02
tags:
  - css
  - gsap
---
截至2026年，**GSAP的所有模块（包括以前收费的会员专属插件）现在都完全免费了**，可以免费用于商业项目。

::gsap-scramble
::

GSAP 被 Webflow 收购后，取消了原来的付费会员（Club GSAP）制度。所有曾经需要付费订阅才能使用的“奖励插件”（Bonus Plugins），现在都已向所有人开放，不再需要授权令牌或私有仓库权限。

以下模块目前**全部免费**，包括但不限于：

- **核心库**：GSAP Core。
    
- **曾经收费的会员插件**：**SplitText**（文字拆分）、**MorphSVG**（形状变形）、**DrawSVG**（SVG 描边）、**ScrambleText**、**Physics2D** 等。
    
- **常用免费插件**：**ScrollTrigger**、**ScrollSmoother**、**Flip**、**Draggable**、**Observer**、**MotionPath**、**CustomEase** 等。

## 学习路径

学习 GSAP 的核心 API，可以按“创建动画 → 组织序列 → 控制与优化”这条主线来掌握。以下是必须熟悉的几个部分

我们最常用的是gsap.to / gasp.from / gsap.fromTo，这三个api可以实现： 让元素从当前状态过渡到制定状态、让元素从指定状态过渡到当前状态、显示指定起点和终点，适合状态切换或避免初始闪烁。以及还需要学习时间线，动画需要按顺序或者重叠执行时，gsap.timeline是信合工具，是一个容器来精确编排tween。

当我们把to/timeline用熟悉，能够用ease、stagger来提升动画的观感之后，再考虑在做响应式或者框架项目的深入。



## 例子

### fade

::gsap-box{animation="fade" :duration="1.2" :delay="0.3"}
这段内容会被 GSAP 动画包裹，并且是fade入场的
::

这段代码在MDC语法渲染之后，等价于Vue模板里的：

```vue
相当于：
<template>
  <div ref="containerRef" class="gsap-box">
    <div class="gsap-controls">
      <button @click="play" :disabled="isPlaying" title="播放">▶</button>
      <button @click="pause" :disabled="!isPlaying" title="暂停">⏸</button>
      <button @click="restart" title="重播">⟲</button>
    </div>
    <div class="animate-target">
      这段内容会被 GSAP 动画包裹。
    </div>
  </div>
</template>

// props 解析后
animation = 'fade'
duration = 1.2
delay = 0.3

// onMounted 里
tl = gsap.timeline({ paused: true, onComplete: ... })

tl.from(containerRef.value.querySelector('.animate-target'), {
  opacity: 0,
  y: 30,
  duration: 1.2,
  delay: 0.3,
  ease: 'power2.out'
})

tl.play()

```

然后我们看看旋转入场的动画：

### rotate

::gsap-box{animation="rotate" :duration="1.2" :delay="0.3"}
这段内容会被 GSAP 动画包裹，并且是rotate入场的
::
在我们的GSAP包装组件里，他使用的GSAP为rotation方法，利用from变成了现在的状态：

```ts
if (props.animation === 'rotate') {
  tl.from(target, {
    rotation: -15,
    opacity: 0,
    duration: props.duration,
    delay: props.delay,
    ease: 'power2.out'
  })
}
```

然后是可拖拽内容：
::gsap-box{animation="drag"}
<div class="drag-me">拖我</div>
::
这个可拖拽的内容我们一般会这样使用：

```ts
case 'drag':
  // 拖拽不依赖 timeline 播放，单独初始化
  draggable = Draggable.create(target, {
	type: 'x,y',
	bounds: containerRef.value,
	inertia: false
  })
  // 拖拽模式没有入场动画，直接标记为“已播放”状态
  isPlaying.value = true
  return // 跳过下面的 tl.play()
```


### splitText

::gsap-box{animation="chars" :duration="0.5" :delay="0.2"}
这段文字会被 SplitText 拆成字符后逐个出现。
::

逐个出现的字符我们会这样使用：

```ts
case 'chars':
split = new SplitText(target, { type: 'chars' })
console.log('chars count:', split.chars.length) // 必须 > 0
tl.from(split.chars, {
	y: 40, opacity: 0,
	duration: 0.5, stagger: 0.03,
	delay: props.delay, ease: props.ease
})
break
```

### stagger

然后就是可以逐个延迟出现的导航项，我们经常这样使用来美化我们的页面，增添更多的动效，这里我们使用的是back.out(1.7)的弹簧振子效果。不要点，点了就是404（
::gsap-box{animation="stagger" :duration="0.4" ease="back.out(1.7)"}
<div style="display: flex; flex-direction: column; gap: 2px; max-width: 260px;">
  <a class="nav-item" href="/" style="display: flex; align-items: center; gap: 10px; padding: 10px 14px; border-radius: 8px; text-decoration: none; color: #1f2937; font-size: 14px; font-weight: 500; background: #eef2ff; transition: background 0.15s;">
    <span style="width: 18px; height: 18px; border-radius: 4px; background: #6366f1; flex-shrink: 0;"></span>
    首页
  </a>
  <a class="nav-item" href="/docs" style="display: flex; align-items: center; gap: 10px; padding: 10px 14px; border-radius: 8px; text-decoration: none; color: #374151; font-size: 14px; transition: background 0.15s;">
    <span style="width: 18px; height: 18px; border-radius: 4px; background: #a5b4fc; flex-shrink: 0;"></span>
    文档
  </a>
  <a class="nav-item" href="/guide" style="display: flex; align-items: center; gap: 10px; padding: 10px 14px; border-radius: 8px; text-decoration: none; color: #374151; font-size: 14px; transition: background 0.15s;">
    <span style="width: 18px; height: 18px; border-radius: 4px; background: #c7d2fe; flex-shrink: 0;"></span>
    指南
  </a>
  <a class="nav-item" href="/components" style="display: flex; align-items: center; gap: 10px; padding: 10px 14px; border-radius: 8px; text-decoration: none; color: #374151; font-size: 14px; transition: background 0.15s;">
    <span style="width: 18px; height: 18px; border-radius: 4px; background: #ddd6fe; flex-shrink: 0;"></span>
    组件
  </a>
  <a class="nav-item" href="/plugins" style="display: flex; align-items: center; gap: 10px; padding: 10px 14px; border-radius: 8px; text-decoration: none; color: #374151; font-size: 14px; transition: background 0.15s;">
    <span style="width: 18px; height: 18px; border-radius: 4px; background: #e9d5ff; flex-shrink: 0;"></span>
    插件
  </a>
  <a class="nav-item" href="/blog" style="display: flex; align-items: center; gap: 10px; padding: 10px 14px; border-radius: 8px; text-decoration: none; color: #374151; font-size: 14px; transition: background 0.15s;">
    <span style="width: 18px; height: 18px; border-radius: 4px; background: #fbcfe8; flex-shrink: 0;"></span>
    博客
  </a>
  <a class="nav-item" href="/about" style="display: flex; align-items: center; gap: 10px; padding: 10px 14px; border-radius: 8px; text-decoration: none; color: #374151; font-size: 14px; transition: background 0.15s;">
    <span style="width: 18px; height: 18px; border-radius: 4px; background: #fecaca; flex-shrink: 0;"></span>
    关于
  </a>
</div>
::
我们在平常的开发中，这样来调用：

```ts
case 'stagger':
  const items = target.querySelectorAll('.nav-item')
  console.log('stagger items:', items.length)
  tl.from(items, {
    x: -50,
    opacity: 0,
    duration: 0.4,
    stagger: 0.1,
    ease: props.ease
  })
  break
```

### flip

这是最值得加的一个。Flip 能记录元素当前状态，在布局改变后自动计算位置差异，并让元素从旧位置“飞”到新位置。适合的场景：卡片从网格布局切换到列表布局、图片从缩略图放大到详情、导航栏在滚动时收缩变形。

::gsap-box{animation="flip"}
<div class="nav-list">
  <a class="nav-item" href="/">首页</a>
  <a class="nav-item" href="/docs">文档</a>
  <a class="nav-item" href="/guide">指南</a>
  <a class="nav-item" href="/components">组件</a>
  <a class="nav-item" href="/plugins">插件</a>
</div>
::

代码实现大致如下：

```ts
<div class="nav-list">
  <a class="nav-item" href="/">首页</a>
  <a class="nav-item" href="/docs">文档</a>
  <a class="nav-item" href="/guide">指南</a>
  <a class="nav-item" href="/components">组件</a>
  <a class="nav-item" href="/plugins">插件</a>
</div>

const isHorizontal = ref(false)
let flipState = null
case 'flip': {
	const list = target.querySelector('.nav-list')
	if (list) {
		list.style.display = 'flex'
		list.style.flexDirection = 'column'
		list.style.gap = '4px'
	}
	isPlaying.value = true
	return
}

function toggleFlipLayout() {
  const target = containerRef.value?.querySelector('.animate-target')
  const items = target?.querySelectorAll('.nav-item')
  const list = target?.querySelector('.nav-list')
  if (!items?.length || !list) return

  // 1. 记录当前状态
  flipState = Flip.getState(items)

  // 2. 切换布局方向
  isHorizontal.value = !isHorizontal.value
  list.style.flexDirection = isHorizontal.value ? 'row' : 'column'
  list.style.flexWrap = isHorizontal.value ? 'wrap' : 'nowrap'

  // 3. 从旧状态补间到新状态
  Flip.from(flipState, {
    duration: 0.6,
    ease: 'power2.inOut',
    stagger: 0.05,
    absolute: true
  })
}
```

除此以外还有swpo element position的功能，用于处理两个或者多个元素在DOM中互换位置时候的平滑过渡效果，实现在于data-flip-id属性。

Flip 插件默认会给每个元素分配一个自动 ID，但当你要在两个**不同的** DOM 元素之间做“交换”动画时，必须手动给它们设置相同的 `data-flip-id`，Flip 才能知道在状态变化前后，哪个元素对应哪个元素


```ts
let swapSelection = null
case 'swap': {
  const items = target.querySelectorAll('.swap-item')
  items.forEach(item => {
	if (!item.dataset.flipId) {
	  item.dataset.flipId = 'swap-' + Math.random().toString(36).slice(2, 8)
	}
	item.addEventListener('click', handleSwapClick)
  })
  isPlaying.value = true
  return
}

function handleSwapClick(e) {
  const item = e.target.closest('.swap-item')
  if (!item) return

  // 第一次点击：选中
  if (!swapSelection) {
    swapSelection = item
    gsap.to(item, { scale: 1.05, duration: 0.2 })
    item.style.outline = '2px solid #6366f1'
    return
  }

  // 再点同一个：取消选中
  if (swapSelection === item) {
    gsap.to(item, { scale: 1, duration: 0.2 })
    item.style.outline = ''
    swapSelection = null
    return
  }

  // 第二次点击另一个：交换
  const first = swapSelection
  const second = item
  const target = containerRef.value.querySelector('.animate-target')
  const items = target.querySelectorAll('.swap-item')
  const state = Flip.getState(items)

  // 交换 DOM 位置
  const parent = first.parentNode
  const placeholder = document.createElement('div')
  parent.insertBefore(placeholder, first)
  parent.insertBefore(first, second)
  parent.insertBefore(second, placeholder)
  parent.removeChild(placeholder)

  Flip.from(state, {
    duration: 0.6,
    ease: 'power2.inOut',
    absolute: true,
    scale: true,
    stagger: 0.02
  })

  // 清理选中状态
  gsap.to([first, second], { scale: 1, duration: 0.2 })
  first.style.outline = ''
  second.style.outline = ''
  swapSelection = null
}
```

看下面例子，请动画结束前不要做重复交换，这里没有代码保护：

::gsap-box{animation="swap"}
<div style="display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px;">
  <div class="swap-item" style="padding: 16px; background: #eef2ff; border-radius: 8px; cursor: pointer; text-align: center;">A</div>
  <div class="swap-item" style="padding: 16px; background: #fce7f3; border-radius: 8px; cursor: pointer; text-align: center;">B</div>
  <div class="swap-item" style="padding: 16px; background: #ecfdf5; border-radius: 8px; cursor: pointer; text-align: center;">C</div>
  <div class="swap-item" style="padding: 16px; background: #fffbeb; border-radius: 8px; cursor: pointer; text-align: center;">D</div>
  <div class="swap-item" style="padding: 16px; background: #f5f3ff; border-radius: 8px; cursor: pointer; text-align: center;">E</div>
  <div class="swap-item" style="padding: 16px; background: #fef2f2; border-radius: 8px; cursor: pointer; text-align: center;">F</div>
</div>
::



高级效果在 GSAP 官方演示中被称为 **“Gallery to Detail Flip”**（画廊到详情页的翻转/无缝放大），本质上是利用 GSAP 的 `Flip` 插件实现**共享元素过渡

它之所以看起来神奇，核心在于“**伪装、记录、还原、补间**”这四个步骤。下面为你拆解它的原理：

详细面板原本是固定定位，占据了大部分屏幕的巨大容器，缩略图是网格里的一个小方块，怎么去无缝对接，我们会使用gsap里提供的一个api叫做Flip.fit方法，`Flip.fit(detail, item, { fitChild: detailImg })` 的作用是：在瞬间通过计算，把巨大的 `detail` 容器缩小、平移，使其精确地盖在 `item` 缩略图上面，连内部的 SVG 尺寸都完全匹配

之后通过记录，拍下快照，现在详情面板虽然看起来像缩略图，但它的 DOM 属性被 GSAP 写满了缩放的 `transform` 内联样式。我们调用 `const state = Flip.getState(detail)`，把这个“伪装成小缩略图”的尺寸、位置和缩放比例记录下来

记录完状态后，我们立刻用 `gsap.set(detail, { clearProps: true })` 把所有内联样式清空。详情面板瞬间恢复成 CSS 里定义的全屏大尺寸。同时把 `visibility` 设为 `visible`

最后，`Flip.from(state)` 会对比“快照里的小尺寸（起点）”和“当前的大尺寸（终点）”，自动计算两者之间的差异，并生成平滑的过渡动画。  
因为内部的 `fitChild: detailImg` 保证了 SVG 的比例一致，所以在用户看来，就是**缩略图被瞬间放大了**

我们把详细代码放出来：

```vue
<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import { gsap } from 'gsap'
import { Flip } from 'gsap/Flip'
gsap.registerPlugin(Flip)

// 直接写死数据，不依赖任何外部传参
const items = [
  { title: 'Hoop', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#0ae448', type: 'circle' },
  { title: 'Cone', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#fec5fb', type: 'triangle' },
  { title: 'Combo', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#00bae2', type: 'rect' },
  { title: 'Star', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#ff8709', type: 'star' },
  { title: 'Sparkle', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#9d95ff', type: 'diamond' },
  { title: 'Squish', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#abff84', type: 'hexagon' },
  { title: 'Spiral', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#0ae448', type: 'circle' },
  { title: 'Tunnel', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#fec5fb', type: 'triangle' },
  { title: 'Lock', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#00bae2', type: 'rect' },
  { title: 'Windmill', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#ff8709', type: 'star' },
  { title: 'Triangle', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#9d95ff', type: 'diamond' },
  { title: 'Hoop', secondary: 'Lorem ipsum dolor sit amet.', text: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.', color: '#abff84', type: 'hexagon' }
]

const appRef = ref(null)
const galleryRef = ref(null)
const detailRef = ref(null)
const detailImgRef = ref(null)
const detailContentRef = ref(null)
const textRefs = {
  title: ref(null),
  secondary: ref(null),
  description: ref(null)
}

const activeItem = ref(null)
const isOpen = ref(false)
const activeData = ref({ title: '', color: '', type: '' })

function getSvg(color, type) {
  const paths = {
    circle: `<circle cx="50" cy="50" r="40" stroke="${color}" stroke-width="8"/>`,
    rect: `<rect x="15" y="15" width="70" height="70" rx="8" stroke="${color}" stroke-width="8"/>`,
    triangle: `<path d="M50 15 L85 80 L15 80 Z" stroke="${color}" stroke-width="8" stroke-linejoin="round"/>`,
    star: `<path d="M50 10 L61 40 L93 40 L67 58 L77 90 L50 70 L23 90 L33 58 L7 40 L39 40 Z" stroke="${color}" stroke-width="6" stroke-linejoin="round"/>`,
    diamond: `<path d="M50 10 L90 50 L50 90 L10 50 Z" stroke="${color}" stroke-width="8" stroke-linejoin="round"/>`,
    hexagon: `<path d="M50 10 L85 30 L85 70 L50 90 L15 70 L15 30 Z" stroke="${color}" stroke-width="8" stroke-linejoin="round"/>`
  }
  return `<svg viewBox="0 0 100 100" fill="none">${paths[type] || paths.circle}</svg>`
}

onMounted(async () => {
  await nextTick()
  if (!detailContentRef.value) return

  gsap.set(detailContentRef.value, { yPercent: -100 })
  gsap.set(detailRef.value, { visibility: 'hidden' })

  gsap.to(appRef.value, { autoAlpha: 1, duration: 0.2 })
  gsap.from(galleryRef.value.querySelectorAll('.item'), {
    autoAlpha: 0, yPercent: 30, stagger: 0.04
  })
})

onUnmounted(() => {
  gsap.killTweensOf("*")
})

function showDetails(item, index) {
  if (activeItem.value) return hideDetails()

  const data = items[index]
  activeData.value = data
  
  textRefs.title.value.innerText = data.title
  textRefs.secondary.value.innerText = data.secondary
  textRefs.description.value.innerText = data.text

  activeItem.value = item
  isOpen.value = true

  nextTick(() => {
    const detail = detailRef.value
    const img = detailImgRef.value
    const content = detailContentRef.value
    const itemEls = galleryRef.value.querySelectorAll('.item')

    Flip.fit(detail, item, { scale: true, fitChild: img })
    const state = Flip.getState(detail)

    gsap.set(detail, { clearProps: true })
    gsap.set(detail, {
      xPercent: -50, top: '50%', left: '50%', yPercent: -50,
      visibility: 'visible', overflow: 'hidden'
    })

    Flip.from(state, {
      duration: 0.5, ease: 'power2.inOut', scale: true,
      onComplete: () => gsap.set(detail, { overflow: 'auto' })
    }).to(content, { yPercent: 0 }, 0.2)

    gsap.to(itemEls, {
      opacity: 0.3,
      stagger: { amount: 0.7, from: itemEls.indexOf(item), grid: 'auto' }
    }).kill(item)
  })
}

function hideDetails() {
  if (!isOpen.value) return
  isOpen.value = false

  const detail = detailRef.value
  const content = detailContentRef.value
  const img = detailImgRef.value
  const itemEls = galleryRef.value.querySelectorAll('.item')
  const currentItem = activeItem.value
  
  if (!detail || !currentItem) {
    if (detail) gsap.set(detail, { visibility: 'hidden' })
    return
  }

  gsap.set(detail, { overflow: 'hidden' })
  const state = Flip.getState(detail)
  Flip.fit(detail, currentItem, { scale: true, fitChild: img })

  const tl = gsap.timeline({
    onComplete: () => {
      activeItem.value = null
      gsap.set(detail, { visibility: 'hidden' })
    }
  })

  tl.to(content, { yPercent: -100 })
    .to(itemEls, { opacity: 1, stagger: { amount: 0.7, from: itemEls.indexOf(currentItem), grid: 'auto' } }, '<')

  Flip.from(state, {
    scale: true, duration: 0.5, delay: 0.2,
    onInterrupt: () => tl.kill()
  })
}
</script>

<template>
  <div class="gallery-root">
    <div ref="appRef" class="app">
      <div ref="galleryRef" class="gallery">
        <div
          v-for="(item, i) in items"
          :key="i"
          class="item"
          @click.stop="showDetails($event.currentTarget, i)"
        >
          <div class="item-svg" v-html="getSvg(item.color, item.type)" />
        </div>
      </div>
    </div>

    <div 
      v-if="isOpen" 
      class="backdrop" 
      @click.self="hideDetails" 
    >
      <div ref="detailRef" class="detail">
        <div ref="detailImgRef" class="detail-svg" @click="hideDetails" v-html="getSvg(activeData.color, activeData.type)" />
        <div ref="detailContentRef" class="content">
          <div :ref="el => textRefs.title.value = el" class="title"></div>
          <div :ref="el => textRefs.secondary.value = el" class="secondary"></div>
          <div :ref="el => textRefs.description.value = el" class="description"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.gallery-root { position: relative; }

.backdrop {
  position: fixed;
  inset: 0;
  z-index: 99;
  background: rgba(0, 0, 0, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
}

.app {
  height: 90vmin; width: 90vmin;
  background: var(--color-just-black, #0e100f);
  position: relative; overflow: auto;
  color-scheme: dark; visibility: hidden;
  margin: 0 auto;
}

.gallery { display: grid; grid-template-columns: repeat(3, 1fr); gap: 1rem; padding: 10px; }
.item { cursor: pointer; }

.item-svg, .detail-svg {
  width: 100%; aspect-ratio: 1 / 1;
  display: flex; align-items: center; justify-content: center;
  background: #1c1d1f; padding: 12%; box-sizing: border-box;
}
.item-svg { border-radius: 10px; }
.detail-svg { border-radius: 10px 10px 0 0; position: relative; z-index: 1; cursor: pointer; }

.item-svg :deep(svg), .detail-svg :deep(svg) { width: 100%; height: 100%; }

.detail {
  width: 90.1vmin;
  display: flex; flex-direction: column;
  visibility: hidden; max-height: 90vh; overflow: auto;
}

.detail .content { background: #232323; padding: 2rem 1.5rem; flex-grow: 1; }
.detail .content > * { margin-bottom: 1rem; }
.detail .title { font-size: 2rem; text-transform: uppercase; color: #fffce1; }
.detail .secondary { color: lightgray; }
.detail .description { line-height: 1.5; color: #bbbaa6; }
</style>
```


效果如下：

::gsap-gallery
::

接下里我们看看其他几个比较受欢迎的插件，比如说滚动驱动的ScrollTrigger和一些文字特效，都是一些能够直接提升内容展示效果的插件。

### scrollTrigger

这是 GSAP 最受欢迎的插件之一。它能让动画不再局限于“页面加载即播放”，而是**由滚动位置触发**。比如文章里的某一段落滚动到视口时再淡入，或者让某个元素在滚动过程中“钉”在屏幕上（pin）并播放一段动画。

- **适用场景**：长文档的渐进式呈现、横向滚动画廊、视差效果。

我们来做个导航栏逐个滑入的效果：

::gsap-box{animation="stagger" trigger="scroll" :duration="0.4" ease="back.out(1.7)"}
<div style="display: flex; flex-direction: column; gap: 8px;">
  <a class="nav-item" href="#" style="padding: 10px; background: #eef2ff; border-radius: 6px;">首页</a>
  <a class="nav-item" href="#" style="padding: 10px; background: #fce7f3; border-radius: 6px;">文档</a>
  <a class="nav-item" href="#" style="padding: 10px; background: #ecfdf5; border-radius: 6px;">指南</a>
</div>
::

默认情况下，当组件顶部到达视口高度的 **85%** 时开始播放（`start="top 85%"`）。如果你希望它更早或更晚触发，可以传入 `start`



这个是屏幕正中间才显示：

<!-- 滚动到屏幕正中（50%）时才播放 -->
::gsap-box{animation="scale" trigger="scroll" start="top 50%"}
我是在屏幕正中间才弹出来的。
::


这个是进入视口才显示：

<!-- 当组件完全进入视口才播放 -->
::gsap-box{animation="chars" trigger="scroll" start="top bottom"}
我是等完全进入屏幕才逐字显示的。
::

有些时候我们不想“触发一次播放”，而是想让**滚动条变成动画的进度条**，也就是“滚多少，动画播多少”。这需要用到 `scrub`

::gsap-scroll-parallax{:height="200vh"}

这是一段随着你的滚动条进度，从缩小、旋转、透明的状态，慢慢放大、旋转回正并显现的文本。

你可以在这里放任何 HTML，比如：
<div style="margin-top: 20px; padding: 10px; background: #0ae448; color: #000; border-radius: 8px;">
  甚至是一个带背景的卡片
</div>
::

代码实现如下：

```vue
<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import { gsap } from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
gsap.registerPlugin(ScrollTrigger)

const props = defineProps({
  // 滚动跟随区域的高度（默认 200vh，留出足够的滚动距离）
  height: { type: String, default: '200vh' },
  // 动画的起始和结束缓动
  ease: { type: String, default: 'none' }
})

const containerRef = ref(null)
const stickyRef = ref(null)
const contentRef = ref(null)

onMounted(async () => {
  await nextTick()
  if (!containerRef.value || !contentRef.value) return

  // 核心逻辑：让内部的内容吸顶，并让动画进度与滚动条绑定
  const tl = gsap.timeline({
    scrollTrigger: {
      trigger: containerRef.value,
      start: 'top top', // 当容器的顶部到达视口顶部时开始
      end: 'bottom bottom', // 当容器的底部到达视口底部时结束
      scrub: 1, // 平滑跟随：1 表示有 1 秒的缓动延迟，让手感更顺滑
      pin: stickyRef.value, // 把内容区域钉在屏幕上
      pinSpacing: false, // 因为外层容器已经有 height，这里不需要额外加 padding
      anticipatePin: 1 // 防止快速滚动时产生 1px 的抖动
    }
  })

  // 下面是你希望随着滚动进度发生的动画
  // 这里以“文字放大、颜色渐变、旋转”为例，你可以任意修改
  tl.fromTo(contentRef.value,
    { 
      scale: 0.5, 
      rotation: -45,
      opacity: 0 
    },
    { 
      scale: 1.2, 
      rotation: 0, 
      opacity: 1,
      ease: props.ease,
      // 你可以在这里继续叠加其他属性
    }
  )
})

onUnmounted(() => {
  // 必须清理 ScrollTrigger，防止页面切换后残留
  ScrollTrigger.getAll().forEach(t => t.kill())
})

function scrollToTop() {
  window.scrollTo({ top: containerRef.value.offsetTop, behavior: 'smooth' })
}
</script>

<template>
  <div 
    ref="containerRef" 
    class="scroll-parallax-container" 
    :style="{ height: props.height }"
  >
    <div ref="stickyRef" class="sticky-wrapper">
      <div ref="contentRef" class="scroll-content">
        <!-- 默认插槽，由 MDC 传入内容 -->
        <slot />
      </div>
    </div>
  </div>
</template>

<style scoped>
.scroll-parallax-container {
  position: relative;
  width: 100%;
  background: var(--ui-backdrop, #f6f6f7);
  border-radius: 12px;
  border: 1px solid var(--ui-line, #e2e2e3);
  margin: 2rem 0;
  overflow: hidden;
}

.sticky-wrapper {
  width: 100%;
  height: 100vh; /* 占满整个屏幕 */
  display: flex;
  align-items: center;
  justify-content: center;
}

.scroll-content {
  /* 这里可以自定义内容的基础样式，动画由 GSAP 控制 */
  padding: 2rem;
  max-width: 800px;
  text-align: center;
  color: #0e100f;
  font-size: 1.5rem;
  font-weight: bold;
  will-change: transform, opacity; /* 提示浏览器开启硬件加速 */
}
</style>
```


然后还有横线滚动的类型：

::gsap-horizontal-scroll
<div class="panel">
  <div class="panel-content">
    <h2>Panel 1</h2>
    <p>向下滚动页面，这整个区域会变成横向滑动。</p>
  </div>
</div>
<div class="panel">
  <div class="panel-content">
    <h2>Panel 2</h2>
    <p>你可以在这里放任意内容。</p>
  </div>
</div>
<div class="panel">
  <div class="panel-content">
    <h2>Panel 3</h2>
    <p>也可以放图片，或者代码块。</p>
  </div>
</div>
<div class="panel">
  <div class="panel-content">
    <h2>Panel 4</h2>
    <p>滑到底部就会恢复正常的垂直滚动。</p>
  </div>
</div>
::

横向滚动非常耗性能（因为是直接修改 layout 属性），所以组件里加了 `will-change: transform` 提示浏览器硬件加速。另外，如果在移动端（手机）上，这种交互经常会和原生的左右滑动冲突，建议在移动端通过媒体查询关掉 `pin` 效果，但这对于 PC 端阅读体验来说，绝对是个加分项

这个在gsap里是一个视觉欺骗的办法，它是把垂直方向的滚动距离映射成了水平方向的移动距离。

在DOM结构上，我们的所有内容都在一个track容器里，通过css的display: flex; flex-wrap: nowrap;强制排列在一行，如果不加限制，那么超出的内容会被挤压，我们需要给track设置width: fit-content，让他撑开，暴露出真实的超长宽度。

由于用户滚轮，页面会一直往下走，横向内容没有人看，那么我们在 `ScrollTrigger` 里设置了 `pin: true`。一旦这个容器顶部碰到屏幕顶部，GSAP 就会给容器加上固定定位（类似于 CSS 的 `position: fixed`），把整个横向轨道“冻”在屏幕正中。同时，GSAP 会在页面底部自动插入等高的占位符，保证页面的滚动条依然存在，用户可以继续往下滚

既然屏幕冻住了，用户往下滚的时候，页面上看不到任何位移。但实际上，用户的滚动事件被 `ScrollTrigger` 拦截了。  
我们在 `gsap.to()` 里写了


```ts
x: () => -(track.scrollWidth - window.innerWidth)
```


这行代码的意思是：**让轨道的 X 轴向左移动，移动的总距离 = 轨道总宽度 - 屏幕宽度**。

结合 `scrub: 1`，这个位移不再是瞬间发生的，而是完全跟你的滚动条深度绑定的。你往下滚 10 像素，轨道就向左移 10 像素（带点平滑延迟）。用户以为自己是在“往下滚”，但在视觉上，内容是从右往左滑过去的



代码实现如下：

```vue
<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import { gsap } from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
gsap.registerPlugin(ScrollTrigger)

const containerRef = ref(null)
const trackRef = ref(null)

onMounted(async () => {
  await nextTick()
  if (!containerRef.value || !trackRef.value) return

  // 获取需要横向滚动的轨道元素
  const track = trackRef.value
  // 获取所有面板项
  const panels = gsap.utils.toArray('.panel', track)

  // 关键逻辑：垂直滚动转换为横向平移
  gsap.to(track, {
    x: () => -(track.scrollWidth - window.innerWidth),
    ease: 'none',
    scrollTrigger: {
      trigger: containerRef.value,
      start: 'top top', // 当容器顶部到达视口顶部时开始
      end: () => '+=' + (track.scrollWidth - window.innerWidth), // 滚动距离等于横向超出的宽度
      pin: true, // 把容器钉在屏幕上
      scrub: 1, // 平滑跟随，1秒延迟让手感更顺
      invalidateOnRefresh: true, // 窗口尺寸变化时重新计算
      anticipatePin: 1
    }
  })

  // 附加效果：每个面板进入屏幕时的轻微动画（可选）
  panels.forEach((panel) => {
    gsap.from(panel.querySelector('.panel-content'), {
      scale: 0.8,
      opacity: 0.5,
      scrollTrigger: {
        trigger: panel,
        containerAnimation: gsap.getTweensOf(track)[0], // 这是横向滚动中触发内部动画的关键
        start: 'left center', // 当面板左侧到达屏幕中心时
        end: 'right center',
        scrub: true,
        toggleActions: 'play none none reverse'
      }
    })
  })
})

onUnmounted(() => {
  ScrollTrigger.getAll().forEach(t => t.kill())
})
</script>

<template>
  <div ref="containerRef" class="horizontal-container">
    <div ref="trackRef" class="horizontal-track">
      <slot />
    </div>
  </div>
</template>

<style scoped>
/* 容器：固定在屏幕上，阻止原生横向滚动 */
.horizontal-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background: #1D1F20;
}

/* 轨道：弹性排布所有面板 */
.horizontal-track {
  display: flex;
  flex-wrap: nowrap;
  width: fit-content;
  height: 100vh;
  align-items: center;
  padding: 0 10vw; /* 两侧留白 */
  will-change: transform;
}

/* 针对 slot 传入的面板默认样式（可被 MDC 内联覆盖） */
.horizontal-track :deep(.panel) {
  width: 80vw;
  height: 60vh;
  flex-shrink: 0;
  margin-right: 5vw;
  border-radius: 16px;
  background: #232323;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
}

.horizontal-track :deep(.panel-content) {
  padding: 2rem;
  text-align: center;
}
</style>
```


GSAP 生态里还有几个非常顶尖的视觉特效，分别叫做沿着路径移动的`MotionPathPlugin`，然后还有SVG形状变形的 `MorphSVGPlugin`，辉光和滤镜`Physics2DPlugin` 或 CSS Filter。接下来有时间我们都会细细讲一下。


## 其他代码的Gallery展示


### 彩带大炮 (Physics2D)

这是一次结合了物理引擎的重力与抛物线模拟。[Physics2D | GSAP | Docs & Learning](https://gsap.com/docs/v3/Plugins/Physics2DPlugin/)

::gsap-cannon
::

### 浮起来的3D卡片

::gsap-tilt
::

### 文字遮罩效果

::gsap-reveal-text
::

### 文字沿贝塞尔曲线飞入

::gsap-motion-path
::

### 流水跑马灯

展示技术栈、品牌 Logo 或合作伙伴最经典的用法。因为它需要**无限循环**，所以实现原理非常简单：**把内容复制两份，然后用 GSAP 让它永远向左平移**


::gsap-marquee
::

### 聚光灯跟随

整个板块背景是暗黑的，一束圆形的“手电筒光芒”会实时跟随着你的鼠标移动，照亮鼠标下方的区域（可以照亮局部文字，也可以照亮一整张图片）。

::gsap-spotlight
::