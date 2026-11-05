​	

​	项目的入口文件是index.html 也就是呈现给给浏览器的主页

​	启动可以看一下短命令，短命令可以看package.json里，里面的script规定了命令等其他指令，比如build构建项目 npm run dev 便可以运行项目，ctrl+c可以退出vite构建工具

​	vite.config.ts是工程配置文件，可以配置插件，代理等

​	main.ts 决定了样式和创建应用app

```ts
import './assets/main.css'

import { createApp } from 'vue' //这个相当于花的盆栽
import { createPinia } from 'pinia' 

import App from './App.vue' //相当于花的根
import router from './router'

const app = createApp(App)

app.use(createPinia())
app.use(router)

app.mount('#app')
```

 index.html 里把app的位置写入了进去！ 可以看作是摆放花盆的位置

### componets

componets文件夹主要放的是组件！ 其中App.vue可以看作是app的根！

icons里的HelloWorld.vue等文件可以看作是植物的叶子，没有App.vue的优先级高！。

### assets

里面存放的是固有的css文件

### main.ts

```ts
import './assets/main.css'

import { createApp } from 'vue' //相当于盆栽
import { createPinia } from 'pinia'

import App from './App.vue' //相当于根（组件）
import router from './router'

const app = createApp(App) //把App组件挂载到app上

app.use(createPinia())
app.use(router)

app.mount('#app') //把app挂载到id为app的div(容器)里

```

可以定义应用，组件，挂载等操作

### App.vue

里面可以定义标签，包括应用所在的位置或是javascript等

## 分别暴露和默认暴露

在 JavaScript 中，模块导出（export）有两种方式：**暴露（Named Exports）\**和\**默认暴露（Default Export）**。这两种导出的方式有一些关键的差别，主要体现在如何导入、命名以及是否支持多个导出等方面。

### 1. **暴露（Named Exports）**

#### 特点：

- **暴露多个值**：一个模块可以暴露多个变量、函数或类。
- **必须使用相同的名称进行导入**：导入时需要明确指定导入的名称，且名称必须和导出的名称一致。

#### 语法：

- **导出**：

  ```
  // 可以暴露多个变量、函数、类等
  export const foo = 'bar';
  export function myFunction() {
    // ...
  }
  export class MyClass {
    // ...
  }
  ```

- **导入**：

  ```
  javascript
  
  import { foo, myFunction, MyClass } from './module';
  ```

#### 特点总结：

- 一个模块可以导出多个成员。
- 导入时需要使用 `{}` 来指定导入的具体成员名称。
- 可以选择只导入某些特定的成员。

#### 示例：

```
// module.js
export const a = 10;
export const b = 20;
export function add(x, y) {
  return x + y;
}

// main.js
import { a, add } from './module.js';
console.log(a); // 10
console.log(add(2, 3)); // 5
```

### 2. **默认暴露（Default Export）**

#### 特点：

- **只能有一个默认导出**：每个模块**只能有一个默认导出**。
- **导入时可以自定义名称**：导入时不需要使用 `{}`，并且可以为默认导出的值自定义名称。

#### 语法：

- **导出**：

  ```
  // 只能有一个默认导出
  export default function() {
    console.log('This is the default export function');
  }
  
  // 或者导出一个值
  export default 42;
  ```

- **导入**：

  ```
  import myFunction from './module';
  import number from './module';
  ```

#### 特点总结：

- 每个模块只能有一个默认导出。
- 导入时不需要使用 `{}`，并且可以自由命名。
- 默认导出通常用于导出单一对象、函数或类。

#### 示例：

```
// module.js
export default function() {
  console.log('Hello from default export');
}

// main.js
import greet from './module.js';
greet(); // Hello from default export
```

### 3. **暴露与默认暴露的对比**

| 特性         | **暴露（Named Exports）**    | **默认暴露（Default Export）**         |
| ------------ | ---------------------------- | -------------------------------------- |
| **导出数量** | 可以导出多个成员             | 只能导出一个成员                       |
| **导入方式** | 需要使用 `{}` 指定导入的名称 | 不需要 `{}`，可以自定义导入名称        |
| **命名要求** | 导入时必须使用相同的名称     | 可以自由命名                           |
| **常见用途** | 导出多个工具函数、常量、类等 | 导出一个主功能或对象，通常是单一的功能 |

## 暴露组件定义

```vue
<script lang="ts">
export default { 
    //用于导出整个组件配置对象，这样其他文件或组件就可以导入并使用它。
  name: 'App', //暴露组件名
}
</script>
```

这段代码是一个 Vue 组件的定义，它使用 TypeScript (`lang="ts"`) 编写。其目的和作用如下：

### 1. **定义组件**：

- `<script lang="ts">` 标签表示在 Vue 单文件组件中使用 TypeScript 作为脚本语言。**Vue 默认使用 JavaScript，但通过 `lang="ts"`，可以让 Vue 支持 TypeScript 特性**，如类型检查、类型推导、接口等。

### 2. **组件的 `name` 属性**：

name: 'App'是 Vue 组件的 name

 选项，用来指定组件的名称。

name选项对于 Vue 来说有几个作用：

- **调试用途**：在浏览器开发者工具中查看 Vue 组件时，**`name` 属性会显示为组件的名称**，这有**助于调试**。
- **递归组件**：如果你需要**在组件内部递归调用自己**（例如在组件中嵌套相同类型的子组件），`name` 是必须的。
- **路由视图**：如果你使用 **Vue Router 并且使用动态路由或异步组件加载**，`name` 也是**帮助 Vue 匹配路由和加载相应组件的关键。**

### 3. **暴露组件名**：

- `name: 'App'` 这行代码的意思是**暴露一个名为 `App` 的 Vue 组件**。这通常是 Vue 项目的**根组件**或者是你**自定义的某个组件**。

  在 Vue 中，每个组件都可以通过 `name` 选项来指定组件的名称，这个名称用于在 Vue 开发工具中识别组件，并且在注册组件时也非常有用。你可以在组件定义时通过 `name` 来指定它的名字，这样在父组件或者其他地方就可以引用该组件。

  ```vue
  <!-- Person.vue -->
  <template>
    <div>
      <h1>{{ name }}</h1>
    </div>
  </template>
  
  <script>
  export default {
    name: 'Person', // 这里暴露了组件名称
    data() {
      return {
        name: 'Alice'
      }
    }
  }
  </script>
  
  ```

  **解释**：

  - `name: 'Person'`：这是 Vue 组件的名称，它**暴露了该组件的名字，其他组件或应用可以通过这个名称来识别它**。

  ​        你可以在父组件中通过 `name` 来引用子组件。父组件通常通过注册组件来使用它。

  ```vue
  <!-- App.vue -->
  <template>
    <div>
      <Person /> <!-- 使用 Person 组件 -->
    </div>
  </template>
  
  <script>
  import Person from './components/Person.vue'
  
  export default {
    components: {
      Person // 注册组件时，使用组件的 `name` 属性
    }
  }
  </script>
  
  ```

  

### 4. **TypeScript 的使用**：

- 虽然这段代码没有涉及 TypeScript 的具体语法，但通过使用 `<script lang="ts">`，你可以在组件内部利用 TypeScript 的强类型特性（例如类型检查、接口、类等）来增强代码的可维护性和安全性。

## vue2和vue3的api风格

​	vue2的api是**Options**（配置）风格的， vue3的api是**composition**(组合)风格的。

​	Options的表现为：数据，方法和属性等都是分布在data,methods,computed这些代码块中的，因此，新增或者修改都要修改这些代码块里的内容。

​	因为数据是写在data中的

![image-20250203143230163](./assets/image-20250203143230163.png)

方法写在methods里，计算属性等就写在computed里，监视等在watch里！

```vue
<template>
  <div class="person">
    <h2>姓名:{{ name }}</h2>
    <h2>年龄:{{ age }}</h2>
    <button @click="showtel">查看联系方式</button>
    <button @click="ChangeName">修改名字</button>
    <button @click="ChangeAge">修改年龄</button>
  </div>
</template>

<script lang="ts">
export default {
  name: 'Person',
  data() {
    return { name: 'kiriyama', age: 20, tel: '123' }
  },
  methods: {
    ChangeName() {
      this.name = 'kiriyama_2'
    },
    ChangeAge() {
      this.age += 1
    },

    showtel() {
      alert(this.tel)
    },
  },
}
</script>
<style scoped>
.person {
  background-color: #ddd;
  box-shadow: 0 0 10px;
  border-radius: 10px;
  padding: 20px;
}
</style>

```

没加一个功能都要逮着这几个代码块猛猛改（很麻烦）

因此vue3的api改变为Composition，可以集中管理一个功能的属性，计算方法以及数据，更方便编写代码和实现

vue3中的定义功能，数据定义，计算等都在setup里进行！

```ts
  setup() {
    //数据
    let name = 'kiriyama'
    let age = 20
    let tel = '123456789'
    return {
      name: name,
      age: age,
      tel: tel,
    }
  },
```

表面数据后 需要用return返回，可以直接写name,age,tel变量名，也可以写如上的键值对！

### `let` 和 `var` 的区别

`let` 和 `var` 都是 **JavaScript 中用来声明变量的关键字**，但有以下区别：

- **`let`**：具有**块级作用域**，**不会污染全局作用域**，并且不会发生变量提升。
- **`var`**：具有**函数作用域**，可能会**出现变量提升的问题**。

在 Vue 的 `setup()` 函数里，**你可以用 `let` 来定义普通变量，但它并不会自动变成响应式数据。如果你想要响应式数据，就需要使用 Vue 提供的 API，比如 `ref` 和 `reactive`**。 setup里就无法使用 this这个关键词了

```ts
setup() {
    //数据
    let name = 'kiriyama'
    let age = 20
    let tel = '123456789'
    //方法
    function ChangeAge() {
      age = age + 1
    }
    function ChangeName() {
      name = 'kiriyama_2'
    }
    function showtel() {
      alert(tel)
    }
    return {
      name: name,
      age: age,
      tel: tel,
      ChangeAge, //交出去方法
      ChangeName,
      showtel,
    }
  },
```

