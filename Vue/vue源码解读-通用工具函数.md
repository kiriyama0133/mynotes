尝试将给定的值宽松地转换为数字/**
 * 通用工具函数库
 *
 * 该模块包含了 Vue 3 中最常用的工具函数，提供了类型检查、字符串处理、
 * 对象操作、数组操作等基础功能。这些函数被 Vue 的各个核心模块广泛使用。
 */

import { makeMap } from './makeMap'

/** 空对象常量，开发环境使用 Object.freeze 冻结 */
export const EMPTY_OBJ: { readonly [key: string]: any } = __DEV__
  ? Object.freeze({})
  : {}
/** 空数组常量，开发环境使用 Object.freeze 冻结 */
export const EMPTY_ARR: readonly never[] = __DEV__ ? Object.freeze([]) : []

/** 空函数，用于默认值 */
export const NOOP = (): void => {}

/**
 * 总是返回 false 的函数
 */
 export const NO = () => false

export const isOn = (key: string): boolean =>
  key.charCodeAt(0) === 111 /* o */ &&
  key.charCodeAt(1) === 110 /* n */ &&
  // uppercase letter
  (key.charCodeAt(2) > 122 || key.charCodeAt(2) < 97)

export const isModelListener = (key: string): key is `onUpdate:${string}` =>
  key.startsWith('onUpdate:')

export const extend: typeof Object.assign = Object.assign

export const remove = <T>(arr: T[], el: T): void => {
  const i = arr.indexOf(el)
  if (i > -1) {
    arr.splice(i, 1)
  }
}

const hasOwnProperty = Object.prototype.hasOwnProperty
export const hasOwn = (
  val: object,
  key: string | symbol,
): key is keyof typeof val => hasOwnProperty.call(val, key)

export const isArray: typeof Array.isArray = Array.isArray
export const isMap = (val: unknown): val is Map<any, any> =>
  toTypeString(val) === '[object Map]'
export const isSet = (val: unknown): val is Set<any> =>
  toTypeString(val) === '[object Set]'

export const isDate = (val: unknown): val is Date =>
  toTypeString(val) === '[object Date]'
export const isRegExp = (val: unknown): val is RegExp =>
  toTypeString(val) === '[object RegExp]'
export const isFunction = (val: unknown): val is Function =>
  typeof val === 'function'
export const isString = (val: unknown): val is string => typeof val === 'string'
export const isSymbol = (val: unknown): val is symbol => typeof val === 'symbol'
export const isObject = (val: unknown): val is Record<any, any> =>
  val !== null && typeof val === 'object'

export const isPromise = <T = any>(val: unknown): val is Promise<T> => {
  return (
    (isObject(val) || isFunction(val)) &&
    isFunction((val as any).then) &&
    isFunction((val as any).catch)
  )
}

export const objectToString: typeof Object.prototype.toString =
  Object.prototype.toString
export const toTypeString = (value: unknown): string =>
  objectToString.call(value)

export const toRawType = (value: unknown): string => {
  // extract "RawType" from strings like "[object RawType]"
  return toTypeString(value).slice(8, -1)
}

export const isPlainObject = (val: unknown): val is object =>
  toTypeString(val) === '[object Object]'

export const isIntegerKey = (key: unknown): boolean =>
  isString(key) &&
  key !== 'NaN' &&
  key[0] !== '-' &&
  '' + parseInt(key, 10) === key

export const isReservedProp: (key: string) => boolean = /*@__PURE__*/ makeMap(
  // the leading comma is intentional so empty string "" is also included
  ',key,ref,ref_for,ref_key,' +
    'onVnodeBeforeMount,onVnodeMounted,' +
    'onVnodeBeforeUpdate,onVnodeUpdated,' +
    'onVnodeBeforeUnmount,onVnodeUnmounted',
)

export const isBuiltInDirective: (key: string) => boolean =
  /*@__PURE__*/ makeMap(
    'bind,cloak,else-if,else,for,html,if,model,on,once,pre,show,slot,text,memo',
  )

const cacheStringFunction = <T extends (str: string) => string>(fn: T): T => {
  const cache: Record<string, string> = Object.create(null)
  return ((str: string) => {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }) as T
}

const camelizeRE = /-\w/g
/**
 * @private
 */
 export const camelize: (str: string) => string = cacheStringFunction(
    (str: string): string => {
    return str.replace(camelizeRE, c => c.slice(1).toUpperCase())
    },
 )

const hyphenateRE = /\B([A-Z])/g
/**
 * @private
 */
 export const hyphenate: (str: string) => string = cacheStringFunction(
    (str: string) => str.replace(hyphenateRE, '-$1').toLowerCase(),
 )

/**
 * @private
 */
  export const capitalize: <T extends string>(str: T) => Capitalize<T> =
    cacheStringFunction(<T extends string>(str: T) => {
    return (str.charAt(0).toUpperCase() + str.slice(1)) as Capitalize<T>
    })

/**
 * @private
 */
 export const toHandlerKey: <T extends string>(
    str: T,
 ) => T extends '' ? '' : `on${Capitalize<T>}` = cacheStringFunction(
    <T extends string>(str: T) => {
    const s = str ? `on${capitalize(str)}` : ``
    return s as T extends '' ? '' : `on${Capitalize<T>}`
    },
 )

// compare whether a value has changed, accounting for NaN.
export const hasChanged = (value: any, oldValue: any): boolean =>
  !Object.is(value, oldValue)

export const invokeArrayFns = (fns: Function[], ...arg: any[]): void => {
  for (let i = 0; i < fns.length; i++) {
    fns[i](...arg)
  }
}

export const def = (
  obj: object,
  key: string | symbol,
  value: any,
  writable = false,
): void => {
  Object.defineProperty(obj, key, {
    configurable: true,
    enumerable: false,
    writable,
    value,
  })
}

/**
 * "123-foo" will be parsed to 123
 * This is used for the .number modifier in v-model
 */
 export const looseToNumber = (val: any): any => {
    const n = parseFloat(val)
    return isNaN(n) ? val : n
 }

/**
 * Only concerns number-like strings
 * "123-foo" will be returned as-is
 */
 export const toNumber = (val: any): any => {
    const n = isString(val) ? Number(val) : NaN
    return isNaN(n) ? val : n
 }

// for typeof global checks without @types/node
declare var global: {}

let _globalThis: any
export const getGlobalThis = (): any => {
  return (
    _globalThis ||
    (_globalThis =
      typeof globalThis !== 'undefined'
        ? globalThis
        : typeof self !== 'undefined'
          ? self
          : typeof window !== 'undefined'
            ? window
            : typeof global !== 'undefined'
              ? global
              : {})
  )
}

const identRE = /^[_$a-zA-Z\xA0-\uFFFF][_$a-zA-Z0-9\xA0-\uFFFF]*$/

export function genPropsAccessExp(name: string): string {
  return identRE.test(name)
    ? `__props.${name}`
    : `__props[${JSON.stringify(name)}]`
}

export function genCacheKey(source: string, options: any): string {
  return (
    source +
    JSON.stringify(options, (_, val) =>
      typeof val === 'function' ? val.toString() : val,
    )
  )
}

它集中了 Vue 框架在所有模块（如响应式、编译器、运行时）中都需要使用的**底层、高性能的基础操作**，涵盖了类型检查、字符串格式化、数组操作和性能优化等多个方面。

## 1. 核心常量与空操作

​	这些是用于提高代码可读性、**避免重复创建对象和保证类型安全的常量**。

