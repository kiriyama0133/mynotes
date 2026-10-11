---
date: 2026-10-11
tags:
  - css
---
css中有几个上下文需要了解一下，我下面列出来。这是最核心的一类，决定盒子如何布局、如何相互影响。CSS里的上下文本质都是作用域的隔离机制，它们共同构成布局和渲染的隔离体系

## 不同上下文

主要包括：

| 上下文     | 全称                        | 作用                            | 触发方式                                                                                                 |
| ------- | ------------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------- |
| **BFC** | Block Formatting Context  | 块级布局隔离，防止 margin 折叠、清除浮动、包含浮动 | `overflow: hidden`、`display: flow-root`、`float`、`position: absolute/fixed`、`display: flex/grid` 的子项等 |
| **IFC** | Inline Formatting Context | 行内元素布局，决定行盒如何排列               | 块级容器内包含行内内容时自动创建                                                                                     |
| **FFC** | Flex Formatting Context   | Flex 布局环境                     | `display: flex` 或 `inline-flex`                                                                      |
| **GFC** | Grid Formatting Context   | Grid 布局环境                     | `display: grid` 或 `inline-grid`                                                                      |
| **TC**  | Table Formatting Context  | 表格布局环境                        | `display: table` 等表格相关值                                                                              |

**BFC 是最常被提到的**，因为它能解决很多经典问题，不过先不讲，后面我再解释。除了上面的，还有层叠上下文，用于决定元素在Z轴上如何堆叠。还有隔离上下文比如isolation: isolate创建独立的层叠上下文，用于mix-blend-mode场景防止混合模式影响外部元素。容器上下文，比如css containment中的contain: layout/paint/size会创建布局/绘制隔离，让元素的内部变化不影响外部，这个也可以看做一种上下文。

