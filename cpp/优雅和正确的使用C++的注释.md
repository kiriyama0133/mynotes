## 前言



这个文档我会讲解如何使用c++的注释以及使用doxygen来生成文档使用，doxygen支持多种风格，但是在现代c++中，我们推荐使用javadoc或者三斜杠。如果我们需要让生成的文档像专业库opencv或者QT一样清晰，那么我们就需要使用使用下面的标签：



| **标签**      | **含义**   | **用法示例**                                |
| ------------- | ---------- | ------------------------------------------- |
| **`@brief`**  | 简要说明   | `@brief 初始化相机驱动`                     |
| **`@param`**  | 参数说明   | `@param width 图像宽度（像素）`             |
| **`@return`** | 返回值说明 | `@return 成功返回 0，失败返回错误码`        |
| **`@note`**   | 特别注意   | `@note 该函数是非线程安全的`                |
| **`@see`**    | 参考引用   | `@see StopCamera()`                         |
| **`@throws`** | 异常说明   | `@throws std::runtime_error 如果硬件未连接` |
| **`@file`**   | 文件声明   | 放在文件开头，用于生成文件列表文档          |

比如说我们需要编写一个日志类：

```cpp
/**
 * @file Logger.hpp
 * @author YourName
 * @brief 高性能日志系统头文件
 */

#pragma once
#include <string>

/**
 * @class Logger
 * @brief 处理系统日志输出的单例类
 * 
 * 该类通过 ANSI 转义序列实现控制台彩色输出。
 */
class Logger {
public:
    /**
     * @brief 打印一条格式化日志
     * 
     * @param level 日志等级 @see LogLevel
     * @param msg   要打印的消息内容内容
     * @attention 确保已调用 SetConsoleOutputCP(CP_UTF8)
     */
    static void Log(LogLevel level, const std::string& msg) noexcept;
};
```

## 生成文档

生成文档的方法也比较简单，我们可以选择doxywizard去导出：

![image-20260501224029730](./assets/image-20260501224029730.png)

这一页的配置比较简单，看一下就知道怎么设置，重要的是后面的设置：

![image-20260501225546263](./assets/image-20260501225546263.png)

目前选的是 **Documented entities only**（仅提取已注释的实体），如果我们的代码还灭有大规模写完`/ ... */` 这种格式的注释，请改选 **All Entities**，选了 All Entities 后，即便你还没写注释，Doxygen 也会把类结构、函数声明都列出来，方便你查看项目的整体架构。



![image-20260501225718307](./assets/image-20260501225718307.png)

output可以选择导出的文档的格式，如果我们想在浏览器里看文档的话可以去掉LaTex，latex用来生成的是pdf打印版，生成比较慢，而且会有很多中间文件。

Diagrams是标签页，这是 Doxygen 自带的简易绘图，只能画简单的类继承关系，样子比较复古。效果如下，我们可以直接在网页中看到：

![image-20260501230109312](./assets/image-20260501230109312.png)

![image-20260501230131524](./assets/image-20260501230131524.png)