| 函数/常量       | 作用             | 目的/原理                                                    |
| --------------- | ---------------- | ------------------------------------------------------------ |
| **`EMPTY_OBJ`** | 空对象常量       | 避免重复创建空对象。在 `__DEV__`（开发环境）下使用 `Object.freeze()` **冻结**，确保对象的不可变性，避免意外修改。 |
| **`EMPTY_ARR`** | 空数组常量       | 与 `EMPTY_OBJ` 相同，提供一个**冻结的空数组**。              |
| **`NOOP`**      | 空函数           | “No Operation”的缩写，用于函数参数的默认值，表示什么都不做。 |
| **`NO`**        | 总是返回 `false` | 用于作为钩子函数的默认值，例如，如果某个选项未提供，则默认为不启用。 |

这里讲一下冻结的空数组的含义：`Object.freeze()` 是 JavaScript 中用于实现**不可变性（Immutability）**的方法。当一个对象（包括数组）被冻结后，以下操作将**被禁止**，并且在严格模式下会抛出错误：

- **修改现有属性的值。**
- **添加新属性。**
- **删除现有属性。**
- **修改数组元素**（因为数组元素本质上是属性）。
- **调用数组变异方法**（如 `push`、`pop`、`splice` 等）。

因此，`Object.freeze([])` 得到的是一个**永远为空、且不能被任何人修改**的数组。在 Vue 源码中，使用冻结的空数组主要有以下两个目的：

A. 运行时安全 (Safety)

当 Vue 组件或**内部函数需要返回一个空数组作为默认值时**，它们会返回 `EMPTY_ARR`。

- **如果没有冻结：** 如果一个函数返回了 `[]`，而调用方不小心对它执行了 `push()` 操作，那么这个**共享**的空数组就被污染了，可能会导致其他依赖这个空数组的地方出现意外的 bug。
- **使用 `EMPTY_ARR`：** 任何试图修改 `EMPTY_ARR` 的操作都会失败或抛出错误，从而确保了**代码的安全性和可预测性**。

B. 性能优化 (Performance)

- **避免重复创建：** 在 Vue 的生命周期中，可能会有成千上万次需要返回一个空数组的场景（例如，组件没有子节点、某个选项没有提供）。通过共享一个全局的 `EMPTY_ARR` 常量，**避免了每次都创建新的数组对象 (`new Array()`) 的内存分配和垃圾回收开销**。



### 3. 代码中的条件判断

## 2. 类型检查工具（Type Checking）

​	这组函数**用于在运行时安全、准确地判断一个值的类型**。Vue 倾向于使用 `Object.prototype.toString` 进行精确类型检查。

| 函数                                                        | 作用                             | 原理                                                         |
| ----------------------------------------------------------- | -------------------------------- | ------------------------------------------------------------ |
| **`toTypeString`**                                          | 获取值的原始类型字符串。         | 使用 `Object.prototype.toString.call(value)`，返回如 `"[object Array]"`。 |
| **`toRawType`**                                             | 提取原始类型名。                 | 从 `[object RawType]` 中提取 `RawType`。                     |
| **`isMap`, `isSet`, `isDate`, `isRegExp`, `isPlainObject`** | 精确判断特定对象类型。           | 依赖 `toTypeString` 的结果进行判断，比 `typeof` 更可靠。     |
| **`isObject`**                                              | 检查是否为非 `null` 的对象。     | `val !== null && typeof val === 'object'`。                  |
| **`isPromise`**                                             | 检查是否为 Promise。             | 检查值是否为对象或函数，并具有 `then` 和 `catch` 方法。      |
| **`isIntegerKey`**                                          | 检查字符串是否是有效的整数索引。 | 用于判断对象属性是否可能是数组的索引，例如在响应式系统中的优化。 |

```js
export const extend: typeof Object.assign = Object.assign
```

上面是别名与模块化封装

- **创建别名：** 将标准的 JavaScript 内建函数 `Object.assign` 赋值给一个新常量 `extend`。
- **模块化封装：** 将这个 `extend` 别名通过 `export` 导出，使其成为 Vue `shared` 工具库的一部分。这使得 Vue 的其他模块（如编译器、运行时）可以统一地从 `shared` 模块中导入 `extend` 来执行对象合并操作，而不是直接访问全局的 `Object.assign`。
  - *示例：* 在其他模块中会写成 `import { extend } from '@vue/shared'`，然后使用 `extend(target, source)`。

这行代码包含的 TypeScript 语法确保了类型安全：

- **`export const extend: typeof Object.assign`**
  - `typeof Object.assign`：这行告诉 TypeScript 编译器，`extend` 这个常量必须具有与内置的 `Object.assign` **完全相同的函数签名**（即参数类型、返回值类型）。
  - 这保证了任何使用 `extend` 的地方都能**得到正确的类型检查和智能提示**，增强了代码的健壮性。

`Object.assign` 是一个标准的 JavaScript 内置函数，它的作用是**将一个或多个源对象的所有可枚举（Enumerable）的自有（Own）属性复制到目标对象（Target Object）上**，并返回目标对象。

### `Object.assign` 的函数签名（Signature）

在 TypeScript 中，它的签名可以大致概括为：

```ts
Object.assign(target: object, ...sources: any[]): any
```

| 参数名           | 描述                                                         | 类型约束                                                     |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **`target`**     | **目标对象。** 属性会被复制到这个对象上。这个对象会被改变（被**修改**）。 | 必须是一个对象 (`object`)。如果传入非对象值，它会被强制转换为对象（例如 `Object(null)` 或 `Object(undefined)` 会抛出错误，但基本类型会被包装）。 |
| **`...sources`** | **源对象。** 属性将**从这些对象复制到目标对象**。可以有零个或多个源对象。 | 可以是任何类型。如果是 `null` 或 `undefined`，则跳过；如果是基本类型，会被包装成对象，但只有**可枚举**的属性会被复制（通常基本类型没有可枚举的自有属性）。 |

运行时修改对象属性（在 JavaScript 中被称为 **Duck Typing** 或 **动态性**）是 JavaScript 语言的内置能力。可以随时对任何对象添加、删除或修改属性，而不需要事先在类的定义中声明。而mixin也是一种实现方式。**Mixin** 是一种设计模式，目的是为了**复用**代码或功能。它的实现方式就是将一个或多个**源对象（或函数）的属性和方法**复制到**目标对象（或类）**上，从而“混入”这些功能。**Mixin 是实现运行时属性修改的** **一种结构化、有目的的方法**。

### Mixin 的实现方式（通常使用 `Object.assign`）

​	Vue 源码中的 `extend` (即 `Object.assign`) 函数，就是实现 Mixin 的核心工具：

```js
const LoggerMixin = {
  log(message) {
    console.log(`[LOG]: ${message}`);
  }
};

const UserMethods = {
  // 注意，这里的 extend 就是 Object.assign
  start: function() {
    this.status = 'running';
  }
};

const componentOptions = {}; // 目标对象

// ⭐️ 使用 Object.assign / extend 进行 Mixin
Object.assign(componentOptions, LoggerMixin, UserMethods);

// componentOptions 现在拥有了 log 和 start 方法
componentOptions.log("Mixin successful!");
```

## 安全地从一个数组中删除指定的元素

```ts
export const remove = <T>(arr: T[], el: T): void => {
  const i = arr.indexOf(el)
  if (i > -1) {
    arr.splice(i, 1)
  }
}
```

### 步骤 A: 查找元素的位置 (`indexOf`)

```ts
const i = arr.indexOf(el)
```

- `arr.indexOf(el)` 遍历数组 `arr`，查找元素 `el` **第一次出现**的索引位置 `i`。
- 如果找到，`i` 是一个大于或等于 `0` 的整数。
- 如果找不到，`i` 的值是 `-1`。

### 步骤 B: 执行删除 (`splice`)

```ts
if (i > -1) {
    arr.splice(i, 1)
}
```

