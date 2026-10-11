
## 样式
### postcss部分

**`autoprefixer`** 插件比较常用，可以根据browserslist 自动加浏览器前缀（`-webkit-`、`-moz-`），**`postcss-preset-env`** 插件可以帮我们把CSS语法降低为兼容性语法。**`cssnano`** 可以帮我们压缩CSS、去掉空格、注释、合并规则。并且要注意的是postcss的插件是按照数组顺序来配置的·，并且是按照顺序执行，而与之不同的是webpack的loader，是从下往上执行。

postcss处理的是标准css，把标准css给解析成AST，然后让插件改，sass/less是预处理器，处理的是他们自己的语法，编译成css之后postcss才接管。


### tailwind


tailwind3开始 jit就是默认引擎了，它**只生成你用到的类**，而不是先生成全部再“清除”未使用的[](https://stackoverflow.com/revisions/db144099-3e0d-4d40-ac2c-1c2372c9caff/view-source)[](https://github.com/tailwindlabs/tailwindcss/discussions/17570?sort=top#1)。所以 `purge` 选项在 v3 里改名叫 `content`，而且 `mode: 'jit'` 这个配置项**已经不需要了**，写了反而会警告[](https://stackoverflow.com/revisions/db144099-3e0d-4d40-ac2c-1c2372c9caff/view-source)。

从版本4开始，@theme取代了tailwind.config.js，我们填写主题配置都在css里面直接配置

```css
@theme {
	--color-primary: #3b82f6
	--breakpoint-sm: 640px;
    --breakpoint-md: 768px;
    --breakpoint-lg: 1024px; %%自定义断点%%
    --breakpoint-xl: 1280px;
    --breakpoint-2xl: 1536px;
}
```

到时候会自动生成bg-primary或者text-等这些类。

有一点比较重要,tailwind不能动态拼接类名，比如说 `text-${color}-600` 是无效的，因为 JIT 扫描的是静态字符串，它不认识运行时才拼出来的类名。正确做法是用完整的类名做条件判断：

```html
<div :class="error ? 'text-red-600' : 'text-green-600'"></div>
```

如果需要动态值，用任意值语法 `text-[#bada55]`，或者用 CSS 变量 `text-(--my-color)`

tailwindv4 通过@import 'tailwindcss'; 一个命令等价于：
```css
@layer theme, base, components, utilities;

@import "tailwindcss/theme.css" layer(theme);
@import "tailwindcss/preflight.css" layer(base);
@import "tailwindcss/utilities.css" layer(utilities);
```

我们从这四个层级来说的话，越靠后优先级越高，我们经常用theme来设计颜色、间距、字体、断体等，会自动生成对应的工具类。
base用来元素重置和全局基础样式，也可以叫prelight，需要重置 HTML 元素默认样式时，比如去掉标题的默认字号、列表的圆点。
components是可复用可覆盖的组件类，封装像 `.btn`、`.card` 这样的组件模式，希望被工具类覆盖时
utilities是原子化工具类比如说flex bg-red-500等，日常写样式的主要方式，优先级最高，能覆盖前面的所有层。css有个规则， @layer，同一层叠层内的规则将层叠在一起，这给予了 web 开发者对层叠机制的更多控制

![层叠层优先级示意图](https://mdn.github.io/shared-assets/images/diagrams/css/at-rules/layer-cascade.svg)
带 `!important` 标志的声明，优先于_普通声明_，即无 `!important` 标志的常规声明，过渡具有最高的优先级。按 **CSS Cascade 5** 规范，优先级从高到低是：

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
看起来像个镜面对称一样，user agent / user/ author | author/ user/ user agent

这里提一嘴用户代理和过渡声明：

用户代理是浏览器自带的默认样式表，你写一个没加任何 CSS 的 HTML：

```html
<h1>标题</h1>
<p>段落</p>
<ul><li>列表项</li></ul>
```

浏览器会自己加上h1的字号、字重、上下margin，p的上下margin，ul的左padding和原点，body的8px margin，因此我们引入@base的时候就会移除默认样式，当然，我们可以在@layer base中加入一些自己的基础样式，比如说：

```css
@import "tailwindcss";

@layer base {
  h1 {
    @apply text-3xl font-bold;
  }
  body {
    @apply bg-gray-50 text-gray-900;
  }
}
```


然后呢，过渡优先级最高这个说法是来自CSS级联里面一个比较特殊的设计，和常规的优先级不是一个维度，过渡不是样式，是状态间插值，color:red -> color:blue 的情况，比如说你加了 transition: color 0.3s，浏览器会在0.3s内持续计算中间值，这个中间值在某一瞬间是 `rgb(128, 0, 128)` 之类的，它**既不是 red 也不是 blue**。如果这个中间值能被其他 CSS 规则覆盖，过渡就会“抖动”或“卡顿”。所以规范规定：**过渡期间计算出来的值，优先级高于任何其他声明，包括 `!important`。**

如果说想要在css里模拟wpf的动画控制，可以使用web Animation API WAAPI来实现，WAAPI 是浏览器提供的**命令式动画 API**，更接近 WPF：

```js
const animation = element.animate(
  [{ transform: 'translateX(0)' }, { transform: 'translateX(100px)' }],
  { duration: 300, easing: 'ease' }
);

// 检测
console.log(animation.playState);  // 'running' | 'paused' | 'finished'

// 打断
animation.cancel();

// 暂停、继续
animation.pause();
animation.play();

// 监听
animation.onfinish = () => { /* ... */ };
```

`@layer` 定义的是**作者样式内部**的优先级。同一层级的作者样式里，`@layer` 的规则是：

- **未分层的样式优先级最高**（也就是没加 `@layer` 的普通 CSS）。
    
- **分层样式按声明的顺序**，后声明的层优先级更高。
    
- 层级内部的样式，再按常规的选择器优先级比较。

```
@layer base, components, utilities;

@layer utilities {
  .text-red { color: red; }
}

@layer components {
  .text-red { color: blue; }
}

.text-red { color: green; }  /* 无层级，优先级最高 */
```

所以，如果想要两个一起使用，我们就得把css也一起放入@layer中，而如果是手写的css组件样式，希望被tailwind工具类覆盖，就得放入@layer components中。
```css
@import "tailwindcss";

@layer components {
  .card {
    @apply rounded-lg bg-white p-4 shadow;
  }
}
```

有个很有趣的争议。叫做@apply的到底该不该用，@apply会把原子类的样式内联到自定义css内， 如果到处用，CSS 体积会膨胀，而且失去了原子化“组合灵活”的优势。所以我们只会在比较高频使用的重复类组合比如.btn来使用，或者需要**给第三方库的类注入样式**。

### CSS


普通作者声明内部优先级从高到低是：

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

/* 选中「不包含任何子元素的空 div」 */
div:not(:has(*)) {
  display: none;
}
```

伪类和伪元素（常和关系选择器配合）

|类型|写法|含义|
|---|---|---|
|伪类|`:hover`|鼠标悬停|
|伪类|`:focus`|获得焦点|
|伪类|`:first-child`|第一个子元素|
|伪类|`:last-child`|最后一个子元素|
|伪类|`:nth-child(n)`|第 n 个子元素|
|伪类|`:not(selector)`|排除某个选择器|
|伪元素|`::before`|元素前插入内容|
|伪元素|`::after`|元素后插入内容|
|伪元素|`::placeholder`|输入框占位符|

@namespace是CSS中一个少用，但是特定场景下需要使用的一个规则，作用是**给选择器里的元素和属性指定 XML 命名空间**。

在 HTML 里，元素名是全局的，`div` 就是 `div`。但在 XML 或 SVG 里，不同命名空间可以有同名元素

```xml
<html:div>HTML 的 div</html:div>
<svg:div>SVG 的 div</svg:div>
```

如果直接用 `div` 选择，浏览器不知道你要选哪个命名空间下的 `div`。`@namespace` 就是用来区分它们的。`@namespace` 必须写在所有样式规则之前（除了 `@charset` 和 `@import`）。

```
/* 给某个命名空间起别名 */
@namespace svg url(http://www.w3.org/2000/svg);

/* 默认命名空间，没有前缀的选择器都属于它 */
@namespace url(http://www.w3.org/1999/xhtml);

/* 之后用 `svg|` 前缀来指定该命名空间 */
svg|circle {
  fill: red;
}
```

HTML5 允许在 `<html>` 里直接写 `<svg>` 和 `<math>`，浏览器会自动给它们分配 SVG 和 MathML 命名空间，这里**不需要**写 `xmlns`，浏览器知道 `<svg>` 是 SVG 命名空间。但在独立 SVG 文件里，`xmlns` 是必须的。

```
<svg width="100" height="100">
  <circle cx="50" cy="50" r="40" fill="red" />
</svg>
```

这里需要了解一个机制，html其实是有命名空间的，HTML5 和 XML 是两套不同的解析模型，只是语法上有相似之处，HTML5 文档里，元素可以属于三个命名空间：

| 命名空间 URI                             | 前缀      | 用在哪些元素                          |
| ------------------------------------ | ------- | ------------------------------- |
| `http://www.w3.org/1999/xhtml`       | 无（默认）   | 所有 HTML 元素（`div`、`p`、`span`...） |
| `http://www.w3.org/2000/svg`         | `svg:`  | `<svg>` 及其子元素                   |
| `http://www.w3.org/1998/Math/MathML` | `math:` | `<math>` 及其子元素                  |
|                                      |         |                                 |

**HTML 解析器是“预定义”的。** 规范里明确写了：当解析器遇到 `<svg>` 标签时，**自动切换到 SVG 命名空间**，直到遇到 `</svg>` 闭合。

在DOM中，属性分为两个类型，一个叫做无命名空间属性，一个叫做有命名空间属性。 前者是 属性名就是普通字符串，没有命名空间，`checked`、`disabled`。 后者属于有命名空间的属性，属性名带前缀，属于某个命名空间，xml:lang，xlink:href。

你可以在devtools中验证：

```js
const input = document.querySelector('input')
input.checked = true

// 查看属性节点
const attr = input.getAttributeNode('checked')
console.log(attr.namespaceURI)  // null
console.log(attr.name)          // "checked"
console.log(attr.localName)     // "checked"
console.log(attr.prefix)        // null
```

那么真正没有默认命名空间，需要xmlns的文档是哪些文件，就比如说Avalonia的axaml。
```
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
```

CSS自定义属性： 属性名以 `--` 开头，值可以是任意 CSS 值。


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

用 `var()` 函数。

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


css的其他重要知识点：

z-index 不是全局的，是在一个层叠上下文里比较，层叠上下文是`relative` / `absolute` / `fixed` / `sticky`，且 `z-index` 不是 `auto`，父元素有transform，子元素的z-index: 9999 压不过父元素外面元素，子元素在父元素的层叠上下文里。

我们可以通过设置position + z-index来创建一个层叠上下文，比如说position: relative + z-index:1 ，或者你可以直接设置position: fixed/sticky也是不需要z-index的。或者opacity < 1 ；trnsform != none 比如transform: translateX(0)；Isolation: isolate （专门创建层叠上下文)。

#### BFC

bfc全称块级格式化上下文，是一个独立的渲染区域，内部布局不影响外部，创建bfc的方法有：

1. overflow 不是 Visible
2. display: flow-root
3. position: absolute/fixed
4. float不是none
5. display: inline-block /flex/ grid

能够解决： 清除浮动，父元素创建bfc，保住浮动子元素。防止margin塌陷，两个相邻的bfc的margin不会合并。阻止元素被浮动元素覆盖。**`display: flow-root`**，它是专门为“创建 BFC 且不影响其他布局”设计的，比 `overflow: hidden` 干净。

margin塌陷，垂直方向两个margin会合并，取比较大那个：

```css
.a { margin-bottom: 20px; }
.b { margin-top: 30px; }
```

父子塌陷，父元素没有padding、border、overflow等元素情况下，子元素的margin-top会溢出到父元素外面。可以让父元素加 `padding-top` 或 `border-top`或者父元素创建bfc，或者在flex/grid容器里面用gap来替代margin。

Flex和Grid是两种不同的布局思想，flex是内容驱动，子元素决定怎么排列，grid是容器定义网格，子元素填入。前者适合的是导航、工具栏、卡片列表，后者适合页面整体布局 复杂网格。CSS的正常布局模式中包含以下几种：
	block布局，叫做块级，inline布局 行内布局， inline-block、Flex布局、Grid布局、定位布局(position)、浮动布局(float)、表格布局(table)
flex grid会改变元素的外部显示类型和内部格式上下文，我们称元素对外表现叫block还是inline，对内部子元素按什么规则排列。
比如你想让元素对内部flex，对外部block，可以这么写:

```css
display: block flex; %%对外block 对内flex%%
display: inline-flex %%对外inline，对内flex%%
```

这里真的有坑，display: flex 是 display: block flex的简写，表示对外block，对内flex。display: block才是display: block flow的简写，表示对外block，对内flow，子元素会受到float的影响，会发生margin塌陷。

ok 我们来谈谈塌陷问题，display: block表示内部格式化上下文为BFC 块级格式化上下文，display: flex 对内flex，会发生FFC flex格式化上下文。

那么bfc为啥要发生，因为它规则之一是相邻块级元素垂直margin会合并，这个上面讲过，这就是规范行为，目的是让段落之间差距更加自然，ffc呢，flex容器里子元素不是块级元素，而是flex item，这样会让item的margin不会和其他item合并，margin差就是它们的和。

grid也是为ui布局设计的，margin语义是明确间距，因此grid item也不会塌陷。如果你想在bfc里避免margin塌陷，就给父元素加padding、border，或者让父元素创建bfc，display: flow-root、用gap替代，其中gap替代是最推荐的，gap是flex、grid原生属性，不受margin塌陷影响。

最后我们再把inline单独拿出来说，inline对应的是ifc 行内格式化上下文，和bfc并列。inline元素会像文字一样排列在一行里，从左到右排列，一行放不下就换行，并且不能设置width height，宽高由内容决定，垂直margin不生效，水平margin是生效的，只是无法撑开行高。每一行文字，浏览器都会创建一个叫做linebox行盒的东西，行盒的高度由这一行最高的元素决定，行盒的垂直对齐vertical-align只有在ifc中有效，在flex、grid中无效，用align-item替代。

常见的inline元素有span 、a、strong、em、img、button、input。

inline-block是折中的方式，表示对外inline，对内flow-root，能设置inline不能设置的宽高。ifc会合并连续空格、换行。

```
<span>a</span>
<span>b</span>
```

这样的渲染出来是a b， 中间只有一个空格，因为html源码里的换行被当成一个空格，如果不想要有空隙，可以压缩到一行去，或者用flex布局，flex不处理whitespace，相对来说，flex默认不换行，如果需要换行，就要设置flex-wrap，flex-wrap: wrap。

然后我们来讲脱离文档流的程度，float是半脱离，absolute是完全脱离，float是原来位置不保留，后面的元素会上移，absolute也是，后面的元素会上移。但是float会影响周围文字，比如说文字环绕，但是absolute是不影响的，文字不会环绕，float不积压周围块级元素，但是会被其他块级元素无视，absolute则是完全独立，不积压。所以说float是保留空间影响，absolute是完全去掉影响，完全脱离文档流。

最后，我们以行/列轨道概念结尾，**`align-content` 操作的是“行/列轨道”，不是单个元素。** 在Flex和Grid中，有两个层级，分别是轨道和项目，英文叫做track和item，比如说flex默认一行轨道，当flex-wrap: wrap时候有多行。

flex里有四个对齐属性，一个叫做justify-content，用于所有item整体，作用方向是主轴；一个叫做align-item，用于处理单个item，方向是交叉轴；align-content作用于track，作用方向是交叉轨道；align-self作用于单个item覆盖align-items，方向是交叉轴。

grid中，justify-content处理整个网格，为行轴；align-content处理整个网格，方向为列轴；justify-items作用于每个item在单元格内，行轴；align-items处理每个item在单元格内，方向列轴。

#### 单位

| 单位              | 参照物             | 适用      |
| --------------- | --------------- | ------- |
| `px`            | 绝对像素            | 边框、小间距  |
| `rem`           | 根元素 `font-size` | 全局缩放、字号 |
| `em`            | 父元素 `font-size` | 局部相对缩放  |
| `%`             | 父元素对应属性         | 宽度、高度   |
| `vw` / `vh`     | 视口宽/高           | 全屏布局    |
| `vmin` / `vmax` | 视口较小/较大边        | 正方形自适应  |
| `ch`            | 字符 `0` 的宽度      | 输入框宽度   |
| `fr`            | Grid 剩余空间       | Grid 布局 |

#### 伪元素

||伪类|伪元素|
|---|---|---|
|语法|单冒号 `:hover`|双冒号 `::before`|
|作用|描述元素的状态|创建虚拟元素|
|例子|`:hover`、`:focus`、`:nth-child`|`::before`、`::after`、`::placeholder`|
|是否在 DOM 里|不创建新元素|创建虚拟元素|

**面试问**：“`:before` 和 `::before` 有什么区别？”答“单冒号是 CSS2 语法，双冒号是 CSS3 语法，现代写法用双冒号”。

#### `display: none` vs `visibility: hidden` vs `opacity: 0`

||占据空间|可交互|触发重排|触发重绘|
|---|---|---|---|---|
|`display: none`|❌|❌|✅|✅|
|`visibility: hidden`|✅|❌|❌|✅|
|`opacity: 0`|✅|✅|❌|✅|

**面试问**：“三者区别？”答“`display: none` 完全移除，`visibility: hidden` 保留空间但不可交互，`opacity: 0` 保留空间且可交互”。

#### 重排和重绘

||重排（Reflow）|重绘（Repaint）|
|---|---|---|
|触发|几何属性变化|颜色、背景变化|
|代价|高|低|
|属性|`width`、`height`、`margin`、`padding`、`display`、`position`|`color`、`background-color`、`visibility`|
|优化|用 `transform`、`opacity`（只触发合成）|—|

**面试问**：“怎么优化动画？”答“用 `transform` / `opacity`，加 `will-change`，避免触发重排”。

#### 滚动相关

| 属性                        | 作用             |
| ------------------------- | -------------- |
| `overflow`                | 控制溢出           |
| `overscroll-behavior`     | 控制滚动边界（防止滚动穿透） |
| `scroll-snap-type`        | 滚动吸附           |
| `scroll-behavior: smooth` | 平滑滚动           |
| `position: sticky`        | 粘性定位           |
| `scrollbar-gutter`        | 预留滚动条空间，避免布局抖动 |
**面试问**：“怎么做滚动吸附？”答“`scroll-snap-type` + `scroll-snap-align`”。没有滚动吸附时，用户滑动轮播图，可能停在两张图中间，一半露一张、一半露另一张，视觉上很别扭。

怎么处理移动端上，1px变成2px的问题，根源是因为设备像素比DPR，高DPR屏幕上，1CSS像素可能对应着2、3个甚至更多的物理像素，导致1px的边框在物理上渲染成了2px或者3px宽。问题根源：

```css
border: 1px solid #ccc;
```

DPR = 1 : 1个CSS像素=1个物理像素
**DPR = 2**：1 个 CSS 像素 = 2×2 个物理像素，1px 边框在物理上占 2 个像素
这个时候可以试试伪元素绘制边框，再缩放：

```css
.border-1px {
  position: relative;
}

.border-1px::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 200%;
  height: 200%;
  border: 1px solid #ccc;
  transform: scale(0.5);
  transform-origin: 0 0;
  pointer-events: none;
}
```

## Pinia

pinia的核心原理是使用vue的核心响应式系统包装一个状态对象，用闭包管理store实例，这个和composable的思想是一致的，用插件机制扩展能力。

`defineStore(id, options)` 返回一个 **`useStore` 函数**，不是 store 本身。


```js
const useCounter = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: {
    double: (state) => state.count * 2,
  },
  actions: {
    increment() { this.count++ },
  },
})
```

`useStore()` 内部做的事，会检查pinia实例上有没有这个id的store，有的话就返回，没有就创建，它会把state变成reactive，把getters变成computed，把actions挂载到store上，然后存进一个叫做`pinia._s` 的Map，key是store id。所以store是单例的，同一个pinia实例里，useCounter()多次调用都只返回同一个对象。

`state` 是一个 **`reactive` 对象**。所有对 state 的读写都经过 Vue 的 Proxy，触发依赖收集和更新，注意这里state是一个Function，而非一个obj，因为store可能被多个组件实例创建，如果state是对象，那么多个store会共享同一个引用，函数每次调用返回新对象，保证每个store都有独立的状态。

### SSR隔离

前面我们提到每个请求都有独立的pinia实例，这个是ssr安全的关键：

```
// 服务端每次请求
const pinia = createPinia()
app.use(pinia)

// 客户端激活时
const pinia = createPinia()
app.use(pinia)
```

在nuxt中，每个请求都会创建新的pinia实例，客户端从payload中恢复store状态。Nuxt 渲染页面时，如果组件在 `setup` 里调用了 `useStore()`，**服务端会执行这个 `setup`**，于是 `useStore()` 也会在服务端执行，store 实例就被创建了。

```vue
<script setup lang="ts">
const userStore = useUserStore()  // 服务端和客户端都会执行
</script>
```

如果store里面有state初始化，getter计算，甚至action里的异步请求，这这些都会在服务端里跑一次。`@pinia/nuxt` 模块会自动处理，让每个 SSR 请求有独立的 `pinia` 实例。`pinia._s` 是请求级的 Map，不同请求的 store 不会互相污染。

## HTML

### Src 和 href 的区别

src是替换当前元素，href是建立关联，src会把外部资源嵌入到当前的元素的位置，替换掉这个元素，它的加载行为：浏览器会阻塞解析，等资源下载完再继续，因为资源是当前文档的一部分，必须等到位。

```html
<img src="a.jpg">
<script src="a.js"></script>
<iframe src="page.html"></iframe>
<video src="a.mp4"></video>
<audio src="a.mp3"></audio>
```

href作用是**建立当前文档和尾部资源的关联**，不会替换元素本身，加载行为：浏览器不阻塞解析，并行下载，资源是关联，非当前文档一部分。

### iframe标签的优缺点

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

### Canvas和SVG

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

### TypeScript的DOM类型定义

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


### 虚拟列表

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

## 浏览器

多标签通信办法，在同源策略下，可以使用BroadcastChannel API，这个是专门为了这个场景而设计的，比localStorage事件、SharedWorker这些老方案更加干净。

```js
const channel = new BroadcastChannel('my-channel')
```

通用的通信逻辑可以放在composables这种，具体的业务响应可以放在组件里：

```ts
// composables/useBroadcast.ts —— 通用封装
export function useBroadcast<T>(name: string) {
  const channel = shallowRef<BroadcastChannel | null>(null)
  const handlers = new Set<(data: T) => void>()

  onMounted(() => {
    channel.value = new BroadcastChannel(name)
    channel.value.onmessage = (e) => {
      handlers.forEach(h => h(e.data))
    }
  })

  onUnmounted(() => {
    channel.value?.close()
  })

  function post(data: T) {
    channel.value?.postMessage(data)
  }

  function on(handler: (data: T) => void) {
    handlers.add(handler)
    onUnmounted(() => handlers.delete(handler))
  }

  return { post, on }
}
```

### localStorage加密邪教技术

cookie和LocalStorage是web开发中最常用的两个存储方式，cookie主要保存在内存中，生命周期和浏览器相关，浏览器关闭之后直接消失（会话cookie），持久cookie是保存在硬盘的特定文件里，生命周期是设置的过期时间。

在Chrome这种浏览器中，持久化cookie存储在用户数据目录里面的cookies SQLITE数据库里，例如 C:\Users\用户名\AppData\Local\Google\Chrome\User Data\Default\cookies

然后看看localStorage，这个始终是存储在硬盘里的独立区域的，Chrome呢，存储在User Data\Default\Local Storage\目录里，- **Firefox**：存储在 `webappsstore.sqlite` 文件中。

cookie的原理是http状态管理，服务器在http响应头里可以加Set-Cookie，浏览器收到后就会保存，每一次向同源服务器发请求的时候浏览器都会自动把Cookie塞进Cookie请求头里，每个Cookie最大4KB，每次请求都会携带，太大会拖慢网络，他有安全属性，HttpOnly让JS读不到，Secure只走HTTPS，SameSite可以控制跨站发送。

localStorage的原理是Web Storage API，是纯客户端，数据只会在浏览器本地，永远不会自动发给服务器，也是按源来隔离的，提供了一些同步API，可以通过setItem、getItem、removeItem等方法直接读写键值对。容量更大，通常为5MB左右，远大于Cookie的4KB，只存字符串，存对象需要先JSON.stringify，取出来再JSON.parse。

聊完了之前技术，有一个好玩的东西，有些库`secure-ls`、`@cyberadityacode/secure-ls` 等可以让我们在localSotrage中存储加密数据，用起来和普通的localStorage差不多，原理是在存入前加密，库会封装setItem和getItem，传入的对象会被AES之类的算法加密成密文，再存进localStorage中，读取的时候自动解密，给我们原始对象。

```ts
// 普通 localStorage
localStorage.setItem('user', JSON.stringify(user))

// 加密库的用法（API 类似，但底层自动加解密）
await storage.setItem('user', user)  // 存进去的是密文
const user = await storage.getItem('user')  // 取出来是明文对象
```

### Service Worker

HTML5的离线存储，也叫做Application Cache，也在已经废弃，继任者叫做Service Worker，ServiceWorker是一个在浏览器后台运行的独立脚本，充当网络请求的代理，可以拦截页面所有请求，并决定是返回缓存资源还是去网络获取，使用分为三步：

注册、安装于缓存、拦截和响应。

注册会在js中注册一个service worker的脚本，安装和缓存会在serviceWorker的install事件中，将需要离线使用的核心资源比如html css js加入到缓存，拦截和响应会在fetch事件中，拦截页面的网络请求，可以制定策略，比如说缓存优先，看看有没有资源，没有再去请求网络，或者网络优先，先请求，失败再回退缓存。

相比 AppCache，Service Worker 提供了**更精细、更灵活的控制能力**。你可以精确管理缓存的版本、更新策略，并且它完全通过 JavaScript 编程实现，不再依赖那个容易出错的 manifest 文件。

service worker不会阻塞主线程，也不能访问dom，靠事件唤醒，空闲时候被终止，能够拦截fetch，再决定怎么响应。注册之后可以长期存在，页面关了也在，必须https或者localhost，只能控制注册路径和子路径。

有service worker之后可以决定

```
页面 → 浏览器 → Service Worker（拦截）→ 决定：
                    ├─ 用缓存响应
                    ├─ 走网络
                    └─ 自定义响应
```

Service Worker 的生命周期和普通脚本完全不同，分几个阶段：

首先是注册：

```js
// 主线程
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')
        .then(reg => console.log('注册成功', reg))
        .catch(err => console.log('注册失败', err));
}
```
注册的时候文件不会加载，不会立即生效，然后install的时候才会。

```js
// sw.js
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open('v1').then(cache => {
            return cache.addAll([
                '/',
                '/index.html',
                '/style.css',
                '/app.js'
            ]);
        })
    );
});
```

install事件在sw首次注册的时候或者文件内容变化的时候触发，通常在预缓存静态资源。 `event.waitUntil(promise)`让浏览器"等这个 Promise 完成再算安装成功"。然后是activate激活阶段：

```js
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then(keys => {
            return Promise.all(
                keys.filter(k => k !== 'v1')
                    .map(k => caches.delete(k))   // 清理旧缓存
            );
        })
    );
});
```

`activate` 在安装成功后、SW 正式接管页面前触发。通常在这里**清理旧版本缓存**。最后是接管：

```js
self.addEventListener('activate', (event) => {
    event.waitUntil(self.clients.claim());   // 立即接管
});
```
默认情况下，sw安装不会立即开始接管已经打开的窗口，需要等待页面刷新，使用这个情况我们可以在这里激活后直接接管。
之后service worker就可以帮我们来完成一系列工作了，比如说拦截请求：

```js
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then(cached => {
            // 缓存命中 → 用缓存
            if (cached) return cached;
            // 没命中 → 走网络
            return fetch(event.request);
        })
    );
});
```

我们可以在这里设置一个缓存优先策略/网络优先策略，这个是fetch事件的常见写法，适合需要最新数据但是也要离线兜底的接口。

```js
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then(cached => {
            return cached || fetch(event.request);
        })
    );
}); // 缓存

self.addEventListener('fetch', (event) => {
    event.respondWith(
        fetch(event.request)
            .then(response => {
                // 网络成功，更新缓存
                const clone = response.clone();
                caches.open('v1').then(cache => cache.put(event.request, clone));
                return response;
            })
            .catch(() => caches.match(event.request))   // 网络失败用缓存
    );
});
```

如果是能够短暂接受旧数据，但是还是想尽快响应的，场景，可以用缓存优先+后台更新：

```js
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then(cached => {
            const fetchPromise = fetch(event.request).then(response => {
                const clone = response.clone();
                caches.open('v1').then(cache => cache.put(event.request, clone));
                return response;
            });
            return cached || fetchPromise;   // 立即返回缓存，后台更新
        })
    );
});
```

sw里面可以用CacheStorage来管理缓存：

```js
// 打开/创建缓存
const cache = await caches.open('v1');

// 添加
await cache.add('/index.html');
await cache.addAll(['/a.js', '/b.css']);
await cache.put(request, response);   // 手动放入

// 查询
const response = await cache.match('/index.html');
const responses = await cache.matchAll();

// 删除
await cache.delete('/index.html');

// 列出所有缓存
const keys = await caches.keys();

// 删除整个缓存
await caches.delete('v1');
```

sw和页面不能直接通信，需要使用postMessage：

```js
// 主线程
navigator.serviceWorker.controller.postMessage({
    type: 'SKIP_WAITING'
});

navigator.serviceWorker.addEventListener('message', (event) => {
    console.log('收到 SW 消息', event.data);
});
```

这个架构设计其实普遍常见，可以看做后端架构里的Worker，比如说Cloudflare Worker或者 Web Workers、后台任务队列：
- **独立执行单元**
    
- **事件驱动**
    
- **无状态或短生命周期**
    
- **通过消息通信**
    
- **不直接访问主进程资源**
**Service Worker 完全符合这个模型**。

提一嘴，`CacheStorage` 存在**磁盘**上，和 `IndexedDB`、`localStorage` 一样是持久化存储。

在vite项目中，我们可以使用vue专用sw包装的插件vite-plugin-pwa，它通过一个虚拟模块 `virtual:pwa-register/vue`，提供了一个专门给 Vue 3 用的 `useRegisterSW` 函数：

```vue
<script setup lang="ts">
import { useRegisterSW } from 'virtual:pwa-register/vue'

const {
  offlineReady, // 应用准备好离线使用（Ref<boolean>）
  needRefresh,  // 有新版本可用（Ref<boolean>）
  updateServiceWorker, // 更新 SW 并刷新页面（函数）
} = useRegisterSW({
  onRegisteredSW(swUrl) {
    console.log(`SW 已注册: ${swUrl}`)
  },
})
</script>

<template>
  <div v-if="offlineReady || needRefresh" class="pwa-toast">
    <span v-if="offlineReady">应用已可离线使用</span>
    <span v-else>有新内容可用，点击刷新</span>
    <button v-if="needRefresh" @click="updateServiceWorker()">刷新</button>
  </div>
</template>
```

### Web Worker

刚才提到的service worker常用于网络代理层，web worker常用于后台计算线程，能够把耗时计算移出主线程，生命周期是随着页面的创建和销毁。这个是html5提供的一个api，允许我们在后台线程运行js代码，从而避免主线程，它是浏览器实现多线程的标准方式。

因为Js是单线程，主线程会负责执行js代码，处理dom渲染，响应用户交互。如果主线程执行一个耗时任务比如大量计算、处理大数组，页面就会卡死，用户点击没反应、动画掉帧。Web Worker 把耗时任务放到**独立线程**，主线程继续响应用户，两者互不阻塞。

```js
// 1. 创建 Worker
const worker = new Worker('worker.js');

// 2. 发送数据给 Worker
worker.postMessage({ type: 'compute', data: [1, 2, 3, 4, 5] });

// 3. 接收 Worker 的结果
worker.onmessage = (e) => {
    console.log('结果:', e.data);
};

// 4. 监听错误
worker.onerror = (err) => {
    console.error('Worker 错误:', err.message);
};

// 5. 终止 Worker
// worker.terminate();

// worker.js
// 接收主线程的消息
self.onmessage = (e) => {
    const { type, data } = e.data;

    if (type === 'compute') {
        // 执行耗时计算
        const result = data.reduce((sum, n) => sum + n * n, 0);

        // 把结果发回主线程
        self.postMessage({ type: 'result', value: result });
    }
};
```


`postMessage` 支持两种传输方式，分别是结构化克隆、转移所有权。默认是结构化克隆：

```js
worker.postMessage({
    num: 1,
    str: 'hello',
    arr: [1, 2, 3],
    obj: { a: 1 },
    map: new Map([['k', 'v']]),
    set: new Set([1, 2]),
    date: new Date(),
    buffer: new ArrayBuffer(8)
});
```

不支持dom节点、error对象（部分）、函数。

转移所有权有点像c++的移动move概念。对于大块数据比如ArrayBuffer，可以用转移替代复制，实现零拷贝：

```js
const buffer = new ArrayBuffer(1024 * 1024 * 100); // 100MB

// 转移所有权，主线程不再拥有该 buffer
worker.postMessage({ buffer }, [buffer]);

// 此时 buffer 在主线程已不可用
console.log(buffer.byteLength); // 0
```

处理大数组、图像数据、音视频时，转移比复制快得多。典型的使用场景就有wasm，我们一般就是在worker中运行wasm模块的。可以用于图像处理、像素操作、canvas离屏渲染。还能用于轮询和长连接，后台定时请求、websocket心跳，这也是为了主线程不会去做大json解析、二进制处理处理占用主线程。

### IndexedDB

这个是浏览器的嵌入式数据库，用kv存储，支持索引、事务、游标。能存储任意结构化数据，localStorage只能存储5-10MB，但是这个可以存储上GB的信息量。一个indexedDB可以创建多个数据库：

```
const request = indexedDB.open('myDB', 1);   // 名字 + 版本号
```

他可以使用对象仓库，类似于表：

```
db.createObjectStore('users', { keyPath: 'id' });
```

可以使用索引加速查询：

```
store.createIndex('email', 'email', { unique: true });
store.createIndex('age', 'age');
store.createIndex('name_age', ['name', 'age']);   // 复合索引
```

支持事务，能确保原子性和自动提交，出错能自动回滚：

```
const tx = db.transaction('users', 'readwrite');
const store = tx.objectStore('users');
store.add({ id: 1, name: 'Alice' });
tx.oncomplete = () => console.log('提交成功');
tx.onerror = () => console.log('失败');
```

### Ajax技术

Ajax是一组技术的组合，用于在不刷新页面的前提下用js异步向服务器请求数据并且更新页面的方法。

我们已知，HTML/CSS用于展示内容，DOM动态更新页面，JS用于逻辑控制，那么XMLHttpRequest可以用于一步请求，XML/JSON可以用于数据展示。

我们都知道现代ajax技术经过了发展从XMLHttpRequest变成了fetch，又变成了axios。XMLHttpRequest是1999发明，es6之后，发布了fetch。

### PWA

**Progressive Web App** 是一组能力的集合，核心是Service Worker + Web App Manifest。他可以做到离线可用，断网也可以打开，可安装，桌面/主屏有图标，独立窗口。可以推送通知，可以后台同步。

service worker之前讲过，是一个跑在浏览器后台独立线程的脚本，可以拦截页面的网络请求、缓存资源、离线响应。


### 浏览器缓存技术

浏览器本身给我们提供了一些缓存技术，最基础的缓存是HTTP响应头机制，通过强缓存可以将带内容哈希的静态资源比如app.a1b2c3.js缓存。

Cache-Control: 这个头我们在缓存里最常用，可以设置max-age= 单位s来以高优先级设置缓存资源，也可以用immutable设置资源永不变，刷新也不请求。头部可以设置Expires来设置绝对时间过期，优先级比较低。

Cache-Control: public, max-age=31536000, immutable 比如这样。

浏览器可以发送请求询问服务器是否可用，这个叫做协商缓存：

`ETag` / `If-None-Match` 可以基于内容哈希，精度比较高，`Last-Modified` / `If-Modified-Since`基于时间戳，精度比较低。响应为304 Not Modified，不传响应体，比较节省带宽。 ETag是服务器为了某个资源生成的唯一标识符，通常基于文件内容计算比如哈希值，这样客户端在下次请求的时候就会通知服务器本地存有这个版本，并判断是否资源最新的。

这是协商的完整过程：

```
第一次请求：
  客户端 ──GET /index.html──→ 服务器
  客户端 ←──200 OK────────── 服务器
              ETag: "abc123"
              Cache-Control: no-cache
// - `no-cache` 表示**每次都要向服务器确认**，不能直接用本地缓存。

第二次请求：
  客户端 ──GET /index.html──→ 服务器
              If-None-Match: "abc123"
  客户端 ←──304 Not Modified── 服务器
              （无响应体）
// 如果没变，返回 `304`，**不传响应体**，节省带宽

如果内容变了：
  客户端 ←──200 OK────────── 服务器
              ETag: "def456"
              （返回新内容）
```


适用于HTML文件、API响应等需要及时更新的资源：

```
Cache-Control: no-cache
ETag: "abc123"
```

vary: Origin表示响应内容依赖请求的Origin头，如果同一个URL被不同 `Origin` 请求，浏览器会分别缓存。这在开发环境下常见，因为 Vite 的 dev server 需要处理 CORS。生产环境下，如果 CDN 配置不当，`vary: Origin` 可能导致缓存命中率下降。

缓存策略组合：

HTML这种资源推荐使用no-cache+ ETag每次协商，js/css推荐用max-age和immutable进行缓存。图片和字体也可以通过max-age+ETag进行缓存。API响应可以通过no-store或者短max-age + ETag进行缓存。

内存缓存是浏览器自己自动管理的，同一页面重复请求同一资源的时候就会直接从内存读取，关闭标签页就会失效。浏览器在收到一个资源的时候，会根据响应头决定存储在哪里。内存缓存的存储位置是RAM，而非DISK，读取速度很快，像js css 图片资源等高频使用的资源可以存，生命周期是当前标签页会话，关闭就失效。磁盘缓存的位置是硬盘，生命周期是由Cache-Control / Expires 决定，读取速度较快，像大文件，不常变动的资源可以缓存。service worker缓存存储的位置是Cache API，独立于HTTP缓存，需要手动控制生命周期，像PWA、离线资源可以存。

```
1. 内存缓存命中？
   ├─ 是 → 直接用（from memory cache）
   └─ 否 → 下一步

2. Service Worker 缓存命中？
   ├─ 是 → 用 SW 缓存
   └─ 否 → 下一步

3. 磁盘缓存命中？
   ├─ 强缓存有效？
   │   ├─ 是 → 直接用（from disk cache）
   │   └─ 否 → 协商缓存（发请求带 If-None-Match）
   │              ├─ 304 → 用本地
   │              └─ 200 → 下载新资源
   └─ 否 → 发请求下载
```

![[Pasted image 20260930221146.png]]

![[Pasted image 20260930221200.png]]

public, max-age=2592000, immutable 会让我们强缓存30天永不变，cf-cache-status: HIT表示这是Cloudflare CDN边缘节点已缓存。etag是协商的缓存标识，last-modified是最后的修改时间。这个组合会让浏览器把图片存入磁盘缓存，下次请求同一URL时候浏览器直接读磁盘，不发网络请求，状态码会显示 `200 (from disk cache)`。

### 标签页和v8

每个标签页其实不会分配一个独立的v8实例，v8实例是绑定在渲染进程RendererProcess上的，而不是直接绑定在标签页上。正确的关系是，一个v8实例可能服务多个标签页和iframe。在早期，确实一个标签页一个进程，现在策略已经变了，如果我们打开两个标签都访问同一个网站比如example.com，chrome可能让它们共享一个渲染进程，也就是共享一个v8实例，用来节省内存。如果网站嵌入了跨站iframe，chrome会为了跨站ifame单独创建一个渲染进程也就是OOPIF，跨进程iframe，一个标签页内部可能对于好几个渲染进程，有好几个v8实例。

可以查看chromium.org内看文档：离线处理架构OOPIFs：

```
这种技术允许页面的子框架由与父框架不同的进程来渲染。采用独立进程内嵌框架的做法主要是出于安全考虑，比如 Site Isolation 项目所提出的解决方案——通过这种方式，即使存在跨域内嵌框架，也能让某个渲染进程专注于处理某个特定的网站内容。不过，独立进程内嵌框架也是一种通用的机制，其应用范围不仅限于安全相关场景，还可以用于其他功能，例如 Chrome 应用程序中的<webview>标签。
```

OOPIF 要求 Chromium 的架构从“以标签页为中心”转向“**以 frame 为中心**”，因为一个 frame 在其生命周期内可能被渲染到不同进程中。浏览器进程现在直接追踪**完整的 frame 树**。`WebContents` 持有一个 `FrameTreeNode` 树，镜像当前页面的 frame 结构。每个 `FrameTreeNode` 包含该 frame 的元信息（名称、origin 等），其 `RenderFrameHostManager` 负责处理该 frame 的跨进程导航，并支持状态复制和消息路由

```
OOPIF 的渲染面临一个核心安全问题：**能合成一个纹理，就意味着能读取它的像素**（Cross Site Texture Stealing）。因此 OOPIF 的完整安全优势依赖于 **Übercompositor** 架构[](https://www.chromium.org/developers/design-documents/oop-iframes/oop-iframes-rendering/)。

在 Übercompositor 下：

- 每个渲染进程的子合成器生成 **CompositorFrame**，包含 quads 和纹理引用。
    
- 渲染进程通过 **IPC 将 CompositorFrame 发给浏览器进程**，而不是直接发给 GPU。
    
- 浏览器进程的父合成器与 GPU 进程交互，完成最终合成[](https://www.chromium.org/developers/design-documents/oop-iframes/oop-iframes-rendering/)。
    

纹理在不同进程间通过 **TextureMailbox** 传递所有权：一个进程用 `glProduceTextureCHROMIUM` 生成纹理，另一个进程用 `glConsumeTextureCHROMIUM` 消费它[](https://www.chromium.org/developers/design-documents/oop-iframes/oop-iframes-rendering/)。
```

这地方了解着玩玩就行，不用急太深，quad是合成器提交给gpu的最小绘制单元，可以理解为一个带纹理矩形，描述了哪块的纹理，用什么变换，绘制到屏幕哪个位置。渲染进程的合成器不直接操作 GPU，而是把一帧画面**拆解成一组 quad**，打包成 `CompositorFrame`，通过 IPC 发给浏览器进程或 GPU 进程。GPU 再按顺序把这些 quad 绘制出来，合成最终画面。

再深入挖掘就是架构图形栈知识了，不讲不讲...

### viz进程架构

viz是chromium渲染管线现代化的核心，主要为了解决GPU进程职责过重，站点隔离下的性能和安全瓶颈，以及图形栈的服务化需求。

在 Spectre 等侧信道攻击的背景下，跨站 iframe 需要严格的进程隔离（OOPIF）。VIZ将核心的合成和光栅化集中到独立进程中，使得来自不同渲染进程的合成器帧可以在一个安全的、收信任环境下统一聚合，从而支持“无额外性能代价的站点隔离”。Viz 是 Chromium 向 Mojo 服务化架构转型的一部分。它将合成、GL、命中测试、媒体等能力封装为标准服务，通过 Mojo 接口对外提供，使得 Browser 进程不再是唯一的核心枢纽，架构上更清晰、可维护性更强。

skia是chromium的底层2D图形库，负责我们几乎我们所有看到的像素绘制，从文字到图形、形状、路径、渐变、阴影、图层混合等。

```
Chromium 图形栈（从上层到下层）：

┌─────────────────────────────────┐
│  Blink（渲染引擎）              │  ← HTML/CSS 解析、布局、绘制指令
├─────────────────────────────────┤
│  cc / viz（合成器）             │  ← 图层合成、光栅化调度、GPU 提交
├─────────────────────────────────┤
│  Skia                           │  ← 2D 绘制：路径、文字、图片、混合
├─────────────────────────────────┤
│  GPU 驱动 / Vulkan / Metal / GL │  ← 硬件加速
└─────────────────────────────────┘
```

## JavaScript

### 0.1 + 0.2 === 0.3 嘛？为什么？

**不等于。** `0.1 + 0.2 === 0.3` 返回 `false`，实际结果是 `0.30000000000000004`，因为 **JavaScript 的数字用 IEEE 754 双精度浮点数表示**，而 `0.1`、`0.2`、`0.3` 这些十进制小数，在二进制里都是**无限循环小数**，无法精确表示。

把十进制的 `0.1` 转成二进制：

```js
0.1 × 2 = 0.2    → 0
0.2 × 2 = 0.4    → 0
0.4 × 2 = 0.8    → 0
0.8 × 2 = 1.6    → 1
0.6 × 2 = 1.2    → 1
0.2 × 2 = 0.4    → 0  ← 回到 0.2，开始循环
...
```

结果是 `0.000110011001100110011...`，无限循环。**双精度浮点数只有 52 位尾数**，存不下无限循环，所以只能**截断或舍入**。存进去的 `0.1` 已经不是精确的 `0.1`，而是一个**非常接近但略有偏差的值**。

JavaScript 的设计目标之一是**简单**。Brendan Eich 在 1995 年设计 JS 时，选择了“一种数字类型”的方案：

- 不用区分 `int` 和 `float`。
    
- 所有数字都是 64 位双精度浮点数。
    
- 整数的最大安全范围是 `2^53 - 1`。


### 运行时类型检测

在js中，在运行时检测对象不能仅仅凭借typeof，因为typeof只能区分原始类型和对象，无法区分具体的对象类型，要判断是不是 Map 这种类型，就需要使用instanceof 或者Object.prototype.toString。


```js
typeof new Map()      // "object"
typeof {}             // "object"
typeof []             // "object"
typeof new Date()     // "object"
typeof new Set()      // "object"
typeof null           // "object"
```

typeof只会返回几种字符串，分别是undefined、object、boolean、number、bigint、string、symbol、function，**除了 `function`，所有引用类型都是 `"object"`。** 所以 `typeof` 无法区分 `Map`、`Set`、`Array`、`Date`。

instanceof检查的是原型链，也就是对象的原型链上有没有这个构造函数的prototype。

```js
new Map() instanceof Map  // true
```

**跨 iframe / 跨 window 失效** ：

```js
// iframe 里的 Map
const iframeMap = iframe.contentWindow.Map

new Map() instanceof iframeMap  // false ❌
```

因为 iframe 有自己的 `Map` 构造函数，`Map.prototype` 不是同一个对象。**原型链上找的是当前 window 的 `Map.prototype`，不是 iframe 的。**

更加可靠的是 使用Object.prototype.toString方法：

```js
Object.prototype.toString.call(new Map())      // "[object Map]"
Object.prototype.toString.call(new Set())      // "[object Set]"
Object.prototype.toString.call([])             // "[object Array]"
Object.prototype.toString.call(new Date())     // "[object Date]"
Object.prototype.toString.call({})             // "[object Object]"
Object.prototype.toString.call(null)           // "[object Null]"
Object.prototype.toString.call(undefined)      // "[object Undefined]"
```

`Object.prototype.toString` 会读取对象内部的 `[[Class]]` 或 `Symbol.toStringTag` 属性，返回 `[object Xxx]`。**跨 iframe 有效** ， 因为 `[[Class]]` 是引擎内部的标记，不受 window 隔离影响。**不受原型链修改影响**：因为不依赖 `prototype`。


```js
export function getType(value: unknown) {
  return Object.prototype.toString.call(value).slice(8, -1).toLocaleLowerCase()
}

console.log(getType(new Map()))

```

### 脚本和模块

在JS中，如果没有export，js就会被当成脚本，但是有export就会当做模块，这是 JavaScript 的一个核心区别。

|维度|Script|Module（ESM）|
|---|---|---|
|**触发条件**|没有 `import` / `export`|有顶层 `import` / `export`|
|**`<script>` 标签**|`<script>`（默认）|`<script type="module">`|
|**作用域**|全局|模块级（文件作用域）|
|**顶层 `this`**|`window`（浏览器）|`undefined`|
|**变量共享**|全局变量挂到 `window`|不挂到 `window`|
|**变量提升**|有|有，但有 TDZ|
|**严格模式**|默认非严格|默认严格|
|**`await` 顶层**|不允许|允许|
|**`import()` 动态导入**|允许|允许|
|**`.js` 默认解析**|Script|Module|

script
```js
// a.js
var foo = 'hello'  // 挂到 window.foo

// b.js
console.log(foo)   // "hello"，能访问
```

module
```js
// a.js
export const foo = 'hello'  // 模块级，不挂到 window

// b.js
console.log(foo)   // ❌ ReferenceError，访问不到
```

判断一个文件是script还是module可以看有没有import/export。也可以看script标签：

```js
<!-- Script -->
<script src="script.js"></script>

<!-- Module -->
<script type="module" src="module.js"></script>
```

也可以看pacakge.json里的type字段，type:module 中.js文件默认当做module，type:commonjs或者没有.js文件默认当做cmj，mjs当做module，cjs当做cjs.

还有一点，我们都知道import esm是异步方法，我们可以看看@types+node的类型定义层面，


```ts
            /**
             * The absolute `file:` URL of the module.
             *
             * This is defined exactly the same as it is in browsers providing the URL of the
             * current module file.
             *
             * This enables useful patterns such as relative file loading:
             *
             * ```js
             * import { readFileSync } from 'node:fs';
             * const buffer = readFileSync(new URL('./data.proto', import.meta.url));
             * ```
             */
            url: string;
```

`import.meta` 在 TypeScript 里的类型定义分几层：

| 类型定义来源                            | 提供什么                                                        |
| --------------------------------- | ----------------------------------------------------------- |
| **TypeScript 内置（`lib.es5.d.ts`）** | `ImportMeta` 空接口                                            |
| **`lib.dom.d.ts`**                | `import.meta.url`（浏览器环境）                                    |
| **`@types/node`**                 | `import.meta.url`、`import.meta.resolve`（Node 环境）            |
| **Vite（`vite/client`）**           | `import.meta.env`、`import.meta.hot`、`import.meta.glob`      |
| **Nuxt**                          | `import.meta.client`、`import.meta.server`、`import.meta.dev` |

我们可以看看Vite / Nuxt 对 `import.meta` 的类型扩展：

```ts
interface ImportMeta {
	browser: boolean;
	client: boolean;
	dev: boolean;
	server: boolean;
	test: boolean;
}
```

这些用来标记代码运行在什么环境。Vite 在构建时把 `import.meta.client` 替换成 `true` 或 `false`，运行时零开销。


### await和事件循环

我们一直说await是等待任务完成，但是原理模型是需要了解的，**`await` 不是“阻塞”，它是“挂起当前函数，让出线程”。**

同步阻塞说的类似于fs.readFileSync这样的node环境方法，线程卡在这里，什么都做不了，await会让当前函数暂停，让线程去做别的事情，也叫做suspending：

```js
// 异步挂起
const data = await fs.readFile('a.txt')  // 当前函数暂停，线程去处理其他任务
console.log('done')                      // 等数据到了，函数恢复执行
```


```js
async function fetchData() {
  console.log('1')
  const data = await fetch('/api')  // 挂起点
  console.log('2')
}

fetchData()
console.log('3')

1. 执行 fetchData，打印 "1"
2. 遇到 await，fetchData 挂起，返回一个 Promise
3. 主线程继续执行，打印 "3"
4. fetch 完成后，fetchData 恢复，打印 "2"
```

我们知道await挂起的是async方法，不是整个程序，await之后的代码会被注册为Promise的.then回调，这个回调进入的是微任务队列，而不是宏任务队列。
**微任务队列**和**宏任务队列** 

| 队列        | 存放内容                                   | 执行时机                 |
| --------- | -------------------------------------- | -------------------- |
| **微任务队列** | Promise.then、await 恢复、MutationObserver | 每个宏任务结束后，**清空所有微任务** |
| **宏任务队列** | setTimeout、setInterval、I/O、UI 渲染       | 每轮事件循环取**一个**宏任务     |

我们知道事件的循环是： 执行调用栈里的同步代码，调用栈清空，开始执行微任务队列，微任务队列清空，执行下一个宏任务，然后继续调用栈清空。

`await` 恢复是微任务，所以**会在调用栈清空后立刻执行**，不会等 UI 渲染或 setTimeout。


```js
console.log('A')

setTimeout(() => console.log('B'), 0)  // 宏任务

Promise.resolve().then(() => console.log('C'))  // 微任务

async function foo() {
  console.log('D')
  await null  // 挂起
  console.log('E')  // 微任务
}

foo()

console.log('F')

A          ← 同步
D          ← 同步
F          ← 同步
C          ← 微任务
E          ← 微任务（await 恢复）
B          ← 宏任务
```

E在B之前，因为await恢复是微任务，setTimeout是宏任务，我们分析就可以知道，微任务队列里面有Promise.then、await恢复，宏任务队列里面有SetTimeout。因此顺序就是同步A同步D同步F微任务C微任务E宏任务B。

>[!tip] **事件循环负责协调调用栈、微任务队列和宏任务队列。同步代码在调用栈中执行；调用栈清空后，先清空所有微任务；然后取一个宏任务执行；如此循环。**


### JS的核心运行机制

刚才讲了同步、异步方法的细节，现在来讲讲执行上下文，这个是理解JS运行原理的核心，await、事件循环、this、变量查找都建立在执行上下文。

执行上下文是代码执行的环境，记录了： 变量环境（var、函数声明）、词法环境（let、const、块级作用域）、this绑定（当前函数的this指向）、外部环境引用（作用域链，用于变量查找）、代码类型（全局、函数、eval）。 每一次函数调用，都会创建一个新的执行上下文。

我们知道，js有三种上下文，分别是全局上下文，这个是脚本加载的时候创建的，是整个脚本的顶层代码。然后是函数上下文，每次调用函数的时候创建。最后是Eval上下文，eval()调用的时候会创建，很少使用。

全局上下文只有一个，函数上下文每次调用都会调用新建。‘


```
1. 创建阶段
   ├── 创建变量环境（var、函数声明）
   ├── 创建词法环境（let、const）
   ├── 确定 this 绑定
   └── 建立作用域链

2. 执行阶段
   ├── 逐行执行代码
   ├── 变量赋值
   └── 函数调用（创建新的执行上下文）

3. 销毁阶段
   └── 函数返回，上下文出栈
```

我举例一段代码：

```
function a() {
  console.log('a')
  b()
}

function b() {
  console.log('b')
  c()
}

function c() {
  console.log('c')
}

a()
```

上下文栈的变化：
```
1. 全局上下文入栈
   [全局]

2. 调用 a()，a 的上下文入栈
   [全局, a]

3. a 里调用 b()，b 的上下文入栈
   [全局, a, b]

4. b 里调用 c()，c 的上下文入栈
   [全局, a, b, c]

5. c 执行完，c 出栈
   [全局, a, b]

6. b 执行完，b 出栈
   [全局, a]

7. a 执行完，a 出栈
   [全局]

8. 全局执行完，全局出栈
   []
```

然后我们来聊聊执行上下文和作用域链的关系：

```js
function outer() {
  const a = 1

  function inner() {
    const b = 2
    console.log(a + b)  // 3
  }

  inner()
}
```

用全局的视角来看，outer函数里包裹一个innre函数，inner的执行上下文（变量环境有b: 2，词法环境有b: 2）里面应该是找不到a的，那么他就会沿着作用域向上寻找，它找到了outer上下文的词法环境（变量环境a:1，外部引用->全局上下文）。

作用域链就是执行上下文的外部引用链，变量查找会沿着这条链向上寻找。

再来深入一点点，刚才我们说了变量环境和词法环境，这两个都是决定变量存放地方和查找、可访问的概念。

```
执行上下文
├── 词法环境（Lexical Environment）
│   ├── 环境记录（Environment Record）
│   │   ├── 声明式环境记录（let、const、class、函数声明）
│   │   └── 对象环境记录（with、global 的 var）
│   └── 外部环境引用（outer）
│
├── 变量环境（Variable Environment）
│   └── 环境记录（主要放 var、函数声明）
│
└── this 绑定
```

词法环境由环境记录、外部环境引用组成，环境记录存放的是let const class 函数声明 catch参数组成。外部环境记录指向外部词法环境，形成了作用域链。全局词法环境的外部引用指向null。块级词法环境的外部引用指向全局词法环境。

var和函数声明进入的是变量环境，是因为var和let/const的作用域规则不同，var的作用域是函数级，变量会提升，初始化为undefined，如果是全局的话，会挂载到window上，let/const作用域是块级，可以提升但是不会初始化（TDZ，这个我们了解过），不会挂载到window。

变量查找顺序

当代码里访问一个变量 `x`：

```
1. 先查当前词法环境的环境记录
2. 找不到 → 查当前变量环境的环境记录
3. 找不到 → 沿词法环境的外部引用往上找
4. 找不到 → 沿变量环境的外部引用往上找
5. 到全局还找不到 → ReferenceError
```

块级作用域实现

```js
{
  let a = 1
  const b = 2
}
```

每进入一个代码块{}，就会创建一个新的词法环境。
```
外层词法环境
  ├── 环境记录: {}
  └── 外部引用: null

内层词法环境（{} 内）
  ├── 环境记录: { a: 1, b: 2 }
  └── 外部引用: → 外层词法环境
```

这个可以多记忆一点，因为块级作用域的本质就是进入块级的时候创建词法环境，离开的时候销毁。let也会创建一个块级词法环境，for循环是let面试经常被问到的，为什么let可以解决var闭包问题。

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0)
}
// 输出：3 3 3
```

var i 是函数级作用域，只有一个i，从结构上来看，全局执行上下文下是变量环境，变量环境里有i: 3（循环结束）。三个setTimeout回调在循环结束后执行，此时i = 3，三个闭包引用的是同一个i。而let在for循环中，每次迭代会创建一个新的词法环境，并复制上一轮的变量值。

>[!tip] **`let` 的“每次声明创建新绑定”语义，既导致了“不能重复声明”，也导致了“`for` 循环每轮新环境”。两者是同一规则的两个表现，不是因果关系。**


函数作用域的实现

```js
function foo() {
  var a = 1
  function bar() {}
}
```

函数在调用时候创建新的执行上下文，foo函数有词法环境、变量环境，这个都知道。然后this指向undefined。

闭包的实现

```js
function outer() {
  let a = 1

  function inner() {
    console.log(a)  // 访问外层的 a
  }

  return inner
}

const fn = outer()
fn()  // 1
```

inner返回后，它的执行上下文应该被销毁的，但是我们发现inner作用域引用了outer的词法环境，所以outer的词法环境不会被回收。闭包就相当于一个函数+他创建时候的词法环境链。

作用域和执行上下文的关系

这个可能在面试会被提及，作用域是静态的词法概念，执行上下文是运行时概念。前者决定能够访问哪些变量，后者是执行时的环境。我们写代码的时候发现inner可以访问outer里的属性，这个就是写代码时候确认的，inner的词法作用域链上包含outer。


### this问题

this是**函数执行时的上下文对象**，它的值取决于**函数怎么被调用**，而不是怎么被定义（箭头函数除外）。

js里有五种绑定this的方法，分别是new绑定，this就会指向新创建的对象，比如说new Foo()，然后是显示绑定，call/apply/bind 指定对象，比如foo.call(this)，再就是隐式绑定，this会指向调用方法的对象，比如obj.foo()，再来是默认绑定，this会指向undefined(严格)/(非严格)，最后是箭头函数，this会指向外层作用域this，()->this。

**优先级高的规则赢**：`new` > 显式 > 隐式 > 默认。
箭头函数比较特殊，它不遵循前面4条规则，而是词法绑定，所以要单独列出来说。new呢，是最高优先级：

```js
function Foo(name) {
  this.name = name
}

const obj = new Foo('张三')
console.log(obj.name)  // '张三'
```

我们在typecsriptteam.native-preview中可以看看Function的定义：

```ts
interface FunctionConstructor {
    /**
     * Creates a new function.
     * @param args A list of arguments the function accepts.
     */
    new (...args: string[]): Function;
    (...args: string[]): Function;
    readonly prototype: Function;
}

declare var Function: FunctionConstructor;
```

new Function()和 Function()效果一样，都返回新函数，这个是Function的特殊之处，Function既是构造函数也是普通函数。readonly prototype: Function 表示Function.prototype是不可重新赋值的，表示Function.prototype是不可重新赋值。

### arguments

arguments是普通函数内部自动可用的一个**类数组对象**，包含了调用该函数时候传入的所有实参，他不是数组，但是长得像数组，可以按索引访问，也有length属性。

```
function foo(a, b) {
    console.log(arguments);        // [1, 2, 3]（类数组）
    console.log(arguments.length); // 3
    console.log(arguments[0]);     // 1
    console.log(arguments[1]);     // 2
    console.log(arguments[2]);     // 3
}

foo(1, 2, 3);
```

而 fn: Function ， fn.length访问的就是这个方法至少需要多少个参数了，并且箭头函数也是有的，你可以访问得到。

### call实现


```ts
export {}

declare global {
  interface Function {
    customBind(thisArg: any, ...args: any[]): (...innerArgs: any[]) => any
  }
}

Function.prototype.customBind = function (thisArg: any, ...args: any[]) {
  const originalFn = this

  if (typeof originalFn !== 'function') {
    throw new TypeError('Function.prototype.customBind is not a function')
  }

  function boundFn(...innerArgs: any[]) {
    const totalArgs = args.concat(innerArgs)

    // new 调用：忽略 thisArg，执行 new originalFn
    if (new.target) {
      return new (originalFn as any)(...totalArgs)
    }

    // 普通调用：绑定 thisArg
    return originalFn.apply(thisArg, totalArgs)
  }

  // 不设置 boundFn.prototype，保持和原生 bind 一致
  return boundFn
}

// 测试
const obj = {
    name: "John",
    sayHello: function() {
        console.log(`Hello, ${this.name}`);
    }
}
const sayHello = obj.sayHello.customBind(obj);
sayHello(); // Hello, John
const sayHello2 = obj.sayHello.bind(obj);
sayHello2(); // Hello, John
```

这里实现非常简单，唯一有些新的是new.target属性，这个是ES6引入的原属性，只能在函数内部使用，值取决于函数怎么被调用。

如果一个函数是new Foo()这样被调用的话，那么new.target代表Foo函数本身，如果是Foo()调用的话，那么new.target代表undefined，所以说这个是用来区分构造调用和普通调用。

如果调用方式是：

```js
const BoundPerson = Person.customBind({name: 'ignored'})
const p = new BoundPerson('real')

而不能写成：
if (this instanceof boundFn) {
  return new originalFn(...totalArgs)
}
```
这里的调用方式会让new.target为BoundPerson，这样thisArgs就会被忽略了。如果写成了下面那种样子，this在new调用的时候是新创建的对象，它的原型是boundFn.prototype，如果boundFn.prototype是undefined，instanceof就会报错。

### call

call的作用是立即调用一个函数，并手动指定函数内部的this指向，参数是逐个传入的。

```js
function greet(greeting, punctuation) {
  console.log(`${greeting}, ${this.name}${punctuation}`)
}

const user = { name: '张三' }

greet.call(user, 'Hello', '!')  // 'Hello, 张三!'
```

我们自己的实现大致如下：
```ts
export {}

declare global {
    interface Function {
        customCall: (thisArgs : any, ...args: any[]) => any
    }
}

Function.prototype.customCall = function(thisArgs, ...args) {
    thisArgs = thisArgs ?? globalThis
    if (typeof thisArgs !== 'object' && typeof thisArgs !== 'function') {
        thisArgs = Object(thisArgs)
    }
    const tempKey = Symbol('temp')
    thisArgs[tempKey] = this
    try{
        return thisArgs[tempKey](...args)
    } finally {
        delete thisArgs[tempKey]
    }
}
```

apply和call有一点细致差别，仅仅是传递的非...args而是没展开的数组，直接传入 `[...args]`即可。

### new的实现

现 `new` 是理解 JS 原型链和构造函数机制的经典练习，一个new分为经典的四步，首先是创建一个新对象obj，然后把obj的原型指向Foo.prototype (obj.__proto__ === Foo.prototype)，紧接以obj为this，执行Foo，传入参数，判断Foo的返回值，如果返回值是对象或者函数，那么就用这个返回值，如果不是，就返回原始值/无返回值，用之前创建的obj。


```ts
export function mynew(Constructor: Function, ...args: any[]) {
    const obj = Object.create(Constructor.prototype)
    const result = Constructor.apply(obj, args) // constructor.
    if (result !== null && (typeof result === 'object' || typeof result === 'function')) return result
    return obj
}
```

前面我们已知，Function是一个特殊的东西，它自己是自己的构造函数，Function.__proto__ === Function.prototype，而Function.prototype.__proto__ === Object.prototype。一般对象的 `__proto__` 指向"创建它的构造函数的 `prototype`"，我们都知道prototype一般是obj，但是Function.prototype 类型是function，这是唯一的例外。

 `Function.prototype.__proto__` 指向 `Object.prototype`也只是刚好说明了函数也是对象，所以原型链指向最终的Object.prototype。**默认情况下，用 `function` 声明的函数，它的 `constructor` 都是 `Function`，它的 `[[Prototype]]`（即 `__proto__`）都指向 `Function.prototype`。**



### promise

ECMA-262标准在Promise instance一节中，明确规定了Promise实例必须包含的核心内部槽，其中包括PromiseState： 一个字符串值，取值明确限定为pending，fulfilled、rejected三种状态，这个正式状态机的状态集合。然后是PromiseResult，用于存储promise完成的值，仅当PromiseState不为pending的时候才有意义。然后是一个PromiseFulfillReactions列表，存储当promise从pending转移到fulfilled的时候需要处理的记录，也就是注册的then回调。**`PromiseRejectReactions`** 是一个列表，用于存储当Promise从pending转移到reject的时候需要处理的记录。

标准的抽象操作定义了状态如果迁移，fulfillPromise可以讲一个状态为pending的promise转变为fulfilled，并设置PromiseResult，同时触发PromiseFulfillReactions中的任务。RejectPromise讲一个状态为pending的Promise转变为rejected。

这些定义构成了一个严格的单向状态机：`pending` → `fulfilled` 或 `pending` → `rejected`，且状态一旦改变便不可逆。

机制是十分简单的，只是说签名稍微复杂一点，特别是then方法等的签名，这里我先写出来：

```ts
    then<TResult1 = T, TResult2 = never>(
        onFulfilled?: ((value: T) => TResult1 | CustomPromise<TResult1>) | null,
        onRejected?: ((reason: any) => TResult2 | CustomPromise<TResult2>) | null
    ): CustomPromise<TResult1 | TResult2>;
```

可以看到，then方法是接受一个回调函数，可以在回调函数里注册onFulfilled和onRejected，然后返回的是我们自己的Promise类型，拆开 `onFulfilled` 的类型扩展：

```ts
((value: T) => TResult1 | CustomPromise<TResult1>) | null | undefined
```

可能是一个函数，可能是null和undefined，类型上onFulfilled只可能是这些，但是T和Promise不在这个类型里面，它们出现在这个函数的**参数**和**返回值**里，不是 `onFulfilled` 本身。

这里需要复习常见类型在布尔运算里的真值表：

|类型|`if (x)` 为真分支|`if (x)` 为假分支|
|---|---|---|
|`unknown`|不变，仍是 `unknown`|不变，仍是 `unknown`|
|`never`|**不进入**（分支不可达）|**不进入**（分支不可达）|
|`null`|排除了|只剩 `null`|
|`undefined`|排除了|只剩 `undefined`|
|`null \| undefined`|排除了|剩 `null \| undefined`|


```ts
type PromiseState = 'pending' | 'fulfilled' | 'rejected';

interface CustomPromise<T> {
    state: PromiseState;
    result: T | undefined;
    PromiseFulfillReactions: ((value: any) => void)[];
    PromiseRejectReactions: ((reason: any) => void)[];

    resolve(value: T): void;
    reject(reason?: any): void;

    then<TResult1 = T, TResult2 = never>(
        onFulfilled?: ((value: T) => TResult1 | CustomPromise<TResult1>) | null,
        onRejected?: ((reason: any) => TResult2 | CustomPromise<TResult2>) | null
    ): CustomPromise<TResult1 | TResult2>;

    catch<TResult = never>(
        onRejected?: ((reason: any) => TResult | CustomPromise<TResult>) | null
    ): CustomPromise<T | TResult>;
}



class MyPromise<T> implements CustomPromise<T> {
    state: PromiseState = 'pending';
    result: T | undefined = undefined;
    PromiseFulfillReactions: ((value: any) => void)[] = [];
    PromiseRejectReactions: ((reason: any) => void)[] = [];

    constructor(
        executor?: (
            resolve: (value: T) => void,
            reject: (reason?: any) => void
        ) => void
    ) {
        if (typeof executor === 'function') {
            try {
                executor(
                    (value) => this.resolve(value),
                    (reason) => this.reject(reason)
                );
            } catch (e) {
                this.reject(e);
            }
        }
    }

    resolve(value: T): void {
        if (this.state !== 'pending') return;
        if (value instanceof MyPromise) {
            value.then(
                (v) => this.resolve(v as T),
                (r) => this.reject(r)
            );
            return;
        }

        this.state = 'fulfilled';
        this.result = value;

        const reactions = this.PromiseFulfillReactions;
        this.PromiseFulfillReactions = [];
        this.PromiseRejectReactions = [];
        for (const reaction of reactions) {
            queueMicrotask(() => reaction(value));
        }
    }

    reject(reason?: any): void {
        if (this.state !== 'pending') return;

        this.state = 'rejected';
        this.result = reason;

        const reactions = this.PromiseRejectReactions;
        this.PromiseFulfillReactions = [];
        this.PromiseRejectReactions = [];
        for (const reaction of reactions) {
            queueMicrotask(() => reaction(reason));
        }
    }

    then<TResult1 = T, TResult2 = never>(
        onFulfilled?: ((value: T) => TResult1 | CustomPromise<TResult1>) | null,
        onRejected?: ((reason: any) => TResult2 | CustomPromise<TResult2>) | null
    ): CustomPromise<TResult1 | TResult2> {
        return new MyPromise<TResult1 | TResult2>((resolve, reject) => {
            const handlerFulfilled = (value: T) => {
                if (typeof onFulfilled !== 'function') {
                    resolve(value as unknown as TResult1);
                    return;
                }
                try {
                    const r = onFulfilled(value);
                    if (r instanceof MyPromise) {
                        r.then(
                            (v) => resolve(v as TResult1),
                            (e) => reject(e)
                        );
                    } else {
                        resolve(r as TResult1);
                    }
                } catch (e) {
                    reject(e);
                }
            };

            const handlerRejected = (reason: any) => {
                if (typeof onRejected !== 'function') {
                    reject(reason);
                    return;
                }
                try {
                    const r = onRejected(reason);
                    if (r instanceof MyPromise) {
                        r.then(
                            (v) => resolve(v as TResult1),
                            (e) => reject(e)
                        );
                    } else {
                        resolve(r as unknown as TResult1);
                    }
                } catch (e) {
                    reject(e);
                }
            };

            if (this.state === 'pending') {
                this.PromiseFulfillReactions.push(handlerFulfilled);
                this.PromiseRejectReactions.push(handlerRejected);
            } else if (this.state === 'fulfilled') {
                queueMicrotask(() => handlerFulfilled(this.result as T));
            } else {
                queueMicrotask(() => handlerRejected(this.result));
            }
        });
    }

    catch<TResult = never>(
        onRejected?: ((reason: any) => TResult | CustomPromise<TResult>) | null
    ): CustomPromise<T | TResult> {
        return this.then(null, onRejected);
    }

    static resolve<T>(value: T): MyPromise<T> {
        return new MyPromise<T>((resolve) => resolve(value));
    }

    static reject<T = never>(reason?: any): MyPromise<T> {
        return new MyPromise<T>((_, reject) => reject(reason));
    }
}

MyPromise.resolve(10).then(
    (v) => console.log('fulfilled', v),   // fulfilled 10
    (r) => console.log('rejected', r)
);
```

await就是专门为Promise设计的，我们一般喜欢await x 这样来使用，如果x不是thenable，就会直接返回x，如果x是thenable，就会等它敲定，比如fulfilled返回成功值，rejected跑出拒绝原因。因此await不要求x是Promise实例，只要有then方法就行，说白了，就是认定鸭子类型。

### 原型链

原型链的最初设计目的是为了用最少的机制来解决几个问题，Brendan Eich 1995年设计js的时候，是让对象能够借用另一个对象的方法和属性，而不需要每个对象都复制一份。原型链的本质就是委托，当你访问b.greet的时候b自己没有，就会委托给它的原型a去找，这是一种运行时动态查找，而不是编译器的类继承。

Brendan Eich收Self语言影响，核心思想是不需要类，对象可以直接从对象继承。这个就和传统OOP倾向的语言比如Java/C++不一样了。Self/JS的原型模型那就是： 对象 -> 对象 -> 对象。继承关系在对象之间定义，没有类中间层，每个对象都可以有自己的原型，可以动态修改。

所以，完全可以得出一个结论： js选择原型，是因为更加简单，更加灵活，更加适合脚本语言。这样就可以解决好几个问题：

1是内存效率问题，通过千万个实例可以共享一个函数，而不是每个都复制一份。2是动态性，原型链本来就是运行时查找，改了原型，所有实例都会立即收到影响，这个是类继承做不到的。

es5之前，js是没有class语法的，定义类的方式就是构造函数+原型。es6之后引入class作为了关键字和语法糖。但是到现在都没有interface关键字，因为js是动态语言，运行时才检查类型，不需要接口作为编译器约束。

**每个函数**都有一个 `prototype` 属性，指向一个对象，`prototype` 是**函数才有的属性**，它的作用是：**当这个函数被用作构造函数（`new Foo()`）时，新对象的原型会指向它。** __proto__就是每个对象内部的一个槽位，叫做[[Prototype]]，这个指向它的原型对象，访问方式是通过__proto__，或者Object.getPrototypeOf方法。 __proto__是对象才有的属性，指向继承。

默认情况下，`Foo.prototype.constructor === Foo`：

```ts
function Foo() {}
Foo.prototype.constructor === Foo   // true
```
`constructor` 让实例能反向找到"创建我的构造函数"。

还记得我们之前提到过的类型检测方法吗：

```ts
Object.prototype.toString.call(new Map())      // "[object Map]"
Object.prototype.toString.call(new Set())      // "[object Set]"
Object.prototype.toString.call([])             // "[object Array]"
Object.prototype.toString.call(new Date())     // "[object Date]"
Object.prototype.toString.call({})             // "[object Object]"
Object.prototype.toString.call(null)           // "[object Null]"
Object.prototype.toString.call(undefined)      // "[object Undefined]"
```

这里可以发现Object本身就是函数，所以他有prototype，这个也很有趣，Object既是函数，是一个构造函数，我们可以断定的是Object.prototype是所有普通对象的原型链顶端：

```ts
const obj = {};
obj.__proto__ === Object.prototype   // true

Object.getPrototypeOf(obj) === Object.prototype   // true

`Object.prototype` 上挂着所有对象共享的方法：

Object.prototype.toString
Object.prototype.hasOwnProperty
Object.prototype.valueOf
Object.prototype.isPrototypeOf
Object.prototype.propertyIsEnumerable
Object.prototype.toLocaleString

Object.prototype.constructor === Object // true , constructor也是指向Object自己
```

instanceof 的原理是： obj instanceof Foo 检查 Foo.prototype 是否在obj的原型链上，你甚至也可以手动去实现：

```ts
function myInstanceof(obj, Constructor) {
    let proto = Object.getPrototypeOf(obj);
    while (proto !== null) {
        if (proto === Constructor.prototype) return true;
        proto = Object.getPrototypeOf(proto);
    }
    return false;
}
```

所以函数是一种特殊的对象，之所以能被调用是因为有 `[[Call]]`，同时具备对象的一切能力，能挂载属性，有原型，能作为参数传递，函数既可以被当作普通函数，也可以被当成构造函数。

**两个 `constructor`，别混：**

|表达式|结果|含义|
|---|---|---|
|`Foo.constructor`|`Function`|`Foo` 这个函数**对象**由谁创建|
|`Foo.prototype.constructor`|`Foo`|`Foo` 的**实例**由谁创建|
new Foo()执行的就是Foo这个函数，而不是Foo.constructor。

### 数组可以调用的方法

数组可以调用的函数分为实例方法，挂载到Array.prototype上的，数组可以直接使用。静态方法是挂载在Array的构造函数上的，通过Array.xxx调用。

实例方法：

|方法|作用|
|---|---|
|`push(...items)`|末尾添加，返回新长度|
|`pop()`|删除末尾，返回被删元素|
|`unshift(...items)`|开头添加，返回新长度|
|`shift()`|删除开头，返回被删元素|
|`splice(start, deleteCount, ...items)`|任意位置增删，返回被删元素数组|
|`reverse()`|反转，返回原数组|
|`sort(compareFn?)`|排序，返回原数组|
|`fill(value, start?, end?)`|填充，返回原数组|
|`copyWithin(target, start?, end?)`|内部复制，返回原数组|

|方法|作用|
|---|---|
|`concat(...items)`|合并，返回新数组|
|`slice(start?, end?)`|截取，返回新数组|
|`join(sep?)`|转字符串|
|`indexOf(item, from?)`|找索引，找不到返回 -1|
|`lastIndexOf(item, from?)`|从后找索引|
|`includes(item, from?)`|是否包含，返回 boolean|
|`find(cb)`|找第一个满足的元素|
|`findIndex(cb)`|找第一个满足的索引|
|`findLast(cb)`|从后找第一个满足的元素（ES2023）|
|`findLastIndex(cb)`|从后找索引（ES2023）|
|`filter(cb)`|过滤，返回新数组|
|`map(cb)`|映射，返回新数组|
|`flat(depth?)`|扁平化|
|`flatMap(cb)`|map + flat|
|`at(index)`|支持负数索引|
|`with(index, value)`|返回替换后的新数组（ES2023）|
|`toReversed()`|反转副本（ES2023）|
|`toSorted(cmp?)`|排序副本（ES2023）|
|`toSpliced(...)`|splice 副本（ES2023）|
静态方法：

|方法|作用|
|---|---|
|`Array.from(iterable, mapFn?, thisArg?)`|从类数组/可迭代对象创建数组|
|`Array.fromAsync(iterable)`|异步版 `from`（ES2023）|
|`Array.of(...items)`|用参数创建数组|
|`Array.isArray(x)`|判断是否数组|

 `Array` 是 `ArrayConstructor` ，这个可以在ts的lib.es5.d.ts里找，大致如下：
 
```ts
interface ArrayConstructor {
    new (arrayLength?: number): any[];
    new <T>(arrayLength: number): T[];
    new <T>(...items: T[]): T[];
    isArray(arg: any): arg is any[];
    from<T>(arrayLike: ArrayLike<T>): T[];
    of<T>(...items: T[]): T[];
    // ...
}

declare var Array: ArrayConstructor;
```

Array是ArrayConstructor类型的变量，它同时是函数。push在Array.prototype上：

```ts
interface Array<T> {
    push(...items: T[]): number;
    pop(): T | undefined;
    map<U>(cb: (v: T, i: number, arr: T[]) => U): U[];
    // ...
}

interface ArrayConstructor {
    // 静态方法
}

declare var Array: ArrayConstructor;
```

可以看到push是定义在`Array<T>`上的，对应Array.prototype上，所有数组实例共享。[]或者new Array()都可以得到Array实例。[]是语法糖，或者严格来说是数组字面量，由规范里的ArrayLiteral生产式定义。两者创建的对象原型链完全一样。

### 生成器与迭代器

[[生成器与迭代器]] 

### 流式操作

流式操作远不止“读取大文件”。围绕 **Web Streams API**（`ReadableStream`、`WritableStream`、`TransformStream`），可以构建出一整套**数据处理管道**

比如说，读取视频文件，你可以把流接到目标：

```ts
const response = await fetch('/video.mp4');
const writable = new WritableStream({
    write(chunk) {
        console.log('收到', chunk.length, '字节');
    }
});

await response.body.pipeTo(writable);
```

流可以经过转换器，body有个方法叫做pipeThrough，能够把字节转换为字符串，字符串转换为大写：

```ts
const response = await fetch('/data.txt');
const stream = response.body
    .pipeThrough(new TextDecoderStream())      // 字节 → 字符串
    .pipeThrough(new TransformStream({         // 字符串 → 大写
        transform(chunk, controller) {
            controller.enqueue(chunk.toUpperCase());
        }
    }));

for await (const chunk of stream) {
    console.log(chunk);
}
```

你甚至可以用tee把流直接一分为2交给两个消费者：

```ts
const response = await fetch('/data');
const [stream1, stream2] = response.body.tee();

// 两个消费者同时处理
stream1.pipeTo(new WritableStream({ write(c) { /* ... */ } }));
stream2.pipeTo(new WritableStream({ write(c) { /* ... */ } }));
```

#### 常用场景

解压操作gz；

```ts
const response = await fetch('/data.gz');
const stream = response.body
    .pipeThrough(new DecompressionStream('gzip'));

const text = await new Response(stream).text();
console.log(text);
```

压缩为gzip：

```ts
const stream = new Blob(['hello world'])
    .stream()
    .pipeThrough(new CompressionStream('gzip'));

const compressed = await new Response(stream).arrayBuffer();
```

文本接编码；

```ts
const response = await fetch('/data.txt');
const stream = response.body
    .pipeThrough(new TextDecoderStream());

for await (const chunk of stream) {
    console.log(chunk);  // 字符串
}

const stream = new ReadableStream({
    start(controller) {
        controller.enqueue('hello');
        controller.enqueue('world');
        controller.close();
    }
}).pipeThrough(new TextEncoderStream());
```

#### 流式数据格式解析

NDJSON，每行有一个json:

```ts
async function* parseNDJSON(stream) {
    const reader = stream
        .pipeThrough(new TextDecoderStream())
        .getReader();
    let buffer = '';

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        buffer += value;
        const lines = buffer.split('\n');
        buffer = lines.pop();
        for (const line of lines) {
            if (line.trim()) yield JSON.parse(line);
        }
    }
}

