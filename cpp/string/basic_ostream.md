
## 前置知识

流式一个基础并且核心的概念，是c++用来抽象数据输入和输出操作的概念，可以看做是一个数据传送带或者水管，数据可以从源头输入数据，叫做输入流，可以从程序流出到目的地，叫做输出流。

它屏蔽了底层硬件设备（键盘、显示器、硬盘、网卡）的差异，让程序员可以用**统一的方式**（如 `<<` 和 `>>` 操作符）来读写数据，而不用关心数据到底去了哪里。

```
// 对程序员来说，写法几乎一样：
std::cout << "Hello";   // 流向控制台
fileStream << "Hello";  // 流向文件
socketStream << "Hello"; // 流向网络
```

我们深入分析的 `basic_ostream` 就是 C++ 标准库流体系的核心基类。整个体系可以看作一个**继承树**：

|层次|类名|作用|
|---|---|---|
|**核心基类**|`ios_base`|管理流状态、格式标志（如进制、宽度）|
|**中间基类**|`basic_ios`|管理流缓冲区指针 `rdbuf()`，处理错误状态|
|**输入流**|`basic_istream`|负责**读**数据（`>>` 操作符）|
|**输出流**|`basic_ostream`|负责**写**数据（`<<` 操作符）|
|**输入输出流**|`basic_iostream`|同时支持读写（继承自上述两者）|
根据数据的目的地来看，可以将流分为几个大类，底层机制是一样的，**区别在于内部绑定的 “流缓冲区（streambuf）” 不同**。

数据流入的是磁盘文件的话，就叫做文件流。 **类名**：`ifstream`（读文件）、`ofstream`（写文件）、`fstream`（读写文件）
    **关联**：它内部绑定了一个 `filebuf`（文件缓冲区），负责将数据真正写入硬盘。

```
#include <fstream>
std::ofstream file("test.txt");
file << "写入文件";   // 用法和 cout 完全一样！
file.close();
```

字符串流是将内存中的字符串上进行读写，用于数据类型转换或者作为内存中的临时存储，避免频繁的IO操作，内部绑定了stringbuf字符串缓冲区。

```
#include <sstream>
std::ostringstream oss;
oss << "年龄: " << 25;
std::string result = oss.str(); // 把数据“流”进字符串里
```

标准流是操作系统默认帮你打开的流，用于连接程序和终端：
- **`std::cout`**（标准输出流）：通常指向控制台屏幕。
    
- **`std::cin`**（标准输入流）：通常指向键盘。
    
- **`std::cerr`**（标准错误流）：无缓冲，立即输出错误信息。



