## c++ 与 c#在类上的不同

## 兼容性

有时候我们编写头文件的时候，我们准备让代码有更好的兼容性，比如说库文件，我们可能会遇到一些宏，类似于如下：

```cpp
#ifdef __cplusplus
extern "C" {
#endif 
```

它们的作用就是确保cpp的编译器在处理戴拿的时候按照c语言的方式来处理函数名，因为C语言是不支持函数重载的，C编译器眼中，函数 `void print(int)` 的符号名就是 `print`。C++ 支持函数重载。为了区分 `print(int)` 和 `print(double)`，C++ 编译器会偷偷修改函数名（这被称为 **Name Mangling**）。

```
print(int)` 可能会变成 `_Z5printi
print(double)` 可能会变成 `_Z5printd
```

如果我们在一个c++中包含了一个使用C编译的库，c++的链接器会去寻找`_Z5printi`，但 C 库里只有 `print`。结果就是：**链接错误（LNK2019）**，提示找不到符号。

如果我们用了__cplusplus这个编译器预定义的宏，那么**c++编译器**编译它的时候，它就存在，如果是用的**C编译器**，那么它就不存在。

上面的exern "C":{} 就是告诉c++的编译器不使用name mangling，直接使用它们原本的名称。最经典的使用场景会包在整个头文件的内容外面：

```cpp
#pragma once

#ifdef __cplusplus
extern "C" {
#endif

// 这里写你的 C 风格函数定义
void Logger_Log(const char* message);
int Calculate_Sum(int a, int b);

#ifdef __cplusplus
}
#endif
```



### 拷贝构造和移动构造

拷贝是对象被当作值传递或者A a = b的时候触发，移动构造是c++11引入的，用于转移资源所有权，就比如说一个从临时对象拿走内存，来避免昂贵的内存拷贝。

在CPP中，是否发生了拷贝是由参数传递方式决定的，比如说究竟是值传递还是引用传递。

```cpp
void Func(const std::string s) { ... }
```

这种就属于按值传递，编译器回去拷贝一份字符串，const作用就是函数内部，不能修改整个拷贝出来的副本，这种方式少见而且不推荐，因为这种方式限制了修改副本的同时还花费了拷贝开销，很烂。

我们可以按常量传递，使用类似 const T& x这种方式：

```cpp
void Func(const std::string& s) { ... }
```

这种不会发生拷贝，函数会直接指向数据的内存地址，这里是左值引用，const的作用是确保函数内部智能读取原数据而不能修改，这个是最常用的，避免了拷贝，还可以保证原数据的安全，是只读的。类似于c#的in关键字。

还有一种是普通引用传递T& x{}：

```cpp
void Func(std::string& s) { ... }
```

这种不会发生拷贝，并且可以随意的修改原数据，修改会直接反应在调用者那里，类似于ref关键字，这个就是我们常说的这个函数会产生副作用



### 静态类的处理

C#中，static class意味着类不能被实例化，且所有成员必须是静态的。 在 **C++** 中：

**类本身不能标记为 `static`**。`static` 关键字在 C++ 中修饰变量或函数时，表示“内部链接”或“类共有成员”，但不能直接修饰 `class` 定义。

如果你想要实现类似 C# 静态类的效果，C++ 通常有两种做法：**使用 `namespace`** 或 **类内全静态成员**。

就比如：

```cpp
// Logger.h
#pragma once
#include <iostream>
#include <string>

enum class LoggerLevel { // 建议使用 enum class 避免命名污染
    Info,
    Debug,
    Warning,
    Error
};

namespace Logger {
    // 声明函数
    void Log(LoggerLevel level, const std::string& message);
}
```

顺带一提， enum class是c++11引入的强类型枚举，也叫有作用域枚举，**C++ 的 `enum class` 在行为上几乎等同于 C# 的 `enum`**。而 C++ 传统的 `enum`（不带 `class` 的）反而更像是一个“披着名字的整数”。

普通的enum中，枚举值是之际额暴露在外部，在enum class中，必须通过类名来引用，因此我们才叫做它可以避免命名污染。



## C++的构造函数



