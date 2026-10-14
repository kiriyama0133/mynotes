---
date: 2026-12-10
tags:
  - browser
  - html
---
这里整理的是html相关技术的笔记，面试可能会考到，所以这里便积累了一些

## Src 和 href 的区别

src是替换当前元素，href是建立关联，src会把外部资源嵌入到当前的元素的位置，替换掉这个元素，它的加载行为：浏览器会阻塞解析，等资源下载完再继续，因为资源是当前文档的一部分，必须等到位。

```html
<img src="a.jpg">
<script src="a.js"></script>
<iframe src="page.html"></iframe>
<video src="a.mp4"></video>
<audio src="a.mp3"></audio>
```

href作用是**建立当前文档和尾部资源的关联**，不会替换元素本身，加载行为：浏览器不阻塞解析，并行下载，资源是关联，非当前文档一部分。

## iframe标签的优缺点

这个标签作用是创建一块独立的文档区域，有自己的DOM、js上下文（不同全局对象），css作用域，cookie和存储。

父页面的css不会污染iframe内容，反之亦然，这在嵌入第三方内容时非常关键。

```
<!-- 第三方广告、地图、视频播放器 -->
<iframe src="https://maps.example.com"></iframe>
```

跨域部分，想在页面中显示另一个域的内容，iframe就是唯一的直接支持，其他方案会受到CORS限制，而且脚本不会执行。

sandbox属性可以精确控制iframe能力：

```
<iframe
  src="https://untrusted.com"
  sandbox="allow-scripts"
></iframe>
```

|`sandbox` 值|允许的能力|
|---|---|
|`allow-scripts`|执行 JS|
|`allow-forms`|提交表单|
|`allow-same-origin`|保持同源（**慎用**）|
|`allow-popups`|打开弹窗|
|`allow-top-navigation`|跳转顶层页面|

iframe的加载和主页面解耦，不会拖慢主页面渲染，如果iframe脚本崩溃，不会影响父页面。但是每一个iframe是一个独立的浏览上下文，是独立的js引擎实例，独立的dom树，独立事件循环，内存开销大，多个iframe会显著增加内存占用。搜索引擎**通常不索引 iframe 内容**。如果关键内容在 iframe 里，SEO 基本失效。

父页面和 iframe 通信只能通过 `postMessage` ，这里一定要和BroadcastChannel分开，这是两个不同技术。

```ts
// 父页面发消息
iframe.contentWindow.postMessage('hello', 'https://child.com')

// 子页面接收
window.addEventListener('message', (e) => {
  if (e.origin !== 'https://parent.com') return  // 必须校验 origin
  console.log(e.data)
})
```

## Canvas和SVG

canvas是位图，svg是矢量图，canvas是立即模式渲染，逐像素绘制，svg是保留模式绘制，DOM节点描述。canvas本质是像素数组，svg本质是dom树。

稍微触底来看看，canvas像素数据在js层面使用的是uint8ClampedArray，不同于wasm里的uint8heap，都是字节数组，但是约束和用途不同。

使用getImageData()获取canvas像素的时候，返回的是ImageData对象，里面有一个data属性，类型就是Uint8ClampedArray。

```js
const imageData = ctx.getImageData(0, 0, width, height);
// imageData.data 是 Uint8ClampedArray
// 顺序是 RGBA：R, G, B, A, R, G, B, A...
```
这个数组的每个元素是一个 **0 到 255 之间的整数**，每 4 个一组（红、绿、蓝、透明度），**“Clamped”就是“钳制”的意思。** 如果你给 `Uint8ClampedArray` 的元素赋值 `300`，它会被自动截断为 `255`；赋值 `-10`，会变成 `0`。而 `Uint8Array` 会做模运算，`300` 会变成 `44`。

而wasm里面的线性内存WebAssembly.Memory中，在js侧是通过ArrayBuffer暴露的，我们可以用不同的TypedArray视图去查看：

```js
// WebAssembly 的内存
const waMemory = waInstance.exports.memory;

// 用 Uint8Array 视图看这块内存
const u8Heap = new Uint8Array(waMemory.buffer);
```

## TypeScript的DOM类型定义

ts中的dom类型定义里，Element继承链是EventTarget-> Node-> element，
```ts
interface EventTarget { }
interface Node extends EventTarget { }
interface Element extends Node { }
```

再往下的话，就是具体元素HTMLElement、SVGElement、MathElement 都是继承自Element了。
```
EventTarget
  └── Node
        └── Element
              ├── HTMLElement
              │     ├── HTMLDivElement
              │     ├── HTMLInputElement
              │     └── ...
              ├── SVGElement
              └── MathMLElement
```

然后接下来可以看看每一层都提供了什么，这也是DOM的原型设计，记了有很多好处，如果后续想要自己开发跨平台组件的话，也可以参考看看。

EventTarget提供了 `addEventListener`、`removeEventListener`、`dispatchEvent` ，让DOM组件有了事件能力。

Node提供了 `parentNode`、`childNodes`、`appendChild`、`removeChild`、`nodeType`、`textContent`，可以让我们随意增删节点和设置文本内容。