```cpp
class basic_ostream : virtual public basic_ios<_Elem, _Traits> { // control insertions into a stream buffer
public:
    using _Myios = basic_ios<_Elem, _Traits>;
    using _Mysb  = basic_streambuf<_Elem, _Traits>;
    using _Iter  = ostreambuf_iterator<_Elem, _Traits>;
    using _Nput  = num_put<_Elem, _Iter>;
    explicit __CLR_OR_THIS_CALL basic_ostream(basic_streambuf<_Elem, _Traits>* _Strbuf, bool _Isstd = false) {
        _Myios::init(_Strbuf, _Isstd);
    }
    __CLR_OR_THIS_CALL basic_ostream(_Uninitialized, bool _Addit = true) {
        if (_Addit) {
            this->_Addstd(this); // suppress for basic_iostream
        }
    }
protected:
    __CLR_OR_THIS_CALL basic_ostream(basic_ostream&& _Right) noexcept(false) {
       _Myios::init();
        _Myios::move(_STD move(_Right));
    } 
    basic_ostream& __CLR_OR_THIS_CALL operator=(basic_ostream&& _Right) noexcept /* strengthened */ {
        this->swap(_Right);
        return *this;
    }
    void __CLR_OR_THIS_CALL swap(basic_ostream& _Right) noexcept /* strengthened */ {
        if (this != _STD addressof(_Right)) {
            _Myios::swap(_Right);
        }
    }
public:
    __CLR_OR_THIS_CALL basic_ostream(const basic_ostream&)            = delete;
    basic_ostream& __CLR_OR_THIS_CALL operator=(const basic_ostream&) = delete;
    __CLR_OR_THIS_CALL ~basic_ostream() noexcept override {}
    using int_type = typename _Traits::int_type;
    using pos_type = typename _Traits::pos_type;
    using off_type = typename _Traits::off_type;
    class _Sentry_base { // stores thread lock and reference to output stream
    public:
        __CLR_OR_THIS_CALL _Sentry_base(basic_ostream& _Ostr) : _Myostr(_Ostr) { // lock the stream buffer, if there
            const auto _Rdbuf = _Myostr.rdbuf();
            if (_Rdbuf) {
                _Rdbuf->_Lock();
            }
        }
        __CLR_OR_THIS_CALL ~_Sentry_base() noexcept { // destroy after unlocking
            const auto _Rdbuf = _Myostr.rdbuf();
            if (_Rdbuf) {
                _Rdbuf->_Unlock();
            }
        }
        basic_ostream& _Myostr; // the output stream, for _Unlock call at destruction
        _Sentry_base& operator=(const _Sentry_base&) = delete;
    };
   class sentry : public _Sentry_base {
    public:
        explicit __CLR_OR_THIS_CALL sentry(basic_ostream& _Ostr) : _Sentry_base(_Ostr) {
            if (!_Ostr.good()) {
                _Ok = false;
                return;
            }
           const auto _Tied = _Ostr.tie();
            if (!_Tied || _Tied == _STD addressof(_Ostr)) {
                _Ok = true;
                return;
            }
            _Tied->flush();
            _Ok = _Ostr.good(); // store test only after flushing tie
        }
        _STL_DISABLE_DEPRECATED_WARNING
        __CLR_OR_THIS_CALL ~sentry() noexcept {
#if !_HAS_EXCEPTIONS
            const bool _Zero_uncaught_exceptions = true;
#elif _HAS_DEPRECATED_UNCAUGHT_EXCEPTION
            const bool _Zero_uncaught_exceptions = !_STD uncaught_exception(); // TRANSITION, ArchivedOS-12000909
#else // ^^^ _HAS_DEPRECATED_UNCAUGHT_EXCEPTION / !_HAS_DEPRECATED_UNCAUGHT_EXCEPTION vvv
            const bool _Zero_uncaught_exceptions = _STD uncaught_exceptions() == 0;
#endif // ^^^ !_HAS_DEPRECATED_UNCAUGHT_EXCEPTION ^^^  
            if (_Zero_uncaught_exceptions) {
                this->_Myostr._Osfx();
            }
        }
        _STL_RESTORE_DEPRECATED_WARNING 
        explicit __CLR_OR_THIS_CALL operator bool() const {
            return _Ok;
        }
        __CLR_OR_THIS_CALL sentry(const sentry&)            = delete;
        sentry& __CLR_OR_THIS_CALL operator=(const sentry&) = delete; 
    private:
        bool _Ok; // true if stream state okay at construction
    };
```

虚继承自basic_ios，`basic_iostream` 同时继承 `basic_istream` 和 `basic_ostream`，而两者都继承自 `basic_ios`。

```
            basic_ios
          /          \
   basic_istream   basic_ostream   ← 这里是虚继承
          \          /
           basic_iostream
```

类型别名：
```
using _Myios = basic_ios<_Elem, _Traits>;   // 基类
using _Mysb  = basic_streambuf<_Elem, _Traits>;  // 流缓冲区
using _Iter  = ostreambuf_iterator<_Elem, _Traits>;  // 输出迭代器
using _Nput  = num_put<_Elem, _Iter>;  // 数值格式化器
```

这里实现不同类型的构造：

绑定流缓冲区，将**输出流绑定到一个具体的流缓冲区**（文件、字符串、控制台等）
```
explicit basic_ostream(basic_streambuf<_Elem, _Traits>* _Strbuf, bool _Isstd = false) {
    _Myios::init(_Strbuf, _Isstd);
}
```

特殊构造，未初始化状态， 用于 `basic_iostream` 这样的派生类，它们需要**先构造**，再初始化基类， `_Uninitialized` 是一个空标记类型，表示"不初始化流缓冲区"。
```
basic_ostream(_Uninitialized, bool _Addit = true) {
    if (_Addit) {
        this->_Addstd(this);  // 注册到标准流列表
    }
}
```

移动构造，支持移动语义，避免不必要拷贝，移动可能需要抛出异常，这个取决流缓冲区：