​	并且你也可以**继续坚持写data对象来定义数据**，并且**data还可以读到setup对象里的定义好的数据！**（用this去访问，）**setup是最早的生命周期，他最早读到，因此setup先行。**

**语法糖写法：**

```ts
<script setup lang="ts">
let name = 'kiriyama'
let age = 20
let tel = '123456789'
//方法
function ChangeAge() {
  age = age + 1
}
function ChangeName() {
  name = 'kiriyama_2'
}
function showtel() {
  alert(tel)
}
</script>
```

那个script标签里的setup相当于return了 很方便，只是记得要加入Lang="ts"让编译器理解到你写的是ts

因此建议保留两个script标签，一个来声明组件名称，也就是

```ts
export default {
  name: 'Person',
  beforeCreate() {
    console.log('beforeCreate')
  },
}
```

​	另一个就是上面的放置功能的！也可以简化一些，通过安装插件：**npm i vite-plugin-vue-setup-extend -D**后再用import去引用进项目，项目配置文件也就是vite-config.ts: **import setup_ from 'vite-plugin-vue-setup-extend'** 之后就可以在script标签里直接写name=''来写出**组件名称**

​	只是这样写的话，数据改变了页面也不会改变（即使数据以及发生了变化），因为这些属性**不是响应式**的。因此我们接下来需要写vue3当中的响应式数据 **import { ref } from 'vue'**来完成响应式。

## 响应式

### ref

想让哪个数据是响应式就把哪个数据去用ref包裹一遍即可。

```ts
<script setup lang="ts">
let name = ref('kiriyama')
let age = ref(20)
let tel = '123456789'
```

这样 name 和age就变成对象的数据类型，

![image-20250203155613566](./assets/image-20250203155613566.png)

```vue
  <h2>姓名:{{ name }}</h2>

  <h2>年龄:{{ age }}</h2>  
```