Element提供了`id`、`className`、`tagName`、`attributes`、`querySelector`、`getAttribute`、`innerHTML`，让我们可以在DOM组件上设置标识、查询后代、读写属性。

**`HTMLElement`** 提供了 `style`、`dataset`、`offsetWidth`、`click()`、`focus()`、`hidden`，让DOM组件可以控制样式，读写data属性，测量尺寸，触发交互、控制可见性。


## 虚拟列表

这是一种前端优化的性能技术，用于高效渲染超长列表，和WPF的虚拟滚动是一种意思，**只渲染视口内可见的列表项，而不是一次性渲染全部数据**。

在遇到超多DOM的情况下可以做到小占用，滚动流程。只不过虚拟列表可能更常用，因为在前端UI组件中更加常用，组件叫做VirtualList。虚拟滚动在技术文章和浏览器API里讨论常见，描述滚动时动态更新内容这个行为。

我们需要保留视口内的容器，让新的数据替换掉，重新渲染。

```vue
<template>
  <div
    ref="viewportRef"
    class="virtual-list"
    @scroll.passive="onScroll"
  >
    <!-- 占位层：撑起总高度，让滚动条正确 -->
    <div class="phantom" :style="{ height: totalHeight + 'px' }"></div>

    <!-- 渲染层：只放可见项 -->
    <div class="render-layer" :style="{ transform: `translateY(${offsetY}px)` }">
      <div
        v-for="item in visibleItems"
        :key="item.id"
        class="list-item"
        :style="{ height: itemHeight + 'px' }"
      >
        {{ item.name }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue';

const props = defineProps({
  list: { type: Array, default: () => [] },
  itemHeight: { type: Number, default: 50 },
  buffer: { type: Number, default: 3 } // 上下缓冲区，防止快速滚动白屏
});

const viewportRef = ref(null);
const scrollTop = ref(0);
const viewportHeight = ref(0);

// 总高度：撑起滚动条
const totalHeight = computed(() => props.list.length * props.itemHeight);

// 可见起始索引（含缓冲区）
const startIndex = computed(() => {
  const raw = Math.floor(scrollTop.value / props.itemHeight);
  return Math.max(0, raw - props.buffer);
});

// 可见结束索引（含缓冲区）
const endIndex = computed(() => {
  const raw = Math.ceil((scrollTop.value + viewportHeight.value) / props.itemHeight);
  return Math.min(props.list.length - 1, raw + props.buffer);
});

// 当前可见的数据切片
const visibleItems = computed(() =>
  props.list.slice(startIndex.value, endIndex.value + 1)
);

// 渲染层的偏移量
const offsetY = computed(() => startIndex.value * props.itemHeight);

// 滚动处理（用 rAF 节流）
let ticking = false;
const onScroll = (e) => {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    scrollTop.value = e.target.scrollTop;
    ticking = false;
  });
};

// 初始化视口高度
const updateViewportHeight = () => {
  if (viewportRef.value) {
    viewportHeight.value = viewportRef.value.clientHeight;
  }
};

onMounted(() => {
  updateViewportHeight();
  window.addEventListener('resize', updateViewportHeight);
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', updateViewportHeight);
});
</script>

<style scoped>
.virtual-list {
  height: 500px;          /* 固定视口高度 */
  overflow-y: auto;
  position: relative;
  border: 1px solid #ddd;
}

.phantom {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  z-index: -1;            /* 不遮挡内容，只撑高度 */
}

.render-layer {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  will-change: transform; /* 提示浏览器优化 */
}

.list-item {
  box-sizing: border-box;
  border-bottom: 1px solid #eee;
  display: flex;
  align-items: center;
  padding: 0 12px;
}
</style>
```

## 首屏防闪

在样式生效前把 .dark 挂到 `<html>` 上就可以防止首次加载的时候闪烁，这个我们在nuxt.config.ts里可能能够看得到相关配置：

```ts
  app: {
    head: {
      script: [
        {
          // 首屏防闪：在样式生效前把 .dark 挂到 <html> 上。
          innerHTML:
            "(function(){try{var s=localStorage.getItem('blog-theme');var d=s?s==='dark':window.matchMedia('(prefers-color-scheme: dark)').matches;if(d)document.documentElement.classList.add('dark')}catch(e){}})()",
          tagPosition: 'head' as const
        }
      ]
    }
  },
```

问题核心就是防止页面在暗色模式初始化前就先闪一下亮色背景，在切换成暗色。浏览器的渲染顺序是1先解析HTML，构建DOM；2加载并解析CSS，构建CSSOM；3渲染出第一帧；4加载并执行Javascript。js通常在第4步才执行，如果暗色模式判断逻辑是第4步的话，比如说读取localstorage，然后给html加.dark类，那么就可能发生闪烁。

这时候我们会在head里内联一段阻塞脚本，把判断逻辑提前到第一帧渲染之前执行。上面的代码干的就是这件事：

```html
<head>
  <script>
    // 在 CSS 渲染前执行，直接给 <html> 加 .dark
    (function () {
      const theme = localStorage.getItem('theme');
      if (theme === 'dark' || (!theme && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
        document.documentElement.classList.add('dark');
      }
    })();
  </script>
  <!-- 之后才是样式表 -->
</head>
```