- **条件检查：** 只有当元素被找到 (`i > -1`) 时，才执行删除操作。这确保了如果元素不存在，程序不会出错。
- **`arr.splice(i, 1)`：** 这是数组删除的核心。
  - 第一个参数 `i`：指定开始修改数组的索引位置。
  - 第二个参数 `1`：指定要移除的元素数量（只移除一个，即找到的 `el`）。

- **高效性：** 尽管 `indexOf` 是 O(N)（线性时间复杂度）的操作，**但它比手动循环查找要简洁得多**，并且对于 Vue 源码中处理的通常较小的依赖数组（例如响应式系统的依赖集合），性能开销可以接受。
- **元素相等性：** 查找依赖于 JavaScript 的严格相等性比较 (`===`)，这意味着对于对象元素，只有当数组中包含**完全相同的对象引用**时，才会将其移除。
- **直接修改：** 该函数是**变异 (Mutating)** 操作，它会修改传入的原始数组 `arr`。

## **安全且高效地检查对象自身是否包含某个属性**的关键函数。

```ts
const hasOwnProperty = Object.prototype.hasOwnProperty
export const hasOwn = (
  val: object,
  key: string | symbol,
): key is keyof typeof val => hasOwnProperty.call(val, key)

```

在 JavaScript 中，最直接的**检查对象自身属性的方法**是 `obj.hasOwnProperty(key)`。然而，这种方法存在一个潜在的风险，尤其是在处理来自外部或不可信来源的对象时：如果一个对象 `val` 自己定义了一个属性名也叫 `hasOwnProperty`，它就会**覆盖（Shadow）**原型链上的原版方法，导致调用出错或返回错误的结果。

```ts
const maliciousObj = {
  prop: 1,
  // 覆盖了原生的方法！
  hasOwnProperty: () => true 
};

maliciousObj.hasOwnProperty('prop'); // 总是返回 true，结果不准确
```

`hasOwn` 函数通过将 **`hasOwnProperty` 方法“借用”**（Borrowing）或 **“借用调用”**（Call Borrowing）到目标对象 `val` 上来解决这个问题：

```ts
const hasOwnProperty = Object.prototype.hasOwnProperty // 缓存原生方法
export const hasOwn = (val, key) => hasOwnProperty.call(val, key)
```

## **类型检测（Type Checking）**

```ts

export const isArray: typeof Array.isArray = Array.isArray
export const isMap = (val: unknown): val is Map<any, any> =>
  toTypeString(val) === '[object Map]'
export const isSet = (val: unknown): val is Set<any> =>
  toTypeString(val) === '[object Set]'

export const isDate = (val: unknown): val is Date =>
  toTypeString(val) === '[object Date]'
export const isRegExp = (val: unknown): val is RegExp =>
  toTypeString(val) === '[object RegExp]'
export const isFunction = (val: unknown): val is Function =>
  typeof val === 'function'
export const isString = (val: unknown): val is string => typeof val === 'string'
export const isSymbol = (val: unknown): val is symbol => typeof val === 'symbol'
export const isObject = (val: unknown): val is Record<any, any> =>
  val !== null && typeof val === 'object'

```

`toTypeString` 依赖于 `Object.prototype.toString.call(val)`。

- **核心优势：** 这是 JavaScript 规范中**最可靠、最稳定的**获取对象内部 `[[Class]]` 属性（或称内部标签）的方法。
- **返回值：** 对于**几乎所有内置类型**，它都会返回一个形如 `[object TypeName]` 的**字符串**。
  - Map 实例 → `[object Map]`
  - Date 实例 → `[object Date]`
  - 数组 → `[object Array]`
  - 普通对象 → `[object Object]`

​	因此，`toTypeString(val) === '[object Map]'` 是确保 `val` 是一个真正的 `Map` 实例的**最可靠、最跨环境**的方式。`val is Map<any, any>` 的作用（类型安全）这部分是纯粹的 **TypeScript 语法**，被称为 **类型谓词（Type Predicate）**。

**`val is Map<any, any>`**这是一个特殊的返回类型。它告诉 TypeScript 编译器：“如果这个函数返回 `true`，那么你可以保证传入的参数 `val` **就是**一个 `Map<any, any>` 类型的值。”**使用场景**当我们在代码中调用 `if (isMap(someValue))` 时，在 `if` 块内部，`someValue` 的类型将**自动被收窄（Narrowing）**为 `Map<any, any>`。

​	你觉得 `val` 设置为 `unknown` 并且使用 `toTypeString` 是多此一举，是因为你可能主要从运行时 JavaScript 的角度来看。但是，在 TypeScript 层面，这种写法是**必须的**，也是**最有意义的**。

------



## 为什么 `val: unknown` 并非多余？

### 1. `unknown` 是最安全的类型

在 TypeScript 中：

- **`unknown`** 类型代表**任何值**。
- 它是**顶级类型**，意味着你可以将任何值赋给它。
- 它的安全性在于：**如果你不先进行类型检查（即类型收窄），你就不能对 `unknown` 类型的值执行任何操作**（除了相等性检查）。

当我们在 Vue 源码中定义一个通用工具函数时，我们不知道用户会传入什么类型的值，所以使用 **`val: unknown`** 是最正确的选择，因为它：

> **“我不知道你传的是什么，但我要对你进行检查。”**

### 2. 运行时 JavaScript 代码（`toTypeString`）的必要性

因为 `val` 是 `unknown`，我们必须使用 **运行时** 的 JavaScript 代码来判断它的实际类型。

toTypeString(val)===′[object Map]′

- 这行代码是 **JavaScript 运行时** 的逻辑，它判断 `val` 是否真的是一个 Map 实例。
- 这个判断的结果（`true` 或 `false`）决定了 **TypeScript 编译器** 如何处理后续的代码。

------



### 3. 为什么 `val is Map<any, any>` 才是核心价值？

这里的核心价值在于 **类型谓词** `val is Map<any, any>`。它负责将运行时的检查结果反馈给编译时的 TypeScript 系统。

| 步骤     | 环境              | 解释                                                         |
| -------- | ----------------- | ------------------------------------------------------------ |
| **输入** | TypeScript 编译时 | `val` 的类型是 `unknown`。编译器不允许你访问 `val.size`。    |
| **执行** | JavaScript 运行时 | `toTypeString` 检查 `val` 的实际值。                         |
| **返回** | TypeScript 编译时 | 如果函数返回 `true`，编译器相信了类型谓词，并**收窄** `val` 的类型到 `Map`。 |

## **鸭子类型（Duck Typing）**检查

它的实现策略是进行**鸭子类型（Duck Typing）**检查，这在 JavaScript 库中是一种常见且灵活的做法。

​	`isPromise` 函数详解:  该函数没有使用 `val instanceof Promise`，而是执行了一系列**运行时检查**，以确定 `val` 是否具备 **Promise 的行为特征**。

| 部分         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| **签名**     | `<T = any>(val: unknown): val is Promise<T>`                 |
| **类型谓词** | `val is Promise<T>` 告诉 TypeScript 编译器，如果此函数返回 `true`，则 `val` 的类型被收窄为 `Promise<T>`。 |
| **输入类型** | `val: unknown` 表明它接受任何类型的值，并对其进行动态检查。  |

鸭子类型（Duck Typing）检查逻辑，该函数返回 `true` 的条件是所有子条件都成立（使用 `&&` 连接）：

#### A. 检查基本类型

​			(isObject(val) || isFunction(val))

- **目的：** 确保 `val` 是一个复合类型，因为 Promise（或 Thenable）必须是一个对象或函数。
- **排除项：** 排除原始类型（如 `string`, `number`, `boolean`, `null`, `undefined`, `symbol`），这些类型不可能是 Promise。

#### B. 检查 `then` 方法

​			isFuction((val as any).then)