​	这里模板里不用写.value,因为模板自动给你后面加了.value的 ，至于script里，就需要加value了，这样我们就可以得到响应式的数据类型了，并且提一嘴，模板这里其实{{}}里面还可以写函数来调用(邪教   。

### reactive写法

import { reactive } from 'vue' 先在script这里引入

```ts
let age = reactive({ age: 20 })
```

然后定义用reactive去包裹一个键值对。

```ts
<template>
  <div class="person">
    <h2>姓名:{{ name }}</h2>
    <h2>年龄:{{ age.age }}</h2>
    <button @click="showtel">查看联系方式</button>
    <button @click="ChangeName">修改名字</button>
    <button @click="ChangeAge">修改年龄</button>
  </div>
</template>

<script lang="ts">
import { ref } from 'vue'
import { reactive } from 'vue'
export default {
  name: 'Person',
  beforeCreate() {
    console.log('beforeCreate')
  },
}
</script>

<script setup lang="ts">
let name = ref('kiriyama')
let age = reactive({ age: 20 })
let tel = '123456789'
//方法
function ChangeAge() {
  age.age = age.age + 1
}
function ChangeName() {
  name.value = 'kiriyama_2'
}
function showtel() {
  alert(tel)
}
</script>
<style scoped>
.person {
  background-color: #ddd;
  box-shadow: 0 0 10px;
  border-radius: 10px;
  padding: 20px;
}
</style>

```

​	只是这样的话，你就要在上面的模板写入.age 等key 来显示他们的value。  这个也很方便！

并且被包裹起来的键值对会变成以下的格式：

![image-20250203160702773](./assets/image-20250203160702773.png)

被包裹的键值对变成了一个响应式对象，展开如下：

![image-20250203160802776](./assets/image-20250203160802776.png)

另外，reactive里面也可以**写入数组等其他数据类型！**比如**数组里再包裹键值对来达到一次性定义的好处**

并且。reactive**只能定义对象类型的响应式数据**，实际上，**ref也可以定义对象型的响应式数据。**

![image-20250203161914081](./assets/image-20250203161914081.png)

只是访问有点麻烦。。。

​	并且的是,reactive的数组可以用push进行压栈操作，可以很轻松完成一系列的任务。

```ts
const sum = ref(0)
const dogList = reactive(['https://images.dog.ceo/breeds/pembroke/n02113023_6269.jpg'])
// 方法：改变 sum 值
async function change() {
  const result = await axios.get('https://dog.ceo/api/breed/pembroke/images/random')
  console.log(result.data)
  dogList.push(result.data.message)
}

```



### toRef和toRefs

![image-20250203162523178](./assets/image-20250203162523178.png)

​	这种写法 可以把name 和age 和person所代表的对象的属性提出来，来给模板这些使用，并且下面的函数也可以正常使用！只不过这个只是把person.name 和 person.age 的响应式数据给读了出来，因此页面不会更新。

那么这里就要讲一个**toRefs**的工具

​	把这个toRefs去给person包裹一下，这样name,page就是响应式的数据了，并且是**ref所定义的响应式数据**。并且这样的话你也会**把原来的person的数据也给改了！也就是双向绑定！**

​	为什么呢，这是因为栈拷贝。**栈拷贝**（Stack Copy）是指在程序中对**数据结构（通常是变量）进行的复制操作**，这种复制操作发生在 **栈** 上，而栈是计算机内存中的一种存储区域。栈是以 **后进先出（LIFO）** 的方式管理内存的，它通常用于**存储局部变量**、**函数的参数**以及**函数调用的返回地址**。

​	在编程中，尤其是涉及到**函数调用和参数传递**时，栈拷贝是一个非常常见的操作。这里的拷贝可以分为 **值拷贝** 和 **地址拷贝** 两种情况。

#### **栈拷贝与值拷贝**：

- 当我们将一个 **基本数据类型**（如整数、浮点数等）从一个函数传递到另一个函数时，通常会进行 **栈拷贝**。这种拷贝是**值拷贝**，意味着**传递的是数据的副本**，而不是**数据的引用（地址）**。

- 例如：

  ```cpp
  void foo(int a) {
      a = 10;  // a的值被改变，但外部的变量不会受到影响
  }
  
  int main() {
      int x = 5;
      foo(x);  // x 的值会被拷贝到 foo 中
      std::cout << x;  // 输出 5
  }
  ```

  这里，**x的值被拷贝到 foo函数的栈帧中**，foo**函数的修改不会影响到 x**

  

#### 2. **栈拷贝与地址拷贝（引用拷贝）**：

- 如果我们传递的是一个 **引用类型**（如对象、数组、指针等），那么传递的是 **地址拷贝**，即拷贝的是数据在内存中的地址，而不是数据的副本。这样修改函数内部的数据会直接影响到外部的数据。

- 例如：

  ```cpp
  void foo(int &a) {
      a = 10;  // a的值被修改，外部的变量会受到影响
  }
  
  int main() {
      int x = 5;
      foo(x);  // 传递的是 x 的地址
      std::cout << x;  // 输出 10
  }
  ```

  在这种情况下，**x 的地址被传递给 foo**，因此**对 a 的修改会直接影响到 x**

#### 3. **栈拷贝与内存管理**：

- 栈拷贝通常发生在**栈内存**中。当**函数调用时，局部变量和参数会被压入栈中**。当函数调用结束时，这些**栈帧（包括栈上的拷贝）会被销毁**。
- 栈的分配和回收非常高效，因此大多数编程语言会优先使用栈来存储局部变量。但栈内存的大小是有限的，**过多的栈拷贝或递归可能会导致栈溢出（stack overflow）。**
- **栈帧**（stack frame）是**栈内存中用于存储函数**调用信息的一个**单位**。每当函数被调用时，操作系统会为该函数创建一个**栈帧**，栈帧中存储着与该函数执行相关的所有数据。这些数据包括：
  1. **函数的局部变量**：函数内部声明的变量。
  2. **函数的参数**：传递给函数的参数。
  3. **返回地址**：函数执行完毕后，程序需要返回的位置。
  4. **保存的寄存器值**：某些寄存器在函数调用前需要保存，以便函数执行后恢复。
  5. **调用者的栈帧指针**：指向调用该函数的栈帧，允许函数返回时能够正确回到调用点。

### 栈拷贝与堆拷贝的对比：

- **栈**：栈是自动管理的，内存的分配和释放由操作系统和编译器管理。栈上的数据在函数调用结束后自动销毁。栈拷贝通常发生在基本类型（如整数、字符等）数据的传递中。
- **堆**：堆是**动态分配的内存区域**，通常需要**手动管理内存（例如使用 `malloc`/`free`，或者在高级语言中使用垃圾回收机制）**。堆上的数据不会自动销毁，需要程序员**显式释放或由垃圾回收机制回收**。堆拷贝通常用于**复杂对象（如类、结构体等）**或需要在多个函数间共享的引用类型。

至于toref

![image-20250203164422922](./assets/image-20250203164422922.png)

就是单独把person里的**响应式属性提取出来**赋值并进行**双向绑定**。

## computed和输入

​	`v-model` 是 Vue.js 中用于实现 **双向数据绑定** 的指令 可以加入在<input>这种输入的标签里，实现与数据的双向绑定。

​	vue3中的计算属性方法是如下定义的，并且这么定义的计算属性是只读的！并且计算属性计算出来也是个ref，是computedref 类型的响应式数据！

```ts
let add_n = computed(() => {
  return name.value + '123'
})  //方便的lamda表达式
```

![image-20250203170249995](./assets/image-20250203170249995.png)

​	注意计算出来的属性是很聪明的，如果上一次与下一次的计算的值是一样的，那么这个**lamda表达式里的函数便不会执行**（不管你是写了什么console.log啥的），它会继续使用上一次的计算结果。

​	想要可以写可以读的就如下的写法：

![image-20250203171302097](./assets/image-20250203171302097.png)

这样写法既有get也有 set的访问器。 就十分方便了 可以修改了 可以给访问器set设置**变量传递** 

![image-20250203171628137](./assets/image-20250203171628137.png)

![image-20250203171918735](./assets/image-20250203171918735.png)

如果通过如上的函数去尝试设置计算属性的值的花便会引起访问器set的执行，会把**'li-si'**这个变量传递进去。

**`get`** 访问器用于**读取计算属性的值**，当你**访问该计算属性时**，`get` 会被触发，**返回属性的值**。

**`set`** 访问器用于**修改计算属性的值**，当你**给计算属性赋值时**，`set` 会被触发，并且会根据你**传入的值来更新对象的数据。** 就和java或者csharp一样

## watch监视

vue3中的watch可以监视以下四种数据：

1. ref定义的数据
2. reactive定义的数据
3. 函数返回一个值(getter函数)
4. 一个包含上述内容的数组



使用的话还是要先把{ref,watch}来引入再使用watch函数！

```ts
watch(sum, (newValue, old) => {
  console.log('newValue', newValue, old)
})
```

注意看watch函数的回调是一个数组，里面有新的值和旧的值！

另外 watch函数可在前定义一个函数，作用是是否停止监视。

```ts
const x = watch(sum, (newValue, old) => {
  console.log('newValue', newValue, old)
  if (newValue > 10) {
    x()
  }
})
```

​	上述所说的x函数其实就是stopwatch的函数，它可以用来判断是否停止监视！

### 监视ref对象的情况

```ts
//监视
watch(person, (newValue, old) => {
  console.log('person变化了', newValue, old)
})
```

​	这种写法watch函数不会监视person对象里的值的变化，只是当所有的属性都发生变化后，他才会发生监视事件并执行后面的函数。

​	页面在刷新一次后，**其实person就变了一次，从undefined**变成了person对象。

![image-20250214184253305](./assets/image-20250214184253305.png)

因此如果你需要监视里面的属性的话，需要在后面加上{ deep: true }

```ts
watch(person, (newValue, old) => {
  console.log('person变化了', newValue, old), { deep: true }
})
```

**具体说明：**

- **没有 `deep: true` 时**：
  - 如果你修改了对象的某个属性（如 `person.name`），`watch` 会触发，但如果你修改了对象内部的嵌套属性（如 `person.address.city`），`watch` **不会** 被触发。
- **设置 `deep: true` 时**：
  - `watch` 会监听对象内部所有属性的变化，不管你修改的是 **嵌套属性** 还是 **数组元素**，都能触发监听。

### 监视reactive定义的数据类型

![image-20250221183256054](./assets/image-20250221183256054.png)

​	用这种Object.assign的方法可以做到不改变属性的地址值，只是改变值。

只要对象的属性值发现变化就会触发watch后的函数。

![image-20250221183726384](./assets/image-20250221183726384.png)

​	如果在监视reactive对象的情况下，会**默认开启深度检测**！并且无法关闭！

### 监视单个属性

![image-20250221185420736](./assets/image-20250221185420736.png)

![image-20250221185636658](./assets/image-20250221185636658.png)

在如上的代码，里面的name 和age属于是响应式数据里的基本数据类型（字符串）

​	**注意watch可以输入的是一个函数，返回一个值，一个ref，一个响应式对象，或是由以上类型的值组成的数组**

直接                        watch(()=>{return person.name},(new,old)=>{})



```ts
watch(
  () => {
    return person.name
  },
  (newValue, oldValue) => {
    console.log('person.name变化了', newValue, oldValue)
  },
)
```

​	注意这里用的是 {return person.name}这个函数  若该属性不是对象类型，则需要写成函数形式！而person.car就是对象类型，可以直接写

### 监视多个数据的情况

​	可以写成数组形式！

​	

![image-20250221191033804](./assets/image-20250221191033804.png)

### watcheffect

![image-20250221192057463](./assets/image-20250221192057463.png)

条件判断！

![image-20250221194611293](./assets/image-20250221194611293.png)

​	在 Vue 中，使用 `ref` 来引用一个组件或 DOM 元素，使得你可以在父组件中通过这个引用来访问子组件的实例或元素。

在你的例子中：

```vue
<Person ref="ren"/>
```

## 含义：

- **`ref="ren"`**：`ref` 是 Vue 提供的一个特殊属性，用来为组件或元素创建一个引用。在这里，`ref="ren"` 为 `Person` 组件创建了一个引用，名字是 `ren`。
- **`<Person />`**：这是一个子组件，`Person` 是该组件的名称。

### 如何访问子组件：

​	通过 `ref` 创建的引用，可以**在父组件中访问子组件的实例**。在父组件（比如 `App.vue`）中，可以通过 `this.$refs.ren` 来访问 `Person` 组件的实例。

假设你在 `App.vue` 中有这样的代码：

```vue
<div>
    <Person ref="ren" />
    <button @click="accessPerson">访问Person组件</button>
  </div>
</template>

<script>
import Person from './components/Person.vue'

export default {
  components: {
    Person,
  },
  methods: {
    accessPerson() {
      // 访问 Person 组件实例
      console.log(this.$refs.ren)
      this.$refs.ren.changename()  // 调用 Person 组件中的 changename 方法
    }
  }
}
</script>
```

### 解释：

- **`this.$refs.ren`**：`$refs` 是 Vue 提供的一个对象，包含所有通过 `ref` 注册的引用。在这里，`this.$refs.ren` 会返回 `Person` 组件的实例。
- 你可以通过这个引用来调用 `Person` 组件中的方法（比如 `changename`）或访问它的属性。

### 使用 `ref` 的一些注意事项：

1. **仅在组件渲染后访问**：`ref` 只在组件渲染后才有效，因此在组件的生命周期中的某些时机（如 `mounted`）使用 `ref` 更合适。
2. **不能用于动态组件**：`ref` 无法在动态组件（通过 `v-if`、`v-for` 渲染的组件）中访问，除非组件被渲染出来。

在其他的情况，比如![image-20250221200542672](./assets/image-20250221200542672.png)

​	这个情况在vue的组件下，给标签Person了一个a的属性，可以在Person组件里接受。需要使用defineProps

```ts
import {defineProps} from 'vue'
defineProps(['a']) //如果是两个属性，则可以在数组后面加就是了！
//然后可以用了
.....
在上面加上<h2>{{a}}</h2>  就可以调用了，只不过无法打印（没有定义）

```

​	如果想要打印，就需要去定义再打印（打印出来是个对象，包含了你所有接受的东西）

```ts
let x = defineProps(['a'])
console.log(x)
```

很巧的是 ，这个特性可以和下面的这个接口一起联合使用

![image-20250221201227015](./assets/image-20250221201227015.png)

在list前面加上：  =》    :list就会去读一个变量去传递。

![image-20250221203206801](./assets/image-20250221203206801.png)

接上：的表里可以绑定有一个表达式，可以得到结果。

### 限制接受：

若list

```ts
import {type Persons} from 'vue'
import {defineProps} from '@/types'
defineProps<{list:Persons}>() //用这种方法去保证父组件传递的规范
```

//不传递，传递类型不规范都是会报错

![image-20250221204056242](./assets/image-20250221204056242.png)

加上?就表示可以为空

如果要设置默认值，需要用到withDefaults的函数

```ts
import {defineProps,withDefaults} from 'vue'
```

![image-20250221204339760](./assets/image-20250221204339760.png)

## TS中的接口

```ts
interface PersonInter{id:string,name:string,age=number}
```

​	这便定义好了一个接口用于限制person对象的具体格式，现在得暴露出去。

可以选择分别暴露。 就是在前面加上 **export**

![image-20250221200005684](./assets/image-20250221200005684.png)

![image-20250221195614664](./assets/image-20250221195614664.png)

​	这里@/types 的意思是站在/src文件夹的视角去看下面的所有资源文件，因此可以快速定位到文件上去。

并且通过接口去实现一个属性必须要符合接口规范

![image-20250221195740677](./assets/image-20250221195740677.png)

![image-20250221195851323](./assets/image-20250221195851323.png)

以及，对于列表情况，可以引用我们**之前定义的数据来定义更为复杂的自定义类型**

![image-20250221200113216](./assets/image-20250221200113216.png)

​	等到要用到响应式数据的时候，比如reactive函数，可以也在后面加上<Persons>来防止写错的情况

![image-20250221200421238](./assets/image-20250221200421238.png)

## 数据展示

### v-for：

![image-20250221203322605](./assets/image-20250221203322605.png)

后面的是数据源，前面的是**每一个数据**，相当于一个形参。

```vue
<img v-for="(dog, index) in dogList" :src="dog" :key="index" />
```

#### 解释：

1. **`v-for="(dog, index) in dogList"`**：
   - `dogList` 是你要循环遍历的数组。
   - `dog` 表示数组中的每一项，也就是每次循环中当前的元素。
   - `index` 是当前元素在 `dogList` 数组中的索引（从 0 开始）。
2. **`:src="dog"`**：
   - `src` 属性绑定到 `dog`。
   - 假设 `dogList` 是一个包含图片 URL 的数组，那么每次循环时 `dog` 就是一个图片的 URL，这会将 `<img>` 标签的 `src` 设置为当前 `dog` 对应的 URL，从而显示对应的图片。
3. **`:key="index"`**：
   - `key` 是 Vue.js 用来跟踪**每个元素的唯一标识符**。这里使用 `index` 作为 `key`，确保每个 `<img>` 标签都有一个唯一的标识符，**以帮助 Vue 更高效地更新和渲染 DOM。**

总结：这段代码会根据 `dogList` 数组的内容，生成一组 `<img>` 标签，每个标签的 `src` 属性值来自于 `dogList` 中的每一项（即图片的 URL）。并通过 `:key` 确保每个 `<img>` 标签在更新时能够高效地定位和管理。

### v-if:

![image-20250221214103699](./assets/image-20250221214103699.png)

可以控制组件的是否显示。

## 生命周期

![image-20250221204713407](./assets/image-20250221204713407.png)

![image-20250221204724000](./assets/image-20250221204724000.png)

### vue2中的生命周期写法：

把创建和挂载写在一起，然后是挂载，更新和销毁

![image-20250221213017410](./assets/image-20250221213017410.png)

4和其实是笼统的说法，具体8个

![image-20250221213102137](./assets/image-20250221213102137.png)

![image-20250221213310067](./assets/image-20250221213310067.png)

![image-20250221213332905](./assets/image-20250221213332905.png)

周期是为了帮助我们在特定的周期完成特定的操作。

### vue3的生命周期：

​	大部分一样，个别的函数名字不一样。

![image-20250221213733930](./assets/image-20250221213733930.png)

setup相当于创建

![image-20250221213900941](./assets/image-20250221213900941.png)

![image-20250221214025552](./assets/image-20250221214025552.png)

### 自定义钩子:

作用是把实现同一个功能的**数据和方法贴合在一起**。

所以要建立一个文件夹，命名为hooks,然后建立各个ts文件，把代码放在里面。

![image-20250221222731370](./assets/image-20250221222731370.png)

![image-20250221222745318](./assets/image-20250221222745318.png)

这样就可以做到很好的分离了！

## 路由

​	在/src文件夹下面创建/router文件夹然后创建index.ts后就可以开始创建路由器并暴露出去。

​	在制定路由的时候一定要制定路由器的工作模式。

先介绍单页面应用的两种路由模式：

路由的 **历史模式** 和 **哈希模式** 是两种常见的路由模式：

### 1. **历史模式（History Mode）**

#### 特点：

- URL 格式

  ：历史模式下，URL 是干净的，没有 

  ```
  #
  ```

   符号。例如：

  ```
  http://example.com/about
  ```

- **浏览器行为**：历史模式使用的是浏览器的 `history.pushState` API 来更新浏览器的地址栏，不会触发页面的重新加载。这样可以保证应用的单页面性质。

- 优势

  ：

  - URL 更加简洁和符合常规的网页结构。
  - 支持浏览器的前进、后退和刷新操作。

- 缺点

  ：

  - 需要服务器支持，因为历史模式会将 URL 中的路径作为请求发送到服务器。例如，访问 `http://example.com/about` 时，服务器需要能够识别这个路径并正确返回页面。如果服务器没有做特别配置，直接访问这些 URL 可能会导致 404 错误。

#### 工作原理：

- 当用户点击链接时，`pushState` 被调用，URL 更新，但页面并不会重新加载。
- 浏览器的前进、后退按钮会工作得和多页面网站一样，适当地更新历史记录。

### 2. **哈希模式（Hash Mode）**

#### 特点：

- URL 格式

  ：哈希模式下，URL 会包含一个 

  ```
  #
  ```

   符号。例如：

  ```bash
  http://example.com/#/about
  ```

- **浏览器行为**：哈希模式**依赖于浏览器的哈希值**（即 URL 中 `#` 后面的部分）。**哈希值的变化不会触发页面刷新**，因此它也能保持单页面应用的特性。

- 优势：

  - 不需要服务器做额外配置。因为哈希值的变化是由浏览器处理的，**服务器只会请求根路径，其他部分不会被发送到服务器，避免了 404 错误。**

- 缺点：

  - URL 中有 `#`，看起来**不如历史模式干净**。
  - 不能利用现代浏览器提供的历史记录 API（如前进和后退按钮会处理哈希而非页面路径）。

#### 工作原理：

- 当用户点击链接时，哈希值发生变化，但不会重新加载页面。
- 浏览器的前进和后退按钮会依据哈希值变化来控制页面的导航。

### 总结：

| 特性               | 历史模式（History）              | 哈希模式（Hash）                     |
| ------------------ | -------------------------------- | ------------------------------------ |
| **URL 格式**       | `/about`                         | `#/about`                            |
| **支持服务器配置** | 需要服务器支持（否则会出现 404） | 无需服务器支持                       |
| **浏览器行为**     | 使用 `history.pushState`，无刷新 | 使用哈希（`#`）更新 URL，页面无刷新  |
| **优点**           | URL 更干净、符合常规 URL 结构    | 无需服务器配置，避免 404 错误        |
| **缺点**           | 需要配置服务器，避免 404 错误    | URL 不够干净，哈希值容易暴露实现细节 |

![image-20250221230559610](./assets/image-20250221230559610.png)

写完后， 记得在main.ts中引用。

​	这样配置完了后 当路径发生了变化，VueRouter就观察到了。然后找规则，并且真的找到了这个组件后，需要实现把它放在对应的地方。

所以要引入RouterView，这样便可以实现视图的更新了

 最后我们要实现button去实现修改路径，需要使用RouterLink

```json
import {RouterLink} from 'vue-router'
```

![image-20250221232640630](./assets/image-20250221232640630.png)

可以设置激活的状态：

![image-20250221232711726](./assets/image-20250221232711726.png)

点击后就会亮

![image-20250221232732247](./assets/image-20250221232732247.png)

### 路由的两个注意点：

​	路由组件通常放在pages或者views文件夹里，一般的组件一般放在components文件夹里。通过点击导航，视觉效果上“消失”了的路由组件通常是被销毁掉了，需要再去挂载。



一般的组件是要亲手写标签出来，比如<demo/>

而路由组件是靠规则渲染出来的

routes:[{path:'/demo',component:Demo}],

### to的两种写法

![image-20250221233517177](./assets/image-20250221233517177.png)

命名路由：

![image-20250221233737884](./assets/image-20250221233737884.png)

就可以是实现通过name去实现跳转

![image-20250221233910074](./assets/image-20250221233910074.png)

### 嵌套路由

需要在routes里添加children:[]来做到嵌套路由

![image-20250221235926946](./assets/image-20250221235926946.png)

![image-20250221235935442](./assets/image-20250221235935442.png)

### 路由query参数进行传参：

可以通过在路由后加入？符号来进行传参

![image-20250222000308354](./assets/image-20250222000308354.png)

&来分割传来的参数，这个写法属于get传参。

![image-20250222000453640](./assets/image-20250222000453640.png)

这里要引入一个hooks userRoute  所有的hooks命名规则都是use+~~~来表示。

![image-20250222000658898](./assets/image-20250222000658898.png)

在网页上可以看到这个变量的详细信息，以及还有query这种传入的参数。

![image-20250222000718192](./assets/image-20250222000718192.png)

​	所以在vue里可以用RounterLink 进行点击网页的方式传入参数，让vue来进行对路径的监控。显示更加完善的消息等。

![image-20250222001115372](./assets/image-20250222001115372.png)

第二种写法：

![image-20250222001216513](./assets/image-20250222001216513.png)

### 路由params参数

特性：比query轻便一些，传递更加方便：

![image-20250222032726448](./assets/image-20250222032726448.png)

直接/去写就ok了。

/news/detail 是路径，  /哈哈/你好/嘿嘿 是传递的参数。需要在路由规则去占位。

![image-20250222033729785](./assets/image-20250222033729785.png)

:id/:title/:content就是占位用的

然后就可以在相关的路由组件去查看效果。

![image-20250222033941300](./assets/image-20250222033941300.png)

如果不能是写死的情况，和query一样 用 :to的方法。

![image-20250222034053527](./assets/image-20250222034053527.png)

第二种 :to写法

![image-20250222034329441](./assets/image-20250222034329441.png)

注意用的不是path 而是name去路由的。

![image-20250222034430855](./assets/image-20250222034430855.png)



简化写法：

![image-20250222034826471](./assets/image-20250222034826471.png)

使用toRefs

```ts
import {toRefs} from 'vue'
let {params} = toRefs(route) //把params给提取出来。
```

其实还可以更加省略，从路由规则下手。

![image-20250222035039174](./assets/image-20250222035039174.png)

### 路由的props配置

​	从上图可以看到在index.ts中定义的路由规则有一个props:true的属性。

它的作用是：

![image-20250222035224333](./assets/image-20250222035224333.png)

将path:中接受的参数 转化为params 给组件，让组件可以接下来的操作。

![image-20250222035325711](./assets/image-20250222035325711.png)

### 路由的replace属性

路由在进行跳转的时候会利用浏览器的历史记录，路由起操作历史记录有

push         和         replace两个操作

push相当于压栈，可以前进和后退的操作

replace是单个历史记录的替换，因此不能前进和后退

![image-20250222035828311](./assets/image-20250222035828311.png)

在RouterLink的地方加上replace就可以开启replace模式。

### 编程式导航

作用是在ts区域编写脚本让SPA实现编程式路由导航的跳转。

可以脱离RoterLink来实现路由跳转。

![image-20250222040524255](./assets/image-20250222040524255.png)

用push进行压栈的跳转。

### 路由重定向

需要在规则上编写。

在一级规则写完后，即可在后面添加重定向：

![image-20250222144803630](./assets/image-20250222144803630.png)

会发现: /会被重定向到/home

## vue3引入markdown编辑器

```vue
<template>
  <MdPreview :editorId="id" :modelValue="text" />
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { MdPreview } from 'md-editor-v3';
import 'md-editor-v3/lib/preview.css';

const id = 'preview-only';
const text = ref(`
# 😊欢迎来到首页
---
这是一个示例标题。

## 示例内容

- 列表项 1
- 列表项 2
- 列表项 3

### 代码块

\`\`\`javascript
// modules/my-vuetify-module
export default defineNuxtModule({
  setup(_options, nuxt) {
    // If you're using Nuxt < 3.8.1, you should add a ts-expect-error here
    nuxt.hook('vuetify:registerModule', register => register({
      moduleOptions: {
        /* module specific options */
      },
      vuetifyOptions: {
        /* vuetify options */
      },
    }))
  },
})
\`\`\`
## 结尾
希望大家能够享受这场游戏！
`)

