---
date: 2026-10-14
tags:
  - css
  - bem
---
BEM 是 `Block__Element--Modifier` 的命名规范，我们经常会在嵌套语法里使用到，核心目的是**让类名自解释、避免样式冲突、方便团队协作**。

## BEM 的三个部分

|部分|英文|含义|写法|
|---|---|---|---|
|**B**|Block|块，独立的功能单元|`.block`|
|**E**|Element|元素，块的一部分|`.block__element`|
|**M**|Modifier|修饰符，块或元素的状态/变体|`.block--modifier`|

我们之前在css优先级里讲过块级元素有哪些（有自己的语义。比如 `.header`、`.button`、`.card`、`.menu`。），这里不复述：

```html
<div class="card">...</div>
```

元素是块级元素内的组成部分，不能脱离块级元素单独使用，需要用下划线，必须说你可以用一个下划线来连接：

```html
<div class="card">
  <h2 class="card__title">标题</h2>
  <p class="card__text">内容</p>
  <img class="card__image" src="..." />
</div>
```

`card__title` 表示“card 这个块里的 title 元素”。它**只对 card 有意义**，换个块就不叫这个名字。

修饰符用来修饰块或者元素的状态、变化、外观变化，我们一般使用双连字符--来连接：

```html
<button class="button button--primary">主要按钮</button>
<button class="button button--large">大按钮</button>
<button class="button button--disabled">禁用按钮</button>
```

`button--primary` 表示“button 这个块的主要变体”。修饰符通常**和基础类一起用**，不单独出现。

这样做的优点很好懂，可以很好避免和普通命名的冲突，bem这种方式可以自带作用域。并且这样的类名自己携带解析，一眼就清楚是哪个组件使用的什么元素。我们在scss里经常使用嵌套语法，这样也是可以很好的偏好使用bem命名法：

```scss
.head {
  margin-bottom: 8px;

  &__title {
    font-size: 14px;
  }

  &__meta {
    font-size: 11px;
  }

  &--large {
    font-size: 18px;
  }
}
```

## 哪些库遵守了bem准则

最经典的Element Plus就是使用的bem，比如el-button_text这样的，一眼就看得出使用的BEM。还有就是Ant Design的 ant-btn... 的中后台经常使用的组件库，还有Bulma部分使用bem的modal-card-head等。