在 C++ 中，构造函数是用于初始化类对象的特殊成员函数。它的名称与类名完全相同，没有返回类型（连 `void` 都没有）。

```cpp
class Logger {
private:
    int _id;
    std::string _name;

public:
    // 1. 默认构造函数 (Default Constructor)
    Logger() {
        _id = 0;
        _name = "Default";
    }

    // 2. 有参构造函数 (Parameterized Constructor)
    Logger(int id, std::string name) {
        _id = id;
        _name = name;
    }
};
```



### 委托构造

如果你有多个构造函数，且它们有重复的逻辑，可以让一个构造函数调用另一个。

```cpp
class Device {
public:
    Device(int id, std::string type) {
        // 复杂的初始化逻辑
    }

    // 调用上面的构造函数
    Device(int id) : Device(id, "Unknown") { } 
};
```

### default和delete

如果你定义了有参构造函数，编译器就不会自动生成默认构造函数。你可以手动强制它生成，我们需要使用deafult关键字。如果你不希望对象被实例化（比如写单例或静态工具类），可以禁用构造函数，我们使用delete。

```cpp
class Utils {
public:
    Utils() = delete; // 禁止创建实例
};

class Data {
public:
    Data() = default; // 显式要求编译器生成默认构造函数
    Data(int x) { }
};
```

### 防止隐式转换

在 C++ 中，如果**构造函数只有一个参数**，编译器会尝试进行隐式类型转换，这有时会导致诡异的 Bug，我们可以使用explicit关键字：

```cpp
class Buffer {
public:
    explicit Buffer(int size) { ... }
};

// 使用：
Buffer b1(10);    // OK
// Buffer b2 = 10; // 报错！因为加了 explicit，禁止将 int 直接“变成” Buffer
```



### 初始化

在 C++ 中，上面的赋值写法（在 `{}` 内部赋值）实际上是“先创建变量再赋值”。**更高效、更推荐**的做法是使用“初始化列表”。

**语法：** `Constructor() : member1(val1), member2(val2) { ... }`

```cpp
class Camera {
private:
    const int _camId; // 常量必须在初始化列表中初始化
    int _width;
    int _height;

public:
    // 推荐写法：在进入构造函数体之前完成初始化
    Camera(int id, int w, int h) 
        : _camId(id), _width(w), _height(h) 
    {
        // 函数体通常为空，或者只写一些逻辑校验
    }
};
```



```cpp
class MyVector {
public:
    // 拷贝构造：参数必须是常量引用
    MyVector(const MyVector& other) {
        // 执行深拷贝
    }

    // 移动构造：参数是右值引用
    MyVector(MyVector&& other) noexcept {
        // 转移指针的所有权
    }
};
```

noexcept关键字是 c++11引入的，用来告诉编译器和开发者，这个函数保证是不会抛出异常的，c#的方法默认也是不声明异常的，不强制使用try-catch语法，但是cpp的noexcept不仅仅是一个文档说明，对性能和稳定性也有很重要的影响。先来说说性能优化的地方：

当 C++ 的标准库容器（比如 `std::vector`）需要扩容时，它会将**旧内存中的对象搬到新内存**中，如果说对象是的移动构造函数是noexcept的话，std::vector就会使用高效的移动操作，如果不是noexcept的话，编译器就会保证移动过程中报错了**原数据就能不丢失**，会保守选择拷贝方式。

可能我们比较奇怪的一点，为什么拷贝构造的形参是一个只读左值引用，但是却叫做拷贝。拷贝构造描述的其实是函数的功能，只读的左值引用是实现的最高效、安全的手段：

### 递归死循环

这个是技术上最根本的原因，假设不使用引用的话，而是真的使用值传递的话：

```cpp
class MyClass {
public:
    // 假设我们不写引用 &
    MyClass(MyClass other) { 
        // ... 实现拷贝逻辑
    }
};
```

当你执行 `MyClass a; MyClass b = a;` 时：编译器发现我们**需要使用b的拷贝构造函数**，为了调用这个函数我们必须先把实参a传递给形参other，那么这个时候按值传递就意味着要把a拷贝一份给other，**然后为了把a拷贝给other，编译器就要调用MyClass的拷贝构造函数**，为了调用这个拷贝构造函数，**又需要再次拷贝实参**。