- **目的：** 检查 `val` 是否具有一个名为 **`then`** 的属性，并且该属性是一个**函数**。
- 这是识别 Thenable 对象的**关键**。根据 **Promise/A+ 规范**，任何具有 `then` 方法的对象都被视为 **Promise-like 对象**。

#### C. 检查 `catch` 方法 (辅助性检查)

​			isFunction((val as any).catch)

- **目的：** 检查 `val` 是否具有一个名为 **`catch`** 的属性，并且该属性是一个**函数**。
- 虽然理论上 Thenable 只需要 `then` 方法，但实际应用中的 Promise 几乎总是具有 `catch` 方法。 Vue 在此加入 `catch` 检查，是为了提高判断的准确性，确保它能被当作一个完整的 Promise 实例来处理。

**鸭子类型 (Duck Typing)** 的哲学是：“如果它走起来像鸭子，叫起来像鸭子，那么它就是一只鸭子。”

在这种情况下：

- **优点：** 这种方法**不依赖于全局的 `Promise` 构造函数**（如 `instanceof Promise`）。它能识别出**所有**遵循 Promise 约定的对象，包括自定义的 Promise 实现（如一些库中的 Thenable 对象），从而提高了框架的兼容性和灵活性。
- **`as any` 的使用：** 由于 `val` 是 `unknown`，TypeScript 不允许直接访问其属性。`val as any` 暂时绕过了 TypeScript 的静态检查，以允许在运行时执行属性检查 (`.then` 和 `.catch`)。因为这是在类型检查工具函数内部，这种逃逸是合理的。

## **最精确、最可靠的运行时类型检测**的基础工具函数

```ts
export const objectToString: typeof Object.prototype.toString =
  Object.prototype.toString
export const toTypeString = (value: unknown): string =>
  objectToString.call(value)

```

`objectToString`：缓存原生方法

```ts
export const objectToString: typeof Object.prototype.toString =
  Object.prototype.toString
```

**目的：** 这行代码是为了**缓存（Caching）和安全**地获取原生的 `toString` 方法。

- **缓存：** 将 `Object.prototype.toString` 赋值给一个常量 `objectToString` 并导出。在后续代码中，可以直接使用这个常量，而不是每次都访问 `Object.prototype`。
- **安全：** 它避免了直接在目标对象上调用 `value.toString()`。如果 `value` 上有自定义的 `toString` 方法，直接调用会执行自定义方法，导致类型检测失败。通过**借用** `Object.prototype.toString`，我们可以**保证执行的是原生方法**。

`toTypeString`：实现精确类型检查

```ts
export const toTypeString = (value: unknown): string =>
  objectToString.call(value)
```

**目的：** 获取值的 **内部 `[[Class]]` 标签**（Internal Tag），返回一个形如 `"[object TypeName]"` 的字符串。

**原理（借用调用）：**

- 使用缓存的 `objectToString`。
- 使用 `.call(value)` **强制**将 `toString` 方法的 `this` 上下文设置为要检查的值 `value`。
- 当 `Object.prototype.toString` 在任何对象上被调用时，**它会读取该对象的内部属性，并返回一个特定格式的字符串。**

这是 JavaScript 规范中**最可靠**的类型检测机制。例如：

| 值 (`value`)        | `objectToString.call(value)` 的返回值 | 实际类型  |
| ------------------- | ------------------------------------- | --------- |
| `[1, 2]` (数组)     | `"[object Array]"`                    | Array     |
| `new Map()`         | `"[object Map]"`                      | Map       |
| `new Date()`        | `"[object Date]"`                     | Date      |
| `{ a: 1 }` (纯对象) | `"[object Object]"`                   | Object    |
| `null`              | `"[object Null]"`                     | Null      |
| `undefined`         | `"[object Undefined]"`                | Undefined |

## 获取值 **原始类型名称（Raw Type Name）** 的工具函数

```ts
export const toRawType = (value: unknown): string => {
  // extract "RawType" from strings like "[object RawType]"
  return toTypeString(value).slice(8, -1)
}

```

`toRawType` 的目的就是从 `toTypeString` 返回的格式化字符串中，提取出纯粹的类型名称。

### 工作原理



| 步骤                    | 代码                           | 解释                                                         |
| ----------------------- | ------------------------------ | ------------------------------------------------------------ |
| **A. 获取格式化字符串** | 隐式调用 `toTypeString(value)` | 获取 `Object.prototype.toString.call(value)` 的结果，格式为 `"[object RawType]"`。 |
| **B. 截取字符串**       | `.slice(8, -1)`                | 使用 JavaScript 的 `slice` 方法进行高效截取。                |

### 截取的位置 (`.slice(8, -1)`)

这个数字不是随意取的，它基于 `"[object RawType]"` 这个固定格式的字符串：

- **`[object `**：这部分字符串固定占用 **8 个字符**。
  - `[` 占 1 位 (索引 0)
  - `o` 占 1 位 (索引 1)
  - `b` 占 1 位 (索引 2)
  - `j` 占 1 位 (索引 3)
  - `e` 占 1 位 (索引 4)
  - `c` 占 1 位 (索引 5)
  - `t` 占 1 位 (索引 6)
  - 空格 占 1 位 (索引 7)
- **`8` (开始索引)：** 从索引 8 开始截取，正好跳过 `[object `，开始于 **`R`**awType。
- **`-1` (结束索引)：** 截取到倒数第一个字符之前，正好跳过末尾的 **`]`**。

## 准确检查一个值是否为普通的 JavaScript 对象

​	它排除了像数组、Date、Map 等所有其他内置对象类型，只接受那些直接或间接继承自 `Object.prototype` 且内部标签为 `[object Object]` 的对象。

```ts
export const isPlainObject = (val: unknown): val is object =>
  toTypeString(val) === '[object Object]'
```

这段代码是确保一个对象是“普通对象”的最可靠方式，因为 **`typeof` 运算符在这个场景中是不可靠的**。

| 检查方式                  | 结果   | 问题                                                         |
| ------------------------- | ------ | ------------------------------------------------------------ |
| `typeof val === 'object'` | `true` | 对于 **`null`**、**数组**、**Date**、**RegExp**、**Map** 等都会返回 `true`，太宽泛。 |
| `val instanceof Object`   | `true` | 对于数组、Date 等所有继承自 `Object` 的对象都会返回 `true`。 |

### 运行时可靠性：`toTypeString`

toTypeString(val)===′[object Object]′

- **依赖：** 它依赖于 `toTypeString`（即借用 `Object.prototype.toString.call(val)`）。
- **排除所有内置类型：** 只有通过 `{}` 字面量创建的，或通过 `new Object()` 创建的对象，以及那些没有被指定特定 `[[Class]]` 内部标签的对象，才会返回 `[object Object]`。
  - **例：** `new Date()` 返回 `[object Date]`，`[]` 返回 `[object Array]`。因此，这种方法可以准确地将普通对象与其他内置类型区分开。

## 判断一个值是否可以作为有效的非负整数键

```ts
export const isIntegerKey = (key: unknown): boolean =>
  isString(key) &&
  key !== 'NaN' &&
  key[0] !== '-' &&
  '' + parseInt(key, 10) === key

```

这个函数接收一个值 `key`，并执行了一系列**严格的检查**来确定它是否是一个合格的整数键。



### 1. 检查是否为字符串 (`isString(key)`)

isString(key)

- **目的：** 强制要求 `key` 必须是**字符串**类型。在 JavaScript 中，对象键和数组索引最终都会被转换为字符串进行访问。
- **排除：** 排除了所有非字符串类型（如 `number`, `symbol`, `object` 等）。



### 2. 排除 `'NaN'` 字符串

key!==′NaN′

- **目的：** 排除字符串 `'NaN'`。虽然 `parseInt('NaN', 10)` 会返回 `NaN`，但这个显式检查是为了确保排除这个特殊值。