const response = await fetch('/logs.ndjson');
for await (const obj of parseNDJSON(response.body)) {
    console.log(obj);
}
```

SSE数据处理，sse服务器可以发送默认消息和自定义事件，消息没有event字段，浏览器默认按照message事件处理，客户端用`es.onmessage` 或 `es.addEventListener('message', ...)` 接收。自定义事件有event字段，指定事件类型，- 客户端用 `es.addEventListener('eventName', ...)` 接收。

```ts
async function* parseSSE(stream) {
    const reader = stream
        .pipeThrough(new TextDecoderStream())
        .getReader();
    let buffer = '';

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        buffer += value;
        const events = buffer.split('\n\n');
        buffer = events.pop();
        for (const event of events) {
            const data = event
                .split('\n')
                .filter(l => l.startsWith('data:'))
                .map(l => l.slice(5).trim())
                .join('\n');
            yield data;
        }
    }
}
```

CSV流式解析，CSV的结构如下：

```csv
name,age,city
Alice,30,Beijing
Bob,25,Shanghai
Charlie,35,Guangzhou
...
```

每条记录以\n或者\r\n结束，列分隔用,分隔，不像json有嵌套结构，不需要等整个文档解析完。无全局头，第一行是列名，之后每行独立。因此我们需要从ReadableStream或者文件读取一块数据，用TextDecorder按UTF-8解码，按照\n切分，最后一行可能不完整，留在缓冲区。把每行按 `,` 切分成字段，每行独立处理，不需要等下一行。

```ts
async function* parseCSV(stream) {
    const reader = stream
        .pipeThrough(new TextDecoderStream())
        .getReader();
    let buffer = '';
    let headers = null;

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        buffer += value;
        const lines = buffer.split('\n');
        buffer = lines.pop();

        for (const line of lines) {
            const cells = line.split(',');
            if (!headers) {
                headers = cells;
            } else {
                yield Object.fromEntries(
                    headers.map((h, i) => [h, cells[i]])
                );
            }
        }
    }
}

