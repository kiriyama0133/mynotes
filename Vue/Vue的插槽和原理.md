
## 样例代码

```vue
<!-- 子组件comA -->
<template>
  <div class='demo'>
    <slot><slot>
    <slot name='test'></slot>
    <slot name='scopedSlots' test='demo'></slot>
  </div>
</template>
<!-- 父组件 -->
<comA>
  <span>这是默认插槽</span>
  <template slot='test'>这是具名插槽</template>
  <template slot='scopedSlots' slot-scope='scope'>这是作用域插槽（老版）{{scope.test}}</template>
  <template v-slot:scopedSlots='scopeProps' slot-scope='scope'>这是作用域插槽（新版）{{scopeProps.test}}</template>
</comA>
```

## 实现原理

`vue`组件实例化顺序为：父组件状态初始化(`data`、`computed`、`watch`...) --> 模板编译 --> 生成`render`方法 --> 实例化渲染`watcher` --> 调用`render`方法，生成`VNode` --> `patch VNode`，转换为真实`DOM` --> 实例化子组件 --> ......重复相同的流程 --> 子组件生成的真实`DOM`挂载到父组件生成的真实`DOM`上，挂载到页面中 --> 移除旧节点

从上述流程中，可以推测出：

1. 父组件模板解析在子组件之前，所以父组件首先会获取到插槽模板内容
2. 子组件模板解析在后，所以在子组件调用`render`方法生成`VNode`时，可以借助部分手段，拿到插槽的`[VNode](https://zhida.zhihu.com/search?content_id=173063685&content_type=Article&match_order=4&q=VNode&zhida_source=entity)`节点
3. 作用域插槽可以获取子组件[内变量](https://zhida.zhihu.com/search?content_id=173063685&content_type=Article&match_order=1&q=%E5%86%85%E5%8F%98%E9%87%8F&zhida_source=entity)，因此作用域插槽的`VNode`生成，是动态的，即需要实时传入子组件的作用域`scope`

以下面代码为例，简要概述插槽运转的过程。

```html
<div id='app'>
  <test>
    <template slot="hello">
      123
    </template>
  </test>
</div>
<script>
  new Vue({
    el: '#app',
    components: {
      test: {
        template: '<h1>' +
          '<slot name="hello"></slot>' +
          '</h1>'
      }
    }
  })
</script>
```

### 父组件编译阶段

编译是将模板文件解析成`AST`语法树，会将插槽`template`解析成如下[数据结构](https://zhida.zhihu.com/search?content_id=173063685&content_type=Article&match_order=1&q=%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84&zhida_source=entity)：

```js
{
  tag: 'test',
  scopedSlots: { // 作用域插槽
    // slotName: ASTNode,
    // ...
  }
  children: [
    {
      tag: 'template',
      // ...
      parent: parentASTNode,
      children: [ childASTNode ], // 插槽内容子节点，即文本节点123
      slotScope: undefined, // 作用域插槽绑定值
      slotTarget: "\"hello\"", // 具名插槽名称
      slotTargetDynamic: false // 是否是动态绑定插槽
      // ...
    }
  ]
}
```

### 父组件生成渲染方法

根据`AST`[语法树](https://zhida.zhihu.com/search?content_id=173063685&content_type=Article&match_order=2&q=%E8%AF%AD%E6%B3%95%E6%A0%91&zhida_source=entity)，解析生成渲染方法字符串，最终父组件生成的结果如下所示，这个结构和我们直接写`render`方法一致，本质都是生成`VNode`, 只不过`_c`或`h`是`this.$createElement`的缩写。

```js
with(this){
  return _c('div',{attrs:{"id":"app"}},
  [_c('test',
    [
      _c('template',{slot:"hello"},[_v("\n      123\n    ")])],2)
    ],
  1)
}
```

### 父组件生成VNode

调用`render`方法，生成`VNode`,`VNode`具体格式如下：

```js
{
  tag: 'div',
  parent: undefined,
  data: { // 存储VNode配置项
    attrs: { id: '#app' }
  },
  context: componentContext, // 组件作用域
  elm: undefined, // 真实DOM元素
  children: [
    {
      tag: 'vue-component-1-test',
      children: undefined, // 组件为页面最小组成单元，插槽内容放放到子组件中解析
      parent: undefined,
      componentOptions: { // 组件配置项
        Ctor: VueComponentCtor, // 组件构造方法
        data: {
          hook: {
            init: fn, // 实例化组件调用方法
            insert: fn,
            prepatch: fn,
            destroy: fn
          },
          scopedSlots: { // 作用域插槽配置项，用于生成作用域插槽VNode
            slotName: slotFn
          }
        },
        children: [ // 组件插槽节点
          tag: 'template',
          propsData: undefined, // props参数
          listeners: undefined,
          data: {
            slot: 'hello'
          },
          children: [ VNode ],
          parent: undefined,
          context: componentContext // 父组件作用域
          // ...
        ] 
      }
    }
  ],
  // ...
}
```

在`vue`中，**组件是页面结构的基本单元**，从上述的`VNode`中，我们也可以看出，`VNode`页面层级结构结束于`test`组件，`test`组件`children`处理会在子组件初始化过程中处理。子组件构造方法组装与属性合并在**vue-dev\src\core\vdom\create-component.js** `createComponent`方法中，组件实例化调用入口是在**vue-dev\src\core\vdom\patch.js** `createComponent`方法中。

### 子组件状态初始化

**实例化子组件时**，会在`initRender` -> `resolveSlots`方法中，将子组件插槽节点挂载到组件作用域`vm`中，挂载形式为`vm.$slots = {slotName: [VNode]}`形式。

### 子组件编译阶段

子组件在**编译阶段**，会将`slot`节点，编译成以下`AST`结构：

```js
{
  tag: 'h1',
  parent: undefined,
  children: [
    {
      tag: 'slot',
      slotName: "\"hello\"",
      // ...
    }
  ],
  // ...
}
```


### 子组件生成渲染方法

生成的**渲染方法**如下，其中`_t`为`renderSlot`方法的简写，从`renderSlot`方法，我们就可以直观的将**插槽内容**与**插槽点**联系在一起。

```js
// 渲染方法
with(this){
  return _c('h1',[ _t("hello") ], 2)
}
// 源码路径：vue-dev\src\core\instance\render-helpers\render-slot.js
export function renderSlot (
  name: string,
  fallback: ?Array<VNode>,
  props: ?Object,
  bindObject: ?Object
): ?Array<VNode> {
  const scopedSlotFn = this.$scopedSlots[name]
  let nodes
  if (scopedSlotFn) { // scoped slot
    props = props || {}
    if (bindObject) {
      if (process.env.NODE_ENV !== 'production' && !isObject(bindObject)) {
        warn(
          'slot v-bind without argument expects an Object',
          this
        )
      }
      props = extend(extend({}, bindObject), props)
    }
    // 作用域插槽，获取插槽VNode
    nodes = scopedSlotFn(props) || fallback
  } else {
    // 获取插槽普通插槽VNode
    nodes = this.$slots[name] || fallback
  }

  const target = props && props.slot
  if (target) {
    return this.$createElement('template', { slot: target }, nodes)
  } else {
    return nodes
  }
}
```

### [作用域插槽](https://zhida.zhihu.com/search?content_id=173063685&content_type=Article&match_order=10&q=%E4%BD%9C%E7%94%A8%E5%9F%9F%E6%8F%92%E6%A7%BD&zhida_source=entity)与具名插槽区别

```html
<!-- demo -->
<div id='app'>
  <test>
      <template slot="hello" slot-scope='scope'>
        {{scope.hello}}
      </template>
  </test>
</div>
<script>
    var vm = new Vue({
        el: '#app',
        components: {
            test: {
                data () {
                    return {
                        hello: '123'
                    }
                },
                template: '<h1>' +
                    '<slot name="hello" :hello="hello"></slot>' +
                  '</h1>'
            }
        }
    })

</script>
```

作用域插槽与普通插槽相比，主要区别在于**插槽内容可以获取到子组件作用域变量**。由于需要注入子组件变量，相比于具名插槽，作用域插槽有以下几点不同：

- 作用域插槽在**组装渲染方法时**，生成的是一个**包含注入作用域的方法**，相对于`createElement`生成`VNode`，多了一层注入作用域方法包裹，这也就决定插槽`VNode`作用域插槽是在子组件生成`VNode`时生成，而具名插槽是在父组件创建`VNode`时生成。`_u`为`resolveScopedSlots`，其作用为将节点配置项转换为`{scopedSlots: {slotName: fn}}`形式。