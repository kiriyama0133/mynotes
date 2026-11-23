## 🎯 学习路径建议

第一阶段：基础架构理解 (1-2周)

1. 从 packages/shared/ 开始packages/shared/src/├── index.ts     *# 工具函数入口*├── utils.ts     *# 基础工具函数*├── types.ts     *# 类型定义*└── constants.ts   *# 常量定义*

为什么从这里开始？

- 代码量相对较小，容易理解

- 被其他所有包依赖，是基础

- 包含大量工具函数，学习价值高

2. 学习 packages/reactivity/packages/reactivity/src/├── index.ts     *# 响应式系统入口*├── reactive.ts    *# reactive API*├── ref.ts      *# ref API*├── computed.ts   *# computed API*├── watch.ts     *# watch API*└── effect.ts    *# 副作用系统*

重点学习：

- reactive() 的实现原理

- ref() 和 reactive() 的区别

- effect 的依赖收集和触发机制

- computed 的缓存机制

第二阶段：运行时核心 (2-3周)

3. 深入 packages/runtime-core/packages/runtime-core/src/├── apiCreateApp.ts  *# 应用创建*├── component.ts   *# 组件定义*├── vnode.ts     *# 虚拟节点*├── renderer.ts   *# 渲染器*├── scheduler.ts   *# 调度器*└── h.ts      *# h 函数*

学习顺序：

1. vnode.ts - 理解虚拟 DOM 结构

1. component.ts - 组件实例和生命周期

1. apiCreateApp.ts - 应用创建流程

1. renderer.ts - 渲染逻辑

1. scheduler.ts - 更新调度

4. 学习 packages/runtime-dom/packages/runtime-dom/src/├── index.ts     *# DOM 运行时入口*├── nodeOps.ts   *# DOM 操作*├── patchProp.ts  *# 属性更新*└── renderer.ts   *# DOM 渲染器*

第三阶段：编译器 (3-4周)

5. 编译器核心 packages/compiler-core/packages/compiler-core/src/├── parse.ts    *# 模板解析*├── transform.ts  *# AST 转换*├── codegen.ts   *# 代码生成*└── transforms/  *# 各种转换器*

6. SFC 编译器 packages/compiler-sfc/packages/compiler-sfc/src/├── parse.ts    *# SFC 解析*├── compileScript.ts *# 脚本编译*├── compileTemplate.ts *# 模板编译*└── compileStyle.ts *# 样式编译*

第四阶段：高级特性 (2-3周)

7. 服务端渲染 packages/server-renderer/

8. 兼容性层 packages/runtime-core/src/compat/

9. 主入口 packages/vue/

🛠️ 学习工具和方法

推荐的学习工具：

1. VS Code 插件：

- TypeScript Importer

- Vue Language Features (Volar)

- GitLens

1. 调试工具：

- Chrome DevTools

- Vue DevTools

- Node.js 调试器

学习方法：

1. 从测试用例开始*# 每个包都有对应的测试*packages/reactivity/__tests__/packages/runtime-core/__tests__/

2. 创建学习项目*# 创建简单的测试项目*mkdir vue-source-learningcd vue-source-learningnpm init -ynpm install vue@next

3. 断点调试*// 在关键函数处打断点*import { reactive, ref } from 'vue'const state = reactive({ count: 0 }) *// 在这里打断点*const count = ref(0) *// 在这里打断点*

📚 学习资源

官方资源：

1. Vue 3 官方文档

1. Vue 3 迁移指南

1. Vue RFC

推荐阅读顺序：

1. 先读 Vue 3 官方文档，理解 API 用法

1. 再看源码实现，理解内部原理

1. 结合测试用例，验证理解

🎯 具体学习建议

第一周：响应式系统*// 学习目标：理解这个简单例子背后的原理*import { reactive, effect } from 'vue'const state = reactive({ count: 0 })effect(() => { console.log(state.count) *// 当 count 变化时自动执行*})state.count++ *// 触发 effect*

关键问题：

- 为什么 state.count++ 会自动触发 effect？

- reactive 是如何工作的？

- 依赖收集是如何实现的？

第二周：组件系统*// 学习目标：理解组件创建和渲染流程*import { createApp, h } from 'vue'const App = { render() {  return h('div', 'Hello Vue 3!') }}createApp(App).mount('#app')

关键问题：

- createApp 做了什么？

- h 函数如何创建虚拟节点？

- 组件是如何渲染到 DOM 的？

## 💡 学习技巧

1. 不要一开始就深入细节，**先理解整体架构**

1. 多画图，用**思维导图整理知识点**

1. 写笔记，**记录关键概念和实现原理**