// 使用
const response = await fetch('/data.csv');
for await (const row of parseCSV(response.body)) {
    console.log(row);  // { name: 'Alice', age: '30', city: 'Beijing' }
}
```



SSR流式HTML渲染，如果这个放在web worker之后就需要使用postMessage把HTML片段传回主线程，由主线程来插入DOM，这是因为Worker不能操作DOM，因为DOM不是线程安全的，多线程操作DOM会导致状态不一致，worker是独立环境，也没有DOM api

这个有点像之前微软设计STA模型的时候同样的根本性考虑，为了保护一个非线程安全的共享资源，比如限制它只能被特定的线程访问。DOM和COM组件都不是线程安全的。

它们的解决方案也很相似，两者都采用了代理proxy+消息传递的模式来安全地转发调用，COM STA中，来自其他线程的调用视图访问STA对象时候，COM运行时不会直接调用，而是通过一个代理将调用编组成一条窗口消息，投递到对象所属STA线程的消息队列里，对象线程通过GetMessage/DispatchMessage取出消息并处理这个消息。

Worker也一样，他必须通过postMessage将数据序列化后发送给主线程，主线程收到了消息之后再自己的执行环境下执行真正的DOM操作。

```ts
const response = await fetch('/page');

const stream = response.body
    .pipeThrough(new TextDecoderStream())
    .pipeThrough(new TransformStream({
        transform(chunk, controller) {
            // 在 HTML 到达时逐块插入 DOM
            document.getElementById('app').insertAdjacentHTML('beforeend', chunk);
            controller.enqueue(chunk);
        }
    }));