>[CSS](https://developer.mozilla.org/zh-CN/docs/Web/CSS) 属性 **`contain`** 标示了元素及其内容尽可能独立于文档树的其余部分。局限使 DOM 的一部分得以被隔离，且通过将布局、样式、绘制、尺寸或其任意组合的计算限制于 DOM 子树而非整个页面使性能受益。局限也可用于限制 CSS 计数器和引号的作用域。它允许你向浏览器声明：**某个元素及其子树是独立的，其内部的布局、样式或绘制变化，不会影响到页面上的其他部分。这样浏览器就能把重排、重绘的工作范围限制在这个元素内部，而不是检查整个页面，从而在处理复杂或动态内容时提升渲染性能，具体看MDN

### 其他相关概念

- **滚动上下文（Scroll Container）**：`overflow` 非 `visible` 且内容溢出时，形成独立的滚动区域。
    
- **裁剪上下文（Clipping Context）**：`clip-path`、`overflow: hidden` 等会裁剪内部内容。
    
- **变换上下文（Transform Context）**：`transform` 非 `none` 会创建新的坐标系统，同时也会创建层叠上下文。


## bfc

**BFC（块级格式化上下文）**：是一个**布局隔离环境**，需要特定条件才能触发，比如 `overflow: hidden`、`display: flow-root`、`float`、`position: absolute` 等。

**块级元素（`display: block`）**：默认 `width: auto`，会**占满父容器的可用宽度**（撑满整行）。这跟它是不是 BFC 无关，普通块级元素也这样。我们常常用display: flow-root来创建BFC，它本身是块级元素，所以默认也**占满整行**。

```css
<div class=" text-black bg-black">123</div>
```

比如这个简单的元素，他就会默认占满父容器宽度，前提是它处于**正常流**中，并且没有设置 `width`、`display: inline-block`、`float` 等改变宽度的属性。只要我们加了inline-block或是float，那么就会出现收缩包裹现象。哦对了，里面的123直接放在块级容器里的文本会被浏览器自动包进一个**匿名行内盒子**，这个盒子参与行内格式化上下文IFC，会从左到右排列。

所以，这里重点强调：占满整行是块级元素的默认行为！！！块级元素的代表我先列出来：

| 默认 `display`       | 代表元素                                                  | 特点                                               |
| ------------------ | ----------------------------------------------------- | ------------------------------------------------ |
| **`block`**        | `<div>`、`<p>`、`<h1>`~`<h6>`、`<ul>`、`<li>`、`<section>` | 独占一行，默认占满父容器宽度                                   |
| **`inline`**       | `<span>`、`<a>`、`<em>`、`<strong>`、`<b>`、`<i>`          | 不独占一行，宽高由内容决定，`width`/`height` 无效，表现为inline的行内元素 |
| **`inline-block`** | `<button>`、`<input>`、`<select>`、`<img>`               | 对外像行内元素，对内像块级元素，可设宽高                             |
| **`list-item`**    | `<li>`                                                | 像块级元素，但会生成列表项标记                                  |
| **`table` 相关**     | `<table>`、`<tr>`、`<td>`                               | 表格布局专用                                           |
| **`none`**         | `<head>`、`<meta>`、`<title>` 等                         | 不生成盒子，不参与布局                                      |
对外像行内元素，对内像块级元素，可设宽高的意思是不独占一行，多个 `inline-block` 元素会排在同一行；宽度默认由内容决定（收缩包裹），不占满父容器。`width`、`height`、`margin`、`padding` 都有效，这是它和纯 `inline` 元素最大的区别。内部可以放块级元素，内部布局按正常流进行。

正常流式CSS中最基础、默认的布局方式，当你不使用 `float`、`position: absolute/fixed` 或 Flex/Grid 等特殊布局时，元素就处于正常流中。表现为把页面上元素分成两类来排列，块级元素会从上到下垂直堆叠，每个独占一行，默认占满父容器宽度，垂直方向的margin折叠。

行内元素inline，从左到右水平排列，排满一行后换行，宽度由内容决定，不独占一行，垂直margin不生效，水平margin生效。

```html
  <div class=" text-black bg-black inline-block">
    <span class=" mr-2">123</span>
    <span class=" mt-10">123</span>
  </div>
```

比如这里的mr-2会生效，mt-10不会生效，你想要生效就要去生成bfc，可以加上block生效，把span的display的inline改为block，这里不是创建bfc的意思哦。

根据 CSS 规范，一个元素满足以下**任意一个**条件，就会创建 BFC：

|触发条件|说明|
|---|---|
|`float: left/right`|浮动元素|
|`position: absolute/fixed`|绝对/固定定位|
|`display: inline-block`|行内块|
|`display: table-cell` / `table-caption`|表格相关|
|`display: flow-root`|专门创建 BFC 的值|
|`overflow` 非 `visible`|如 `hidden`、`auto`、`scroll`|
|`contain: layout/paint/content/strict`|容器隔离|
|Flex/Grid 子项|某些情况下|
一个普通 `block` 元素**不能**包含浮动子元素（父元素会高度塌陷），因为它是正常流，浮动子元素会脱离它。但如果你给它加 `overflow: hidden`，它就变成 BFC，就能包含浮动子元素了。


```html
<div class="box bg-yellow-500">
  <div style="float: left; width: 100px; height: 100px; background: red;"></div>
</div>
```

这种情况下，会出现高度塌陷，因为刚才才说浮动元素会脱离高度计算是吧，这个脱离了正常流之后box就显示不出黄色了，box不包含它。那么我们这时候加上 overflow-hidden试试，这样就会发现之前的黄色出来了，这就意味着overflow-hidden确实创建了BFC，解决了高度塌陷的问题，之前的浮动元素参与了高度计算！！当然你输入flow-root也是一样的效果。

## ffc

当一个元素变成 Flex 容器后，它内部的子元素**不再遵循正常流**，而是变成 **Flex 项（flex item）**，按 Flex 规则排列。

这个时候就会分为主轴排列和交叉轴两个方向，子元素默认按照主轴从左到右排列，然后默认flex-wrap：nowrap不换行，挤在一行，可能压缩。浮动会消失，子元素的float会直接被忽略，不生效。垂直上，margin不会折叠，flex项之间的margin不会折叠。子元素可以设置宽高，flex项是块级化的，width和height都生效。flex容器的高度会包含flex项，不会高度塌陷，flex容器对外是块级，内部布局独立。


```html
<div class="flex bg-amber-400">
  <div class="item bg-blue-400 mt-2">1</div>
  <div class="item bg-blue-600 mt-3">2</div>
  <div class="item">3</div>
</div>
```

这里的mt-2 和mt-3不会被折叠，两个都可以生效。

你还可以使用行内flex: inline-flex，创建行内flex容器。flex-row或者flex-col设置主轴方向。除此以外，flex和流式都可能出现高度拉伸，但触发条件和表现完全不同。

在正常流中，父元素有固定高度+子元素占据100%高度情况下，子元素自己是会拉伸到指定高度的，。在flex中，如果父元素display: flex + 默认align-items:stretch下，所有flex项会默认拉伸，就算子元素不设置高度，依旧会去拉伸。

flex在设置flex-wrap换行之后，就会flex行和项，分辨flex行和项之后才可以清除之后的justify-content等布局操作的主体是什么。

justify-content操作的是每一flex行的内部，控制该行中所有 Flex 项沿**主轴**的分布，**不控制行与行之间的分布**——那是 `align-content` 的职责。align-items操控的每一个flex项，作用于交叉轴。

比如你控制三个item项在一个flex行中居中：

```html
<div class="flex flex-wrap justify-center bg-amber-400 h-20">
  <div class="item bg-blue-400 mt-2">1</div>
  <div class="item bg-blue-600 mt-3">2</div>
  <div class="item">3</div>
</div>
```

我们使用gap-y-6可以设置flex行的距离，`gap-y-6` 是 Tailwind 的 `row-gap: 1.5rem`（24px）。在 Flex 布局中，它控制的是**相邻 Flex 行之间的固定间距**。如果你仔细观察这几行代码，会发现两行高度上没有贴在一起，而是分开的，这是因为align-content: stretch发力了，当 Flex 容器在交叉轴方向有**剩余空间**时，把剩余空间**平均分配**给每一行，让每行的高度被拉大，从而填满整个容器。**如果 `align-content` 是 `flex-start`，两行就会贴在一起，靠在容器顶部。**


控制 Flex **行与行之间**在交叉轴（垂直方向）分布的属性是 **`align-content`**，对应的 Tailwind 类是：

| Tailwind 类        | CSS 属性                         |
| ----------------- | ------------------------------ |
| `content-start`   | `align-content: flex-start`    |
| `content-end`     | `align-content: flex-end`      |
| `content-center`  | `align-content: center`        |
| `content-between` | `align-content: space-between` |
| `content-around`  | `align-content: space-around`  |
| `content-evenly`  | `align-content: space-evenly`  |
| `content-stretch` | `align-content: stretch`（默认）   |
## 层叠上下文

层叠上下文和z-index相关，是 CSS 中决定**元素在 Z 轴上如何堆叠**的核心机制。理解它，就能解释“为什么 `z-index: 9999` 还是被盖住”这类经典问题。

它是一个独立的堆叠环境，可以想象为一个盒子，内部元素是按规则堆叠，整体作为一个单位，参与外层父级层叠上下文的堆叠，子元素的z-index只是在同一个层叠上下文内比较，无法跨上下文越级，层叠上下文可以嵌套，形成一棵树。

在一个层叠上下文内部，元素从低到高堆叠：

|顺序|内容|说明|
|---|---|---|
|1（最低）|背景和边框|当前层叠上下文根元素的背景、边框|
|2|`z-index` 为负的子元素|如 `z-index: -1`|
|3|块级元素的背景和边框|普通流中的块级盒子|
|4|浮动元素|`float: left/right`|
|5|行内元素的背景和边框|普通流中的行内盒子|
|6|`z-index: auto` 或 `0` 的定位元素|定位但未设 z-index|
|7（最高）|`z-index` 为正的子元素|如 `z-index: 1`、`z-index: 999`|
