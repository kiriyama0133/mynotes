---
date: 2026-10-11
tags:
  - css
---
css有个规则， @layer，同一层叠层内的规则将层叠在一起，这给予了 web 开发者对层叠机制的更多控制

![层叠层优先级示意图](https://mdn.github.io/shared-assets/images/diagrams/css/at-rules/layer-cascade.svg)
带 `!important` 标志的声明，优先于_普通声明_，即无 `!important` 标志的常规声明，过渡具有最高的优先级。

## 层叠图

按 **CSS Cascade 5** 规范，优先级从高到低是：


```
1. 过渡（Transition）声明
2. 重要用户代理声明（!important user agent）
3. 重要用户声明（!important user）
4. 重要作者声明（!important author）
5. 动画（Animation）声明
6. 普通作者声明（Normal author）
7. 普通用户声明（Normal user）
8. 普通用户代理声明（Normal user agent）
```

这是层叠的**最高优先级**。按从低到高排列：

| 优先级   | 来源                 | 说明                |
| ----- | ------------------ | ----------------- |
| 1（最低） | 用户代理样式             | 浏览器默认样式           |
| 2     | 用户普通样式             | 用户在浏览器设置里写的       |
| 3     | 作者普通样式             | 你写的普通 CSS         |
| 4     | 动画（CSS Animations） | `@keyframes` 中的样式 |
| 5     | 作者 `!important`    | 你写的 `!important`  |
| 6     | 用户 `!important`    | 用户写的 `!important` |
| 7（最高） | 用户代理 `!important`  | 浏览器强制样式           |

上面最高是过渡声明，CSS 的层叠顺序首先看**来源与重要性（Origin & Importance）**，`@layer` 只是在**作者样式（Author Styles）** 内部调整优先级的工具，它无法跨越来源的层级去挑战更高优先级的声明。

最容易混淆的是用户声明和作者声明。作者是网页开发者，写CSS/HTML的人，用户是网页分访问者，用户在浏览器设置里自定义的样式，或通过扩展/用户样式表注入的 CSS。

过渡优先级最高这个说法是来自CSS级联里面一个比较特殊的设计，和常规的优先级不是一个维度，过渡不是样式，是状态间插值，color:red -> color:blue 的情况，比如说你加了 transition: color 0.3s，浏览器会在0.3s内持续计算中间值，这个中间值在某一瞬间是 `rgb(128, 0, 128)` 之类的，它**既不是 red 也不是 blue**。如果这个中间值能被其他 CSS 规则覆盖，过渡就会“抖动”或“卡顿”。所以规范规定：**过渡期间计算出来的值，优先级高于任何其他声明，包括 `!important`。**

## 普通作者声明内部优先级

| 优先级 | 来源              | 例子                              |
| --- | --------------- | ------------------------------- |
| 1   | **行内 style**    | `<div style="color: red">`      |
| 2   | **ID 选择器**      | `#header`                       |
| 3   | **类 / 属性 / 伪类** | `.btn`、`[type="text"]`、`:hover` |
| 4   | **元素 / 伪元素**    | `div`、`::before`                |
| 5   | **通配符 / 继承**    | `*`、继承来的值                       |
CSS有一个叫做(a, b, c) 三元组表示法来表示选择器优先级，行内 style 的优先级是 `(1, 0, 0, 0)`，比任何选择器都高，`!important` 的优先级**高于所有普通声明**，包括行内 style。比较规则：**从左到右逐位比，大的赢**。

```
#header .nav a { }     /* (1, 1, 1) */
.nav a { }             /* (0, 1, 1) */
a { }                  /* (0, 0, 1) */
```

常用选择器：

后代选择器： 空格
直接子选择器： >
相邻兄弟： + 
通用兄弟： ~

上面的会改变锚点，在最右边，前面都是限定路径，而has不会，has的锚点在最左侧。

父选择器： :has ，它本质是**关系选择器**，可以做很多以前需要 JS 才能做的事：

```css
/* 选中「内部包含 img 的 div」 */
div:has(img) {
  border: 1px solid #ccc;
}

/* 表单里有 input 聚焦时，给表单加样式 */
form:has(input:focus) {
  border-color: blue;
}

/* 选中「后面跟着 p 的 h2」 */
h2:has(+ p) {
  margin-bottom: 0;
}
如果你想选中一个元素，条件是它左右紧邻的兄弟元素和它标签相同，CSS 没有直接的“左右都判断”选择器，但可以用组合方式实现。
/* 选中一个元素，它的前一个兄弟和后一个兄弟都是同类标签 */
div + div:has(+ div) { ... }

/* 选中「不包含任何子元素的空 div」 */
div:not(:has(*)) {
  display: none;
}
```

然后我再补充一些伪类选择器。

## 其他选择器

| `:first-child`  | 结构位置 | 是否是父元素第一个子元素   |
| --------------- | ---- | -------------- |
| `:last-child`   | 结构位置 | 是否是父元素最后一个子元素  |
| `:nth-child(n)` | 结构位置 | 是否是父元素第 n 个子元素 |
属性选择器CSS中根据HTML元素的属性或者属性值来选中元素的选择器，它不依赖标签名、类名或 ID，而是直接看元素身上“带了什么属性”。


```html
[属性名]           /* 有这个属性就选中 */
[属性名="值"]      /* 属性值完全等于 */
[属性名~="值"]     /* 属性值包含某个单词（空格分隔） */
[属性名|="值"]     /* 属性值等于该值或以该值开头后跟 - */
[属性名^="值"]     /* 属性值以该值开头 */
[属性名$="值"]     /* 属性值以该值结尾 */
[属性名*="值"]     /* 属性值包含该子串 */
<a href="https://example.com">链接</a>
<input type="text" placeholder="用户名">
<img src="logo.png" alt="网站logo">
<div data-role="admin">管理员</div>

/* 1. 有 href 属性的 a 元素 */
a[href] { color: blue; }

/* 2. type 正好是 text 的 input */
input[type="text"] { border: 1px solid #ccc; }

/* 3. alt 里包含 "logo" 这个词（空格分隔的单词） */
img[alt~="logo"] { outline: 2px solid red; }

/* 4. data-role 以 "admin" 开头 */
div[data-role^="admin"] { background: gold; }

/* 5. src 以 .png 结尾 */
img[src$=".png"] { border: 1px solid green; }

/* 6. href 里包含 "example" */
a[href*="example"] { font-weight: bold; }
```

CSS自定义属性： 属性名以 `--` 开头，值可以是任意 CSS 值。

```css
<div class="item" style="--i: 0"></div>
<div class="item" style="--i: 1"></div>
<div class="item" style="--i: 2"></div>
:root {
  --primary-color: #3b82f6;
  --spacing: 16px;
  --i: 1;
}
```

用 `var()` 函数。

```css
<div class="item" style="--i: 0"></div>
<div class="item" style="--i: 1"></div>
<div class="item" style="--i: 2"></div>
.button {
  color: var(--primary-color);
  padding: var(--spacing);
  z-index: var(--i);
}
```

css变量最大优势是运行时可变，可以让主题切换、动态计算、响应式调整变得更加容易：

```js
// 读
const value = getComputedStyle(element).getPropertyValue('--i')

// 写
element.style.setProperty('--i', 3)
```