编译器就会报错！

或者我们可以从结果来说，拷贝构造定义的是函数执行之后的结果，内存里多了个和原对象一模一样的新对象。

接下来说说移动构造，为什么必须是右值引用：

移动构造本质是资源所有权的转移，我们可以脱离拷贝，直接将buffer指针地址拿来赋值给自己，然后将原来的指针设置为nullptr，为什么必须是右值，因为左值是一个持久的对象，它是有名字有地址的对象，如果你把一个正在使用的变量之际额给拿走，原程序就会崩溃，右值由于是临时对象，没有名字会马上销毁，那么它占用的内存资源就可以拿来利用，这个对原程序没有任何影响。

如果说**没有右值引用**，那么编译器就没法在语法层面上面区分是想要拷贝还是移动：

```cpp
class Logger {
public:
    // 拷贝构造：我承诺不改你，所以我用 const &
    Logger(const Logger& other); 

    // 移动构造：我需要改你（把你置空），所以我不能用 const
    // 我需要专门处理临时变量，所以用 &&
    Logger(Logger&& other) noexcept; 
};
```



## 访问符号的区别

C++区别于C#，他把权限分的很细，我们使用C#的时候一直都是使用.来访问成员，但是CPP不一样。 . 这样的点运算符是对象成员访问，用于直接操作对象实例，左侧必须是一个具体的对象，也就是在栈上定义的变量或者**解引用后的对象**，对应C#的是访问普通类属性或方法的逻辑：

```cpp
Logger myLogger;         // 对象在栈上
myLogger.Log(level, ".."); // 使用点访问
```

->箭头运算符用于指针访问成员，这是C++处理指针时的专属符号，左侧必须是一个指针，比如说unique_ptr，或者是shared_ptr或者原始指针*。

它是(*ptr)的缩写，他回去解引用找到对应的对象后，再去访问成员。C#的引用类型变量比如class在底层都是指针，知识c#在帮你自动转换，在cpp中必须手动区分，如果手里是地址，就得用->。

```cpp
auto loggerPtr = std::make_unique<Logger>(); // loggerPtr 是个智能指针
loggerPtr->Log(level, "..");                  // 使用箭头访问
```

::是作用域解析运算符，用于静态/全局访问，左侧可以是命名空间，也可以是类名或者枚举名称。我们经常用它来访问静态成员变量/方法，访问命名空间里的类或者函数，访问enum class里的成员。对应的是c#里面中的话，访问静态方法比如说Console.WriteLine或者命名空间也用的是. ，但是c++里面必须用::。

```cpp
Loagger::Logger myLogger;        // 命名空间 :: 类名
LoggerLevel level = LoggerLevel::info; // 枚举 :: 成员
std::cout << "Hi";               // 命名空间 :: 对象
```

## 禁止构造函数隐式转换

`explicit` 用于**禁止构造函数或转换运算符的隐式转换**。它强制开发者必须**显式**地调用构造函数，从而避免编译器自动进行不期望的类型转换。


```cpp
    explicit __CLR_OR_THIS_CALL basic_ostream(basic_streambuf<_Elem, _Traits>* _Strbuf, bool _Isstd = false) {
        _Myios::init(_Strbuf, _Isstd);
    }
```

这里是std::basic_ostream的构造函数，作用是**流缓冲区初始化输出流**，我们平常这样使用：

```cpp
// 你平时这样用：
std::stringstream ss;           // 内部有自己的 streambuf
std::ostream os(&ss);           // ← 这里调用了这个构造函数
```

_Strbuf 是指向流缓冲区的指针，实际数据存储和传输的地方。_Isstd默认false，标记是否为标准流，标准流有cout、cerr、clog等，用于特殊处理。

这里用explicit防止隐式转换：

```cpp
std::stringbuf buf;
std::ostream os = &buf;   // ❌ 如果没有 explicit，这居然能编译！
std::stringbuf buf;
std::ostream os(&buf);    // ✅ 加了explicit需要显式构造，清晰明确
```