### 3. 排除负号开头

key[0]!==′−′;

- **目的：** 排除所有以负号开头的字符串。数组索引必须是非负整数。
- **排除：** 例如，排除了字符串 `'-1'`、`'-100'`。



### 4. 核心检查：是否为纯粹的整数字符串



” + parseInt(key,10)===key

这是最关键的一步，用于检查字符串是否**完全**由整数数字组成，且没有多余的字符。

- **`parseInt(key, 10)`：** 尝试将字符串 `key` 解析为十进制（基数 10）整数。
  - **如果 `key` 是 `'123'`：** 解析结果是数字 `123`。
  - **如果 `key` 是 `'123a'`：** 解析结果是数字 `123`（`parseInt` 会停止于非数字字符）。
  - **如果 `key` 是 `'01'`：** 解析结果是数字 `1`。
- **`'' + ...`：** 将解析后的数字**重新转换回字符串**。
  - `123` → `'123'`
  - `123` → `'123'`
  - `1` → `'1'`
- **`=== key`：** 将这个**重新转换的字符串**与**原始 `key` 字符串**进行严格比较。

| 原始 `key`   | `parseInt` 结果 | 重新转换字符串 | 比较结果                   | **判断**       |
| ------------ | --------------- | -------------- | -------------------------- | -------------- |
| **`'123'`**  | `123`           | `'123'`        | `'123' === '123'` (true)   | ✅              |
| **`'123a'`** | `123`           | `'123'`        | `'123' === '123a'` (false) | ❌              |
| **`'01'`**   | `1`             | `'1'`          | `'1' === '01'` (false)     | ❌ (排除前导零) |
| **`'1.2'`**  | `1`             | `'1'`          | `'1' === '1.2'` (false)    | ❌              |

## 高效地检查一个属性名（`key`）是否是 **Vue 框架内部保留使用的属性**

```ts
export const isReservedProp: (key: string) => boolean = /*@__PURE__*/ makeMap(
  // the leading comma is intentional so empty string "" is also included
  ',key,ref,ref_for,ref_key,' +
    'onVnodeBeforeMount,onVnodeMounted,' +
    'onVnodeBeforeUpdate,onVnodeUpdated,' +
    'onVnodeBeforeUnmount,onVnodeUnmounted',
)
```

`isReservedProp` 检查的属性是 Vue 内部使用的特殊键，它们**不应该**被视为普通的组件 `prop` 或传递给子组件。

这个列表主要包含两类属性：

| 类型                   | 属性示例                                   | 作用                                                         |
| ---------------------- | ------------------------------------------ | ------------------------------------------------------------ |
| **内部控制键**         | `key`, `ref`, `ref_for`, `ref_key`         | 用于**虚拟 DOM（VNode）的差异对比和模板引用的核心属性。**    |
| **VNode 生命周期钩子** | `onVnodeBeforeMount`, `onVnodeMounted`, 等 | 用于**监听 VNode 自身生命周期的特殊钩子**，不能作为普通 `prop` 传递。 |

如果 Vue 在模板中发现一个**属性是保留属性**，它会对其进行特殊处理（例如，将 `key` 属性附加到 VNode 上，而不是作为组件 `prop` 传递下去）。

## 性能与代码结构分析

### A. 使用 `makeMap` (高性能查找)

export const isReservedProp: (key: string) => boolean=/*@__PURE__*/makeMap(…)

- **机制：** `makeMap` 将长字符串列表转换为一个内部的 **哈希表**（`Object.create(null)` 字典）。
- **效果：** 检查一个属性是否保留，从 O(N)（遍历数组）优化为平均 **O(1)（常数时间）** 的哈希表查找，保证了运行时的极高效率。



### B. `/*@__PURE__*/` (构建时优化)

- **魔术注释：** 这是告诉 Tree-Shaking 工具（如 Rollup/Webpack）这个函数调用 **没有副作用**。
- **效果：** 如果 Vue 应用程序没有用到 `isReservedProp` 检查的某些功能（尽管不太可能），或者打包器能够确定它的结果是常量，就可以安全地进行优化或移除。



### C. 列表内容与特殊处理

’,key,ref,ref_for,ref_key,...’

- **前导逗号（Leading Comma）：** 源码注释中提到了 `// the leading comma is intentional so empty string "" is also included`。

## 检查一个属性名（`key`）是否是 **Vue 框架内置的指令**

```ts
export const isBuiltInDirective: (key: string) => boolean =
  /*@__PURE__*/ makeMap(
    'bind,cloak,else-if,else,for,html,if,model,on,once,pre,show,slot,text,memo',
  )

```

`isBuiltInDirective` 用于在 Vue 模板编译器或运行时识别那些不需要作为普通组件属性或 HTML 属性处理的特殊指令。

当模板编译器解析到一个属性时，它会首先检查这个属性是否是内置指令：

- **如果是内置指令**：它会被编译成特定的渲染函数逻辑（例如 `v-if` 编译成三元表达式，`v-model` 编译成属性设置和事件监听）。
- **如果不是内置指令**：它可能是一个自定义指令（如 `v-focus`）或一个普通的组件 `prop`/HTML 属性。

### 2. 列表内容

列表中包含了 Vue 提供的所有核心指令（在模板中通常以 `v-` 开头，但在内部存储时，`v-` 前缀被移除）：

- **结构化指令 (Structural):** `if`, `else-if`, `else`, `for`
- **内容绑定指令 (Content):** `html`, `text`
- **属性绑定指令 (Attribute):** `bind` (对应 `:prop`), `on` (对应 `@event`), `model` (对应 `v-model`)
- **修改指令 (Modifier):** `once`, `memo`
- **控制流指令 (Control):** `show`, `cloak`, `pre`, `slot`

### 3. 性能优化 (和 `isReservedProp` 相同)

export const isBuiltInDirective: (key: string) => boolean=/*@__PURE__*/makeMap(…)

- **`makeMap`：** 将所有内置指令名转换为一个**哈希表**（`Object.create(null)` 字典），确保对任何键的查找都是平均 **O(1)** 的常数时间，速度极快。
- **`/\*@__PURE__\*/`：** 告诉构建工具，这个函数调用**没有副作用**。这使得 Tree-Shaking 能够在最终的生产包中安全地移除任何未使用的相关代码。

简而言之，这段代码是为了让 Vue 能够以最高的效率，在数千次的模板解析中，快速准确地判断一个指令是否是它自己内置的核心功能。

## **记忆化（Memoization）** 模式

```ts
const cacheStringFunction = <T extends (str: string) => string>(fn: T): T => {
  const cache: Record<string, string> = Object.create(null)
  return ((str: string) => {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }) as T
}
```

该函数接收一个字符串处理函数 `fn`，并返回一个新的、**带有缓存功能的函数**。

**工作原理：**

1. **第一次调用：** 当新的字符串 `str` 传入时，它会在内部的 `cache` 中查找。由于第一次找不到（`hit` 为 `undefined`），它会执行原始函数 `fn(str)` 来计算结果。
2. **存储结果：** 结果会被赋值给 `cache[str]`，同时作为整个表达式的值被返回 (`cache[str] = fn(str)` 的结果就是赋给 `cache[str]` 的值)。
3. **后续调用：** 再次传入相同的 `str` 时，`cache[str]` 会立即命中（`hit` 有值），直接返回缓存的结果，**跳过**原始函数 `fn` 的重复计算。