1. 动手实践，修改源码看效果

1. 参与社区，在 Vue 官方 Discord 或 GitHub 讨论

## 🚀 进阶学习

当你掌握了基础后，可以学习：

- Vue 3 的性能优化策略

- 编译器的优化技术

- 服务端渲染的实现

- 兼容性层的设计思路

记住：学习源码不是为了记住每一行代码，而是理解设计思路和实现原理。从简单到复杂，循序渐进，你一定能掌握 Vue 3 的精髓！

### 脚本内容

​	首先是脚本部分，下面的是vuejs框架的管理、构建和发布维护项目工程使用的脚本：

![image-20251013173755237](./assets/image-20251013173755237.png)

aliases.js - 模块别名配置脚本

- 用于配置 vitest 和 rollup 之间的**模块路径映射**

- 动态**扫描 packages 目录**并生成别名映射

build.js - Vue 核心包**生产环境构建脚本**

- 支持并行构建多个包

- 生成不同格式的输出文件

- 集成文件大小报告和枚举内联优化

dev.js - Vue 核心包**开发环境构建脚本**

- 使用 esbuild 进行**快速构建**

- 支持多种**输出格式和热重载**

inline-enums.js - 枚举**内联优化脚本**

- 优化 TypeScript 枚举使用

- 减少最终打包文件大小

- 将枚举转换为全局替换

pre-dev-sfc.js - SFC 开发前检查脚本

- 确保 SFC 相关包已构建完成

- 防止开发过程中出现模块找不到的问题

release.js - Vue 核心包**发布脚本**

- 自动化发布流程

- 版本管理、依赖更新、构建、测试、Git 操作和 npm 发布

setup-vitest.ts - Vitest 测试环境设置文件

- 配置测试环境

- 添加自定义匹配器和全局模拟

size-report.js - 文件**大小报告生成脚本**

- 比较当前版本与之前版本的文件大小

- 支持多种压缩格式对比

usage-size.js - Vue **用法大小分析脚本**

- 分析**不同用法场景下的包大小**

- 测试 tree-shaking 效果

utils.js - 构建工具通用函数库

- 包含构建脚本中的通用工具函数

- 包管理、目标匹配、命令执行等

verify-commit.js - Git **提交信息验证脚本**

- 验证提交信息**是否符合项目规范**

- 确保自动生成变更日志

verify-treeshaking.js - Tree-shaking 验证脚本

- 验证生产构建的 tree-shaking 效果

- 确保**不包含不应该存在的代码**

###  Vue 核心项目的私有包和工具

​	这些包不会发布到 npm，主要用于内部开发、测试和调试。

1. dts-test/ - TypeScript 类型定义测试

- 用于测试 Vue 核心包的 TypeScript 类型定义

- 确保类型定义在源码和构建后都能正常工作

- 包含各种 Vue API 的类型测试文件



1. dts-built-test/ - 构建后类型测试

- 专门用于**测试外部库使用 Vue 核心类型的情况**

- 验证构建后的类型定义是否正常工作

- 解决类似 Vuetify 等**第三方库的类型兼容性问题**



1. template-explorer/ - **模板编译器在线工具**

- Vue 模板编译器的**实时探索工具**

- 部署在 https://template-explorer.vuejs.org

- 用于展示和调试 Vue 模板编译过程。

- Vue 文件（`.vue`）本质上是一个自定义的文件格式，它将 `<template>`、`<script>` 和 `<style>` 块组合在一起。

  - **输入：** 原始 `.vue` 文件内容。
  - **解析器：** `@vue/compiler-sfc`
  - **过程：** SFC 解析器将文件内容**分割**成独立的块（称为 Block 或 Descriptor）。
  - **输出：** 一个描述符对象，包含：
    - `template` 块的内容 (HTML 字符串)。
    - `script` 块的内容 (JS/TS 代码)。
    - `style` 块的内容 (CSS 代码)。

  #### 优化和代码生成

  - **输入：** 模板 AST。
  - **编译器：** `@vue/compiler-dom`
  - **过程：**
    1. **静态提升 (Static Hoisting)：** 编译器会遍历 AST，识别出那些**内容和属性永不改变的节点**（即静态节点）。
    2. **代码生成：** 编译器根据优化后的 AST，生成 **JavaScript 渲染函数（Render Function）**。
       - 这个渲染函数在运行时会被执行，并调用 `createElementVNode` (创建虚拟 DOM 节点 VNode) 等 API 来构建虚拟 DOM 树。
  - **输出：** 包含 `render()` 函数的 JavaScript 代码字符串。

  

  ###  最终产物

  在构建工具（如 Vite 或 Webpack 的 Vue Loader）中，最终的 `.vue` 文件会被转换为一个 **标准的 JavaScript 模块**，这个模块通常 `export` 出组件的定义对象，其中包含了：

  1. 来自 `<script>` 块的组件选项或 `setup` 函数（**JS/TS 代码**）。
  2. 来自 `<template>` 块的 **编译后的 `render` 函数**（**JS 代码**）。

  

  ## 总结：为什么不是“静态文件”？

  

  - **不是静态 HTML/CSS：** 编译器没有生成浏览器可以直接加载的 `.html` 或 `.css` 文件。它生成的是 **JavaScript 代码**。
  - **是可执行的 JavaScript 渲染函数：** 这个生成的 JavaScript 代码（渲染函数）是**动态的**，它在组件挂载和更新时执行，根据组件的状态（响应式数据）创建或修补虚拟 DOM 树。