let scrollElement: HTMLElement | null = null;

onMounted(() => {
  // 确保在客户端渲染时才访问 document
  scrollElement = document.documentElement;
});
</script>

```

这段代码的实际含义是，确保在 Vue 组件的客户端渲染阶段，正确访问浏览器中的 `document.documentElement` 元素（即 `<html>` 元素），而不会在服务器端渲染时出错。以下是更详细的解释：

### 代码分解与含义：

1. **`let scrollElement: HTMLElement | null = null;`**
   - 这行代码声明了一个变量 `scrollElement`，类型为 `HTMLElement` 或 `null`。
   - 初始值是 `null`，表示这个变量一开始没有指向任何 DOM 元素。
   - `HTMLElement` 是 HTML 文档中元素的基类，它代表所有的 HTML 元素（如 `div`、`p`、`html`、`body` 等）。
2. **`onMounted(() => { ... })`**
   - `onMounted` 是 Vue 3 的生命周期钩子，表示在组件挂载到页面后调用这个函数。具体来说，`onMounted` 会在组件的 DOM 元素已经被插入到页面中时被触发。
   - 在 Vue 的服务器端渲染（SSR）中，`onMounted` 只有在客户端渲染时才会执行。因此，这个钩子确保访问 `document` 只会发生在客户端，而不会在服务器端引发错误。
3. **`scrollElement = document.documentElement;`**
   - 这行代码将 `scrollElement` 设置为 `document.documentElement`，即整个 HTML 文档的根元素，通常就是 `<html>` 标签。
   - `document.documentElement` 代表的是整个 HTML 文档的根节点，通常在浏览器中是 `<html>` 元素。通过它，你可以访问页面的根元素，以便对页面的布局、样式等进行操作。

### 代码的实际含义：

- **客户端渲染时获取 `html` 元素**：在 `onMounted` 钩子内，确保只有在组件挂载到 DOM 后才去访问浏览器的 `document.documentElement`。这样，`scrollElement` 会指向 `<html>` 元素，并且**你可以在后续的逻辑中操作该元素**。
- **防止 SSR 错误**：如果在 Vue 应用中启用了服务器端渲染（SSR），在服务器渲染阶段 `document` 并不存在，因此不能访问 `document.documentElement`。通过将代码放在 `onMounted` 钩子内，确保它只会在客户端执行，从而避免在服务器端渲染时访问 `document` 导致的错误。

## Package.json讲解

### scripts:

`scripts` 字段定义了一组可以通过命令行运行的脚本命令。这些脚本通常用于开发、构建、测试和部署等任务，简化了常用命令的执行。

### dependencies和devDependencies

![image-20250313153916978](./assets/image-20250313153916978.png)

#### **为什么要区分这两者？**

- **优化生产环境**:
  在生产环境中，`devDependencies` 不会被安装，从而减少了依赖的体积，提升了部署效率。
- **明确依赖的用途**:
  将开发工具和运行时依赖分开，可以让团队更清楚每个依赖的作用，便于维护。

------

#### **5. 注意事项**

- **误用问题**:
  如果将开发工具（如 `sass` 或 `eslint`）放入 `dependencies`，会导致它们在生产环境中也被安装，增加了不必要的体积。

- **混合使用问题**:
  某些依赖（如 UI 库）可能既在开发阶段使用，也在运行时使用，需要根据实际情况决定放在哪个字段中。

  

## 动画制作框架：

### GSAP、

### Lottie、

### Matter、

## Vue-complier（VUE编译器讲解）:

### **为什么需要 Vue Compiler 将 `.vue` 文件转换为 render 函数？**

​	在 Vue.js 中，`.vue` 文件（单文件组件，SFC）是开发者用来编写组件的主要方式，它包含了 `template`、`script` 和 `style` 三个部分。为了让浏览器能够运行这些组件，Vue 需要将 `.vue` 文件中的 `template` 转换为 JavaScript 的 **render 函数**。这个过程由 **Vue Template Compiler** 完成。

​	Vue Compiler（如 `vue-template-compiler`）的主要作用是将 `.vue` 文件中的 `template` **转换为 render 函数**，并将其**注入到组件的 JavaScript 模块**中。

- **编译过程**：
  1. 解析 `.vue` 文件，提取 `template` 部分。
  2. 将 `template` 转换为 render 函数。
  3. 将 render 函数与组件的其他部分（如 `script` 和 `style`）整合，**生成最终的 JavaScript 模块**。
  

### **什么是 render 函数？**

​	Render 函数是 Vue.js 中的一种**底层机制**，用于**描述组件的虚拟 DOM（Virtual DOM）**。它是一个**纯 JavaScript 函数，返回一个虚拟 DOM 节点树（VNode）**。与模板语法相比，**render 函数提供了更高的灵活性和更接近底层的控制**。

例如，一个简单的模板：

```vue
<template>
  <div>Hello, {{ name }}</div>