```
basic_ostream(basic_ostream&& _Right) noexcept(false) {
    _Myios::init();           // 先初始化基类（无缓冲区）
    _Myios::move(_STD move(_Right));  // 移动基类资源
}
```

由于流对象不可拷贝，这是设计上的选择，std::cout不能拷贝，因此我们需要删除拷贝操作
```
basic_ostream(const basic_ostream&) = delete;
basic_ostream& operator=(const basic_ostream&) = delete;
```

内部有一个叫做Sentry的对象，是流操作的防护，每次执行<<操作的时候需要检查流状态是否good，刷新tie，如果有绑定的输入流，先刷新，加锁在多线程的环境下保护流缓冲区，还可以解析的时候解锁刷新确保数据输出。

```
class _Sentry_base {  // 基类：负责加锁/解锁
public:
    _Sentry_base(basic_ostream& _Ostr) : _Myostr(_Ostr) {
        const auto _Rdbuf = _Myostr.rdbuf();
        if (_Rdbuf) {
            _Rdbuf->_Lock();  // ← 加锁！
        }
    }
    
    ~_Sentry_base() noexcept {
        const auto _Rdbuf = _Myostr.rdbuf();
        if (_Rdbuf) {
            _Rdbuf->_Unlock();  // ← 解锁！
        }
    }
    
    basic_ostream& _Myostr;
};

class sentry : public _Sentry_base {
public:
    explicit sentry(basic_ostream& _Ostr) : _Sentry_base(_Ostr) {
        // 1. 检查流状态
        if (!_Ostr.good()) {
            _Ok = false;
            return;
        }
        
        // 2. 刷新绑定的输入流（tie）
        const auto _Tied = _Ostr.tie();
        if (!_Tied || _Tied == &_Ostr) {
            _Ok = true;
            return;
        }
        
        _Tied->flush();  // 刷新绑定的流
        _Ok = _Ostr.good();
    }
    
    ~sentry() noexcept {
        // 析构时调用 _Osfx()（输出刷新和清理）
        if (_Zero_uncaught_exceptions) {
            this->_Myostr._Osfx();
        }
    }
    
    explicit operator bool() const { return _Ok; }  // 检查是否成功
    
private:
    bool _Ok;
};
```


## 模板构造


```
#include <fmt/format.h>

template<typename CharT, typename Traits>
std::basic_ostream<CharT, Traits>& operator<<(
    std::basic_ostream<CharT, Traits>& os,
    const Person& p
) {
    os << "Person{name: " << p.name 
       << ", age: " << p.age 
       << ", hobbies: ["
       << fmt::join(p.hobbies, ", ")
       << "]}";
    return os;
}
```

这里是个利用函数模板和fmt库实现的自定义数据结构的流输出操作。`CharT` 和 `Traits` 是 C++ 标准库流设计的核心，理解了它们，你就理解了 `basic_ostream` 为什么能同时支持 `char`、`wchar_t` 甚至自定义字符类型。

---

CharT是字符类型，决定了流中**每个字符用什么类型**去存储：

|实例化|`CharT`|说明|
|---|---|---|
|`std::ostream`|`char`|单字节 ASCII/UTF-8|
|`std::wostream`|`wchar_t`|宽字符（Windows 上 2 字节，Linux 上 4 字节）|
|`std::u8ostream` (C++20)|`char8_t`|UTF-8 字符|
|`std::u16ostream` (C++20)|`char16_t`|UTF-16 字符|
|`std::u32ostream` (C++20)|`char32_t`|UTF-32 字符|

```
// 1. char 版本
std::ostream& os = std::cout;      // CharT = char
os << "Hello";                     // 存储 char 序列

// 2. wchar_t 版本
std::wostream& wos = std::wcout;   // CharT = wchar_t
wos << L"Hello";                   // 存储 wchar_t 序列

// 3. 模板函数自动适配
template<typename CharT, typename Traits>
void print(std::basic_ostream<CharT, Traits>& os, const CharT* str) {
    os << str;  // 自动适配 char 或 wchar_t
}

print(std::cout, "Hello");     // CharT = char
print(std::wcout, L"Hello");   // CharT = wchar_t
```

Traits是策略类，用于定义字符类型CharT的各种操作和行为，比如说比较两个字符，计算字符串长度，EOF值是多少，怎么复制字符数组。