1. sfc-playground/ - SFC 在线游乐场

- Vue 单文件组件的在线编辑和预览工具

- 部署在 https://play.vuejs.org

- 支持 Vue 3 的 SFC 开发体验



1. vite-debug/ - Vite 调试工具

- 用于调试与 @vitejs/plugin-vue 相关的问题

- 模拟生产环境，使用构建后的 dist 文件而非源码

- 解决只能在 Vite 环境中复现的问题



### Vue 3 核心框架的所有核心包

#### 1. 运行时包 (Runtime Packages)

- vue/ - 主入口包，整合所有功能

- 包含**完整的 Vue 框架**

- 支持多种构建格式（ESM、CJS、Global）

- 包含**编译器、服务端渲染等子包**

- runtime-core/ - 运行时核心

- Vue 的**核心运行时逻辑**

- 组件系统、响应式系统、**调度器**等

> [!NOTE]
>
> 这里调度器和组件系统等可以着重看一下



- **平台无关的运行时实现**

- runtime-dom/ - DOM 运行时

- 浏览器环境的 DOM 操作

- 事件处理、属性绑定等

- 基于 runtime-core 的浏览器实现

- runtime-test/ - 测试运行时

- 用于测试环境的运行时

- 提供测试工具和模拟环境

#### 2. 编译器包 (Compiler Packages)

- compiler-core/ - 编译器核心

- 模板解析、AST 转换、代码生成

- 平台无关的编译逻辑

- 支持**各种指令和语法**

- compiler-dom/ - DOM 编译器

- 针对**浏览器环境的编译优化**

- DOM 特定的编译逻辑

- compiler-ssr/ - SSR 编译器

- 服务端渲染的编译支持

- 静态优化和序列化

- compiler-sfc/ - 单文件组件编译器

- .vue 文件的编译处理

- 模板、脚本、样式的分离和编译

#### 3. 响应式系统包

- reactivity/ - 响应式系统

- Vue 3 的**响应式核心**

- reactive、ref、computed、watch 等 API

- 可独立使用的响应式库

#### 4. 服务端渲染包

- server-renderer/ - 服务端渲染器

- 服务端渲染支持

- **流式渲染**、**序列化**等

#### 5. 兼容性包

- vue-compat/ - Vue 2 兼容层

- 提供 **Vue 2 到 Vue 3 的迁移支持**

- 兼容 Vue 2 的 API 和语法

#### 6. 共享工具包

- shared/ - **共享工具**

- **各包之间共享的工具函数**

- 类型定义、常量、工具函数等

## 通用工具函数库：shared

#### 1. index.ts - 入口文件

- 作用：了解整个共享工具库的结构

- 重点：理解各个模块的依赖关系

- 时间：15分钟

#### 2. makeMap.ts - 映射表工具

- 作用：Vue 3 中最基础的工具函数

- 重点：理解性能优化策略（Object.create(null)、tree-shaking）

- 时间：30分钟