await new Response(stream).text();
```

流式Markdown:

```ts
const response = await fetch('/article.md');
const stream = response.body
    .pipeThrough(new TextDecoderStream())
    .pipeThrough(new TransformStream({
        transform(chunk, controller) {
            const html = markdownToHTML(chunk);
            document.getElementById('content').insertAdjacentHTML('beforeend', html);
            controller.enqueue(chunk);
        }
    }));
```

 AI 流式输出
 
```ts
const response = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({ prompt: 'Hello' })
});

const reader = response.body
    .pipeThrough(new TextDecoderStream())
    .getReader();

while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    // 逐字输出
    document.getElementById('output').textContent += value;
}
```

#### 音视频处理

音频流式

```ts
const response = await fetch('/audio.mp3');
const audioContext = new AudioContext();
const source = audioContext.createMediaStreamSource(
    new MediaStream([/* ... */])
);
```

视频流式录制：

```ts
const stream = await navigator.mediaDevices.getUserMedia({ video: true });
const recorder = new MediaRecorder(stream);

const writable = new WritableStream({
    write(chunk) {
        // 把录制的数据块上传或保存
    }
});

recorder.ondataavailable = (e) => {
    writable.getWriter().write(e.data);
};
```


vueuse提供了两个处理web worker的组合式函数：useWebWorker和useWebWorkerFn，分别对应托管现成Worker脚本和直接包装函数体两种场景。


```ts
import { useWebWorkerFn } from '@vueuse/core'