</template>
```

会被 Vue Compiler 转换为类似这样的 render 函数：

```js
render(h) {
  return h('div', {}, `Hello, ${this.name}`);
}
```

### **为什么需要将模板转换为 render 函数？**

#### **(1) 浏览器无法直接理解模板语法**

​	`.vue` 文件中的 `template` 是一种**高层的声明式语法**，浏览器无法直接解析。为了让浏览器能够运行，**Vue 必须将这些模板转换为 JavaScript 代码（即 render 函数）**，因为浏览器可以直接执行 JavaScript。

#### **(2) 提高运行时性能**

​	模板语法是开发者友好的，但**它需要在运行时被解析为虚拟 DOM。通过在构建阶段（build time）将模板预编译为 render 函数**，可以**避免在运行时解析模板的开销，从而提高性能**。

- **开发阶段**：Vue Compiler 将模板转换为 render 函数。
- **生产阶段**：运行时**直接使用预编译的 render 函数**，减少了运行时的计算量。



![image-20250311192325277](./assets/image-20250311192325277.png)

​	只要代码一变化，就会由工具自动帮你完成以上的工作，这个就是构建工具。

## **Babel 语法降级**

​	在现代 JavaScript 开发中，**Babel** 是一个非常重要的工具，用于将现代 JavaScript 代码（ES6+）转换为向后兼容的旧版本 JavaScript（如 ES5），以便在不支持最新语法的浏览器或运行环境中运行。这一过程被称为 **语法降级（transpilation）**。

------

### **为什么需要 Babel 语法降级？**

1. **浏览器兼容性问题**
   不同的浏览器对 JavaScript 的支持程度不同，尤其是旧版浏览器（如 IE11）不支持许多现代 JavaScript 特性（如箭头函数、`let`/`const`、模块化等）。通过 Babel，可以将这些现代语法转换为旧版浏览器支持的语法。
2. **支持更广泛的用户群体**
   如果你的应用需要支持较老的设备或浏览器（如企业内部系统或面向全球用户的应用），语法降级是必要的。
3. **使用最新的 JavaScript 特性**
   Babel 允许开发者使用最新的 JavaScript 特性（如 ES2020、ES2021），即使这些特性尚未被所有浏览器支持。Babel 会将这些特性转换为兼容的旧语法。



![image-20250311193725306](./assets/image-20250311193725306.png)

## ES6语法：

### ESM和CJS

​	Nuxt 3 默认使用的是 **ES Modules (ESM)**，而不是 CommonJS (CJS)。它**完全基于现代的 ECMAScript 模块化系统构建**，旨在利用现代 JavaScript 的特性和性能优势。不过，Nuxt 3 仍然可以兼容使用 CommonJS 模块的库，但需要一些额外的处理。

- **CommonJS (CJS)** 是 Node.js 的模块系统，主要用于服务端。它的模块导入（`require`）是在运行时由服务端执行的。
- **ES Modules (ESM)** 是现代 JavaScript 的模块系统，最初设计用于浏览器，但现在也被 Node.js 支持。ESM 的模块导入（`import`）**在浏览器中是静态的**，**直接从服务端加载模块文件**，而在 Node.js 中则是服务端解析的。

**ES Modules 的导入机制：**

- ESM 是现代 JavaScript 的模块化标准，最初设计用于浏览器，后来被 Node.js 采纳。
- 使用 `import` 导入模块，模块的解析是静态的（编译时确定依赖关系）。
- 在浏览器中，ESM 模块通过 `<script type="module">` 标签加载，浏览器会直接从服务端请求模块文件。

因此 如果全局导入，会出现卡顿等问题，所以模块化的优点很大！

### const let:

ES6 新增了`let`命令，用来声明变量。它的用法类似于`var`，但是所声明的变量，只在`let`命令所在的代码块内有效。

```javascript
{
  let a = 10;
  var b = 1;
}

a // ReferenceError: a is not defined.
b // 1
```

上面代码在代码块之中，分别用`let`和`var`声明了两个变量。然后在代码块之外调用这两个变量，结果`let`声明的变量报错，`var`声明的变量返回了正确的值。这表明，`let`声明的变量只在它所在的代码块有效。

## ESM和CommonJs模块的导出：

​	基本差异：

- CommonJS 模块输出的是一个值的拷贝，ES6 模块输出的是值的引用。
- CommonJS 模块是运行时加载，ES6 模块是编译时输出接口。

​	常见的Commonjs的导出模式有命名导出、函数[导出类](https://zhida.zhihu.com/search?content_id=217245956&content_type=Article&match_order=1&q=导出类&zhida_source=entity)导出和类的实例导出几种。Commonjs导出的关键词是**exports**或**[module.exports](https://zhida.zhihu.com/search?content_id=217245956&content_type=Article&match_order=1&q=module.exports&zhida_source=entity)**。

命名导出模式示例如下：

```js
//namedexport.js
//命名的导出模式是commonjs的标准模式
exports.info=(msg)=>{
    console.log(`info:${msg}`);
}
exports.verbose=(msg)=>{
    console.log(`verbose:${msg}`);
}
```

函数导出模式示例如下：

```js
//funcexport.js
//函数导出模式是commonjs的扩展
module.exports=(msg)=>{
    console.log(`info:${msg}`);
}

module.exports.verbose=(msg)=>{
    console.log(`verbose:${msg}`);
}
```

类导出模式示例如下：

```js
//classexport.js
//类导出模式
class Logger{
    constructor(name) {
        this.name=name;
    }
    log(msg){
        console.log(`[${this.name}]-${msg}`);
    }
    info(msg){
        this.log(`info:${msg}`);
    }
    verbose(msg){
        this.log(`verbose:${msg}`);
    }
}

module.exports=Logger;
```

类的实例导出模式：

```js
//instanceexport.js
//类的实例导出
class Logger{
    constructor(name){
        this.count=0;
        this.name=name;
    }
    info(msg){
        this.count++;
        console.log(`[${this.name}] ,count=${this.count}:${msg}`);
    }
}