| 代码段                                              | 机制               | 作用和优化                                                   |
| --------------------------------------------------- | ------------------ | ------------------------------------------------------------ |
| **`<T extends (str: string) => string>(fn: T): T`** | **泛型与类型安全** | 确保了传入的 `fn` 和返回的函数 `T` 具有完全相同的签名（输入 `string`，输出 `string`），保证了类型安全。 |
| **`const cache = Object.create(null)`**             | **高性能哈希表**   | 使用 `Object.create(null)` 创建一个**没有原型链的纯净对象**作为缓存容器。这使得查找速度最快（平均 O(1)），避免了原型链查找的开销。 |
| **`const hit = cache[str]`**                        | **快速查找**       | 直接在哈希表中查找结果。                                     |
| **`return hit                                       |                    | (cache[str] = fn(str))`**                                    |
| **闭包**                                            | **状态保持**       | 内部的匿名函数通过 **闭包** 机制，保持对外部 `cache` 变量的引用，确保了缓存状态的持久性。 |

Vue 源码中的 `cacheStringFunction` 所实现的作用，与 Python 标准库 `functools` 中的 **`@lru_cache` 装饰器** 所提供的 **记忆化（Memoization）功能** 是**完全一致的**，尽管实现细节和环境有所不同。

### 关键区别：LRU 策略

这是它们之间最大的技术区别：

- **`@lru_cache`**：默认或配置后具有 **LRU (Least Recently Used)** 策略。这意味着当缓存达到最大容量时，它会自动丢弃**最久未使用**的条目，从而控制内存占用。
- **`cacheStringFunction`**：Vue 的这个简单实现**没有** LRU 策略。它会为遇到的每个不同的输入字符串都保留一个缓存条目。

### 为什么 Vue 不用 LRU？

在 Vue 的特定场景中，`camelize`、`hyphenate` 等函数处理的键通常是：

1. 组件的 `prop` 名。
2. HTML 属性名/事件名。

这些名字的集合在实际应用中是**有限且相对稳定**的。由于这些键会被反复使用，Vue 倾向于**永久缓存**它们，以获得最高的查找速度，而不需要复杂的 LRU 逻辑来管理一个不会无限膨胀的集合，从而避免了额外的性能开销。

## **字符串格式化**（连字符式与驼峰式互相转换）

```ts
const camelizeRE = /-\w/g
/**
 * @private
 */
export const camelize: (str: string) => string = cacheStringFunction(
  (str: string): string => {
    return str.replace(camelizeRE, c => c.slice(1).toUpperCase())
  },
)

const hyphenateRE = /\B([A-Z])/g
/**
 * @private
 */
export const hyphenate: (str: string) => string = cacheStringFunction(
  (str: string) => str.replace(hyphenateRE, '-$1').toLowerCase(),
)
```

它们都是通过 **`cacheStringFunction`** 进行封装的，以实现高性能的缓存。

## 字符串**首字母大写（Capitalize）**的工具函数

```ts
/**
 * @private
 */
export const capitalize: <T extends string>(str: T) => Capitalize<T> =
  cacheStringFunction(<T extends string>(str: T) => {
    return (str.charAt(0).toUpperCase() + str.slice(1)) as Capitalize<T>
  })

```

## **普通事件名**（如 `'click'`）转换为 **标准事件处理函数属性名**（如 `'onClick'`）的工具函数

```ts
/**
 * @private
 */
export const toHandlerKey: <T extends string>(
  str: T,
) => T extends '' ? '' : `on${Capitalize<T>}` = cacheStringFunction(
  <T extends string>(str: T) => {
    const s = str ? `on${capitalize(str)}` : ``
    return s as T extends '' ? '' : `on${Capitalize<T>}`
  },
)

```

### `toHandlerKey` 的核心作用

`toHandlerKey` 的作用是：**统一处理组件事件名与 VNode 属性名的转换。**

当你在 Vue 模板中使用 `@click` 或 `v-on:click` 时，模板编译器**需要将这个事件名转换为一个 JavaScript 对象属性**，以便在 **VNode 上存储事件监听器**。

### 运行时 JavaScript 逻辑

核心逻辑位于 `cacheStringFunction` 内部的匿名函数中：

```javascript
(str: T) => {
  const s = str ? `on${capitalize(str)}` : ``
  return s as T extends '' ? '' : `on${Capitalize<T>}`
}
```

1. **空字符串检查：** `str ? ... : ""`
   - 如果输入 `str` 是空字符串，则直接返回空字符串 `""`。
2. **转换逻辑：** ``on${capitalize(str)}``
   - **`capitalize(str)`：** 调用了前面定义的 `capitalize` 函数，将输入字符串的首字母大写（如 `'click'` → `'Click'`）。
   - **模板字面量：** 在首字母大写的字符串前添加前缀 `'on'`，得到最终的处理函数键（如 `'onClick'`）。
3. **高性能缓存：** 整个函数被 `cacheStringFunction` 封装，确保任何事件名（如 `'click'`）的转换只计算一次，后续调用直接从缓存中获取结果。

**条件类型 (`T extends '' ? '' : ...`)：** 这是一个类型级别的**三元表达式**：

- **如果** 输入类型 `T` 是空字符串字面量类型 (`''`)，则返回类型是 `''`。
- **否则**（如果 `T` 是任何非空字符串），则执行下一个类型构造。

**模板字面量类型 (`on${Capitalize<T>}`):**

- 它告诉 TypeScript 编译器：返回的类型是字符串 `'on'` 后面连接着一个由 `Capitalize<T>` 工具类型处理后的字符串（即首字母大写的输入类型）。
- **例如：** 如果传入的类型是 `'foo'`，编译器推断返回的类型是 `'onFoo'`。

## 安全地比较两个值是否发生变化

```ts
// compare whether a value has changed, accounting for NaN.
export const hasChanged = (value: any, oldValue: any): boolean =>
  !Object.is(value, oldValue)

```

### 为什么要用 `!Object.is()` 而不是 `!==`？

在大多数情况下，`!Object.is(a, b)` 的结果与 `a !== b` 是一致的。然而，`Object.is()` 解决了标准 JavaScript 严格相等性（`===`）的两个关键缺陷：

| 场景         | 标准比较 (`===`)             | `Object.is()`                       | `hasChanged` 结果 | 目的                                                         |
| ------------ | ---------------------------- | ----------------------------------- | ----------------- | ------------------------------------------------------------ |
| **NaN**      | `NaN === NaN` 为 **`false`** | `Object.is(NaN, NaN)` 为 **`true`** | **`false`**       | **核心目的：** 确保如果响应式值从 `NaN` 变为 `NaN`，**不**触发更新。 |
| **有符号零** | `-0 === +0` 为 **`true`**    | `Object.is(-0, +0)` 为 **`false`**  | **`true`**        | 允许 Vue 在极少数需要区分 `-0` 和 `+0` 的场景下，正确识别为“已变化”。 |
| **其他值**   | `1 === 1` 为 `true`          | `Object.is(1, 1)` 为 `true`         | **`false`**       | 正常判断值未变化。                                           |

### 核心价值：处理 NaN

在响应式系统中，一个值从 `NaN` 变为另一个 `NaN`，从用户的角度来看，它**没有发生有意义的变化**。

- 如果使用 `!==`，由于 `NaN !== NaN` 为 `true`，系统会错误地认为值已变化并触发不必要的更新。
- 使用 `!Object.is()`，由于 `Object.is(NaN, NaN)` 为 `true`，所以 `!Object.is(...)` 返回 **`false`**，**正确地阻止了不必要的更新**。

## **按顺序、安全地调用一个函数数组中的所有函数**，并将相同的参数传递给它们

```ts
export const invokeArrayFns = (fns: Function[], ...arg: any[]): void => {
  for (let i = 0; i < fns.length; i++) {
    fns[i](...arg)
  }
}
```

该函数专门用于处理 Vue 内部存储的**回调函数列表**。在 Vue 的生命周期、响应式系统的副作用（effect）清理、或 VNode 钩子等场景中，一个事件或一个状态变化可能需要触发多个注册的回调函数。