const { workerFn } = useWebWorkerFn((numbers) => {
  // 这里是耗时的重计算
  return numbers.sort((a, b) => a - b)
})

// 调用 workerFn，返回 Promise
const sorted = await workerFn(hugeArray)

```

### 闭包的实现

闭包主要是通过上下文对象和作用域链实现的，**内部函数引用了外部函数的变量的时候**，v8会把这些变量从栈上搬到堆上的上下文对象中，这样内部的函数可以通过作用域链来访问它们。正常情况下函数执行的时候局部变量会分配在调用栈上，函数一旦返回，变量就会随着栈帧一起销毁。但是闭包会打破这个规则，v8发现一个内部函数引用了外部函数变量，那么就让这个变量在外部函数返回后仍需存活，v8会把这些被引用的变量以及可能其他被引用的变量从栈提升到堆上，封装成一个上下文对象。

可以把上下文理解为一个特殊的环境记录，用来保存那些需要活过外部函数生命周期的变量。每个函数对象内部都有一个隐藏的 `[[Environment]]`槽，称为Scope内部属性，当函数被创建的时候，这个属性会指向它定义时候所处的词法环境，也就是外层context对象。闭包执行并需要访问外部的变量，v8就会沿着这条链去找。

v8不会盲目保存所有变量，而是依靠惰性解析，v8首次解析代码的时候遇到**函数声明**会跳过其内部代码，**只生成顶层的AST和字节码**，然后进行静态分析，只把内部函数实际引用的变量提升到上下文对象，而不是外部函数所有的局部变量，如果某个变量没有被闭包引用，他依然留在栈上，执行完销毁。

结合之前的js核心运行机制可以推断出：

词法环境和变量环境是抽象概念，v8层面，它们会被具象化为context对象，变量提升到context对象，**发生在外层函数执行的时候**，而不是内层函数查找的时候。V8 在编译外层函数时，就已经通过**静态分析**知道了 哪些变量会被内层函数引用，这些变量应该放在上下文的哪个槽位，所以外层函数一执行，V8 就直接在堆上创建 Context，把这些变量放进去。内层函数创建时，直接拿到这个 Context 的引用。

静态分析和逃逸分析

静态分析和逃逸分析是包含关系：

```
编译优化
  ├── 静态分析（Static Analysis）        ← 大类：编译期分析代码性质
  │     ├── 逃逸分析（Escape Analysis）   ← 子类：分析变量是否“逃逸”
  │     ├── 死代码消除
  │     ├── 常量折叠
  │     └── 内联分析
  └── 动态分析（Dynamic Analysis）        ← 运行时分析