module.exports=new Logger('DEFAULT');
```

通过Commonjs导出模块的导入示例

```js
//命名导出的模块导入方法示例
const logger1=require('./namedexport');
logger1.info('This is a named export example');
logger1.verbose('With a verbose message');

//函数导出的模块导入方法示例
const logger2=require('./funcexport');
logger2('This is a function export example');
logger2.verbose('With a verbose message');

//类导出的模块导入方法示例
const Logger3=require('./classexport');
let logger3=new Logger3('DB');
logger3.info('This is a class export example');
logger3.verbose('With a verbose message');

//类的实例导出的模块导入方法示例
const logger4=require('./instanceexport');
logger4.info('This is a class instance export example');
const logger5=require('./instanceexport');
logger5.info('This is a class instance export example');

//可以通过以下方式向导入的模块添加customMessage方法
require('./namedexport').customMessage=(msg)=>{
    console.log(`This is a patched msg :${msg}`);
}
logger1.customMessage('logger 1 is patched');
```

##  ECMAScript模块导入与导出的方法

​	ESMAScript方式仅在**较新的Nodejs版本中受支持**，此外，**Nodejs默认采用Commonjs语法导入导出模块**，直接使用ECMAScript模块会报错，在文件对应的`package.json`中添加`"type":"module",`字段，才能正常使用ECMAScript模块。而一旦确定了`type`字段后，就不能再使用Commonjs语法，需要参考下一节的内容进行调整。

- ECMAScript模块导出

```js
//esmexport.js
//采用ESM导出示例
//导出函数
export function defaultlog(msg){
    console.log(msg);
}
//导出定义的常数
export const DEFAULT_VALUE='info';

export const LEVELS={
    error:0,
    debug:1,
    warn:2,
    data:3,
    infor:4,
    verbose:5
}

//导出类
export class DefaultClass{
    constructor(name){
        this.name=name;
    }
    log(msg){
        console.log(`[${this.name}]:${msg}`);
    }
}
```

- 默认导出

```js
//defaultexport.js
//默认类导出
export default class Logger{
    constructor(name){
        this.name=name;
    }
    log(msg){
        console.log(`[${this.name}]:${msg}`);
    }
}
```

- 模块导入示例

```js
// 导入整个模块
import * as loggerModule from './esmexport.js';
console.log(loggerModule);
const defaultClass=new loggerModule.DefaultClass('ESMA');
defaultClass.log(`test info,DEFAULTVALUE is ${loggerModule.DEFAULT_VALUE}`);

//导入模块的某个对象
import { defaultlog } from './esmexport.js';
defaultlog('esma export example');

//导入默认的模块，自定义Logger2
import Logger2 from './defaultexport.js';
const logger2 = new Logger2('default');
logger2.log('sample message');

//导入其他目录下的Commonjs模块（在同一个目录下会失败）
import { info as newinfo,verbose as newverbose } from '../namedexport.js';
newinfo('newinfo message');
newverbose('new verbose message');

//命名导出的模块导入方法示例
//将.js后缀改为.cjs，可以让node将该文件当作Commonjs解析，可以通过import进行导入
//类似地，如果要让Nodejs将一个文件当作ECMAScript，需要将.js后缀修改为.mjs
import logger1 from './namedexport2.cjs';
logger1.info('This is a named export example');
logger1.verbose('With a verbose message');
```

## 3. 模块异步导入导出

​	Nodejs支持模块的异步导入与导出，本例通过一个根据[命令行参数](https://zhida.zhihu.com/search?content_id=217245956&content_type=Article&match_order=1&q=命令行参数&zhida_source=entity)选择模块导入来进行说明。
创建`el.js`、`en.js`等文件。

```js
//el.js希腊语
export const HELLO='Γειά σου Κόσμε';
//en.js英语
export const HELLO='Hello world';
//es.js西班牙语
export const HELLO='Hola mundo';
//it.js意大利语
export const HELLO='Ciao mondo';
//pl.js波兰语
export const HELLO='Witaj świecie';
```

​	异步导入模块

```js
//根据命令行参数选择解析的语言，例如
// node ./main.js el
const SUPPORTED_LANGUAGES=['el','en','es','it','pl'];
const selectedLanguage=process.argv[2];

if(!SUPPORTED_LANGUAGES.includes(selectedLanguage)){
    console.error(`The selected language ${selectedLanguage} is not supported yet`);
    process.exit(0);
}

const translationModule=`./${selectedLanguage}.js`;
import(translationModule)
.then((str)=>{ //str是导入的模块
    console.log(`Your selected language is ${selectedLanguage} \n ${str.HELLO}`)
})
```

## Commonjs和ECMAScript部分区别

​	ECMAScript是在**严格模式下运行**的，他也**不支持Commonjs提供的一些引用**，例如require,exports,__filename,__dirname。如果要使用这些内容，**需要进行替换**。

```js
import {fileURLToPath} from 'url';
import {dirname} from 'path';
const __filename=fileURLToPath(import.meta.url);
const __dirname=dirname(__filename);
```

## 5. TypeScript的情况

​	TypeScript是**微软主导的基于JavaScript的语言**，**可用于更严谨完整的编写JavaScript程序。在TypeScript中主要使用两种导入导出的方式**，分别继承自ECMAScript和兼容Commonjs。使用`export`关键词导出的模块，使用`import`导入，而使用`export=`导出的模块则需要通过`import Module=require('module')`。在TypeScript[官方文档](https://link.zhihu.com/?target=https%3A//www.tslang.cn/docs/handbook/modules.html)（或[中文文档](https://link.zhihu.com/?target=https%3A//www.tslang.cn/docs/handbook/modules.html)）可以看到关于导入导出的详细示例。

## vite-create .  vite

​	**Vite** 是一个现代化的前端构建工具，提供**快速的开发服务器和高效的构建能力**。而 **create-vite** 是一个用于快速创建 Vite 项目的脚手架工具。

1. **Vite 是核心工具，create-vite 是辅助工具**
   Vite **本身是一个构建工具**，专注于**开发和构建的性能优化**。而 create-vite 是一个命令行工具，用于**快速生成基于 Vite 的项目模板**。它简化了项目初始化的过程，帮助开发者快速搭建一个 Vite 项目。
2. **create-vite 的作用**
   create-vite **提供了多种模板选项，支持不同的框架和语言，比如 Vue、React、Svelte、Solid 等**，以及它们的 TypeScript 版本。**通过 create-vite，开发者可以直接选择所需的模板并生成项目**，而无需手动配置。

\# 使用 npm 创建一个 Vite + Vue 项目 npm create vite@latest my-vue-app -- --template vue

**如果使用vue-cli会创建wabpack的项目,会内置wabpack!**

![image-20250313151428997](./assets/image-20250313151428997.png)

### 生产环境和开发环境：

vite会全权交给一个叫做rollup的库去完成生产环境的打包。

### 依赖预构建：

​	vite会找到对应的依赖和调用esbuild(一个用**go语言**写的**对js语法进行处理的库**)，可以帮我们把其他规范比如cmj规范的代码全部转换为esmould规范的代码，让我们使用。这是 Vite 的一个重要功能，旨在解决现代开发中 CJS 和 ESM 模块混用的问题。

​	在 **CommonJS** 模块规范中，`module.exports` 是用来定**义当前模块对外暴露的接口的**。其他文件在加载这个模块时，实际上读取的就是 `module.exports` 中的内容。

- 当其他模块通过 `require` 加载这个模块时，返回的就是 `module.exports` 的值。

而Vite：

- Vite 使用 **esbuild** 和 **Rollup** 作为底层工具。
- 当 Vite 发现**项目中引入了 CommonJS 模块时**，它会通过 **esbuild** 或 Rollup 的 `@rollup/plugin-commonjs` 插件，将 CommonJS 模块转换为 ESM 格式。

```js
// CommonJS 模块
module.exports = {
  foo: 'bar',
  add: (a, b) => a + b,
};
```

- 转换后的模块会变成类似以下的 ESM 格式：

```js
// 转换后的 ESM 模块
const foo = 'bar';
const add = (a, b) => a + b;