```ts
/**
 * 创建映射表工具函数
 *
 * 该函数用于创建高效的字符串映射表，主要用于 Vue 3 中的各种配置检查，
 * 如 HTML 标签检查、SVG 标签检查、保留属性检查等。
 *
 * 性能优化：
 * - 使用 Object.create(null) 创建纯净对象，避免原型链查找
 * - 支持 tree-shaking，所有调用必须添加 PURE 注释
 * - 使用 in 操作符进行快速查找
 *
 * 使用示例：
 * ```typescript
 * const isHTMLTag = makeMap('div,span,p')
 * isHTMLTag('div') // true
 * isHTMLTag('custom') // false
 * ```
 */

/*@__NO_SIDE_EFFECTS__*/
export function makeMap(str: string): (key: string) => boolean {
  const map = Object.create(null)
  // 创建一个没有原型链的对象。避免了在使用 map[key] 或 key in map 时，JavaScript 引擎去查找 Object.prototype 上的继承属性（如 toString），从而加快查找速度。
  
  for (const key of str.split(',')) map[key] = 1
    //将列表转换为一个 Hash Map (哈希表)，查找时间复杂度接近 O(1)。比使用数组的 Array.prototype.includes() (O(N)) 快得多。赋值只需要一个简单的 1，比赋值 true 更简洁（尽管差异微小）。
    
  return val => val in map
}

/*@__NO_SIDE_EFFECTS__*/ 的代码实现：/*@__NO_SIDE_EFFECTS__*/ export function makeMap(...) 
	这是一个 Rollup/Webpack 等打包工具识别的魔术注释（Magic Comment）。它告诉打包工具：调用这个函数没有副作用（Side Effect），如果该函数的返回值（即 isHTMLTag 检查器）在最终代码中没有被使用，那么整个 makeMap 函数以及它内部的逻辑都可以安全地被 Tree-Shaking 移除。	

```

​	这个用途是用于快速查找，是否一个字符串存在于一个预定定的列表之中，并做了很多的优化。`makeMap` 接收一个用逗号分隔的字符串 (`str`)，并返回一个**函数**。这个返回的函数接受一个字符串 `key`，并判断这个 `key` 是否包含在原始列表中。核心用途：配置和检查      在 Vue 3 源码中，`makeMap` 被广泛用于创建各种配置检查器：

1. **HTML 标签检查：** `isHTMLTag = makeMap('div,span,a,p,...')`
2. **SVG 标签检查：** `isSVGTag = makeMap('svg,path,g,...')`
3. **保留属性检查：** `isReservedProp = makeMap('key,ref,v-model,...')`
4. **事件属性检查：** `isOn = makeMap('onUpdate,onClick,...')`

通过这种方式，Vue 可以在编译或运行时快速判断一个标签名或属性名是否有效、是否需要特殊处理。

#### 3. general.ts - 通用工具函数

- 作用：最常用的工具函数集合

- 重点：类型检查、字符串处理、对象操作

- 时间：1-2小时

### 第二阶段：核心算法 (2-3天)

#### 4. looseEqual.ts - 宽松比较

- 作用：响应式系统的核心算法

- 重点：递归比较、类型处理、性能优化

- 时间：1小时

#### 5. toDisplayString.ts - 显示字符串转换

- 作用：模板插值的核心功能

- 重点：各种数据类型的转换逻辑

- 时间：45分钟

#### 6. escapeHtml.ts - HTML 转义

- 作用：安全防护机制

- 重点：XSS 防护、字符转义算法

- 时间：30分钟

### 第三阶段：标志位系统 (1-2天)

#### 7. shapeFlags.ts - 形状标志位

- 作用：虚拟节点类型标识

- 重点：位运算、节点类型判断

- 时间：30分钟

#### 8. patchFlags.ts - 补丁标志位

- 作用：Vue 3 性能优化的核心

- 重点：位运算、优化策略、编译器提示

- 时间：1-2小时

#### 9. slotFlags.ts - 插槽标志位

- 作用：插槽优化策略

- 重点：插槽稳定性、更新策略

- 时间：30分钟

### 第四阶段：属性处理 (1-2天)

#### 10. normalizeProp.ts - 属性标准化

- 作用：处理 class 和 style 属性

- 重点：样式解析、属性合并算法

- 时间：1小时

#### 11. cssVars.ts - CSS 变量处理

- 作用：CSS-in-JS 功能支持

- 重点：CSS 变量标准化、类型检查

- 时间：30分钟

### 第五阶段：DOM 相关 (1-2天)

#### 12. domTagConfig.ts - DOM 标签配置

- 作用：HTML/SVG/MathML 标签识别

- 重点：标签分类、性能优化

- 时间：45分钟

#### 13. domAttrConfig.ts - DOM 属性配置

- 作用：属性处理和安全检查

- 重点：属性映射、安全检查

- 时间：1小时

### 第六阶段：高级工具 (1天)

#### 14. typeUtils.ts - TypeScript 类型工具

- 作用：类型系统增强

- 重点：高级类型、类型推导

- 时间：1小时

#### 15. globalsAllowList.ts - 全局变量白名单

- 作用：模板安全机制

- 重点：安全策略、全局变量控制

- 时间：30分钟

#### 16. codeframe.ts - 代码框架生成

- 作用：错误定位工具

- 重点：字符串处理、错误显示

- 时间：45分钟