```

静态分析是“在编译期分析代码性质”的总称，逃逸分析是其中专门用来判断“变量是否会逃出当前作用域”的技术。逃逸本意是指一个变量或者对象的生命周期是否会超出它被创建时的作用域，变量只在当前函数内使用，函数返回即销毁就叫做不逃逸，反之就是逃逸到堆。变量被传递到其他线程就叫做逃逸到其他线程。

### GC机制

js的垃圾回收是引擎自动管理内存的机制，负责识别并回收不再被使用的内存，防止内存泄漏，v8的gc策略是分代回收+增量标记+并发清理的组合策略。

gc不关心这个变量叫什么，只关心这个对象还能不能访问到，从根对象出发，沿着引用链遍历，**能达到的对象就是可达的**，不能达到的就是不可达，会被回收掉。

根对象包括了window/globalThis，调用栈上的变量（当前执行函数的局部变量、参数）。闭包捕获的变量、DOM引用。

v8把堆内存分为了新生代和老生代，针对不同生命周期采用不同策略，新生代比较小可能就1-8MB，老生代比较大，可能几百MB甚至GB，新生代的生命周期比较短，老生代周期比较长。

新生代使用Scavenge复制算法进行回收，老生代使用mark-sweep+mark-compact。把新生代分为两个半区：**From 空间**和**To 空间**。

```
From 空间（使用中）    To 空间（空闲）
┌─────────────┐      ┌─────────────┐
│ 活对象       │      │             │
│ 死对象       │      │             │
└─────────────┘      └─────────────┘
```

从根出发，标记from空间中的活对象，把活对象复制到To空间，清空From空间，交换From和To的对象，如果对象经过多次 Scavenge 仍然存活，会被**晋升到老生代**。

分代思想是个很成熟的模型，就像CLR也是采用的三代模型的gc回收。第一代充当缓冲区，短期存活的对象有机会再0代回收后被快速清理，而不必升到2代，clr有大型对象堆LOH，任何大于85KB的对象会直接分配到LOH上，并被标记为2代， 它**只在完全 GC（第 2 代回收）时才会被清理**，且**默认不会被压缩**[](https://github.com/rambhawan/Computer-Compiler-Lang/blob/825822d3e676b2dc5a93d649ae8a4555cb606b93/Pro%20.Net%20Performance.pdf?_x_output_type_b6db407c4_=embedded_pdf#53#19)[](https://m.yisu.com/zixun/102253.html)。这是因为移动大对象内存的代价很高，V8 没有专门的 LOH，所有对象都在新生代或老生代中处理。
### TS的类型

在ts之中，函数会被看做值，类型是函数类型(x: number) => string。方法是对象里的函数属性，类型也是函数类型。类方法是类原型上的函数，类型也是函数类型。因此可以推理得出：方法的泛型 = 函数类型的泛型，不是所有方法都有独立的类型系统。

泛型参数 `<T>`可以写在函数签名上，而不是写在类型别名/接口/类上。我们知道泛型有两种写法：

```ts
type Box<T> = {
    value: T;
};
//   ^^^ T 是 Box 的类型参数
const b: Box<number> = { value: 1 };   // 用类型时确定 T

function identity<T>(x: T): T {
    //           ^^^ T 是函数的类型参数
    return x;
}
identity<number>(1);   // 调用函数时确定 T
```

两种泛型位置不同，分别叫做类型级泛型和调用级泛型。

### Ts的基本模式

ts有个非常常见的模式，使用interface来描述实例，一份interface描述构造函数并且包含静态方法，然后class同时满足。

我们使用的Array Function Promise全都是按照这个模式来编写的：

```ts
// 1. 实例接口
interface User {
    name: string;
    age: number;
    greet(): string;
}

// 2. 构造函数接口（含静态方法）
interface UserConstructor {
    new (name: string, age: number): User;   // 构造签名
    create(name: string, age: number): User; // 静态方法
    readonly MAX_AGE: number;                 // 静态属性
}

// 3. 类实现两者
class UserImpl implements User {
    static MAX_AGE = 150;
    
    constructor(public name: string, public age: number) {}
    
    greet() {
        return `Hi, I'm ${this.name}`;
    }
    
    static create(name: string, age: number): User {
        return new UserImpl(name, age);
    }
}

// 4. 用构造函数接口约束类本身
const UserClass: UserConstructor = UserImpl;   // ✅
```

这样的方式可以让我们解耦接口和实现，还支持依赖注入、工厂函数和类统一。以及还忘记提了，可以仿照Promise一样，使用declare var Promise: PromiseConstructor

这是lib.es2015.promise.d.ts里的做法，用一个var声明把值（构造函数）和类型（实例）分开，这个是ts声明库类型时的经典模式。


```ts
// ===== 1. 实例接口 =====
interface Promise<T> {
    then<TResult1 = T, TResult2 = never>(
        onfulfilled?: ((value: T) => TResult1 | PromiseLike<TResult1>) | null,
        onrejected?: ((reason: any) => TResult2 | PromiseLike<TResult2>) | null
    ): Promise<TResult1 | TResult2>;
    catch<TResult = never>(
        onrejected?: ((reason: any) => TResult | PromiseLike<TResult>) | null
    ): Promise<T | TResult>;
    finally(onfinally?: (() => void) | null): Promise<T>;
    readonly [Symbol.toStringTag]: string;
}

// ===== 2. 构造函数接口 =====
interface PromiseConstructor {
    readonly prototype: Promise<any>;
    
    // new 签名
    new <T>(
        executor: (
            resolve: (value: T | PromiseLike<T>) => void,
            reject: (reason?: any) => void
        ) => void
    ): Promise<T>;
    
    // 静态方法
    resolve(): Promise<void>;
    resolve<T>(value: T | PromiseLike<T>): Promise<T>;
    reject<T = never>(reason?: any): Promise<T>;
    all<T extends readonly unknown[] | []>(values: T): Promise<...>;
    race<T>(values: Iterable<T | PromiseLike<T>>): Promise<T>;
    allSettled<T>(values: Iterable<T | PromiseLike<T>>): Promise<...>;
    any<T>(values: Iterable<T | PromiseLike<T>>): Promise<T>;
}

// ===== 3. 用 declare var 声明"值" =====
declare var Promise: PromiseConstructor;
```

这种三段式方法很有用，declare var....这种一般用来告诉ts运行时有个全局变量叫做Promsie，类型为PromiseConsturctor，但是声明不会生成代码，如果没有declare的话，ts就把他当做变量声明了，导致结果就是编译成js，代码执行，全局Promise覆盖为undefined。
## vue

### vue-router

vue-router是提供了一系列组价比如说router-link，它是官方的路由库，核心职责是把URL和组件映射起来，但URL变化渲染对应的组件。

```ts
const routes = [
    { path: '/', component: Home },
    { path: '/about', component: About },
    { path: '/user/:id', component: User }
];
```

当你访问/about的时候就会渲染Aboud组件，渲染/user/1的时候就会渲染User，id=1。SPA单页应用没有真正的页面跳转，URL变化由Vue Router接管，用JS切换组件，这个就是前端路由。


```js
import { createRouter, createWebHistory } from 'vue-router';

const router = createRouter({
    history: createWebHistory(),   // 用 HTML5 History 模式
    routes
});

export default router;
```

挂载到app:

```js
const app = createApp(App);
app.use(router);
app.mount('#app');
```

组件渲染的位置：

```vue
<template>
    <router-view />   <!-- 当前路由的组件渲染在这 -->
</template>
```

router-view是路由的出口，URL匹配到哪个组件，渲染就会在这里。

router-link标签是导航链接。

active-class这个也比较重要，因为用户怎么知道自己在哪个页面，导航栏对应的链接应该高亮，这个就是active-class的由来，当前的URL匹配某个router-link的时候，vue-router就会自动给他的a加一个类，然后你使用CSS让他高亮。

```css
.router-link-active {
    color: red;
    font-weight: bold;
}
```

动态路由

刚才我们讲了:id的路由形式，这个是动态路由参数，是vue-router的最常用的动态形式，但是这个词其实涵盖了好几种类别。

:开头的叫做参数占位符，能够匹配任意非/字符串，通过route.params可以访问，?表示可选，+表示一个或者多个，可以匹配 匹配 `/user/1/2/3`。params.ids是数组。`*` 表示**零个或多个**，匹配 `/user` 和 `/user/1/2/3`。

```
{ path: '/user/:id?', component: User }
{ path: '/user/:ids+', component: User }
{ path: '/user/:ids*', component: User }
```

路由也是支持自定义正则的：


```
{ path: '/user/:id(\\d+)', component: User }
```

括号里是**正则**，限制参数格式。`(\\d+)` 只匹配数字。

路由也可以使用alias来设置别名，我们可以做到一个路由可以有多个路径：

```js
{
    path: '/user',
    component: User,
    alias: '/u'   // /u 也渲染 User
}
{
    path: '/user',
    component: User,
    alias: ['/u', '/member']   // 多个别名
}
```

命名路由可以给路由起名字，导航的时候用名字而不是路径。

```js
const routes = [
    { path: '/user/:id', name: 'user', component: User }
];
<router-link :to="{ name: 'user', params: { id: 123 } }">用户</router-link>
```

嵌套路由：

```js
{
    path: '/user',
    component: UserLayout,
    children: [
        { path: '', component: UserHome },
        { path: 'profile', component: UserProfile }
    ]
}
```

这不过我们在nuxt中可能更加习惯使用layout来解决。

路由模式

Vue-Router有两种主要路由模式，外加一个SSR使用的。首先是HASH模式，可以通过createWebHashHistory来实现，特征是url中带#。#后面是路由路径，#后面的变化不会触发页面刷新。 优点有： 兼容性好、不需要服务端配置、部署简单。缺点有：Url丑、seo不友好、锚点错误。

历史模式则是使用html5 history api （pushState/repelaceState）来管理，url变化也不会刷新页面，监听poststate事件。缺点和优点和上面优点相反，这个需要服务单配置，刷新页面的时候服务端必须要返回index.html，要配置nginx/apache的fallback。

下面来讲底层原理：

hash模式下 我们知道url组成为 `URL: example.com/#/user/123` ,#后面的部分变化不会触发页面刷新，#是url的片段标识符fragment，本来用于页面内锚点(阅览文档有时候会发现，就是锚点可以自动滚动那个功能)，浏览器不会吧#后面的内容发给服务端，所以服务端只会永远看到/，不需要配置。静态托管包含github pages/netify/vercel静态模式、对象存储、CDN，特点就是只提供文件，请求什么路径就返回什么文件。不能写服务端配置，没有nginx配置，也没有fallback规则。没有路由重写能力，不能把/user/123重写成/index.html。如果你在静态托管使用历史模式的话，前端跳转到/user/123会让url变化无刷新，然后用户手动刷新会发现/user/123文件不存在-> 404。哈希不一样，就算你请求example.com/#/user/123，浏览器也只会请求example.com，这样永远都能找到了，然后前端去读取片段符号后的内容/user/123，来渲染对应组件。

`URL: example.com/user/123`
history不一样，pushState能改url但是不刷新页面，刷新的时候浏览器真的会请求/user/123，服务端必须返回index.html
```
history.pushState({}, '', '/a');   // 不触发 popstate
history.back();                     // 触发 popstate
```
用户访问的时候nginx不把/user/123当文件找，而是返回index.html让前端路由接管，类似于这样：

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/my-app;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

实际项目往往还有 API 请求、HTTPS、gzip 等。

导航钩子

导航守卫有三类，分别是全局守卫、路由独享的守卫、组件内守卫。

全局守卫在每次路由的时候都会触发，通常用于全局路由检验，比如说用于全局的权限检验、登录检查或者页面标题更新。

**`beforeEach`**：**全局前置守卫**。在导航被确认之前触发，是进行权限验证（如检查登录状态）最常用的钩子。**`beforeResolve`**：**全局解析守卫**。它在所有组件内守卫和异步路由组件被解析之后、导航被确认之前触发。适合在最终确认导航前执行一些数据获取或权限校验。**`afterEach`**：**全局后置钩子**。导航完成后触发，**没有 `next` 函数**，不能改变导航本身。常用于日志记录、分析或更新页面标题

路由独享守卫定义为具体的路由配置中，只对特定路由生效。**`beforeEnter`**：定义在路由的配置对象里。**仅在进入该路由时触发**，如果只是参数（`params`、`query`）发生变化（例如从 `/users/1` 到 `/users/2`），它**不会**被触发。它和全局前置守卫有相同的函数签名

组件内收尾在路由组件内部，用于控制该组件相关的导航：
**`beforeRouteEnter`**：在**进入**该组件对应的路由前调用。**注意**，此时组件实例**尚未创建**，所以不能访问 this，不过，可以通过给 `next` 传一个回调函数来访问最终的组件实例。 **`beforeRouteUpdate`**：在当前路由改变，但**该组件被复用时**调用。例如，对于一个动态路由 `/foo/:id`，从 `/foo/1` 导航到 `/foo/2` 时会触发。**`beforeRouteLeave`**：在**离开**该组件对应的路由时调用。通常用于**防止用户在未保存更改时离开页面**，可以弹出确认框。

在 Vue 3 的 `<script setup>` 中，组件内守卫通过 `onBeforeRouteUpdate` 和 `onBeforeRouteLeave` 这两个函数来使用

在nuxt中，导航钩子的实现被称为路由中间件，提供了比原生vue-router守卫更好的开发体验，它最大的特点是**去掉了 `next()` 函数**，改用**返回值**来控制导航流程。可以做到发行：返回nothing；重定向：返回navigationTo('/path')；终止导航：返回abortNavigation()等。