export default {
  foo,
  add,
};
```

![image-20250313161442778](./assets/image-20250313161442778.png)

​	同时会对**esmodules规范**的各个模块进行统一的集成。在生产环境中，Vite 使用 Rollup 作为打包工具。Rollup 会**根据动态导入的代码自动进行模块分割**，将每个动态导入的模块**打包成单独的文件（chunk）**。只有在需要时才会加载对应的模块文件，从而实现懒加载。

###  **Vue 组件导出辅助函数**：

```js
x var _export_sfc = (sfc, props) => { //src:Vue 组件对象（通常是 .vue 文件中定义的组件）。  //一个包含键值对的数组，表示需要绑定到组件上的额外属性。  const target = sfc.__vccOpts || sfc; // 获取组件的选项对象  for (const [key, val] of props) {   // 遍历传入的属性    target[key] = val;                // 将属性绑定到组件选项对象上  }  return target;                      // 返回修改后的组件选项对象};export { _export_sfc as default };	
```

​	这段代码是一个 **Vue 组件导出辅助函数**，通常由工具（如 Vite 或 Rollup）在打包 Vue 项目时自动生成，用于处理 Vue 组件的导出和属性绑定。以下是对代码的详细解析：

**`_export_sfc` 函数**

- 作用：
  - 这是一个辅助函数，用于将额外的属性（`props`）绑定到 Vue 组件的选项对象上。
  - 它通常在 Vue 3 的单文件组件（SFC）中被使用，用于处理组件的导出。

- 参数：
  - `sfc`：Vue 组件对象（通常是 `.vue` 文件中定义的组件）。
  - `props`：一个包含键值对的数组，表示需要绑定到组件上的额外属性。

- 返回值：
  - 返回修改后的组件对象。

**`sfc.__vccOpts || sfc`**

- `sfc.__vccOpts`：
  - 在 Vue 3 中，`__vccOpts` 是**组件的内部选项对象**，包含组件的配置（如 `props`、`methods`、`setup` 等）。
  - 如果 `sfc` 是一个经过编译的 Vue 组件对象，则会有 `__vccOpts` 属性。
- `sfc`：
  - 如果 `sfc` 没有 `__vccOpts` 属性（例如**未经过编译的组件**），则直接使用 `sfc` 本身。

**3. `for (const [key, val] of props)`**

- 遍历 `props` 数组，将每个键值对绑定到组件的选项对象上。
- `props` 的结构：
  - `props` 是一个数组，形如：`[['key1', value1], ['key2', value2]]`。
  - 每个键值对会被解构为 `key` 和 `val`。

**4. `target[key] = val`**

- 将 `props` 中的每个键值对绑定到组件的选项对象上。
- 例如，如果 `props` 是 `[['name', 'MyComponent']]`，那么 `target.name` 会被设置为 `'MyComponent'`。

**5. `export { _export_sfc as default }`**

- 将 `_export_sfc` 函数作为默认导出，供其他模块使用。

假设有一个 Vue 组件 `MyComponent.vue`：

```vue
<template>
  <div>Hello, Vue!</div>
</template>

<script>
export default {
  name: 'MyComponent',
};
</script>
```

在打包时，工具可能会生成以下代码：

```js
import { _export_sfc } from './plugin-vue_export-helper.mjs';

const MyComponent = {
  name: 'MyComponent',
  render() {
    return h('div', 'Hello, Vue!');
  },
};

// 使用 _export_sfc 函数绑定额外属性
export default _export_sfc(MyComponent, [['name', 'MyComponent']]);
```

​	以上的函数和代码都来自：**`plugin-vue_export-helper.mjs`**

是一个常见的辅助文件，用于处理 Vue 组件的导出。

###  **Vue 3 插件安装辅助工具**：

代码：

```js
// 引入 Vue 的 NOOP（空函数），用于定义空的 install 方法
import { NOOP } from '@vue/shared';

/**
 * 为组件添加全局注册功能的辅助函数
 * @param {Object} main - 主组件对象，通常是 Vue 组件
 * @param {Object} [extra] - 可选的额外子组件对象，键为子组件名称，值为子组件对象
 * @returns {Object} - 返回增强后的主组件对象，带有 install 方法
 */
const withInstall = (main, extra) => {
  // 为主组件添加 install 方法，用于全局注册组件
  main.install = (app) => {
    // 遍历主组件和额外子组件，将它们注册为全局组件
    for (const comp of [main, ...Object.values(extra != null ? extra : {})]) {
      app.component(comp.name, comp); // 使用 Vue 的 app.component 方法注册组件
    }
  };

  // 如果存在额外子组件，将它们挂载到主组件上，便于通过主组件访问
  if (extra) {
    for (const [key, comp] of Object.entries(extra)) {
      main[key] = comp; // 将子组件作为主组件的属性
    }
  }

  // 返回增强后的主组件对象
  return main;
};

/**
 * 为函数添加全局注册功能的辅助函数
 * @param {Function} fn - 需要注册的函数
 * @param {string} name - 注册到 Vue 全局属性的名称（如 $myFunction）
 * @returns {Function} - 返回增强后的函数，带有 install 方法
 */
const withInstallFunction = (fn, name) => {
  // 为函数添加 install 方法，用于全局注册
  fn.install = (app) => {
    fn._context = app._context; // 保存 Vue 应用的上下文
    app.config.globalProperties[name] = fn; // 将函数挂载到 Vue 的全局属性
  };

  // 返回增强后的函数
  return fn;
};

/**
 * 为指令添加全局注册功能的辅助函数
 * @param {Object} directive - 需要注册的指令对象
 * @param {string} name - 指令的名称
 * @returns {Object} - 返回增强后的指令对象，带有 install 方法
 */
const withInstallDirective = (directive, name) => {
  // 为指令添加 install 方法，用于全局注册
  directive.install = (app) => {
    app.directive(name, directive); // 使用 Vue 的 app.directive 方法注册指令
  };

  // 返回增强后的指令对象
  return directive;
};

/**
 * 为组件添加一个空的 install 方法的辅助函数
 * @param {Object} component - 需要添加空 install 方法的组件
 * @returns {Object} - 返回增强后的组件，带有空的 install 方法
 */
const withNoopInstall = (component) => {
  component.install = NOOP; // 定义一个空的 install 方法
  return component;
};

// 导出所有辅助函数，供其他模块使用
export { withInstall, withInstallDirective, withInstallFunction, withNoopInstall };
```



#### shallowRef:

​	`shallowRef` 是 Vue 3 中的一个响应式 API，用于创建**浅层响应式引用**。它与 `ref` 类似，但有一个关键区别：**`shallowRef` 只对其引用的值本身进行响应式处理，而不会递归地将对象内部的属性转为响应式**。

1. 浅层响应式：
   - `shallowRef` 只追踪引用本身的变化，而不会递归地追踪对象内部的属性。
   - 如果引用的值是一个对象，`shallowRef` 不会将对象的内部属性转为响应式。

1. 适用场景：
   - 适用于需要存储大型不可变数据结构（如对象或数组），并且不希望 Vue 对其内部属性进行深度响应式处理的场景。
   - 可以减少性能开销，尤其是在处理复杂数据时。

1. 与 `ref` 的区别：
   - `ref` 会递归地将对象的所有属性转为响应式。
   - `shallowRef` 只处理引用本身的响应式，而不会递归处理对象内部的属性。

详细请看深度响应式的区别。

## tsconfig.app.json:

```json
{
  // 继承 Vue 官方提供的 TypeScript 配置文件
  // 该配置文件包含了 Vue 项目中常用的 DOM 类型定义和最佳实践
  "extends": "@vue/tsconfig/tsconfig.dom.json",

  // TypeScript 编译器的选项配置
  "compilerOptions": {
    // 指定 TypeScript 增量编译的缓存文件路径
    // 增量编译会将编译信息存储在这个文件中，以便下次编译时加快速度
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",

    // 指定要包含的类型声明文件
    // 这里引入了 @element-plus/icons-vue 的类型定义，方便在项目中使用该库的类型
    "types": ["@element-plus/icons-vue"],

    /* Linting 配置项，用于提高代码质量和可维护性 */

    // 启用 TypeScript 的严格模式
    // 严格模式会启用一系列严格的类型检查规则（如 noImplicitAny、strictNullChecks 等）
    "strict": true,

    // 是否生成编译后的文件
    // 设置为 false，表示允许生成编译后的文件（如 .js 文件）
    "noEmit": false,

    // 是否允许显式导入 .ts 文件
    // 设置为 false，表示不允许在导入模块时使用 .ts 扩展名（如 import './file.ts'）
    "allowImportingTsExtensions": false,

    // 启用 TypeScript 项目引用功能
    // 当一个项目被另一个项目引用时，composite 必须设置为 true
    // 它还要求 declaration 和 declarationMap 选项也被启用（通常在继承的配置中定义）
    "composite": true,

    // 禁止在代码中声明未使用的局部变量
    // 如果有未使用的变量，编译器会报错，帮助保持代码整洁
    "noUnusedLocals": true,

    // 禁止在函数中声明未使用的参数
    // 如果有未使用的参数，编译器会报错，帮助保持代码整洁
    "noUnusedParameters": true,

    // 禁止 switch 语句中的 case 分支没有 break 或 return 导致意外执行下一个分支的情况
    // 如果没有明确的 break 或 return，编译器会报错
    "noFallthroughCasesInSwitch": true,

    // 禁止导入模块时未显式使用的副作用
    // 例如，import './file' 会被认为是有副作用的导入
    // 如果没有显式使用导入内容，编译器会报错
    "noUncheckedSideEffectImports": true
  },

  // 指定要包含在编译中的文件
  "include": [
    // 包括 src 目录下的所有 TypeScript 文件
    "src/**/*.ts",

    // 包括 src 目录下的所有带 JSX 的 TypeScript 文件
    "src/**/*.tsx",

    // 包括 src 目录下的所有 Vue 单文件组件
    "src/**/*.vue"
  ]
}
```

## tsconfig.node.json

```json
{
  "compilerOptions": {
    // 指定 TypeScript 增量编译的缓存文件路径
    // 增量编译会将编译信息存储在这个文件中，以便下次编译时加快速度
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",

    // 指定 ECMAScript 的目标版本
    // 这里设置为 ES2022，表示编译后的代码将符合 ES2022 的规范
    "target": "ES2022",

    // 指定要包含的库
    // "ES2023" 包含最新的 ECMAScript 特性
    // "DOM" 包含浏览器环境的 DOM 类型定义
    "lib": ["ES2023", "DOM"],

    // 指定模块系统
    // "ESNext" 表示使用最新的 ECMAScript 模块系统
    "module": "ESNext",

    // 跳过库文件的类型检查
    // 例如，跳过 node_modules 中的类型检查，以加快编译速度
    "skipLibCheck": true,

    // 启用 TypeScript 项目引用功能
    // 当一个项目被另一个项目引用时，composite 必须设置为 true
    "composite": true,

    /* Bundler mode 配置项 */

    // 指定模块解析策略
    // "bundler" 表示使用打包工具（如 Vite、Webpack）的模块解析方式
    "moduleResolution": "bundler",

    // 强制每个文件都被视为独立的模块
    // 这是为了兼容打包工具的行为
    "isolatedModules": true,

    // 强制模块检测
    // "force" 表示即使没有显式的导入或导出，也会将文件视为模块
    "moduleDetection": "force",

    // 是否生成编译后的文件
    // 设置为 false，表示允许生成编译后的文件（如 .js 文件）
    "noEmit": false,

    /* Linting 配置项，用于提高代码质量和可维护性 */

    // 启用 TypeScript 的严格模式
    // 严格模式会启用一系列严格的类型检查规则（如 noImplicitAny、strictNullChecks 等）
    "strict": true,

    // 禁止在代码中声明未使用的局部变量
    // 如果有未使用的变量，编译器会报错，帮助保持代码整洁
    "noUnusedLocals": true,

    // 禁止在函数中声明未使用的参数
    // 如果有未使用的参数，编译器会报错，帮助保持代码整洁
    "noUnusedParameters": true,

    // 禁止 switch 语句中的 case 分支没有 break 或 return 导致意外执行下一个分支的情况
    // 如果没有明确的 break 或 return，编译器会报错
    "noFallthroughCasesInSwitch": true,

    // 禁止导入模块时未显式使用的副作用
    // 例如，import './file' 会被认为是有副作用的导入
    // 如果没有显式使用导入内容，编译器会报错
    "noUncheckedSideEffectImports": true
  },

  // 指定要包含在编译中的文件
  "include": [
    // 包括 vite.config.ts 文件
    // 这是 Vite 的配置文件，通常需要单独编译
    "vite.config.ts"
  ]
}
```







## 音频接入

HTML5 <audio>或<video>元素生成的音频源,是[AudioBufferSourceNode](https://developer.mozilla.org/zh-CN/docs/Web/API/AudioBufferSourceNode)接口。

​	**`AudioBufferSourceNode`** 接口继承自 [`AudioScheduledSourceNode`](https://developer.mozilla.org/zh-CN/docs/Web/API/AudioScheduledSourceNode)，表现为一个**音频源**，它包含了一些写在内存中的音频数据，通常储存在一个 ArrayBuffer 对象中。在处理有严格的时间精确度要求的回放的情形下它尤其有用。比如播放那些需要满足一个指定节奏的声音或者那些储存在内存而不是硬盘或者来自网络的声音。为了播放那些有时间精确度需求但来自网络的流文件或者来自硬盘，则使用 [`AudioWorkletNode`](https://developer.mozilla.org/zh-CN/docs/Web/API/AudioWorkletNode) 来实现回放。

所有使用的变量：

```js
// 创建和管理音频上下文
const audiocontext = window.AudioContext
const audioctx = new audiocontext()

// 创建音量控制节点
const gain = audioctx.createGain()

// 音量值绑定
const gainvalue_before = ref(30)
const gainvalue = ref(gainvalue_before.value / 100)

// 定义全局变量
let source: AudioBufferSourceNode | null = null 
let isPlaying = ref(false)
let duration_isplaying = ref(0)
let audioBuffer: AudioBuffer | null = null
let currentTime = ref(0) // 添加当前播放时间的引用
let pausedAt = 0 // 记录暂停时的播放位置
// 添加一个变量来记录开始播放的时间
let startTime = 0
// 监听音量值变化
```

监听：

```js
// 监听音量值变化
watch(gainvalue, (newValue) => {
    gain.gain.value = newValue // 实时更新音量
})
watch(gainvalue_before, (newValue) => {
    gainvalue.value = newValue / 100
})
watch(() => music_store.mp3_data, (newData) => {
    if (newData && newData.length > 0) {
        music_meata.value = newData[music_store.music_index]
        loadAudio()
    }
}, { immediate: true })

watch(() => music_store.music_index, (newIndex) => {
    if (newIndex >= 0 && newIndex < music_store.mp3_data.length+1) {
        // 直接使用新的索引获取对应的音乐数据
        music_meata.value = music_store.mp3_data[newIndex]
        // 如果正在播放，先停止当前播放
        if (isPlaying.value) {
            source?.stop()
            source?.disconnect()
            isPlaying.value = false
        }
        // 重置播放相关的状态
        currentTime.value = 0
        pausedAt = 0
        // 加载新的音频
        loadAudio()
    } else {
        console.log("超出索引，重置为 0")
        music_store.music_index = 0
    }
})
```

函数;

```js
// 切换歌曲 下一首 上一首可以同理写
const nextmusic = () => {
    console.log("下一首")
    music_store.music_index += 1
    console.log("当前序列:",music_store.music_index)
}

// 格式化时间 让秒->分和秒显示
const formatTime = (time: number) => {
    const minutes = Math.floor(time / 60)
    const seconds = Math.floor(time % 60)
    return `${minutes}:${seconds.toString().padStart(2, '0')}`
}

//处理进度条拖动的函数
const handleTimeChange = (newTime: number) => {
    currentTime.value = newTime
    if (isPlaying.value) {
        // 如果正在播放，先停止当前播放
        source?.stop()
        source?.disconnect()
        
        // 创建新的音源节点从新位置开始播放
        source = audioctx.createBufferSource() //让source获得缓冲区源
        source.buffer = audioBuffer   //将音乐的数据加载进缓冲区
        source.connect(gain)
        source.start(0, newTime)
        
        // 更新开始时间和暂停位置
        startTime = audioctx.currentTime
        pausedAt = newTime
    } else {
        // 如果暂停状态，只更新暂停位置
        pausedAt = newTime
    }
}

// 播放音乐
const playfunction = () => {
    if (!audioBuffer) {
        console.error("音频未加载")
        return
    }

    if (isPlaying.value) {
        console.log("音频已经在播放中")
        return
    }

    // 创建新的音源节点
    source = audioctx.createBufferSource()
    source.buffer = audioBuffer
    duration_isplaying.value = audioBuffer.duration
    source.connect(gain)  //source和增益器相连
    gain.connect(audioctx.destination) //增益器输出到接口处播放
    gain.gain.value = gainvalue.value //gain的增益--（0-1）

    // 记录开始播放的时间
    startTime = audioctx.currentTime

    // 从暂停位置开始播放
    if (pausedAt > 0) {
        source.start(0, pausedAt)
        console.log(`从暂停位置 ${pausedAt} 秒继续播放`)
    } else {
        source.start(0)
        console.log("从头开始播放")
    }

    isPlaying.value = true
    
       // 修改更新时间的计算方式
       const updateTime = () => {
        if (isPlaying.value && audioBuffer) {  // 添加 audioBuffer 检查
            // 计算实际播放时间 = 暂停位置 + (当前时间 - 开始播放时间)
            currentTime.value = pausedAt + (audioctx.currentTime - startTime)
            requestAnimationFrame(updateTime)
            
            // 添加空值检查
            if(currentTime.value >= audioBuffer?.duration){
                console.log("音频播放结束")
                currentTime.value = 0
                pausedAt = 0
                isPlaying.value = false
                source = null
            }
        }
    }
    updateTime()
}

// 暂停音乐
const pauseAudio = () => {
    if (!source || !isPlaying.value) {
        console.log("音频未在播放中")
        return
    }

    // 使用计算后的实际播放时间
    pausedAt = currentTime.value
    console.log("暂停播放")
    console.log("暂停位置：", pausedAt)
    source.stop()
    source.disconnect()
    source = null
    isPlaying.value = false
}

// 加载音频数据
const loadAudio = async () => {
    if (!music_meata.value) {
        console.error("没有音乐数据")
        return
    }
    const response = await fetch(music_meata.value.filepath)
    console.log("加载音频数据", music_meata.value.filepath)
    const arrayBuffer = await response.arrayBuffer() //表示原始二进制数据的通用固定长度缓冲区
    //是 JavaScript 中处理二进制数据的基础类型，常用于文件读取、网络请求（如 fetch）、音频处理等场景。
    audioBuffer = await audioctx.decodeAudioData(arrayBuffer) //AudioBuffer 是 Web Audio API 提供的一种高级数据结构，用于表示解码后的音频数据。
    duration_isplaying.value = audioBuffer.duration
    console.log("音频加载完成")
}

```



## element-plus 和 element-plus-icons接入方式

引入element-plus需要引入ElementPlus组件库

**import ElementPlus from 'element-plus'**

然后引入组件样式：

**import 'element-plus/dist/index.css';**

icons引入方式：

```js
import * as Icon from '@element-plus/icons'

const app = createApp(App)

for (let i in Icon){

  console.log(i)

  console.log((Icon as any)[i])

}

```

这样就可以打印出所有的icons图标。

![image-20250327000845647](./assets/image-20250327000845647.png)

​	可以看到所有的icon都是大写字母开头。因此为了让我们不用跟个傻逼一样一个一个去导入，因此可以用遍历和正则化表达式去匹配和导入，我们需要把所有的图标都**注册为<el-icon-......的组件才行**，在utils里**新建index.ts**方法去正则表达式匹配首字符大写并且转换为全小写。

```js
// /src/utils/index.ts
export const toline = (str: string) => {
  return str.replace(/(A-Z)g/,'-$1').toLocaleLowerCase()
};
```

```js
import { createApp } from 'vue'
import './style.css'
import router from './router/inedx'
import App from './App.vue'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css';
import * as Icon from '@element-plus/icons'
import { toline } from './utils'
const app = createApp(App)
//全局注册图标 加载较慢
for (let i in Icon){
    app.component(`el-icon-${toline(i)}`,(Icon as any)[i]) //注册全局组件
    console.log(i)
    console.log((Icon as any)[i])
}
app.use(ElementPlus)
app.use(router)
app.mount('#app')
```

这样就可以正常使用了，顺便一提，所有的图标都是svg格式。我们可以设置svg格式图标的大小。

另外还有通过class去导入的方法

![image-20250327004046730](./assets/image-20250327004046730.png)



## element二次封装

#### 常用的单位：

**px:**     px单位的特点是固定不变，一旦设置了就无法因为适应页面大小而改变。因此，px单位适用于需要精确控制元素尺寸和位置的场合，如设计图标、按钮等。

**rem和em是相对单位**   :**rem单位是相对于根元素的字体尺寸来定义的**，而**em单位则是相对于当前元素内文本的字体尺寸来定义的**。这意味着，如果我们改变了根元素或父元素的字体尺寸，rem和em单位的元素尺寸也会相应地改变。因此，**rem和em单位适用于需要自适应不同屏幕尺寸和分辨率的响应式布局。**

**%、vmin和vmax**   :%单位相对于**父元素的相关尺寸**，适用于**需要相对于父元素进行布局的场合**。vmin和vmax单位则是**vw**和**vh**中的**较小值和较大值**，**适用于需要根据视窗大小自适应调整元素尺寸的场合**。

​	*vw/vh 是一个相对单位*（类似em 和rem 相对单位） vw 是：viewport width 视口宽度单位 vh 是：viewport height 视口高度单位相对视口的尺寸