| 场景           | `fns` 数组中存储的内容                                 |
| -------------- | ------------------------------------------------------ |
| **生命周期**   | 组件注册的所有 `onMounted` 或 `onUpdated` 钩子函数。   |
| **副作用**     | 响应式系统 `effect` 依赖变化时需要重新执行的所有函数。 |
| **VNode 钩子** | 在 VNode 挂载、更新或卸载时需要运行的函数。            |

## **安全且不可见地**给对象添加新属性

```ts
export const def = (
  obj: object,
  key: string | symbol,
  value: any,
  writable = false,
): void => {
  Object.defineProperty(obj, key, {
    configurable: true,
    enumerable: false,
    writable,
    value,
  })
}

```

### 1. `def` 函数的核心目的

`def` 函数的目的是在不影响对象**可迭代性**和**响应式**的前提下，**为对象定义一个新属性**。

它在 Vue 内部有两个主要应用场景：

1. **添加元数据（Metadata）：** 用于给响应式对象（由 `Proxy` 代理的对象）添加 **Vue 内部的元数据或特殊标记**。
2. **避免枚举：** 确保添加的属性不会出现在 `for...in` 循环或 `Object.keys()` 的结果中，从而**保持数据的纯净**。

------



### 2. `Object.defineProperty` 参数分析

`def` 函数接收四个参数，并在内部调用 `Object.defineProperty(obj, key, descriptor)`。

| 参数名         | 值                             | 描述                         |
| -------------- | ------------------------------ | ---------------------------- |
| **`obj`**      | 传入的对象                     | 目标对象，要添加属性的对象。 |
| **`key`**      | 传入的键                       | 要定义或修改的属性的名称。   |
| **`value`**    | 传入的值                       | 属性的值。                   |
| **`writable`** | 传入的布尔值（默认为 `false`） | 控制属性是否可被重新赋值。   |



### 描述符（Descriptor）的设置

`def` 函数的强大之处在于它对属性描述符的**硬编码**设置，这决定了新属性的行为：

| 描述符             | 设置值          | 效果                                                         |
| ------------------ | --------------- | ------------------------------------------------------------ |
| **`configurable`** | `true`          | **可配置。** 允许在将来通过 `defineProperty` 再次修改描述符，或删除该属性。这提供了灵活性。 |
| **`enumerable`**   | **`false`**     | **不可枚举。** **这是关键！** 这意味着属性不会出现在 `for...in` 循环、`Object.keys()` 或 `JSON.stringify()` 的结果中。确保了 Vue 内部属性的**隐蔽性**。 |
| **`writable`**     | `writable` 变量 | **可写性。** 默认为 `false`（只读）。如果需要修改，调用者可以显式传入 `true`。 |
| **`value`**        | `value` 变量    | 属性的实际值。                                               |

### 3. 为什么在响应式系统中使用它？

在 Vue 2 的响应式系统中，`Object.defineProperty` 是实现响应式的核心。但在 Vue 3 中，它主要用于 **`Proxy` 无法接管或不需要响应式处理**的场景：

1. **防止响应式追踪：** 由于不可枚举 (`enumerable: false`)，当遍历对象进行响应式追踪时，这些内部属性会被跳过。
2. **避免泄露：** 内部状态（如 `__v_isRef` 等标记）不会污染用户的数据迭代结果。

因此，`def` 是一种创建**“隐形”**和**“只读”**内部属性的标准化、安全且高效的方式。

## 尝试将给定的值**宽松地**转换为数字

```ts
/**
 * "123-foo" will be parsed to 123
 * This is used for the .number modifier in v-model
 */
export const looseToNumber = (val: any): any => {
  const n = parseFloat(val)
  return isNaN(n) ? val : n
}

```

在 Vue 中，当用户在 `<input>` 上使用 `v-model.number` 时，Vue 需要确保输入的值（通常是字符串）被转换为真正的 JavaScript 数字类型。然而，这个转换是“宽松”的：

- **如果值以数字开头**，它应该被解析成数字。
- **如果值无法解析为数字**（例如，一个非数字字符串），则应**保持原样**（字符串）。

`looseToNumber` 函数正是实现了这种宽松转换的逻辑。

### 代码逻辑分析

export const looseToNumber=(val:any):any⇒{…}

1. **`const n = parseFloat(val)`**
   - 使用 JavaScript 的内置函数 **`parseFloat()`** 尝试解析输入值 `val`。
   - **`parseFloat()` 的特性：** 它会从字符串的开头开始解析数字，遇到第一个非数字字符时停止。
     - `'123'` → `123`
     - `'123-foo'` → `123`
     - `'foo123'` → `NaN`
     - `''` (空字符串) → `NaN`
2. **`return isNaN(n) ? val : n`**
   - **检查结果：** 使用 `isNaN()` 检查 `parseFloat` 的结果 `n` 是否为 `NaN`（Not a Number）。
   - **如果是 `NaN`：** 说明 `val` 无法从开头解析出任何有效的数字（如 `'foo'` 或 `''`）。此时，函数返回**原始值 `val`**，保持其类型不变（例如，仍是字符串）。
   - **如果不是 `NaN`：** 说明成功解析出了数字。此时，函数返回**解析后的数字 `n`**。

| 输入值 (`val`) | `parseFloat(val)` (n) | `isNaN(n)` | 返回值           | **用途**                 |
| -------------- | --------------------- | ---------- | ---------------- | ------------------------ |
| `'123'`        | `123`                 | false      | `123` (number)   | 成功转换                 |
| `'123-foo'`    | `123`                 | false      | `123` (number)   | **宽松转换**（忽略后缀） |
| `'foo'`        | `NaN`                 | true       | `'foo'` (string) | 保持原样                 |
| `''`           | `NaN`                 | true       | `''` (string)    | 保持原样                 |

这种宽松的转换对于 `v-model.number` 非常实用，它能处理像用户输入 `$123` 或 `123px` 时，仍然能提取出前面的数字部分（尽管通常 `v-model` 建议输入纯数字）。



### 与 `toNumber` 的区别

在 Vue 源码中通常还有一个 `toNumber` 函数，它与 `looseToNumber` 的区别在于严格程度：

- **`toNumber` (严格)：** 只有当字符串**完全**是一个数字时才返回数字，否则返回 `NaN` 或原始值。
- **`looseToNumber` (宽松)：** 只要字符串**以数字开头**，就会解析并返回这个数字。

## 解决 TypeScript 在**非 Node.js 环境**下编译时的类型检查问题。

```ts
// for typeof global checks without @types/node
declare var global: {}

```

你提供的这行代码：

TypeScript

```
declare var global: {}
```

它不是 Vue 源码的**执行逻辑**，而是一个纯粹的 **TypeScript 声明（Declaration）**，目的是为了解决 TypeScript 在**非 Node.js 环境**下编译时的类型检查问题。

------



## `declare var global: {}` 详解

### 1. 目的：解决 TypeScript 编译时的类型错误

在 JavaScript 生态中，存在多种全局对象：

- **浏览器环境：** 全局对象是 `window` 或 `self`。
- **Node.js 环境：** 全局对象是 `global`。

当你编写一个需要在 Node.js 中运行的模块时，你需要访问 `global`。但是，如果你的 TypeScript 项目配置（例如，`tsconfig.json` 中的 `lib` 属性）没有包含 Node.js 的类型定义（通常来自 `@types/node`），那么 TypeScript 编译器会报告一个错误：**“找不到名称 ‘global’”。**

这个 `declare var global: {}` 就是用来**欺骗** TypeScript 编译器的。



### 2. `declare` 关键字的作用

- `declare` 告诉 TypeScript 编译器：“**别担心，在程序实际运行时，这个变量（`global`）已经存在于环境中。**”
- 它只提供了**类型信息**（在这个例子中是空对象 `{}`），而**不生成任何实际的 JavaScript 代码**。



