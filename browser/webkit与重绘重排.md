---
date: 2026-10-01
tags:
  - browser
  - webkit
  - css
---
重排和重绘是浏览器渲染引擎的工作机制，而 WebKit 正是最著名、历史最悠久的渲染引擎之一，

## webkit里真实名称

虽然我们常把“重排”叫 Reflow，但在 WebKit 的源码和核心架构里，它更标准的叫法是 **Layout（布局）**。

- **重排 (Reflow / Layout)**：在 WebKit 中，对应的是 `RenderObject`（渲染对象）树去计算每一个盒子几何位置的过程。WebKit 有一个专门的内核类叫 `RenderView` 和 `RenderBlock` 来负责执行 Layout 算法。
    
- **重绘 (Repaint)**：在 WebKit 中，对应的是 `GraphicsContext` 去重新填充像素的过程。

_(注：Reflow 这个词最早是 Firefox 的 Gecko 引擎提出的，后来因为太形象，大家就把它当成了重排的通用代名词。)_

在 WebKit 的架构里，一次页面的渲染大致是：`DOM 树 -> 样式计算 -> Render 树 -> Layout (重排) -> Paint (重绘) -> Composite (合成)`。

- 当 JS 修改了元素的 `width` 时，WebKit 内部的 `RenderObject` 会标记自己为 `needsLayout`。
    
- 在下一帧到来时，WebKit 会调用 `layout()` 方法，重新计算整棵树，这就是**重排**。
    
- 重排完成后，WebKit 发现外观变了，会标记 `needsPaint`，然后调用 `paint()` 方法，把像素画到内存的位图里，这就是**重绘**。


重排和重绘是很消耗性能的，所以webkit以及后来的blink引擎，选择引入了渲染层和图形层概念：

- 如果你只修改了 `transform` 或 `opacity`，WebKit 根本不会触发 Layout 和 Paint，而是直接让 GPU 重新组合已有的图层（Composite）。
    
- 这就是为什么现代 WebKit 引擎能做到 60fps 丝滑动画的底层秘密——**绕开了重排和重绘**。

渲染层和图形层：

Renderlayer是浏览器在dom树和渲染树之后建立的逻辑分层结构，负责解决元素在视觉树的堆叠和覆盖问题，让决定元素在z轴上的堆叠顺序，比如z-index，确保透明、重叠元素能够正确显示，并处理页面内容的渲染层级。

文档根节点和就具有明确的css定位比如relative、absolute、transform 、透明、overflow、mask、filter或者包含3d/加速2d canvas及video元素的节点才会创建。renderlayerf不参与最终的gpu渲染，它的树结构是为了下一步的合成做逻辑准备的。

grapicslayer才是renderlayer的物理存储实现，也是真正作为纹理上传给gpu进行合成的层，负责管理实际的后端存储空间，将renderlayer的内容绘制成位图，最终交给合成器compositor进行gpu合成，输出最终的屏幕图像。一个 RenderLayer 可能对应一个或多个 GraphicsLayer（用于管理前景、背景等不同存储），而 GraphicsLayer 则构成了与 RenderLayer 树并行的另一棵树。

它们协同的核心，就是浏览器为了极致性能而构建的**硬件加速合成化渲染**流程。可以简化为：`DOM Tree` -> `RenderObject Tree` -> `RenderLayer Tree`（逻辑分层）-> `GraphicsLayer Tree`（物理存储）-> GPU 合成

对于实践来说，我们得迎合渲染引擎，实现极致的高性能渲染，核心是想尽一切办法让CSS的变化停留在合成composite阶段，彻底跳过layout和paint环节。如果说你就用with、height、padding、margin、border、top、left、bottom、right、font-size做动画的话，浏览器的每一帧都在经历重排和重绘。

这种情况下我们会把动画交给gpu处理，webkit引擎在处理transform和opacity的时候会把他们提升为一个独立的合成层，叫做compositing layer：

```css
/* 极致性能：不重排、不重绘，仅在 GPU 合成阶段处理 */
.box {
  transform: translateX(100px) scale(1.1);
  transition: transform 0.3s ease;
}
```

如果说一个复杂组件，比如说长列表的其中一项，或者一个复杂的图表，它的内部发生了重排，但是你不喜欢它影响到整个页面，你可以使用contain属性，强制浏览器把该元素以及其子树的布局、绘制与外部隔离：

```css
.card {
  /* 告诉引擎：我内部变化不要影响外面，我也尽量不受外面影响 */
  contain: layout paint style;
  /* 或者直接用最强隔离 */
  /* contain: strict; */
}
```

- **`contain: layout`**：内部布局变化，绝对不会引发外部元素重排。
    
- **`contain: paint`**：内部元素超出边界绝对不会画到外面（类似 overflow: hidden 但不创建 BFC）。
    
- **`content-visibility: auto`**：这是配合 `contain` 的大杀器。对于屏幕外（未滚动到可视区域）的 DOM，浏览器直接**跳过其渲染树构建、布局和绘制**，性能提升极其恐怖。

如果是颜色，背景等必须重绘的内容，我们会尽量减少重绘的面积，用box-shadow来代替外部图片或者大体积元素，避免大面积使用极高模糊度的filter: blur，这个极度消耗gpu性能，尤其是移动端。如果某个元素需要频繁改变样式的话，并且他在屏幕中很小，可以把他的z-index设置为负数或者将其相对定位，避免触发大范围的脏矩形重绘。

由于webkit的布局计算时批量异步处理的，在下一帧执行，因此js操作css容易踩坑，如果你在同一个循环里先修改了样式又立即读取了样式，浏览器为了给你正确的结果，会立即打断异步队列，强制执行重排：

```js
// 极其糟糕的代码：循环内读写交替，导致强制连续重排
for (let i = 0; i < 100; i++) {
  el.style.left = el.offsetLeft + 1 + 'px'; // 读(引发重排) -> 写 -> 读(引发重排)...
}

// 读写分离：先集中读，再集中写
const lefts = [];
for (let i = 0; i < 100; i++) lefts.push(el[i].offsetLeft); // 统一读
for (let i = 0; i < 100; i++) el[i].style.left = lefts[i] + 1 + 'px'; // 统一写
```

优化技巧：

如果说发现某个动画掉帧，你可以认为创建一个单独的渲染层，RenderLayer->GraphicsLayer：

```css
.fast-animation {
  will-change: transform; /* 提前告诉引擎：我马上要变了，把层建好 */
  /* 或者用 3D 变换欺骗引擎 */
  transform: translateZ(0); 
}
```

>这个属性**绝不能滥用**。合成层极其消耗内存，层建得太多，会导致内存爆炸和页面卡顿。**只对即将发生动画、或者频繁变化的元素使用**。


## WebKit 和 Blink 的关系

Google Chrome 早期用的就是 WebKit 引擎，后来 Google 觉得 WebKit 的代码太老了，于是从 WebKit 中 Fork 出了一个分支，命名为 **Blink**（现在 Edge、Opera 等都在用）。  
所以，你在 Chrome 开发者工具（Performance 面板）里看到的绿色 `Layout` 和紫色 `Paint` 块，其底层祖传代码和架构理念，正是来源于 WebKit