`typedef` 是 C 语言中的“老常客”，也是 C 程序员为了让代码更具可读性而最常用的“救星”之一，但是随着C++对的发展，现在地位主键被更为强大的using关键字取代了。

纯C语言中，我们经常使用typedef来定义类型，比如你可以在C中，定义一个结构体，一般这样写：

```cpp
typedef struct {
    int x, y;
} Point;

Point p1; // 这样就清爽多了

struct Point {
    int x, y;
};
```

下面这种写法需要通过struct:struct Point p1这样写，顺便一提 c++中，struct定义完后直接就是类型名了，不需要typedef也可以直接写Point p1，这个是和C不一样的，需要分辨。

另一个作用就是优化函数指针，我们都知道原本C语言中的函数指针的原始语法反人类，一般这样写：

```c
void (*callback)(int, int);
```

那么我们使用typedef可以优化对应的写法：

```c
typedef void (*GLFWkeyfun)(int, int); // 给这种类型的函数起名叫 GLFWkeyfun
GLFWkeyfun my_callback; // 瞬间看懂
```

函数指针是一个变量，可以存储函数在内存中的起始地址，由于它指向的是内存中的指令（可以直接执行），因此可以直接调用。有个有点像的概念叫做匿名函数，它们两个有很多不同。

函数指针是一个地址变量，指向一个已经定义好了的有名字的函数，匿名函数定义的时候是没有名字的。函数指针捕获能力有限，只能访问全局变量或者传入的参数，匿名函数则是可以捕获外部作用域的局部变量，如果一个匿名函数没有捕获任何外部变量，那么它可以隐式转换为一个函数指针。

但是函数指针也是真的丑也非常反直觉，就像凭空生成一个名称，这是因为C语言的typedef采取了模仿变量声明逻辑，只要你能写出一个变量声明，那么前面加上typedef，这个变量名就变成了类型名，如果你定义一个指向缓冲生成函数的变量my_func_ptr:

```C
void (*my_func_ptr) (GLsizei, GLuint*);
```

这里编译器知道my_func_ptr是一个变量名，如果我们在前面加上typedef，编译器就会知道不创建一个叫 `GL_GENBUFFERS` 的变量了，而是把 `GL_GENBUFFERS` 定义为一个**新的类型名称**，这样就不需要**前向声明**，因为这一行本身就是定义。

你可以试试using来替代一下：

```cpp
using GL_GENBUFFERS = void(*) (int, int);
```

注意这里的`(*)`，它的出现可以有三种情况：

没有括号的情况下，比如 `void *func(int);`  这个就被会解析一个名为func的函数，返回一个
`void*`指针，如果有括号，类似于`void (*func)(int)`，这种情况下会被解析为一个名叫func的指针，指向一个返回void的函数。如果是`(int*) ptr`，这个叫做显示转换，在C++中，我们倾向于使用`static_cast<T>`这种方式或者`reinterpret_cast<T>`这种方式，这种带括号的权利太大，很容易出现bug。

现在我们说说日常最长使用的两个转换：


```cpp
double pi = 3.14159;
int rounded_pi = static_cast<int>(pi); // 安全地截断

// 在 OpenCV 中的应用
void* rawData = getSomeData(); 
uchar* pixelData = static_cast<uchar*>(rawData); // 明确告诉编译器：我知道这是 uchar 指针
```



这个适应基础的类型转换，类层次结构中的上行转换：将派生类指针/引用转换为基类，这个是安全的。也可以实现类层次的下行转换：将**基类转换为派生类**，注意，不进行运行时检查，如果确信这个**基类指针指的是派生类**，那么可以用它来换取高性能。

```cpp
class Base { virtual void func() {} }; // 必须有虚函数
class Derived : public Base { void extra() {} };

Base* b = new Base();
Derived* d = dynamic_cast<Derived*>(b); // 转换失败，b 并不是 Derived 类型

if (d) {
    d->extra();
} else {
    // 转换失败的处理逻辑，类似于 C# 的 if (d == null)
}
```

这个dynamic_cast近似于c#中的as操作符，专门用于处理多态那中带有virtual函数的类，它会在程序运行时检查对象的真实类型，如果转换是合法的话，就会返回指针，如果失败，就会返回nullptr，它适用于安全的下行转换，不确定基类指针到底指向哪个子类的时候使用。

```cpp
// 1. 定义一个普通函数
void MyKeyCallback(int key, int action) { 
    /* 处理按键 */ 
}

// 2. 声明一个函数指针变量
// 格式：返回类型 (*指针名)(参数列表)
void (*ptr)(int, int); 

// 3. 赋值（函数名本身就是地址）
ptr = MyKeyCallback;

// 4. 通过指针调用
ptr(10, 1);
```



虽然说typedef在C++中依然有效，但是现代C++程序员更倾向使用using，主要是因为using的可读性更好，遵循从左到右的赋值逻辑。

```c++
// typedef 逻辑是：定义一个变量，前面加个关键字变类型
typedef std::vector<std::string> StringList;

// using 逻辑是：这个名字 = 那个类型（像赋值一样直观）
using StringList = std::vector<std::string>;
```

