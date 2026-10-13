---
date: 2026-12-10
tags:
  - javascript
---
本笔记摘选了一些面试的时候可能会经常被问到的js问题，可以尽量背熟悉。


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