可以写在页面组件内，仅对当前页面生效，通过definePageMeta来定义；可以放在app/middleware目录里，在多个页面复用，然后在页面中通过 `definePageMeta({ middleware: 'auth' })` 引入。全局中间件同样放在middleware目录里，文件名需要以.global.ts结尾，会每次路由变化的时候自动执行

```
<script setup lang="ts">
definePageMeta({
  middleware: [
    function (to, from) {
      // 仅对当前页面生效的钩子逻辑
    }
  ]
})
</script>

// app/middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  if (!useCookie('token').value) {
    return navigateTo('/login')
  }
})

// app/middleware/analytics.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  // 每次导航都会执行
})
```

要说明的是，如果你项目里有 `app/pages/` 目录，Nuxt 底层依然会基于 `vue-router` 构建路由实例，`useRouter()` 返回的也是完整的 Router 实例，它依然提供 `beforeEach` 等原生守卫方法。但官方明确建议：**绝大多数导航拦截需求都应优先使用路由中间件**，而不是直接调用 `router.beforeEach`，因为中间件在 SSR 处理、重定向语义和开发体验上都做了更好的封装


### v-model双向绑定原理

v-model是vue语法糖，本质是:value和@input组合，这里要讲到模板编译和运行时绑定。

`<input v-model="msg">`在编译之后会等价于
`<input :value="msg" @input="msg = $event.target.value">`
:value="msg"是数据->视图的表现，@input是视图->数据，每次input变化的时候就会把新的值写回msg。`$event` 是原生 DOM 事件对象，`$event.target.value` 是输入框的当前值。

Vue 会根据元素类型，自动选**合适的 prop 和事件**

|元素|绑定的 prop|监听的事件|
|---|---|---|
|`<input type="text">`|`value`|`input`|
|`<input type="checkbox">`|`checked`|`change`|
|`<input type="radio">`|`checked`|`change`|
|`<select>`|`value`|`change`|
|`<textarea>`|`value`|`input`|
比如说
```
<input type="checkbox" v-model="checked">
展开就是：
<input
    type="checkbox"
    :checked="checked"
    @change="checked = $event.target.checked"
>
SELECT:
<select v-model="selected">
    <option value="a">A</option>
    <option value="b">B</option>
</select>
展开就是：
<select
    :value="selected"
    @change="selected = $event.target.value"
>
```

`v-model` 支持几个修饰符，改变行为，在v-model之后加上.lazy，就把input事件改为change事件，失焦或者回车才会更新。加上.number，会自动把输入转成数字 `<input v-model.number="age" type="number">`。.trim会自动去掉收尾空格。

@input是vue模板里的事件绑定语法糖，等价于v-on:input，不是原生DOM属性。 `<input @input="handler">` 展开就是 `<input v-on:input="handler">` ，**它编译后不是 DOM 属性，而是 `addEventListener('input', handler)`。** 

我们知道vue模板最终会编译为render方法，@input会编译为onInput这个prop，大致如下：

```js
h('input', {
    value: msg,
    onInput: (e) => { msg = e.target.value; }
})
```

onInput不是DOM属性，是vue渲染函数里的一个特殊prop，vue运行时识别到on开头的prop之后会用addEventListener来绑定，就和onClick一样。而$event是DOM事件对象，组件事件 $event是emit的参数。

然后这个vnode对象大致为

```
{
    type: 'input',           // 标签名 或 组件
    props: { onInput: fn },  // 属性/事件
    children: null,
    key: null,
    el: null                 // 真实 DOM，初始为 null
}
```

经过@vue/runtime-core里的patch之后，vnode变成dom：

```js
// 1. 创建
const el = document.createElement('input');

// 2. 设置 value（DOM property，不是 setAttribute）
el.value = msg;

// 3. 绑定事件（addEventListener）
el.addEventListener('input', (e) => {
    msg = e.target.value;
});

// 4. 插入
container.appendChild(el);
```


从数据->视图的实现是:value是吧，数据变化的时候vue自动重新执行渲染函数，生成新的vnode，和旧的vnode进行对比，将diff应用到真实dom中。

:value之所以可以绑定到ref和自动解包，运用的是编译器的转换。:value = "msg"编译后不是value:msg，而是value:msg.value。

```js
<script setup>
import { ref } from 'vue';
const msg = ref('hello');
</script>

<template>
    <input :value="msg">
</template>

编译后大致是：

import { ref, unref } from 'vue';

const msg = ref('hello');

function render() {
    return h('input', {
        value: unref(msg)   // ← 关键：编译器加了 unref
    });
}

function unref(r) {
    return isRef(r) ? r.value : r;
}
```

这个里面主要就看sfc编译器实现了，有兴趣可以看看编译器的源码。

### mvvm mVC

mvvm和mvc是个经常提起的话题，作为一个合格的客户端开发工程师，那必须熟悉。mvvm原来就三个分离： model、view、viewmodel。

view就对应vue的模板，描述样式。viewmodel要注意一下，原本含义是view和model之间的桥梁，可以把model的数据暴露给view，view的操作也可以传递给model。在vue里，composables是一部分，但是viewmodel还要其他几个组成，比如说script setup里使用的ref reactive 或者watch操作。 script setup整体那就是viewmodel，它把 Model 的数据"包装"成响应式，暴露给模板。model是数据+业务逻辑，所以组成就应该是user post实体这类数据结构+ 业务逻辑比如计算、校验、转换和数据操作比如crud、持久化等。

后端中MVC用的比较多，view就是模板，中间层就是Controller，视图开发中mvc有些不一样，在vue/react这类现代视图开发中mvc的分层思想虽然在，但是控制器手动更新view被淘汰了。
经典mvc:

```
用户操作 → Controller → 改 Model → Model 通知 View → View 更新

// 经典 MVC 的写法
class UserController {
    handleClick() {
        const user = userModel.getUser();
        userView.render(user);   // 手动调 View 的 render
    }
}
```

现代后端也发展了很多，用asp.net来说，2009支持了asp.net mvc，使用mvc来分层、可测试、restful。从2021的.net6开始，又支持了minimal api，极简风格，用于几行代码的服务。2022+之后又增强到可以使用路由组、过滤器、AOT。


## 计算机网络
### RESTful API和GraphQL

**RESTful API 和 GraphQL 是两种对立的 API 设计范式** ，restfulapi的哲学是一切皆资源，用url标识资源：

```
/users          → 用户集合
/users/123      → 单个用户
/users/123/posts → 用户的文章
```

http方法表意动词操作资源，比如GET方法读取 GET /user/123，POST创建，POST /users，PUT表全量更新 PATCH部分更新 DELETE删除。

**方法 + URL = 完整语义**。`DELETE /users/123` 一看就知道"删除 123 号用户"。

每个请求都是独立的，服务端不会保存客户端状态，有统一接口，状态码表意。

REST好用是好用，也有几个缺点，一个是过渡获取，客户端要的字段少服务器返回一堆，比如你可能只要name，但是一次性拿到的是全部（其实和这个要后端端点的设计），一个页面需要多个资源可能要多个请求。后端要版本管理，加字段可能破坏兼容性。

GraphQL核心思想是描述客户端需求之后服务端再返回这些。只需要单一端点：

```
POST /graphql
```

客户端指定好字段就行：

```graphql
query {
    user(id: 123) {
        name
        posts {
            title
        }
    }
}
```

直接返回

```
{
    "data": {
        "user": {
            "name": "Alice",
            "posts": [
                { "title": "文章 1" },
                { "title": "文章 2" }
            ]
        }
    }
}
```

**只要 `name` 和 `posts.title`，就只返回这些。** 

缺点是缓存复杂，rest可以用HTTP天然缓存，GRAPHQL要客户端缓存，查询有复杂度，嵌套查询可能触发大量数据库查询。

**很多项目 REST + GraphQL 并存**：

- 简单接口用 REST
    
- 复杂查询用 GraphQL
    
- 文件上传用 Request


除此外，还有好几种其他的： gRPC、tRPC、WS、SOAP。


### 请求头里的缓存

这里有个容易搞错的误区，我们知道http请求头里Cache-Control、ETag等控制的是浏览器HTTP缓存，但不是CacheStorage，两者是完全独立的存储系统。

HTTP由浏览器网络层管理，CacheStorage是通过sw管理的。http缓存会由浏览器自动写入完成，读取也是浏览器自动的，存储在磁盘/内存。你可以在devtools里的disk cache里显示。

我们常见的fetch操作会默认走http缓存，如果http缓存有且未过期，就会直接返回缓存，而不经过网络请求。

如果要让资源进入cacheStorage里面的话，需要手动注册sw，在install或者fetch里显示设置cache.put()/cache.addAll()

```js
// install 时预缓存
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open('v1').then(cache => 
            cache.addAll(['/style.css', '/app.js'])   // 显式写入
        )
    );
});
```

至于性能上，数据读取上CacheStorage比http缓存慢，因为多了一层sw的启动和js执行开销。我们做pwa应用的时候可能会使用到。

### http

HTTP是工作在应用层的协议，用于在客户端比如浏览器和服务器之间传输数据，是web的基石，采用的是请求-响应模型，无状态、可扩展。

状态码来说：

1xx比如101 Switching Protocols。2xx开头是成功的，比如200 OK 201 Created、204 No Content。3xx开头为重定向，301是永久重定向，302是临时的。4xx开头是客户端错误，5开头的是服务端错误。

 常见请求头/响应头

| 头部                                    | 作用                   |
| ------------------------------------- | -------------------- |
| `Host`                                | 目标主机（HTTP/1.1 必需）    |
| `Content-Type`                        | 报文体的 MIME 类型         |
| `Content-Length`                      | 报文体长度                |
| `Accept`                              | 客户端可接受的类型            |
| `Authorization`                       | 认证凭证（如 Bearer Token） |
| `Cookie` / `Set-Cookie`               | 会话状态管理               |
| `Cache-Control`                       | 缓存策略                 |
| `ETag` / `If-None-Match`              | 协商缓存                 |
| `Last-Modified` / `If-Modified-Since` | 协商缓存                 |
| `Access-Control-Allow-Origin`         | CORS 跨域许可            |
浏览器同源策略限制跨域请求。CORS 通过响应头授权：

```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Credentials: true
```

**预检请求**：非简单请求（如 PUT、自定义头）会先发 `OPTIONS` 请求确认。OPTIONS请求，cors预检请求的响应状态码通常是204 No Content。当使用了非简单方法比如PUT、DELETE、PATCH等的时候就会自动发送这个，或是使用了自定义请求头Authorization、X-Custom-Header也会。

### http的长连接

http长连接核心是复用一个tcp连接发送多个请求，避免每次都重新握手等。短连接是http/1.0默认，每次都要经过tcp握手。长连接是http/1.1默认，多个请求会复用一个tcp。但是http/1.1会有队头阻塞问题，同一个连接上，前一个请求还没有响应完，后面的请求要等待。

http/2的多路复用解决了这个问题：

```
一个 TCP 连接
    ├── Stream 1: 请求 A
    ├── Stream 2: 请求 B
    ├── Stream 3: 请求 C
    └── 并发传输，互不阻塞
```

通过一个连接并发多个请求，将请求/响应拆成帧，交错传输和头部压缩的方式解决了，http/2的长连接更加高效，一个连接搞定所有请求。

讲一下长连接和头部压缩：

长连接通过Connection: keep-alive声明，依靠tcp keep-alive传输层：

```
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &on, sizeof(on));
```

可以检测连接是否还或者，对端挂了，网络断了。应用层自己会发送心跳包，比如说30s一个，来保持链接活跃。如果说应用崩溃，连接不会通过FIN关闭的话，内核会替他关闭，可能是RST，也可能是FIN，比如说进程如果被SIGKILL 杀死，还有未发送的消息，他就会发RST，处理完了就会发FIN。

头部压缩式http/2的核心优化，因为http/1.1头部每次都会携带大量重复头部：

```
GET /index.html
Host: example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
Accept: text/html,application/xhtml+xml,...
Accept-Language: zh-CN,zh;q=0.9
Accept-Encoding: gzip, deflate, br
Cookie: session=abc123; theme=dark; ...
Connection: keep-alive
```

下一个请求也是一样。每个头部都会重复传递这些，显然会浪费带宽，**HTTP/1.1 用 gzip 压缩 body，但不压缩头部**——头部一直是明文。**HTTP/2 引入头部压缩**——**HPACK**。61个常见头部，然后给编号压缩，这就是静态表，对于连接期间动态维护的头部，使用索引来压缩，这就是静态表。对字符串使用哈夫曼编码，这样就可以让头部压缩很大一部分。

### https

我们都知道http是明文传输，数据裸奔，中间人可以直接看到密码、cookie等信息 还能篡改，https可以用tls解决这些问题，https分为四层：

```
HTTP          ← 应用层
  ↓
TLS/SSL       ← 安全层（加解密、认证、完整性）
  ↓
TCP           ← 传输层
  ↓
IP            ← 网络层
```

https = http + tls，http报文会被tls加密之后走tcp传输。tls使用非对称加密，在握手阶段解决。公钥是公开的，私钥是服务器自己保留，握手阶段会协商出**会话秘钥**，对称加密使用一个秘钥加解密，在**非对称加密协商**的时候，这个秘钥会被协商出来。

我们用tls1.2版本解释：

```
客户端                                服务器
  │                                    │
  │ ① Client Hello                    │
  │   - 支持的 TLS 版本                 │
  │   - 支持的加密套件                   │
  │   - 客户端随机数（Random1）          │
  │ ─────────────────────────────────> │
  │                                    │
  │ ② Server Hello                    │
  │   - 选定的 TLS 版本                 │
  │   - 选定的加密套件                   │
  │   - 服务器随机数（Random2）          │
  │ <───────────────────────────────── │
  │                                    │
  │ ③ Certificate                     │
  │   - 服务器证书（含公钥）             │
  │ <───────────────────────────────── │
  │                                    │
  │ ④ Server Key Exchange（可选）      │
  │   - 密钥交换参数                    │
  │ <───────────────────────────────── │
  │                                    │
  │ ⑤ Server Hello Done               │
  │ <───────────────────────────────── │
  │                                    │
  │ ⑥ 客户端验证证书                    │
  │   - 检查证书是否可信                 │
  │   - 检查域名是否匹配                 │
  │   - 检查是否过期                    │
  │                                    │
  │ ⑦ Client Key Exchange             │
  │   - 用服务器公钥加密"预主密钥"       │
  │ ─────────────────────────────────> │
  │                                    │
  │ ⑧ Change Cipher Spec              │
  │   - 通知"后续用协商的密钥加密"       │
  │ ─────────────────────────────────> │
  │                                    │
  │ ⑨ Finished                        │
  │   - 加密的握手摘要                  │
  │ ─────────────────────────────────> │
  │                                    │
  │ ⑩ Change Cipher Spec              │
  │ <───────────────────────────────── │
  │                                    │
  │ ⑪ Finished                        │
  │ <───────────────────────────────── │
  │                                    │
  │ ⑫ 开始加密通信（对称加密）           │
  │ <════════════════════════════════> │
```

客户端会检查证书链，证书是否是可信CA签发，浏览器会内置CA列表。证书里面的域名是否匹配访问的域名，以及是否过期。客户端会生成**预主秘钥**，用于服务器公钥加密，发送给服务器，服务器用**私钥解密**，拿到预主密钥。

```
主密钥 = PRF(预主密钥, "master secret", Random1 + Random2)
```

主密钥用来生成会话秘钥，也就是对称加密的秘钥，这样握手就完成了。

### tcp的可靠

tcp工作在不可靠的IP层之上，它之所以可靠是因为依靠 序列号、ACK、超时重传、快速重传、校验和、连接管理。

传输过程中每个字节都有自己的编号：

```
发送方：字节 1-1000 是第 1 段，1001-2000 是第 2 段
接收方：收到后按序号重组，乱序也能还原
```

这样重复的段会按照序号丢弃，乱序到达之后会按照序号重排，确认应答后告诉对方收到了哪些。接收方收到数据之后会回复一个ACK：

```
发送方 → [SEQ=1, 1000字节] → 接收方
发送方 ← [ACK=1001]        ← 接收方
```

ACK=1001 意思是"1000 及之前的字节我都收到了，下一个期望 1001。然后这样累计确认即可。发送发发完数据之后会启动定时器，接着等待ACK，如果没有就会重传SEQ=1， **超时时间（RTO）动态计算**：根据 RTT（往返时间）动态调整。快速重传不需要等待超时，收到3个重复ACK就重传。
```
发送 [1][2][3][4][5]
接收方收到 [1][3][4][5]，缺 [2]
接收方回 ACK=2（重复 3 次）
发送方收到 3 个 ACK=2 → 立即重传 [2]
```
每个TCP都有校验和，用于检测数据是否损坏。连接管理让双方完成三次握手建立连接、四次挥手关闭，能够确保双方都准备好，确保数据都传输完成。

顺便讲解udp，udp是故意不做可靠性保证的，他把控制器交给了应用层，udp也有校验和，但是没有序列号、ack、重传、排序、去重、流量控制等等行为。

### tcp窗口

窗口是tcp的流量控制机制，接收方可以提醒发送方自己还能接受多少。接受窗口rwnd是接收方的剩余缓冲区大小：

```
接收方缓冲区 64KB
已用 16KB
剩余 48KB → 告诉发送方"窗口 = 48KB"
```

