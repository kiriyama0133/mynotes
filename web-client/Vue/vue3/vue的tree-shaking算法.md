### Tree-Shaking 的起源

​	Tree-Shaking 起源于 [ES6 模块系统](https://zhida.zhihu.com/search?content_id=247751425&content_type=Article&match_order=1&q=ES6+模块系统&zhida_source=entity)。在 ES6 模块中，每个模块都有一个明确的入口点，即模块的顶层作用域。当模块被导入时，顶层作用域中的所有变量和函数都会被导入。

​	然而，**有些变量和函数可能没有被使用到，这就导致了代码的冗余**。Tree-Shaking 的目的是通过分析模块之间的依赖关系，**找出那些没有被使用的代码，并将其从最终的打包文件中移除**。

### Tree-Shaking 的原理

​	Tree-Shaking 的原理是基于[静态分析](https://zhida.zhihu.com/search?content_id=247751425&content_type=Article&match_order=1&q=静态分析&zhida_source=entity)。在打包过程中，**打包工具会遍历所有的模块，并记录下每个模块的导出和导入关系。然后，通过分析这些关系，找出那些没有被使用的模块和代码**。最后，将这些无用代码从打包文件中移除。

​	具体来说，Tree-Shaking 的核心思想是“只打包需要的代码”。**当一个模块被导入时，打包工具会检查该模块中的所有导出变量和函数**。**如果这些变量和函数没有被其他模块导入，那么它们就不会被打包到最终的文件中**。这样，就可以有效地去除冗余代码，减少文件大小。

### Tree-Shaking 技术实现

### 打包工具支持

要实现 Tree-Shaking，首先需要使用支持 Tree-Shaking 的打包工具。目前主流的前端打包工具如 [Webpack](https://zhida.zhihu.com/search?content_id=247751425&content_type=Article&match_order=1&q=Webpack&zhida_source=entity)、[Rollup](https://zhida.zhihu.com/search?content_id=247751425&content_type=Article&match_order=1&q=Rollup&zhida_source=entity) 等都支持 Tree-Shaking。以 Webpack 为例**，可以通过在 Webpack 配置文件中设置 mode 为 production 来启用 Tree-Shaking。**

### [代码拆分](https://zhida.zhihu.com/search?content_id=247751425&content_type=Article&match_order=1&q=代码拆分&zhida_source=entity)

在 Tree-Shaking 中，代码拆分是一种重要的优化策略。通过将代码拆分成更小的模块，可以减少冗余代码的数量，并提高 Tree-Shaking 的效率。

### 摇树优化注意事项

Tree Shaking 只支持 ESM 的引入方式，不支持 CommonJS 引入方式

- ESM: export + import
- CommonJS: module.exports + require

### 定义三种类型的方法

- func.js 定义函数

```js
// 引用并被调用的方法
export function quoteAndUse() {
  console.log("这是use专属的方法，别人都");
  return;
  console.log("这是use中return后面的");
}
// 引用但是未被调用的方法
export function quoteButNotUse() {
  console.log("这是quoteButNotUse方法");
}
// 未被引用的方法
export function notQuote() {
  console.log("这是未被引用的方法");
}
```

- 在页面中不同场景进行使用 notQuote 未被引用的方法 quoteButNotUse 引用但是未被调用的方法 quoteAndUse 引用却已被调用的方法

```js
import { quoteButNotUse, quoteAndUse } from './func'

testUse() {
  quoteAndUse()
}
```

可以设置**生产环境生效 dev 环境打包**： 打包结果发现，所有的代码都被打包了，三种方法均存在，但是下面两个未被引用或使用的方法被标注了 unused harmony export（声明未使用）

prod 环境打包： 无法到达的代码（例如 return 后面的代码）、引用未被使用的方法、 未被引用的方法、注释信息， 都不会被打包到 bundle.js 中

### 摇树优化副作用

**sideEffects 指的是有副作用的代码，假如模块 A 中包含一些影响全局作用域（非模块作用域）的代码**，如果改变了全局的变量或对象的变量，**模块 B 引入了模块 A，但并未使用。在这种情况下，模块 A 就被认为是有副作用的，webpack 是不会删除模块 A 中所有未使用的代码的**，它还会保留模块 A 中立即执行并对全局环境有影响的代码。

- 新建 about.js 文件： 编写 about 方法，挂载到全局，在 main.js 中引入，但未真正调用

![img](./assets/v2-10d1031dcb4e5968e18456cc9d917251_1440w.jpg)

在 main.ts 中引入,未使用 about 方法

```ts
import { about } from "./about.js";
```

- 打包后：我们会发现 about 会被打包在 bundle.js 中

![img](./assets/v2-7d3d00cd185345f21eb093a7f3164867_1440w.jpg)

- 如果**设置了"sideEffects": false，看下面总结说明 请注意： 任何导入的文件都会收到树抖动的影响**。如果 css-loader 在项目中使用类似的东西并导入 css 文件，则需要将其添加到副作用列表中，这样它就不会在生产模式下无意中被删除。 如果有某些模块是有副作用的，那么可以将它的路径加入 package.json 的 sideEffects 数组选项中，这样也能提高删除未使用代码的速度。
- 通过以上描述，我们可以把样式文件和我们最开始的 about 文件添加在副作用数组中

```json
{
  "name": "your-project",
   "sideEffects": ["about.js", "*.css"],
}
```

使用示例对比： 新建 index.css 文件，用于编写样式

![img](./assets/v2-cc4f7099ccbd0f1e697935bbf0c82d9f_1440w.jpg)

在 main.js 中进行引入

![img](./assets/v2-3fc15d652ebabd4d5362fb2a1abd5089_1440w.jpg)

**总结**

```js
// 所有文件都有副作用，全都不可 tree-shaking
{
 "sideEffects": true
}

// 没有文件有副作用，全都可以 tree-shaking，即告知 webpack，它可以安全地删除未用到的 export。
{
 "sideEffects": false
}

// 除了数组中包含的文件外有副作用，所有其他文件都可以 tree-shaking，但会保留符合数组中条件的文件
{
 "sideEffects": [
   "*.css",
   "*.less"
 ]
```