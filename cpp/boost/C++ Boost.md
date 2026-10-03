
[Boost C++ 库](http://www.boost.org/doc/libs/) 是一组**基于C++标准的现代库**。 其源码按 [Boost Software License](http://www.boost.org/LICENSE_1_0.txt) 来发布，允许任何人自由地使用、修改和分发。 这些库是平台独立的，且支持大多数知名和不那么知名的编译器。

Boost 社区负责开发和发布 Boost C++ 库。 社区由一个很大的C++开发人员群组组成，这些开发人员来自于全球，他们通过网站 [www.boost.org](http://www.boost.org/) 以及几个邮件列表相互协调。 社区的使命是开发和收集高质量的库，作为C++标准的补充。 那些被证实**有价值且对于C++应用开发非常重要的库，将会有很大机会在某天被纳入C++标准中**。

Boost 社区在1998年左右出现，当时刚刚发布了C++标准的第一个版本。 从那时起，社区就不断地扩大，现在已成为C++标准化工作中的一个重要角色。 **虽然 Boost 社区与标准化委员会之间没有直接的关系，但有部分开发者同时活跃于两方**。 下一个版本的C++标准很大可能在2011年通过，其中将扩展一批库，**这些库均起源于 Boost 社区**。

要**增强C++项目的生产力，除了C++标准以外，Boost C++ 库是一个不错的选择**。 由于当前版本的C++标准在2003年修订之后，C++又有了新的发展，所以 Boost C++ 库提供了许多新的特性。 由于有了 Boost C++ 库，我们无需等待下一个版本的C++标准，就可以立即享用C++演化中取得的最新进展。

Boost C++ 库具有良好的声誉，这基于它们的使用已被证实是非常有价值的。 在面试中询问关于 Boost C++ 库的知识是不常见的，因为知道这些库的开发人员通常也清楚C++的最新创新，并且能够编写和理解现代的C++代码。

## 智能指针

我们之前经常见到有一个词语叫做 RAII ：资源申请即初始化。**智能指针只是这个习语的其中一例——当然是相当重要的一例**。 智能指针确保在任何情况下，**动态分配的内存都能得到正确释放**，从而将**开发人员从这项任务中解放了出来**。 这包括程序因为异常而中断，原本用于释放内存的代码被跳过的场景。 **用一个动态分配的对象的地址来初始化智能指针**，在析构的时候释放内存，就确保了这一点。 **因为析构函数总是会被执行的，这样所包含的内存也将总是会被释放**。

一个**作用域指针**独占一个**动态分配的对象**。 对应的类名为 `boost::scoped_ptr`，它的定义在 `boost/scoped_ptr.hpp` 中。 不像 `std::auto_ptr`，一个**作用域指针不能传递它所包含的对象的所有权到另一个作用域指针**， 一旦**用一个地址来初始化**，这个**动态分配的对象将在析构阶段释放**。

因为一个作用域指针只是简单保存和独占一个内存地址，所以 `boost::scoped_ptr` 的实现就要比 `std::auto_ptr` 简单。 **在不需要所有权传递的时候**应该优先使用 `boost::scoped_ptr` 。 在这些情况下，比起 `std::auto_ptr` 它是一个更好的选择，因为可以避免不经意间的所有权传递。
### 共享指针

最常用的是标准库中的std::unique_ptr和boost库中的共享指针，命名为boost::shared_ptr。后者不会像前者一样必须独占一个对象，他可以和同类型的指针去共享所有权，这种情况下，当引用类型的最后一个智能指针被销毁，对象才会释放（计数器模式），但是值得注意的是多个线程读写同一个对象的时候依然需要加lock，因为指针并不会保证线程安全。

```cpp
#include <boost/shared_ptr.hpp> 
#include <vector> 

int main() 
{ 
  std::vector<boost::shared_ptr<int> > v; 
  v.push_back(boost::shared_ptr<int>(new int(1))); 
  v.push_back(boost::shared_ptr<int>(new int(2))); 
}
```

多亏了有 `boost::shared_ptr`，我们才能像上例中展示的那样，在**标准容器中安全的使用动态分配的对象**。 因为 `boost::shared_ptr` 能够**共享它所含对象的所有权**，所以保存**在容器中的拷贝**（包括容器在需要时额外创建的拷贝）都是和原件相同的。如前所述，`std::auto_ptr`做不到这一点，所以绝对不应该在容器中保存它们。

类似于 `boost::scoped_ptr`， `boost::shared_ptr` 类重载了以下这些操作符：`operator*()`，`operator-&gt;()` 和 `operator bool()`。另外还有 `get()` 和 `reset()` 函数来获取和重新初始化所包含的对象的地址。

```cpp
#include <boost/shared_ptr.hpp> 

int main() 
{ 
  boost::shared_ptr<int> i1(new int(1)); 
  boost::shared_ptr<int> i2(i1); 
  i1.reset(new int(2)); 
}
```

### 共享数组

共享数组的行为类似于**共享指针**。 关键不同在于**共享数组在析构时**，默认使用 `delete[]` 操作符来**释放所含的对象**。 因为这个操作符**只能用于数组对象**，共享数组必须通过**动态分配的数组的地址**来初始化。

共享数组对应的类型是 `boost::shared_array`，它的定义在 `boost/shared_array.hpp` 里。
```cpp
#include <boost/shared_array.hpp> 
#include <iostream> 

int main() 
{ 
// 1. 在堆上分配内存，并将首地址交给 i1 管理
  boost::shared_array<int> i1(new int[2]); 
  // 2. 拷贝构造函数：i2 和 i1 共享同一块内存地址
  boost::shared_array<int> i2(i1); 
  // 3. 修改 i1 指向的内存，i2 自然也能看到（因为它们指向同一个地方）
  i1[0] = 1; 
  std::cout << i2[0] << std::endl; 
}
```

在 C# 中，数组自带长度信息（`arr.Length`）。但在 C++ 的 `shared_array` 中：

- **它不记录长度**：你传递了 `new int[2]`，但 `i1` 并不知道它里面有 2 个元素。如果你访问 `i1[5]`，程序会直接崩溃或产生随机错误（越界访问）。
    
- **现代化替代方案**：在现代 C++（C++11 及以后）中，我们更倾向于使用 **`std::vector<int>`** 或 **`std::shared_ptr<std::vector<int>>`**，因为 `vector` 会像 C# 的 `List` 或数组一样记住自己的长度，并且更加安全。

### 弱指针

`weak_ptr` 的设计初衷不是为了独立存在，而是为了解决 `shared_ptr` 的两个痛点：

- **循环引用（Circular Dependency）**：两个对象互相持有对方的 `shared_ptr`，导致引用计数永远不为 0，内存永远无法释放。
    
- **对象生命周期探测**：你想知道一个对象是否还活着，但你不想因为你的“观察”而强行让它一直活着。
就比如说：
```cpp
#include <windows.h> 
#include <boost/shared_ptr.hpp> 
#include <boost/weak_ptr.hpp> 
#include <iostream> 

DWORD WINAPI reset(LPVOID p) 
{ 
  boost::shared_ptr<int> *sh = static_cast<boost::shared_ptr<int>*>(p); 
  sh->reset(); 
  return 0; 
} 

DWORD WINAPI print(LPVOID p) 
{ 
  boost::weak_ptr<int> *w = static_cast<boost::weak_ptr<int>*>(p); 
  boost::shared_ptr<int> sh = w->lock();  // 检查该 `weak_ptr` 观察的对象是否还活着（即强引用计数是否 > 0）,活着就会创建一个新的共享指针，强引用计数+1，若是对象已经销毁，则会返回空的共享指针，内部为nullptr。
  
  if (sh) 
    std::cout << *sh << std::endl; 
  return 0; 
} 

int main() 
{ 
  boost::shared_ptr<int> sh(new int(99)); 
  boost::weak_ptr<int> w(sh); 
  HANDLE threads[2]; 
  threads[0] = CreateThread(0, 0, reset, &sh, 0, 0); 
  threads[1] = CreateThread(0, 0, print, &w, 0, 0); 
  WaitForMultipleObjects(2, threads, TRUE, INFINITE); 
}
```

第一个线程函数 `reset()` 的参数是**一个共享指针的地址**。 第二个线程函数 `print()` 的参数是一个**弱指针的地址**。 这个弱指针是之前通过共享指针初始化的。

一旦程序启动之后，`reset()` 和 `print()` 就都开始执行了。 不过**执行顺序是不确定的**。 这就导致了一个潜在的问题：`reset()` 线程在销毁对象的时候`print()` 线程可能正在访问它。

通过调用弱指针的 `lock()` 函数可以解决这个问题：**如果对象存在**，那么 `lock()` 函数**返回的共享指针指向这个合法的对象**。否则，返回的共享指针被设置为0，**这等价于标准的null指针**。

**弱指针本身对于对象的生存期没有任何影响**， `lock()` 返回一个共享指针，`print()` 函数就可以**安全的访问对象了**。 这就保证了——**即使另一个线程要释放对象——由于我们有返回的共享指针**，对象**依然存在**。

### 介入式指针

介入式指针的工作方式和共享指针完全一样。 `boost::shared_ptr` 在内部记录着引用到某个对象的共享指针的数量，可是对介入式指针来说，程序员就得自己来做记录。 对于框架对象来说这就特别有用，**因为它们记录着自身被引用的次数**。

 核心区别：计数器放在哪？

- **`shared_ptr` (非介入式)**：当你创建一个 `shared_ptr` 时，系统会在堆上额外开辟一块内存（控制块）来存计数器。这意味着**一个对象会有两块内存**：对象本身 + 控制块。
    
- **`intrusive_ptr` (介入式)**：它假设你的类里已经有一个成员变量（比如 `int ref_count`）用来存计数了。它只是负责在构造/析构时去调用你定义的增加/减少计数的函数。

```cpp
#include <boost/intrusive_ptr.hpp>
#include <atlbase.h>
#include <iostream>
// 当 intrusive_ptr 被复制或创建时，Boost 会自动调用这个函数

void intrusive_ptr_add_ref(IDispatch *p)
{
    // 调用 COM 原生的增加引用计数方法
    p->AddRef();
}
// 当 intrusive_ptr 离开作用域销毁时，Boost 会自动调用这个函数
void intrusive_ptr_release(IDispatch *p)
{
    // 调用 COM 原生的减少引用计数方法。当计数归零，对象自动在内存中销毁
    p->Release();
}
void check_windows_folder()
{
    CLSID clsid; // 存储组件的唯一标识符 (GUID)
    // 第一步：根据字符串名称 (ProgID) 获取对应的 ID
    // CComBSTR 是 ATL 提供的宽字符串包装类，专门用于 COM 交互
    CLSIDFromProgID(CComBSTR("Scripting.FileSystemObject"), &clsid);
    void *p;
    // 第二步：正式创建 COM 对象实例 (相当于 C# 的 new)
    // CLSCTX_INPROC_SERVER 表示该组件作为 DLL 在进程内运行
    CoCreateInstance(clsid, 0, CLSCTX_INPROC_SERVER, __uuidof(IDispatch), &p);
    // 第三步：用介入式指针接管原始指针
    // 注意：CoCreateInstance 返回时计数已为 1，disp 创建时又会调用 AddRef 变成 2
    // 工业级写法通常会加参数 false 以避免内存泄漏：boost::intrusive_ptr<IDispatch> disp(static_cast<IDispatch*>(p), false);
    boost::intrusive_ptr<IDispatch> disp(static_cast<IDispatch*>(p));
    // 第四步：使用 ATL 辅助类来调用方法 (类似于 C# 的反射调用)
    CComDispatchDriver dd(disp.get());
    // 第五步：准备参数和返回值
    // CComVariant 对应 C# 的 object/dynamic 类型，它可以存储数字、字符串、布尔值等
    CComVariant arg("C:\\Windows"); // 要检查的路径
    CComVariant ret(false);         // 接收返回值的变量
    // 第六步：执行方法。调用 "FolderExists" 方法，传入 1 个参数，结果存入 ret
    dd.Invoke1(CComBSTR("FolderExists"), &arg, &ret);

    // 输出结果：将 COM 的 VARIANT_BOOL 转换为 C++ 的 bool 并打印
    std::cout << "文件夹是否存在: " << (ret.boolVal != 0) << std::endl;
} // 函数结束时，disp 自动析构，调用 Release()，COM 对象安全释放

void main()

{
    // 初始化 COM 库。这相当于在当前线程开启“COM 模式”
    CoInitialize(0);
    check_windows_folder();
    // 释放 COM 库资源，关闭环境
    CoUninitialize();
}
```

### 指针容器

见过 Boost C++ 库的各种智能指针之后，应该能够编写安全的代码，来**使用动态分配的对象和数组**。多数时候，**这些对象要存储在容器里**——如上所述——使用 `boost::shared_ptr` 和 `boost::shared_array` 这就相当简单了。

```cpp
#include <boost/shared_ptr.hpp> 
#include <vector> 

int main() 
{ 
  std::vector<boost::shared_ptr<int> > v; 
  v.push_back(boost::shared_ptr<int>(new int(1))); 
  v.push_back(boost::shared_ptr<int>(new int(2))); 
}
```

上面例子中的代码当然是正确的，智能指针确实可以这样用，然而因为某些原因，实际情况中并不这么用。 第一，反复声明 `boost::shared_ptr` **需要更多的输入**。 其次，将 `boost::shared_ptr` 拷进，拷出，或者**在容器内部做拷贝**，**需要频繁的增加或者减少内部引用计数**，这肯定效率不高。 由于这些原因，Boost C++ 库提供了 [指针容器](http://www.boost.org/libs/ptr_container/) 专门用来**管理动态分配的对象**。

```cpp
#include <boost/ptr_container/ptr_vector.hpp> 

int main() 
{ 
  boost::ptr_vector<int> v; 
  v.push_back(new int(1)); 
  v.push_back(new int(2)); 
}
```

## 事件处理

很多开发者在听到术语'事件处理'时就会想到GUI：点击一下某个按钮，相关联的功能就会被执行。 **点击本身就是事件**，而功能就是**相对应的事件处理器**。

这一模式的使用当然不仅限于GUI。 一般情况下，任意对象都可以调用基于特定事件的专门函数。 本章所介绍的 [Boost.Signals](http://www.boost.org/libs/signals) 库提供了一个简单的方法在 C++ 中应用这一模式。

严格来说，Boost.Function 库也可以用于事件处理。 不过，Boost.Function 和 Boost.Signals 之间的一个主要区别在于，Boost.Signals 能够将一个以上的事件处理器关联至单个事件。 因此，**Boost.Signals 可以更好地支持事件驱动的开发**，当需要进行**事件处理**时，应作为**第一选择**。

### 信号 Signals

 Boost.Signals 所实现的模式被命名为 '信号至插槽' (signal to slot)，它基于以下概念：**当对应的信号被发出时，相关联的插槽即被执行**。 原则上，你可以把单词 '信号' 和 '插槽' 分别替换为 '事件' 和 '事件处理器'。 不过，由于信号可以在任意给定的时间发出，**所以这一概念放弃了 '事件' 的名字**。

```cpp
#include <boost/signals2.hpp>
#include <iostream>
void func()
{
  std::cout << "Hello, world!" << std::endl;
}
int main()
{  
  boost::signals2::signal<void ()> sig;
  sig.connect(func);
  sig();
  return 0;
}
```

boost/signals2 实际上**被实现为一个模板函数**，具有被用作为事件处理器的函数的签名，该签名也是它的**模板参数**。 在这个例子中，只有签名为 `void ()` 的函数**可以被成功关联至信号** `sig`。

函数 `func()` 被通过 `connect()` 方法关联至信号 ， 由于 `func()` 符合所要求的 `void ()` 签名，所以该关联成功建立，因此当信号被触发时，`func()` 将被调用。

`boost::signals2::signal` 的核心定义中，是一个**设计及其精良的观察者模式的实现**，他手动实现了C#的事件系统。

同一例子也可以用 Boost.Function 来实现。

```cpp
#include <boost/function.hpp> 
#include <iostream> 

void func() 
{ 
  std::cout << "Hello, world!" << std::endl; 
} 

int main() 
{ 
  boost::function<void ()> f; 
  f = func; 
  f(); 
}
```

和前一个例子相类似，`func()` 被关联至 `f`。 当 `f` 被调用时，就会相应地执行 `func()`。Boost.Function 仅限于这种情形下适用，而 **Boost.Signals 则提供了多得多的方式**，如**关联多个函数**至**单个特定信号**，示例如下。

```cpp
#include <boost/signals2.hpp>
#include <iostream>
void func1()
{
  std::cout << "Hello" << std::flush;
}
void func2()
{
  std::cout << ", world!" << std::endl;
}
int main()
{
  boost::signals2::signal<void ()> s;
  s.connect(func1);
  s.connect(func2);
  s();
}
```

可以通过反复调用 `connect()` 方法来把多个函数赋值给单个特定信号。 当该信号被触发时，这些函数被按照之前用 `connect()` **进行关联时的顺序来执行**。

要释放某个函数与给定信号的关联，可以用 `disconnect()` 方法。

```cpp
#include <boost/signal.hpp> 
#include <iostream> 

void func1() 
{ 
  std::cout << "Hello" << std::endl; 
} 

void func2() 
{ 
  std::cout << ", world!" << std::endl; 
} 

int main() 
{ 
  boost::signal<void ()> s; 
  s.connect(func1); 
  s.connect(func2); 
  s.disconnect(func2); 
  s(); 
}
```
除了 `connect()` 和 `disconnect()` 以外，`boost::signal` 还提供了几个方法。
```cpp
#include <boost/signals2.hpp>
#include <iostream>
void func1()
{
  std::cout << "Hello" << std::flush;
}
void func2()
{
  std::cout << ", world!" << std::endl;
}
int main()
{
  boost::signals2::signal<void ()> s;
  s.connect(func1);
  s.connect(func2);
  std::cout << s.num_slots() << std::endl;
  if (!s.empty())
    s();
  s.disconnect_all_slots();
}
```
`num_slots()` 返回已关联函数的数量。**如果没有函数被关联**，则 `num_slots()` 返回0。 在这种特定情况下，可以用 `empty()` 方法来替代。 `disconnect_all_slots()` 方法所做的实际上正是它的名字所表达的：释放所有已有的关联。

弄明白了信号被触发时会发生什么事之后，还有一个问题：**这些函数的返回值去了哪里**？ 以下例子回答了这个问题。

```cpp
#include <boost/signals2.hpp>
#include <iostream>
int func1()
{
  return 1;
}
int func2()
{
  return 2;
}
int main()
{
  boost::signals2::signal<int ()> s;
  s.connect(func1);
  s.connect(func2);
  std::cout << s() << std::endl;
}
```

`func1()` 和 `func2()` 都具有 `int` 类型的返回值。 `s` 将处理两个返回值，并**将它们都写出至标准输出流**。 那么，到底会发生什么呢？

以上例子**实际上会把 `2` 写出至标准输出流。 两个返回值都被 `s` 正确接收**，但除了最后一个值，其它值都会被忽略。 缺省情况下，**所有被关联函数中，实际上只有最后一个返回值被返回**。

你可以定制一个信号，令每个返回值都被相应地处理。 为此，要把一个称为**合成器(combiner)的东西**作为第二个参数传递给 `boost::signal`。

```cpp
#include <boost/signals2.hpp>
#include <iostream>
#include <algorithm>
int func1()
{
  return 1;
}
int func2()
{
  return 2;
}
template <typename T>
struct min_element
{
  typedef T result_type;
  template <typename InputIterator>
  T operator()(InputIterator first, InputIterator last) const
  {
    return *std::min_element(first, last);
  }
};
int main()
{
  boost::signals2::signal<int (), min_element<int> > s;
  s.connect(func1);
  s.connect(func2);
  std::cout << s() << std::endl;
}
```
合成器是一个重载了 `operator()()` 操作符的类。这个操作符会**被自动调用**，**传入两个迭代器**，指**向某个特定信号的所有返回值**。 以上例子使用了标准 C++ 算法 `std::min_element()` 来确定并返回最小的值。

除了对返回值进行分析以外，合成器也可以保存它们。
```cpp
#include <boost/signal.hpp> 
#include <iostream> 
#include <vector> 
#include <algorithm> 

int func1() 
{ 
  return 1; 
} 

int func2() 
{ 
  return 2; 
} 

template <typename T> 
struct min_element 
{ 
  typedef T result_type; 

  template <typename InputIterator> 
  T operator()(InputIterator first, InputIterator last) const 
  { 
    return T(first, last); 
  } 
}; 

int main() 
{ 
  boost::signal<int (), min_element<std::vector<int> > > s; 
  s.connect(func1); 
  s.connect(func2); 
  std::vector<int> v = s(); 
  std::cout << *std::min_element(v.begin(), v.end()) << std::endl; 
}
```

### 连接 Connections

函数可以通过由 `boost::signal` 所提供的 `connect()` 和 `disconnect()` 方法的帮助来进行管理。 由于 `connect()` 会返回一个类型为 `boost::signals::connection` 的值，它们可以通过其它方法来管理。
```cpp
#include <boost/signal.hpp> 
#include <iostream> 

void func() 
{ 
  std::cout << "Hello, world!" << std::endl; 
} 

int main() 
{ 
  boost::signal<void ()> s; 
  boost::signals::connection c = s.connect(func); 
  s(); 
  c.disconnect(); 
}
```

除了 `disconnect()` 方法之外，`boost::signals::connection` 还提供了其它方法，如 `block()` 和 `unblock()`。

```cpp
#include <boost/signal.hpp> 
#include <iostream> 

void func() 
{ 
  std::cout << "Hello, world!" << std::endl; 
} 

int main() 
{ 
  boost::signal<void ()> s; 
  boost::signals::connection c = s.connect(func); 
  c.block(); 
  s(); 
  c.unblock(); 
  s(); 
}
```
以上程序只会执行一次 `func()`。 虽然**信号 `s` 被触发了两次**，但是在第一次触发时 `func()` 不会被调用，因为连接 `c` 实际上**已经被 `block()` 调用所阻塞**。 由于在第二次触发之前调用了 `unblock()`，所以之后 `func()` 被正确地执行。

## 字符算法

Boost C++ 字符串算法库 [Boost.StringAlgorithms](http://www.boost.org/doc/libs/1_36_0/doc/html/string_algo.html) 提供了很多**字符串操作函数**。 字符串的类型可以是 `std::string`， `std::wstring` 或任何其他模板类 `std::basic_string` 的实例。

这些函数分类别在不同的头文件定义。 例如，大小写转换函数定义在文件 `boost/algorithm/string/case_conv.hpp` 中。 因为 Boost.StringAlgorithms 类中包括超过20个类别和相同数目的头文件， 为了方便起见，头文件 `boost/algorithm/string.hpp` 包括了所有其他的头文件。 后面所有例子都会使用这个头文件。

正如上节提到的那样， Boost.StringAlgorithms 库中许多函数 都可以接受一个类型为 `std::locale` 的对象作为附加参数。 而此参数是可选的，如果**不设置将使用默认全局区域设置**。

```cpp
#include <boost/algorithm/string.hpp> 
#include <locale> 
#include <iostream> 
#include <clocale> 

int main() 
{ 
  std::setlocale(LC_ALL, "German"); 
  std::string s = "Boris Schäling"; 
  std::cout << boost::algorithm::to_upper_copy(s) << std::endl; 
  std::cout << boost::algorithm::to_upper_copy(s, std::locale("German")) << std::endl; 
}
```

函数 `boost::algorithm::to_upper_copy()` 用于 转换一个字符串为**大写形式**，自然也有提供相反功能的函数 —— `boost::algorithm::to_lower_copy()` 把字符串转换为**小写形式**。 这两个函数都返回转换过的字符串作为结果。 如果作为参数传入的字符串自身需要被转换为大（小）写形式，可以使用函数 `boost::algorithm::to_upper()` 或 `boost::algorithm::to_lower ()`。

显然后者的转换是正确的， **因为小写字母 'ä' 对应的大写形式 'Ä' 是存在的**。 而在 C 区域设置中， **'ä' 是一个未知字符所以不能转换**。 为了能得到正确结果，必须明确传递正确的区域设置参数或者在调用 `boost::algorithm::to_upper_copy()` 之前**改变全局区域设置**。
```cpp
#include <boost/algorithm/string.hpp> 
#include <locale> 
#include <iostream> 

int main() 
{ 
  std::locale::global(std::locale("German")); 
  std::string s = "Boris Schäling"; 
  std::cout << boost::algorithm::to_upper_copy(s) << std::endl; 
  std::cout << boost::algorithm::to_upper_copy(s, std::locale("German")) << std::endl; 
}
```

## 多线程

本章将介绍C++ Boost库 [Boost.Thread](http://www.boost.org/libs/thread/)，它可以开发独立于**平台的多线程应用程序**。
在这个库最重要的一个类就是 `boost::thread`，它是在 `boost/thread.hpp` 里定义的，用来创建一个新线程。下面的示例来说明如何运用它。

```cpp
#include <boost/thread.hpp> 
#include <iostream> 

void wait(int seconds) 
{ 
  boost::this_thread::sleep(boost::posix_time::seconds(seconds)); 
} 

void thread() 
{ 
  for (int i = 0; i < 5; ++i) 
  { 
    wait(1); 
    std::cout << i << std::endl; 
  } 
} 

int main() 
{ 
  boost::thread t(thread); 
  t.join(); 
}
```
新建线程里**执行的那个函数的名称**被传递到 `boost::thread` 的构造函数。 一旦上述示例中的变量 `t` 被创建，该 `thread()` 函数就在其所在线程中被立即执行。 同时在 `main()` 里也**并发地执行**该 `thread()` 。

为了防止程序终止，就需要对**新建线程调用 `join()` 方法**。 `join()` 方法是一个**阻塞调用**：它可以暂停当前线程，直到调用 `join()` 的线程运行结束。 这就使得 `main()` 函数一直会等待到 `thread()` 运行结束。

在上述例子中，**使用一个循环把5个数字写入标准输出流**。 为了减缓输出，每一个循环中调用 `wait()` 函数让执行延迟了一秒。 `wait()` 可以调用一个名为 `sleep()` 的函数，这个函数也来自于 Boost.Thread，位于 `boost::this_thread` 名空间内。`sleep()` 要么在预计的一段时间或一个特定的时间点后时**才让线程继续执行**。 通过传递一个类型为 `boost::posix_time::seconds` 的对象，**在这个例子里我们指定了一段时间**。 `boost::posix_time::seconds` 来自于 Boost.DateTime 库，它被 Boost.Thread 用来**管理和处理时间的数据**。

下面的例子演示了如何**通过所谓的中断点**让一个线程中断。
```cpp
#include <boost/thread.hpp> 
#include <iostream> 

void wait(int seconds) 
{ 
  boost::this_thread::sleep(boost::posix_time::seconds(seconds)); 
} 

void thread() 
{ 
  try 
  { 
    for (int i = 0; i < 5; ++i) 
    { 
      wait(1); 
      std::cout << i << std::endl; 
    } 
  } 
  catch (boost::thread_interrupted&) 
  { 
  } 
} 

int main() 
{ 
  boost::thread t(thread); 
  wait(3); 
  t.interrupt(); 
  t.join(); 
}
```
在一个线程对象上调用 `interrupt()` 会**中断相应的线程**。 在这方面，中断意味着一个类型为 `boost::thread_interrupted` 的异常，**它会在这个线程中抛出**。 然后这只有在**线程达到中断点时才会发生**。
**如果给定的线程不包含任何中断点**，简单调用 `interrupt()` 就不会起作用。 每当一个线程中断点，它就会检查 `interrupt()` 是否被调用过。 只有被调用过了， `boost::thread_interrupted` 异常才会相应地抛出。
Boost.Thread定义了一系列的中断点，例如 `sleep()` 函数。 由于 `sleep()` 在这个例子里被调用了五次，**该线程就检查了五次它是否应该被中断**。 然而 `sleep()` 之间的调用，却不能使线程中断，Boost.Thread定义包括上述 `sleep()`函数十个中断， **有了这些中断点**，线程可以**很容易及时中断**。
然而，他们并不总是最佳的选择，因为**中断点必须事前读入以检查** `boost::thread_interrupted` 异常。
为了提供一个对 Boost.Thread 里提供的**多种函数的整体概述**，下面的例子将会再介绍两个。

```cpp
#include <boost/thread.hpp> 
#include <iostream> 

int main() 
{ 
  std::cout << boost::this_thread::get_id() << std::endl; 
  std::cout << boost::thread::hardware_concurrency() << std::endl; 
}
```
使用 `boost::this_thread`命名空间，能**提供独立的函数应用于当前线程**，比如前面出现的 `sleep()` 。 另一个是 `get_id()`：它会**返回一个当前线程的ID号**。 它也是由 `boost::thread` 提供的。

`boost::thread` 类提供了一个**静态方法** `hardware_concurrency()` ，它**能够返回基于CPU数目或者CPU内核数目的刻在同时在物理机器上运行的线程数**。 在常用的双核机器上调用这个方法，返回值为2。 这样的话就**可以确定在一个多核程序可以同时运行的理论最大线程数**。

### 同步

虽然**多线程的使用可以提高应用程序的性能**，但也增加了复杂性。 如果**使用线程在同一时间执行几个函数，访问共享资源时必须相应地同步**。 一旦应用达到了一定规模，这**涉及相当一些工作**。 本段介绍了Boost.Thread提供**同步线程**的类。
```cpp
#include <boost/thread.hpp> 
#include <iostream> 
void wait(int seconds) 
{ 
  boost::this_thread::sleep(boost::posix_time::seconds(seconds)); 
} 
boost::mutex mutex; 
void thread() 
{ 
  for (int i = 0; i < 5; ++i) 
  { 
    wait(1); 
    mutex.lock(); 
    std::cout << "Thread " << boost::this_thread::get_id() << ": " << i << std::endl; 
    mutex.unlock(); 
  } 
} 
int main() 
{ 
  boost::thread t1(thread); 
  boost::thread t2(thread); 
  t1.join(); 
  t2.join(); 
}
```
**`boost::mutex mutex;`**：定义一个全局互斥量，**作用**：建立一个“临界区”。当线程 t1 执行到 `lock()` 时，如果 t2 已经持有锁，t1 会在此处**阻塞**（挂起），直到 t2 调用 `unlock()`。**目的**：保护标准输出流 `std::cout`。在 C++ 中，`std::cout` 不是线程安全的，如果**不加锁**，两个线程的**输出可能会交织在一起**（例如：`ThrThreadead 12: : 01`）。

>[!tip] join调用的时候主线程会处于阻塞状态，直到子线程执行完毕，如果要实现的是后台化一个线程，比如说后台线程用于计算任务，可以用`detach()`：真正的“守护/后台”化，**风险**：在 C++ 中，`detach` 非常危险。如果**主线程结束导致进程退出**，而子线程还在**读写资源**，会导致**非法访问或资源泄露**。

上面的示例使用一个类型为 `boost::mutex` 的 `mutex` 全局互斥对象。 `thread()` 函数获取此对象的**所有权**才在 `for` 循环内使用 `lock()` 方法**写入到标准输出流的**。 一旦信息被写入，使用 `unlock()` 方法**释放所有权**，由于两个线程试图在写入标准输出流前获得互斥体，实际上只能保证一次只有一个线程访问 `std::cout`。 不管哪个线程成功调用 `lock()` 方法，其他所有线程必须等待，直到 `unlock()` 被调用。


我们可以使用boost::lock_guard，在其内部的构造和析构函数分别自动调用 `lock()` 和 `unlock()` 。 访问**共享资源是需要同步的**，因为它**显示地被两个方法调用**。 `boost::lock_guard` 类是另一个出现在 [第 2 章 _智能指针_](https://wizardforcel.gitbooks.io/the-boost-cpp-libraries/content/smartpointers.html "第 2 章 智能指针") 的RAII用语。
```cpp
#include <boost/thread.hpp> 
#include <iostream> 

void wait(int seconds) 
{ 
  boost::this_thread::sleep(boost::posix_time::seconds(seconds)); 
} 

boost::mutex mutex; 

void thread() 
{ 
  for (int i = 0; i < 5; ++i) 
  { 
    wait(1); 
    boost::lock_guard<boost::mutex> lock(mutex); 
    std::cout << "Thread " << boost::this_thread::get_id() << ": " << i << std::endl; 
  } 
} 

int main() 
{ 
  boost::thread t1(thread); 
  boost::thread t2(thread); 
  t1.join(); 
  t2.join(); 
}
```

![[Pasted image 20251225171719.png]]

除了`boost::mutex` 和 `boost::lock_guard` 之外，Boost.Thread也提供**其他的类支持各种同步**。 其中一个重要的就是 `boost::unique_lock` ，相比较 `boost::lock_guard` 而言，它**提供许多有用的方法**。
```cpp
#include <boost/thread.hpp> 
#include <iostream> 

void wait(int seconds) 
{ 
  boost::this_thread::sleep(boost::posix_time::seconds(seconds)); 
} 

boost::timed_mutex mutex; 

void thread() 
{ 
  for (int i = 0; i < 5; ++i) 
  { 
    wait(1); 
    boost::unique_lock<boost::timed_mutex> lock(mutex, boost::try_to_lock); 
    if (!lock.owns_lock()) 
      lock.timed_lock(boost::get_system_time() + boost::posix_time::seconds(1)); 
    std::cout << "Thread " << boost::this_thread::get_id() << ": " << i << std::endl; 
    boost::timed_mutex *m = lock.release(); 
    m->unlock(); 
  } 
} 

int main() 
{ 
  boost::thread t1(thread); 
  boost::thread t2(thread); 
  t1.join(); 
  t2.join(); 
}
```
`boost::unique_lock` 通过**多个构造函数来提供不同的方式获得互斥体**。 这个期望获得互斥体的函数简单地调用了 `lock()` 方法，一直等到获得这个互斥体。 所以它的行为跟 `boost::lock_guard` 的那个是一样的。

普通的 `mutex` 只有“等”和“不等”两种状态，而 `timed_mutex` 增加了一个“等多久”的选项，### 核心区别：从**“死等”到“限时等”**

| **特性**      | **boost::mutex**          | **boost::timed_mutex**                    |
| ----------- | ------------------------- | ----------------------------------------- |
| **锁定行为**    | `lock()`：如果锁被占用，线程会无限期挂起。 | 支持 `try_lock_for()` 和 `try_lock_until()`。 |
| **返回值**     | 无（或者阻塞直到成功）。              | 返回 `bool`（成功为 true，超时为 false）。            |
| **典型风险**    | 容易导致**永久死锁**。             | 可以通过超时逻辑**打破死锁**。                         |
| **.NET 类比** | `lock(obj) { ... }`       | `Monitor.TryEnter(obj, TimeSpan)`         |
上面的程序向 `boost::unique_lock` 的**构造函数**的第二个参数传入`boost::try_to_lock`。 然后通过 `owns_lock()` 可以**检查是否可获得互斥体**。 如果不能， `owns_lock()` 返回 `false`。 这也用到 `boost::unique_lock` 提供的另外一个函数： `timed_lock()` 等待一定的时间以**获得互斥体**。 **给定的程序等待长达1秒**，应较足够的时间来**获取更多的互斥**。

### 线程本地存储

线程本地存储（TLS）是一个**只能由一个线程访问的专门的存储区域**。 TLS的变量可以被看作是一个只对某个特定线程而非整个程序可见的全局变量。 下面的例子显示了这些变量的好处。
```cpp
#include <boost/thread.hpp> 
#include <iostream> 
#include <cstdlib> 
#include <ctime> 

void init_number_generator() 
{ 
  static bool done = false; 
  if (!done) 
  { 
    done = true; 
    std::srand(static_cast<unsigned int>(std::time(0))); 
  } 
} 

boost::mutex mutex; 

void random_number_generator() 
{ 
  init_number_generator(); 
  int i = std::rand(); 
  boost::lock_guard<boost::mutex> lock(mutex); 
  std::cout << i << std::endl; 
} 

int main() 
{ 
  boost::thread t[3]; 

  for (int i = 0; i < 3; ++i) 
    t[i] = boost::thread(random_number_generator); 

  for (int i = 0; i < 3; ++i) 
    t[i].join(); 
}
```

## 异步输入和输出

使用 Boost.Asio **进行异步数据处理的应用程序基于两个概念**：I/O 服务和 I/O 对象。 I/O 服务**抽象了操作系统的接口**，允许**第一时间进行异步数据处理**，而 I/O 对象则用于**初始化特定的操作**。 鉴于 Boost.Asio 只提供了一个名为 `boost::asio::io_service` 的类作为 I/O 服务，它针对所支持的每一个操作系统都分别实现了优化的类，**另外库中还包含了针对不同 I/O 对象的几个类**。 其中，类 `boost::asio::ip::tcp::socket` 用于通过**网络发送和接收数据**，而类 `boost::asio::deadline_timer` 则提供了一个**计时器**，用于**测量某个固定时间点到来或是一段指定的时长过去了**。 以下第一个例子中就使用了计时器，因为与 Asio 所提供的其它 I/O 对象相比较而言，它不需要任何有关于网络编程的知识。

```cpp
#include <boost/asio.hpp>
#include <iostream>
void handler(const boost::system::error_code &ec)
{
  std::cout << "5 s." << std::endl;
}
int main()
{
  boost::asio::io_context io_context;
  boost::asio::steady_timer timer(io_context, std::chrono::seconds(5));
  timer.async_wait(handler);
  io_context.run();
}
```
对于拥有 .NET 经验的开发者，这段代码的逻辑可以用一个词总结：**Event Loop（事件循环）**。它在底层非常类似于 **WPF/WinForms 的 UI 消息泵**，或者 Node.js 的事件模型。

我们定义了一个异步任务的中心boost::asio::io_context io_context;它负责**与操作系统打交道**（处理排队、通知等），并决定什么时候该调用你的 `handler`，**C# 类比**：类似于 `SynchronizationContext` 或隐藏在 `async/await` 背后的 `TaskScheduler`。

boost::asio::steady_timer timer(...)是一个定时器，可以**将其绑定**到io_context上，用参数设置5s后触发，timer.async_wait(handler)就是**非阻塞调用**，**C# 类比**：类似于 `Task.Delay(5000).ContinueWith(t => handler())`，但它是基于回调（Callback）的。

而io_context.run则是**阻塞调用**，程序会告诉**主线程处理异步任务直到所有任务处理完成再返回**。

‘

### 可扩展性与多线程

用 **Boost.Asio 这样的库来开发应用程序**，与一般的 C++ 风格不同。 那些可能**需要较长时间才返回的函数不再是以顺序的方式来调用**。 不再是**调用阻塞式的函数**，Boost.Asio 是启动一个异步操作。 而那些**需要在操作结束后调用的函数则实现为相应的句柄**。 这种方法的缺点是，本来顺序执行的功能变得在物理上分割开来了，从而令相应的代码更难理解。

可扩展性是指，**一个应用程序从新增资源有效地获得好处的能力**。 如果那些**执行时间较长的操作不应该阻塞其它操作的话**，那么建议使用 Boost.Asio. 由于现今的**PC机通常都具有多核处理器**，所以线程的应用可以进一步提高一个基于 Boost.Asio 的应用程序的可扩展性。

我们可以定义一个io_context，但是启动两个线程来运行，这个在异步编程中叫做**IO线程池模型**，当我们调用io_context.run的时候，线程就会进入一个循环，询问io_context有无任务需要处理。

- **任务分发**：现在你有两个定时器任务（`timer1` 和 `timer2`）。5 秒钟后，这两个任务都会变为“就绪”状态。
    
- **并发执行**：
    
    - `thread1` 可能会抓取到 `handler1` 并**开始执行**。
        
    - 同时，`thread2` 可能会抓取到 `handler2` 并**开始执行**。
        
- **结果**：这两个 "5 s." 的打印动作**可能是在不同的 CPU 核心上同时发生的**。

这种方法的优点很明显，可以负载均衡，**如果handler1里包含着耗时的计算任务**，thread1会被占用，**但是thread2依然可以处理后续的定时器和网络数据**，这样可以提高多核CPU的性能，不需要手动表写复杂的线程切换代码，可以提高吞吐量。

这种模型在C#中非常类似于Task.WhenAll配合自定义的TaskScheduler。

### make_work_guard

```cpp
boost::asio::make_work_guard
```
它为 `io_context` 提供了一种“人造”的挂起任务，**使得即使当前没有任何实际的异步操作**（如 Socket 读写或定时器），`io_context` 的**事件循环**也会保持运行状态。
![[Pasted image 20251231021048.png]]



`make_work_guard` 是一个非常关键的工具，它的核心作用是**防止 `io_context::run()` 函数在任务执行完之前退出** ， 要理解它，我们就得理解io_context的生命周期，我们使用了io_context.run()的时候会去**阻塞当前的线程并且开始处理异步操作的回调**。



### 网络编程

虽然 Boost.Asio 是一个可以异步处理任何种类数据的库，但是它主要被用于网络编程。 这是由于，**事实上 Boost.Asio 在加入其它 I/O 对象之前很久就已经支持网络功能了**。 网络功能是异步处理的一个很好的例子，因为**通过网络进行数据传输可能会需要较长时间，从而不能直接获得确认或错误条件**。

可以试着去写一个完整的异步网络客户端：

```cpp
#include <boost/asio.hpp>
#include <boost/array.hpp>
#include <iostream>
#include <string>
boost::asio::io_context io_context;
boost::asio::ip::tcp::resolver resolver(io_context);
boost::asio::ip::tcp::socket sock(io_context);
boost::array<char, 4096> buffer;
void read_handler(const boost::system::error_code &ec, std::size_t bytes_transferred)
{
  if (!ec)
  {
    std::cout << std::string(buffer.data(), bytes_transferred) << std::endl;
    sock.async_read_some(boost::asio::buffer(buffer), read_handler);
  }
}
void connect_handler(const boost::system::error_code &ec)
{
  if (!ec)
  {
    boost::asio::write(sock, boost::asio::buffer("GET / HTTP 1.1\r\nHost: highscore.de\r\n\r\n"));
    sock.async_read_some(boost::asio::buffer(buffer), read_handler);
  }
}
void resolve_handler(const boost::system::error_code &ec, boost::asio::ip::tcp::resolver::results_type endpoints)
{
  if (!ec)
  {
    sock.async_connect(*endpoints.begin(), connect_handler);
  }
}
int main()
{
  resolver.async_resolve("www.highscore.de", "80", resolve_handler);
  io_context.run();
}
```
这里我们的思想非常简单，将io_context io_context当做异步操作的引擎，用来直到操作系统底层的事件，比如说解析完成、数据的到达和分发给对应的处理函数。
**tcp::resolver resolver则是负责将域名转换为IP地址**，因为DNS很可能慢，因此必须为异步。
tcp::socket sock则是代表了**与服务器之间的连接**，所有的**发送和接受都必须通过它完成**。
我们使用了一个array数组来当做缓冲区，暂存读取到的字节数据。

这里使用了**同步**写 `boost::asio::write`，因为它通常很快。它把 **HTTP 请求发送给服务器**，然后使用async_read_some来**阻塞主线程并等待dns的运行成果**...，使用的是回调模式。

我们可以用C++ 20 的**协程**将这套嵌套的**回调改写为顺序执行的代码**，这样就可以达到和C#的await一样的效果。
```cpp
#include <boost/asio.hpp>
#include <boost/asio/awaitable.hpp>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/use_awaitable.hpp>
#include <boost/asio/this_coro.hpp>
#include <boost/array.hpp>
#include <iostream>
#include <string>
#include <exception>
namespace asio = boost::asio;
asio::awaitable<void> read_loop(asio::ip::tcp::socket& sock, boost::array<char, 4096>& buffer)
{
    try
    {
        while (true)
        {
            std::size_t bytes_transferred = co_await sock.async_read_some(
                asio::buffer(buffer), asio::use_awaitable);
            std::cout << std::string(buffer.data(), bytes_transferred) << std::endl;
        }
    }
    catch (const boost::system::system_error& e)
    {
        if (e.code() != asio::error::eof)
        {
            std::cerr << "Read error: " << e.what() << std::endl;
        }
    }
}
asio::awaitable<void> http_client()
{
    try
    {
        auto executor = co_await asio::this_coro::executor;
        asio::ip::tcp::resolver resolver(executor);
        asio::ip::tcp::socket sock(executor);
        boost::array<char, 4096> buffer;
        // 解析域名
        auto endpoints = co_await resolver.async_resolve(
            "www.highscore.de", "80", asio::use_awaitable);

        // 连接服务器
        co_await sock.async_connect(*endpoints.begin(), asio::use_awaitable);

        // 发送 HTTP 请求
        std::string request = "GET / HTTP/1.1\r\nHost: www.highscore.de\r\nConnection: close\r\n\r\n";
        co_await asio::async_write(sock, asio::buffer(request), asio::use_awaitable);

        // 读取响应
        co_await read_loop(sock, buffer);
    }
    catch (const std::exception& e)
    {
        std::cerr << "Exception: " << e.what() << std::endl;

    }

}

int main()
{
    asio::io_context io_context;
    asio::co_spawn(io_context, http_client(), [](std::exception_ptr e)
    {
        if (e)
        {
            try
            {
                std::rethrow_exception(e);
            }
            catch (const std::exception& ex)
            {
                std::cerr << "Unhandled exception: " << ex.what() << std::endl;
            }
        }
    });
    io_context.run();
    return 0;
}
```
这里的[[asio::awaitable<void>]]就和C#的Task一样的，我们可以使用co_await（类似于await），然后asio::co_spawn类似于Task.Run()的方法来启动异步任务，或者上下文的方法则是asio::this_coro::executor，类似于同步上下文：SynchronizationContext.Current。


## 进程间通信

进程间通讯描述的是**同一台计算机的不同应用程序之间的数据交换机制**。 但不包括网络通讯方式。 如果需要经由网络，在彼此运行在不同计算机上的应用程序之间交换数据，请看[第 7 章 _异步输入输出_](https://wizardforcel.gitbooks.io/the-boost-cpp-libraries/content/asio.html "第 7 章 异步输入输出")，该章讲述了 Boost.Asio 库。

虽然 Boost.Asio 也可以用来在同一台计算机的应用程序间交换数据，但是使用 Boost.Interprocess 库通常性能更好。 Boost.Interprocess 库实际上是使用操作系统的功能优化了同一台计算机的应用程序之间数据交换，所以它应该是任何不需要网络时应用程序间数据交换的首选。

### 共享内存

共享内存通常是**进程间通讯最快的形式**。 它提供一块在应用程序间共享的内存区域。 一个应用能够在另一个应用读取数据时写数据。
这样一块内存区用 Boost.Interprocess 的 `boost::interprocess::shared_memory_object` 类表示。 为使用这个类，需要包含 `boost/interprocess/shared_memory_object.hpp` 头文件。

```cpp
#include <boost/interprocess/shared_memory_object.hpp> 
#include <iostream> 

int main() 
{ 
  boost::interprocess::shared_memory_object shdmem(boost::interprocess::open_or_create, "Highscore", boost::interprocess::read_write); 
  shdmem.truncate(1024); 
  std::cout << shdmem.get_name() << std::endl; 
  boost::interprocess::offset_t size; 
  if (shdmem.get_size(size)) 
    std::cout << size << std::endl; 
}
```
`boost::interprocess::shared_memory_object` 的构造函数需要三个参数。 第一个参数指定共享内存是要创建或打开。 上面的例子实际上是指定了两种方式：用 `boost::interprocess::open_or_create` 作为**参数**，共享内存如果存在就将其打开，否则创建之。

假设之前已经创建了共享内存，现**打开前面已经创建的共享内存**。 **为了唯一标识一块共享内存，需要为其指定一个名称**，传递给 `boost::interprocess::shared_memory_object` 构造函数的第二个参数指定了这个名称。

第三个，也就是最后一个参数**指示应用程序如何访问共享内存**。 例子**应用程序能够读写共享内存**，这是因为最后的一个参数是 `boost::interprocess::read_write`。

在创建一个 `boost::interprocess::shared_memory_object` 类型的对象后，**相应的共享内存就在操作系统中建立了**。 可是此共享内存区域的大小被初始化为0.为了使用这块区域，需要调用 `truncate()` 函数，**以字节为单位传递请求的共享内存的大小**。 对于上面的例子，共享内存提供了1,024字节的空间。

请注意，`truncate()` 函数只能在共享内存以 `boost::interprocess::read_write` 方式打开时调用。 **如果不是以此方式打开**，将抛出 `boost::interprocess::interprocess_exception` 异常，为了调整共享内存的大小，`truncate()` 函数可以被重复调用。

创建共享内存后，`get_name()` 和 `get_size()` 函数可以分别用来查询共享内存的名称和大小。
由于共享内存被用于应用程序之间交换数据，**所以每个应用程序需要映射共享内存到自己的地址空间上**，这是通过 `boost::interprocess::mapped_region` 类实现的。

下面的例子使用共享内存写入并读取一个数字。
```cpp
#include <boost/interprocess/shared_memory_object.hpp> 
#include <boost/interprocess/mapped_region.hpp> 
#include <iostream> 

int main() 
{ 
  boost::interprocess::shared_memory_object shdmem(boost::interprocess::open_or_create, "Highscore", boost::interprocess::read_write); 
  shdmem.truncate(1024); 
  boost::interprocess::mapped_region region(shdmem, boost::interprocess::read_write); 
  int *i1 = static_cast<int*>(region.get_address()); 
  *i1 = 99; 
  boost::interprocess::mapped_region region2(shdmem, boost::interprocess::read_only); 
  int *i2 = static_cast<int*>(region2.get_address()); 
  std::cout << *i2 << std::endl; 
}
```

通常，不会在同一个应用程序内使用多个 `boost::interprocess::mapped_region` 访问同一块共享内存。 **实际上在同一个应用程序内将同一个共享内存映射到不同的内存区域上没有多大的意义**，上面的例子只用于说明的目的。

为了删除指定的共享内存，`boost::interprocess::shared_memory_object` 对象提供了静态的 `remove()` 函数，此函数带有一个要被删除的共享内存名称的参数。

Boost.Interprocess 类的RAII概念支持，明显来自关于智能指针的章节，并使用了另外的一个类名称 `boost::interprocess::remove_shared_memory_on_destroy`。 它的构造函数需要一个已经存在的共享内存的名称。 **如果这个类的对象被销毁了，那么在析构函数中会自动删除共享内存的容器**。
```cpp
#include <boost/interprocess/shared_memory_object.hpp> 
#include <iostream> 

int main() 
{ 
  bool removed = boost::interprocess::shared_memory_object::remove("Highscore"); 
  std::cout << removed << std::endl; 
}
```

如果 `remove()` 没有被调用, 那么，即使进程终止，共享内存还会一直存在，而不论共享内存的删除是否依赖底层操作系统。 多数Unix操作系统，包括Linux，一旦系统重新启动，都会自动删除共享内存，在 Windows 或 Mac OS X上，`remove()` 必须调用，**这两种系统实际上将共享内存存储在持久化的文件上，此文件在系统重启后还是存在的。**


Windows 提供了一种特别的共享内存，**它可以在最后一个使用它的应用程序终止后自动删除**。 为了使用它，提供了 `boost::interprocess::windows_shared_memory` 类，定义在 `boost/interprocess/windows_shared_memory.hpp` 文件中。
```cpp
#include <boost/interprocess/windows_shared_memory.hpp> 
#include <boost/interprocess/mapped_region.hpp> 
#include <iostream> 

int main() 
{ 
  boost::interprocess::windows_shared_memory shdmem(boost::interprocess::open_or_create, "Highscore", boost::interprocess::read_write, 1024); 
  boost::interprocess::mapped_region region(shdmem, boost::interprocess::read_write); 
  int *i1 = static_cast<int*>(region.get_address()); 
  *i1 = 99; 
  boost::interprocess::mapped_region region2(shdmem, boost::interprocess::read_only); 
  int *i2 = static_cast<int*>(region2.get_address()); 
  std::cout << *i2 << std::endl; 
}
```


### 托管共享内存

Boost.Interprocess 提供了一个名为“托管共享内存”的概念，通过定义在 `boost/interprocess/managed_shared_memory.hpp` 文件中的 `boost::interprocess::managed_shared_memory` 类提供。 这个类的目的是，**对于需要分配到共享内存上的对象**，它能够**以内存申请的方式初始化**，并使其自动为使用同一个共享内存的其他应用程序可用。
```cpp
#include <boost/interprocess/managed_shared_memory.hpp> 
#include <iostream> 

int main() 
{ 
  boost::interprocess::shared_memory_object::remove("Highscore"); 
  boost::interprocess::managed_shared_memory managed_shm(boost::interprocess::open_or_create, "Highscore", 1024); 
  int *i = managed_shm.construct<int>("Integer")(99); 
  std::cout << *i << std::endl; 
  std::pair<int*, std::size_t> p = managed_shm.find<int>("Integer"); 
  if (p.first) 
    std::cout << *p.first << std::endl; 
}
```
在常规的共享内存中，为了读写数据，**单个字节被直接访问**，托管共享内存使用诸如 `construct()` 函数，**此函数要求一个数据类型作为模板参数**，此例中声明的是 `int` 类型，函数缺省**要求一个名称来表示在共享内存中创建的对象**。 此例中使用的名称是 "Integer"。

为了访问托管共享内存上的一个特定对象，用 `find()` 函数。 通过传递要查找对象的名称，返回或者是一个指向这个特定对象的指针，或者是0表示给定名称的对象没有找到。正如前面例子中所见，`find()` 实际返回的是 `std::pair` 类型的对象，`first` 属性提**供的是指向对象的指针**，那么 `second` 属性提供的是什么呢？
```cpp
#include <boost/interprocess/managed_shared_memory.hpp> 
#include <iostream> 

int main() 
{ 
  boost::interprocess::shared_memory_object::remove("Highscore"); 
  boost::interprocess::managed_shared_memory managed_shm(boost::interprocess::open_or_create, "Highscore", 1024); 
  int *i = managed_shm.construct<int>("Integer")[10](99); 
  // 10长度的int数组全部初始化为99
  std::cout << *i << std::endl; 
  std::pair<int*, std::size_t> p = managed_shm.find<int>("Integer"); 
  if (p.first) 
  { 
    std::cout << *p.first << std::endl; 
    std::cout << p.second << std::endl; 
  } 
}
```
第二个返回的是10，它返回的是**元素的个数**（在例子中是 10），为什么 `managed_shared_memory` 能记住名字和长度？因为它在映射的 1024 字节头部维护了一套管理结构：
- **Segment Manager**：位于内存块的最前端，负责管理剩余空间。
    
- **Index Table**：存储了 `"Integer"` 这个字符串与**实际物理偏移地址的映射关系**。
    
- **Metadata**：记录了每个分配块的大小（这就是为什么 `p.second` 能返回 10）。


### 同步

Boost.Interprocess 允许**多个应用程序并发使用共享内存**。 由于共享内存被定义为在应用程序之间“共享”，所以 Boost.Interprocess 需要支持一些同步方式。

正如在 [第 6 章 _多线程_](https://wizardforcel.gitbooks.io/the-boost-cpp-libraries/content/multithreading.html "第 6 章 多线程") 所见，Boost.Thread 确实提供了各种概念，如**互斥对象和条件变量来同步线程**。 可惜的是，这些类只能用来同步同一个应用程序内的**线程**，它们不支持同步不同的应用程序。 由于二者面临的问题相同，所以在概念上没有什么差别。

Boost.Interprocess 提供了两种同步对象，**匿名对象被直接存储在共享内存上**，这使得他们自动对所有应用程序可用。 **命名对象由操作系统管理**，所以它们**不存储在共享内存上**，它们**可以被应用程序通过名称访问**。

```cpp
#include <boost/interprocess/managed_shared_memory.hpp> 
#include <boost/interprocess/sync/named_mutex.hpp> 
#include <iostream> 

int main() 
{ 
  boost::interprocess::managed_shared_memory managed_shm(boost::interprocess::open_or_create, "shm", 1024); 
  int *i = managed_shm.find_or_construct<int>("Integer")(); 
  boost::interprocess::named_mutex named_mtx(boost::interprocess::open_or_create, "mtx"); 
  named_mtx.lock(); 
  ++(*i); 
  std::cout << *i << std::endl; 
  named_mtx.unlock(); 
}
```
我们用`boost::interprocess::named_mutex` 创建并使用一个命名互斥对象，除了一个参数用来指定互斥对象是被**创建或者打开之外**，`boost::interprocess::named_mutex` 的构造函数还需要一个名称参数。 每个知道**此名称的应用程序能够访问这同一个对象**。

由于**互斥对象**在任意时刻**只能被一个应用程序拥有**，其他应用程序需要等待，**直到互斥对象被第一个应用程序使用 `lock()` 函数释放**。 一旦应用程序获得互斥对象的所有权，它可以获得互斥对象保护的资源的排他访问。 在上面例子中，资源是`int`类的变量被递增并写到标准输出流中。