然后发送方就可以根据窗口大小决定能发多少，这是为了防止接收方被大量的流量淹没，有窗口才能让双方能够按照节奏来发送。

窗口为0的话，发送方就会停止发送，并启动零窗口探测定时器，询问接收方窗口是否开了，当接收方处理完之后窗口才会>0，然后通知发送方继续，如果接收方窗口开了但通知丢了，发送方靠探测恢复。

并且有指标可以计算：  **吞吐量 = 窗口大小 / RTT**

```
窗口 64KB，RTT 100ms
吞吐量 = 64KB / 0.1s = 640KB/s ≈ 5Mbps

窗口 64KB，RTT 10ms
吞吐量 = 64KB / 0.01s = 6.4MB/s ≈ 51Mbps
```

RTT越大，窗口越重要，RTT越大的话，我们自然需要窗口越大，不然吞吐量上不去。

TCP头部窗口字段只有16位，也就是最大64KB，如果还不够，可以使用窗口缩放选项RFC 1323，能够把窗口扩大到8MB，现代TCP窗口都会启用窗口缩放。

tcp窗口分几个层次：内核自动管理、系统参数调整、应用层通过socket选项来间接影响。应用层**看不到也改不了** TCP 窗口字段本身。窗口是内核根据**接收缓冲区剩余空间**自动计算的。

linux可以通过命令查看：

```
# 看系统级参数
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem

# 输出示例：
# net.ipv4.tcp_rmem = 4096 131072 6291456
#                    min  default  max
```

### 三次握手、四次挥手

TCP 的三次握手建立连接，四次挥手关闭连接。这两个过程的核心是**序列号同步**和**双向确认**。


```
客户端                                服务器
  │                                    │
  │ ① SYN                              │
  │   SEQ=x（客户端初始序列号）           │
  │   SYN=1                            │
  │ ─────────────────────────────────> │
  │                                    │
  │ ② SYN + ACK                        │
  │   SEQ=y（服务器初始序列号）           │
  │   ACK=x+1（确认收到 x）              │
  │   SYN=1, ACK=1                     │
  │ <───────────────────────────────── │
  │                                    │
  │ ③ ACK                              │
  │   SEQ=x+1                          │
  │   ACK=y+1（确认收到 y）              │
  │   ACK=1                            │
  │ ─────────────────────────────────> │
  │                                    │
  │ 连接建立，开始传数据                  │
```

第一次握手客户端进入SYN_SENT状态，发送SYN=1，SEQ=X。第二次服务器发送SYN=1 ACK=X+1  SEQ=Y,服务器进入SYN_RCVD。第三次握手客户端收到Y，连接建立，回复ACK=Y+1，双方进入ESTABLISHED。

初始序列号是随机的，是**防止历史连接的数据混入新连接**。当新连接的序列号和旧连接不同，旧包会被丢弃，ISN生成基于时钟+哈希防止被猜测。

四次挥手用于关闭连接：

```
客户端                                服务器
  │                                    │
  │ ① FIN                              │
  │   SEQ=u                            │
  │   FIN=1                            │
  │ ─────────────────────────────────> │
  │                                    │
  │ ② ACK                              │
  │   SEQ=v                            │
  │   ACK=u+1                          │
  │   ACK=1                            │
  │ <───────────────────────────────── │
  │                                    │
  │   （服务器可能还在发数据）            │
  │                                    │
  │ ③ FIN                              │
  │   SEQ=w                            │
  │   ACK=u+1                          │
  │   FIN=1, ACK=1                     │
  │ <───────────────────────────────── │
  │                                    │
  │ ④ ACK                              │
  │   SEQ=u+1                          │
  │   ACK=w+1                          │
  │   ACK=1                            │
  │ ─────────────────────────────────> │
  │                                    │
  │ 连接关闭                            │
```

第一次挥手，客户端发送FIN=1 SEQ=U，客户端进入FIN_WAIT_1，第二次挥手服务端收到，发送ACK=U+1 SEQ=V，服务器进入CLOSE_WAIT，客户端进入FIN_WAIT_2，第三次挥手，服务器已经发完数据，发送FIN=1 ACK=1 SEQ=W ACK=U+1，服务器进入LAST_ACK状态，最后一次客户端收到FIN，发送SEQ=U+1 ACK=W+1，客户端进入TIME_WAIT状态，2MSL之后关闭。

### SSE

全称叫做服务器推送事件，允许服务器单向、持续地向客户端推送文本数据的技术，基于HTTP协议，专门用于实现服务器到客户端的实时通信。

只有服务器能够主动推送给客户端，而不允许客户端通过同一连接反向发送。使用标准的HTTP请求，无需特殊协议或者端口，只支持UTF8文本，二进制数据需要编码比如base64,。浏览器内置了断线重连机制，默认约3s后重试，相比ws，实现更加简单，适合推送场景。

客户端发起请求的时候会通过EventSource API向服务器发起一个普通HTTP GET请求，服务器响应头设置Content-Type: text/event/stream，之后就可以保持连接不关闭，服务器会按照SSE格式不断向客户端发送数据，客户端接受事件之后浏览器会解析数据流，出发对应的事件回调。


```ts
const eventSource = new EventSource('/api/stream');

// 接收默认消息
eventSource.onmessage = (event) => {
    console.log('收到消息:', event.data);
};

// 接收自定义事件
eventSource.addEventListener('update', (event) => {
    console.log('更新事件:', event.data);
});

// 错误处理
eventSource.onerror = (err) => {
    console.error('连接错误:', err);
};

// 关闭连接
// eventSource.close();
```

响应头：

```cpp
std::string get_sse_headers() {
    response << "HTTP/1.1 200 OK\r\n"
        << "Content-Type: text/event-stream\r\n"   // 必需：SSE 的 MIME 类型
        << "Cache-Control: no-cache\r\n"           // 禁止缓存
        << "Connection: keep-alive\r\n"            // 保持长连接
        << "Access-Control-Allow-Origin: *\r\n"    // CORS 跨域
        << "\r\n";
}
```

 `Content-Type: text/event-stream` 是 SSE 的**标志性头部**，浏览器据此识别为事件流。 头部以 `\r\n` 分隔，最后用一个空行 `\r\n\r\n` 结束。


```cpp
void static send_sse_event(SOCKET client, const std::string& event, const std::string data) {
	std::string message = "event: " + event + "\r\n"
		"data: " + data + "\r\n\r\n";
	send(client, message.c_str(), message.size(), 0);
}
```

这个是SSE中自定义事件的完整报文格式，由两行字短组成，用空行结束：

```
event: heartbeat\r\n
data: ping count: 5\r\n
\r\n
```

浏览器侧：

```ts
const es = new EventSource('/api/stream');

// 监听默认消息
es.onmessage = (e) => {
    console.log('默认消息:', e.data);
};

// 监听 heartbeat 自定义事件
es.addEventListener('heartbeat', (e) => {
    console.log('心跳事件:', e.data);
});
```



## 跨端和高性能

### electron

Electron 的 IPC是它的核心架构，主进程是一个node环境，用于管理窗口、声明周期、系统api，能够访问完整的nodejs api，比如说fs、path、child_process，不能直接操作dom，渲染进程是chromium环境，每个窗口一个，能够操作dom、跑前端代码，默认不能访问nodejs。

每个渲染进程在启动前，会加载一个预加载脚本，我们一般叫做preload.js，用于定义能访问的部分nodejs api，是主进程和渲染进程的bridge。

它提供的有三种模式，分别是渲染进程->主进程，有invoke和handle，渲染进程请求、主进程响应：

```js
// 主进程
const { ipcMain } = require('electron');

ipcMain.handle('read-file', async (event, filePath) => {
    const content = await fs.promises.readFile(filePath, 'utf-8');
    return content;   // 返回给渲染进程
});

// 预加载脚本
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('api', {
    readFile: (path) => ipcRenderer.invoke('read-file', path)
});

// 渲染进程
const content = await window.api.readFile('/path/to/file');
```

它的特点是promise风格，异步有返回值。

另一种是渲染进程->主进程，使用send和on完成，是单向的，渲染进程发，主进程收，但是不返回。


```js
// 主进程
ipcMain.on('log', (event, message) => {
    console.log('渲染进程说:', message);
    // 不返回
});

// 预加载
contextBridge.exposeInMainWorld('api', {
    log: (msg) => ipcRenderer.send('log', msg)
});

// 渲染进程
window.api.log('hello');
```

主进程单向到渲染进程：

```js
// 主进程
mainWindow.webContents.send('update', { version: '1.0.0' });
// 预加载
contextBridge.exposeInMainWorld('api', {
    onUpdate: (callback) => ipcRenderer.on('update', (event, data) => callback(data))
});
// 渲染进程
window.api.onUpdate((data) => {
    console.log('收到更新:', data);
});
```

ipc的底层是chromium的mojo ipc，mojo是chromium的ipc框架，管理二进制序列化、异步、跨进程，electron封装了mojo，能够提供ipcRenderer、ipcMain。

mojo在win32平台的底层传输是基于命名管道实现的，函数在 Windows 下会调用 `CreateNamedPipeW` 来创建管道。一个专门抓取 Chromium IPC 的工具也明确指出，它捕获的就是 **Named Pipe** 的流量

值得一提的是，从 Chrome 112 开始，Chromium 逐渐用新的 **ipcz** 协议替代了 Mojo Core。ipcz 为了优化性能，在 Windows 上可能会更多地使用**共享内存**来传递数据，而不是完全依赖命名管道，但这属于更底层的优化实现。

安全方面：

contextIsolation是默认开启的，渲染进程和预加载脚本上下文隔离。nodeInteration也是默认关闭的，渲染进程肯定不能访问node。sandbox沙箱默认开启，渲染进程在沙箱里。webSecurit默认开启，同源策略。

生命周期方面：

app.whenReady()是app初始化完成的时候，创建窗口。
window-all-closed是所有窗口都关闭，macos不退出。
before-quit: 退出前，可以拦截
will-quit:即将推出
quit:已退出
ready:app就绪
second-instance:第二个实例启动

窗口管理：

BrowserWindow:创建窗口
webPreferences:安全配置preload、contextisloation、nodeintergration
loadURL/loadFile：加载远程URL/本地文件
ready-to-show:窗口准备好显示，避免白屏
frame:false:无边框窗口，自定义标题栏
show:false + read-to-show 先隐藏，准备好在显示，可以消除白屏
transparent:true透明窗口
alwaysOnTop：置顶
webContents:窗口的web内容，可以`send`、`openDevTools`

preload脚本是渲染进程启动前，DOM加载前执行，能够访问`contextBridge`、`ipcRenderer`、部分 Node API。

### Emscripten

emscripten可以把c/c++代码翻译为浏览器和nodejs能直接运行的wasm模块，并附带一个js胶水文件。

emscripten的核心工具是emcc，内部串联了多个工具，分别是前端clang/llvm，可以将我们的c/c++代码编译为llvm的ir码，目标是WebAssembly平台。后端llvm wasm backend将llvm ir转换为WebAssembly目标文件.o或者.a。最后链接和优化，wasm-ld链接器将所有目标文件打包成一个.wasm二进制模块，之后Binaryen工具链会对这个模块进行优化，让他的体积更小，运行更快。emcc会生成一个js胶水文件，作用是加载、编译、实例化.wasm文件，并处理c/c++与js之间的所有通信，比如内存分配，函数调用和文件系统模拟。

C/C++ 代码常需要读写文件，但浏览器出于安全考虑，不允许 JS 直接访问本地文件。Emscripten 的解决方案是**虚拟文件系统**

胶水层代码是自动生成，但什么一些情况下需要手动补充js：
比如说调用导出函数，我们在前端要自己调用；字符串和数组之间需要转换，c的 `char*`和js的string不互通，因此需要手动转换，生成的js提供了 `stringToNewUTF8`、`UTF8ToString` 这些工具，你负责组合；然后C要调js函数比如事件回调，你需要在C侧写

```
// C 侧
typedef void (*Callback)(int);
void set_callback(Callback cb);

// JS 侧
const callback = Module.addFunction((value) => {
    console.log('C 调用了 JS:', value);
}, 'vi');
Module._set_callback(callback);
```

所以我们一般分为core、common、platform的实现，core是平台无关核心实现，然后common是包装c abi，最后platform是平台适配。
## 认证和授权

### OAuth


Oauth2.0是一套授权框架，目标是解决第三方应用如何在不获取用户密码的前提下，代表用户去访问受保护的资源。oauth体系里面，有四个角色，分别是资源所有者、客户端、授权服务器、资源服务器。

资源所有者一般就是用户，是数据拥有者，有权决定是否授权，客户端是想要访问用户数据的第三方应用，比如有个工具想要读取你gayhub仓库列表的项目管理。授权服务器用于验证身份、征得同意、颁发令牌，gayhub也有oauth登录服务。资源服务用于存储受保护的api服务，如github有restapi，需要令牌，不认密码。

然后是授权模式，首先是授权码模式，这个是最安全、最常用的模式。用户需要发请请求，比如阿紫客户端点击使用github登录，重定向到授权服务器中，客户端把用户的浏览器重定向到github的授权服务器，请求用户授权。用户需要登录并授权，同意授权给客户端相应的权限。完成之后授权服务器把浏览器重定向会客户端，并在URL上带上一个授权码，客户端在后台用授权码+客户端秘钥向授权服务器换取访问令牌，客户端就可以通过访问令牌去资源服务器上请求用户的数据了。

隐式模式不推荐，它是令牌慧姐通过URL返回，容易泄漏，无法刷新。密码模式已禁止，用户需要把密码直接给客户端，从根本上违背了OAuth的设计。

客户端机密模式是服务间调用，用户无参与，客户端会以自己的身份获取令牌，用于访问自己的资源，不涉及用户。

OAuth本身留下了很多安全细节，最新实践RFC 9700明确表示强制使用PKCE，所有的客户端类型，尤其是公开客户端SPA 移动APP都需要使用PKCE来防止授权码被拦截。授权服务器必须对重定向uri进行精确的字符串匹配，不能使用通配符或者模式匹配，防止授权码被重定向到恶意地址。

注意： OAuth只负责授权，不负责认证，使用Google登录这种选项实际上是OAuth2.0 + OIDC的组合。**OIDC** 是 OAuth 2.0 之上的一个**身份层**，它额外提供了一个 **ID 令牌 (ID Token)** ，这是一个包含用户身份信息（如姓名、邮箱）的 JWT，用来证明用户身份，之后会讲。

为什么访问资源需要客户端秘钥，这个是为了证明客户端身份，因为授权服务器需要确认客户端是否真的是我们自己规定的客户端，如果说授权码被拦截也无所谓，有了客户端秘钥才能换令牌。

### OIDC

然后我们来谈谈oidc部分，它是在oauth2之上的身份认证层，核心目标是让客户端能验证用户，它在 OAuth 2.0 的授权流程上，额外返回一个 **ID Token**（JWT）。

因为oauth2只会给access_token，也就是用于访问资源的访问令牌，oidc额外返回了id_token，id_token是jwt，能够包含用户信息，客户端解 id_token。然后客户端是可以验证id token里面的签名的，需要使用JWKS，我们可以通过一个get请求，去访问公钥集合：

```
GET https://auth.example.com/.well-known/jwks.json

{
    "keys": [
        {
            "kty": "RSA",
            "kid": "key-1",
            "use": "sig",
            "n": "...",
            "e": "AQAB"
        }
    ]
}
```

并且scope里面必须包含openid，不然就不是oidc。**OIDC 定义了标准 scope**：

|scope|含义|返回的 claim|
|---|---|---|
|`openid`|**必须**，表示 OIDC|`sub`|
|`profile`|基本资料|`name`、`family_name`、`picture` 等|
|`email`|邮箱|`email`、`email_verified`|
|`address`|地址|`address`|
|`phone`|电话|`phone_number`、`phone_number_verified`|

oidc的使用场景：

SSO 单点登录，用户登录一次，可以多个应用共享身份，每个应用用oidc来验证用户。比如企业sso，一个idp，多个应用。

社交登录：使用google登录，是使用oauth2.0+oidc，拿id_token->验证用户。

移动App：调用系统浏览器访问授权服务器返回id_token。然后客户端是可以验证id


## 工程

### 微前端

这是一种将大型前端应用拆分成多个小型、独立、可独立开发和部署的子应用的架构模式，核心理念借鉴了后端微服务，可以把一个庞大的单体前端，拆分为多个可以独立开发、测试、部署的小块，最后在运行时或者构建的时候组合为一个完整应用。

这样的优点很明显，子应用独立构建，互不影响。多团队协作，技术栈可以不同，但是技术栈升级困难，发布风险高，耦合严重。

子应用会被作为npm包引入，构建的时候打包在一起。实现方案也有很多种，比如可以用iframe，每个子应用一个，简单但是通信、样式、路由受限。

可以用web component，自定义元素封装子应用，元素隔离，但是生态兼容性有限。

webpack5支持module federation，构建的时候+运行时结合，生态比较成熟。

qiankun是一个微前端实现库，能够无痛构建一个生产可用的微前端框架系统，qiankun原子蚂蚁金融科技基于微前端架构的云产品统一接入平台，现在已经非常完备了。qiankun 对于用户而言只是一个类似 jQuery 的库，你需要调用几个 qiankun 的 API 即可完成应用的微前端改造。同时由于 qiankun 的 HTML entry 及沙箱的设计，使得微应用的接入像使用 iframe 一样简单。