### 3. `global: {}` 的意义

- `var global`: 声明了一个名为 `global` 的全局变量。
- `{}`: 简单地将它的类型定义为一个空对象。这是一种宽松的类型定义，表示它是一个对象，但不关心它内部有什么属性。



### 4. 在 Vue 源码中的上下文

在 Vue 源码（尤其是用于寻找全局对象的 `getGlobalThis` 函数）中，这段声明的作用是：

> 确保即使开发者没有安装或配置 `@types/node`，代码中的运行时检查（例如：`typeof global !== 'undefined'`）也不会导致 TypeScript 编译器报错。它让 Vue 的工具函数在进行跨环境兼容性检查时，可以安全地引用 `global` 这个标识符。

## 可靠地找到所有可能的 JavaScript 环境（浏览器、Node.js、Web Worker 等）中的全局对象。

```ts
let _globalThis: any
export const getGlobalThis = (): any => {
  return (
    _globalThis ||
    (_globalThis =
      typeof globalThis !== 'undefined'
        ? globalThis
        : typeof self !== 'undefined'
          ? self
          : typeof window !== 'undefined'
            ? window
            : typeof global !== 'undefined'
              ? global
              : {})
  )
}
```

这段代码定义了 **`getGlobalThis`** 工具函数，它在像 Vue.js 这样的 JavaScript 库中至关重要，目的是**可靠地找到所有可能的 JavaScript 环境（浏览器、Node.js、Web Worker 等）中的全局对象。**

这个函数结合了特定的检查顺序和**缓存机制**，以确保最大的兼容性和效率。

### 1. 核心目的：通用全局对象访问

这个函数解决的核心问题是 JavaScript 环境中全局对象名称的不一致性：

| 环境                          | 全局对象名称               |
| ----------------------------- | -------------------------- |
| **现代标准 JS**               | `globalThis` (ES2020 标准) |
| **Web Worker/Service Worker** | `self`                     |
| **浏览器窗口**                | `window`                   |
| **Node.js**                   | `global`                   |

`getGlobalThis` 按**优先级顺序系统地检查这些名称，直到找到一个，确保库无论在哪里运行都能正确访问全局 API**（如 `setTimeout`、`Promise` 等）。

------



### 2. 代码逻辑和回退链

该函数使用逻辑 OR 运算符 (`||`) 和一系列嵌套的三元运算符 (`? :`) 构成一个**有优先级的回退链**：

1. **缓存检查（记忆化）：**

   JavaScript

   ```js
   _globalThis || ( /* ... 回退逻辑 ... */ )
   ```

   - 首先检查缓存变量 `_globalThis` 是否已被赋值。
   - 如果已赋值，立即返回该值，**跳过所有后续的环境检查**。这是**缓存机制**，确保函数只在第一次运行时执行耗时的环境检查。
   - 如果 `_globalThis` 是未定义的，则继续执行回退逻辑，并将结果赋值给 `_globalThis`。

2. **现代标准检查（ES2020）：**

   JavaScript

   ```
   typeof globalThis !== 'undefined' ? globalThis : /* ... */
   ```

   - 优先尝试使用 **`globalThis`** 变量。这是访问全局对象的官方标准化方式。

3. **Web Worker 检查：**

   JavaScript

   ```
   typeof self !== 'undefined' ? self : /* ... */
   ```

   - 如果 `globalThis` 未定义，则检查 **`self`**，它通常是 Web Worker 和 Service Worker 中的全局对象。

4. **浏览器窗口检查：**

   JavaScript

   ```
   typeof window !== 'undefined' ? window : /* ... */
   ```

   - 如果 `self` 也未定义，则检查 **`window`**，这是标准浏览器环境中的经典全局对象。

5. **Node.js 检查：**

   JavaScript

   ```
   typeof global !== 'undefined' ? global : /* ... */
   ```

   - 如果 `window` 也缺失，则检查 **`global`**，这是 Node.js 中使用的全局对象。

6. **最终回退：**

   JavaScript

   ```
   : {}
   ```

   - 如果**所有检查都失败**（在实际环境中极不可能），则返回一个**空对象 (`{}`)** 作为最终、无害的回退，防止运行时错误。

------



### 3. 缓存机制 (`_globalThis`)

该函数是 **延迟初始化（Lazy Initialization）结合记忆化** 的典型例子：

```js
_globalThis || (_globalThis = /* 结果 */)
```

第一次调用 `getGlobalThis()` 时，它执行完整的环境检查链，并将最终结果存储在 `_globalThis` 中。所有后续调用都直接从 `_globalThis` 中以 O(1) 的时间复杂度返回缓存值，保证了极高的效率。

## **生成访问组件 `props` 对象的 JavaScript 表达式字符串**。

```ts
export function genPropsAccessExp(name: string): string {
  return identRE.test(name)
    ? `__props.${name}`
    : `__props[${JSON.stringify(name)}]`
}
```

它的核心目的是根据 `prop` 名称（`name`）的有效性，选择最快、最简洁的属性访问方式。

------



### `genPropsAccessExp` 函数详解

该函数接收一个 `prop` 名称（`name`），并返回一个表示如何从内部的 `__props` 对象中读取该属性值的字符串。

| 部分     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| **输入** | `name: string`，要访问的属性名，例如 `'id'`, `'onClick'`, 或 `'data-abc'`。 |
| **输出** | `string`，可直接在生成的渲染函数中使用的 JavaScript 表达式。 |
| **依赖** | `identRE`（未在提供的代码块中定义，但通常是一个**正则表达式**，用于检查字符串是否为有效的 JavaScript 标识符）。 |



### 1. 检查是否为有效标识符 (`identRE.test(name)`)

`identRE` 通常用于检查 `name` 是否符合 JavaScript 中变量或属性名的命名规范：

- 以字母、下划线（`_`）或美元符号（`$`）开头。
- 后续字符可以是字母、数字、下划线或美元符号。
- 不能包含连字符（`-`）、空格或保留关键字。



### 2. 生成点式访问（Dot Notation）

$$\text{? `\_\_props.${name}`}$$

- **条件：** 当 `name` 是一个有效的 JavaScript 标识符时（例如 `'id'`, `'onClick'`）。
- **结果：** 返回点式访问字符串，例如 `__props.id`。
- **优势：** **点式访问**是 JavaScript 中**最快、最简洁**的属性访问方式，也是 Vue 优先选择的方式。



### 3. 生成方括号访问（Bracket Notation）

$$\text{: `\_\_props[${JSON.stringify(name)}]`}$$

- **条件：** 当 `name` **不是**一个有效的 JavaScript 标识符时（例如 `'data-abc'`, `'aria-label'`, 或带有空格的键）。
- **结果：** 返回方括号访问字符串，例如 `__props["data-abc"]`。
- **`JSON.stringify(name)` 的作用：** 确保 `name` 字符串被正确地用引号包裹，如果 `name` 字符串本身包含引号或其他特殊字符，`JSON.stringify` 会进行适当的转义，以保证生成的表达式是**语法正确**的 JavaScript 字符串。



### 示例对比

| 输入 `name`    | `identRE.test(name)`     | 输出表达式              | 访问方式 |
| -------------- | ------------------------ | ----------------------- | -------- |
| `'id'`         | `true`                   | `__props.id`            | 点式     |
| `'onClick'`    | `true`                   | `__props.onClick`       | 点式     |
| `'data-test'`  | `false` (包含连字符 `-`) | `__props["data-test"]`  | 方括号式 |
| `'aria-label'` | `false` (包含连字符 `-`) | `__props["aria-label"]` | 方括号式 |
| `'for'`        | `false` (保留关键字)     | `__props["for"]`        | 方括号式 |