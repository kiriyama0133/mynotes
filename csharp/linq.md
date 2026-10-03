语言集成查询（LINQ）是.Net 3.5和Visual Studio 2008引入的功能强大的查询语言。**LINQ可与C＃或Visual Basic一起使用**，**以查询不同的数据源。**

LINQ（语言集成查询）是C＃和VB.NET中的统一查询语法，用于从不同的源和格式检索数据。它集成在C＃或VB中，从而消除了编程语言和数据库之间的不匹配，并为不同类型的数据源提供了单个查询接口。

例如，SQL是一种结构化查询语言，用于保存和检索数据库中的数据。同样，LINQ是C＃和VB.NET中内置的结构化查询语法，用于从不同类型的数据源（例如集合，ADO.Net DataSet，XML Docs，Web服务和MS SQL Server和其他数据库）中检索数据。  

LINQ查询将结果作为对象返回。它使您可以在结果集上使用面向对象的方法，而不必担心将不同格式的结果转换为对象。

## 什么是委托类型：

### 1. **委托（Delegate）**

**委托是 C# 中一种类型安全的函数指针**，允许你**将方法作为参数传递**、**存储和调用**。委托定义了方法的签名（返回类型和参数类型），并可以绑定到任何匹配此签名的方法。

例如：

```c#
public delegate int MyDelegate(int x, int y);

```

这个 `MyDelegate` **委托**可以**指向任何接受两个 `int` 参数并返回一个 `int` 的方法。**

### 委托类型的基本形式

1. **无参数的委托**:

   - `Action`

     : 用于没有返回值的方法。

     ```
     
     Action myAction = () => Console.WriteLine("Hello, World!");
     ```

2. **有返回值的委托**:

   - `Func<T, TResult>`

     : 用于有返回值的方法，

     ```
     T
     ```

      是输入参数类型，

     ```
     TResult
     ```

      是返回值类型。

     ```
     
     Func<int, int, int> add = (x, y) => x + y;
     ```

3. **有条件返回值的委托**:

   - `Predicate<T>`

     : 用于条件测试，返回 

     ```
     bool
     ```

     。

     ```
     
     Predicate<int> isEven = number => number % 2 == 0;
     ```

4. **自定义委托类型**:

   - 可以定义自己的委托类型，以适应特定的需求。

     ```
     public delegate bool MyPredicate(int number);
     
     MyPredicate isGreaterThanTen = number => number > 10;
     ```

### 委托类型与函数参数类型的关系

委托类型的作用是定义方法的参数和返回类型。这使得你可以将方法作为参数传递，或者用委托类型来调用方法。它们是方法签名的类型安全表示，确保了方法调用的一致性。



```c#
public static void Main()
{
    // 使用委托类型定义方法
    MathOperation add = (x, y) => x + y;
    MathOperation multiply = (x, y) => x * y;

    // 使用委托调用方法
    int sum = add(5, 3);         // sum = 8
    int product = multiply(5, 3); // product = 15

    Console.WriteLine($"Sum: {sum}");
    Console.WriteLine($"Product: {product}");
}
```
}

**示例**:

### 2. **Lambda 表达式**

**Func<int, int, int> add = (x, y) => x + y;**

Lambda 表达式是一个简化的语法，**用于创建匿名方法或委托**。**它允许你以更简洁的方式定义方法**，**而无需显式声明一个方法或委托类型**。Lambda 表达式常用于 LINQ 查询、事件处理和其他需要委托的场景。

这里的 `(x, y) => x + y` 是一个 Lambda 表达式，它表示**一个接受两个 `int` 参数并返回它们和的匿名方法**。

### `Func<T1, T2, TResult>` 解释

- **`Func<T1, T2, TResult>`** 是一个预定义的泛型委托类型，它表示一个接受两个参数 `T1` 和 `T2`，并返回一个结果 `TResult` 的方法。
- `Func` 是 .NET 中提供的一组委托类型之一，它用于**定义接受参数并返回值的委托**。



**（x，y）是输入参数，x+y是返回的参数**

### 3. **委托类型与 Lambda 表达式**

Lambda 表达式实际上与委托类型相关联。**Lambda 表达式**会**根据其参数列表和返回类型自动推导出一个委托类型**。常用的委托类型包括：

- **`Action<T>`**: 表示一个**不返回结果的委托**，通常用**于无返回值的方法**。
- **`Func<T, TResult>`**: 表示一个有返回结果的委托，`T` 是输入参数的类型，`TResult` 是返回结果的类型。
- **`Predicate<T>`**: 表示一个带有返回值为 `bool` 的委托，通常用于条件测试。

### 参数和返回值

- **第一个 `int`**: 这是 `Func` 委托的第一个参数类型。在此例中，`x` 是第一个 `int` 类型的参数。
- **第二个 `int`**: 这是 `Func` 委托的第二个参数类型。在此例中，`y` 是第二个 `int` 类型的参数。
- **第三个 `int`**: 这是 `Func` 委托的返回值类型。在此例中，`x + y` 的结果是一个 `int` 类型的值，作为这个委托的返回结果。

### 更多 `Func` 类型

`Func` 委托可以接受多达 16 个参数，具体取决于泛型参数的数量。例如：

- **`Func<int>`**: 表示一个没有参数且返回一个 `int` 的方法。
- **`Func<string, bool>`**: 表示一个接受一个 `string` 参数并返回一个 `bool` 的方法。
- **`Func<double, double, double, double>`**: 表示一个接受三个 `double` 参数并返回一个 `double` 的方法。

### 示例代码

以下是一个完整的示例，展示了如何使用 `Func` 委托类型：

```c#
using System;

public class Program
{
    public static void Main()
    {
        // 定义一个 Func 委托，接受两个 int 参数并返回它们的和
        Func<int, int, int> add = (x, y) => x + y;
        
        // 调用 Func 委托
        int result = add(3, 4); // result = 7
        
        // 输出结果
        Console.WriteLine(result);
    }
}

```

在这个例子中，`add` 是一个 `Func<int, int, int>` 类型的委托，它接受两个 `int` 参数并返回它们的和。调用 `add(3, 4)` 会返回 `7`。

### 总结

`Func<int, int, int>` 是一个泛型委托类型，表示一个接受两个 `int` 类型参数并返回一个 `int` 类型结果的方法。Lambda 表达式 `(x, y) => x + y` 匹配这个委托类型，因为它接受两个 `int` 参数并返回它们的和。**Lambda 表达式是一个语法糖，用于创建匿名方法或委托**，它可以**与多种不同类型的委托配合使用**。`Func` 和 `Action` 是最常用的两种委托类型，但实际上你可以使用 Lambda 表达式来创建任何符合特定签名的委托类型。

# LINQ API（.Net）

我们可以为实现`IEnumerable <T>`或 `IQueryable <T>`接口的类编写LINQ查询。*System.Linq的*命名空间包括下列类和接口要求对LINQ查询。当你实现 **`IEnumerable<T>` 或 `IQueryable<T>`** 接口的类时，你可以**编写 LINQ 查询来操作数据。`IEnumerable<T>`** 提供了对内存中的数据进行查询的能力，而 `IQueryable<T>` 允许你在查询时将表达式树转化为数据库查询

![img](https://www.cainiaojc.com/static/upload/201229/1137310.png)LINQ API在Visual Studio中添加新类时，默认包含 System.Linq 命名空间。。

LINQ查询对实现IEnumerable或IQueryable接口的类使用扩展方法。Enumerable和Queryable是两个静态类，它们包含编写LINQ查询的扩展方法。

## 可枚举类(Enumerable)

Enumerable类包括用于实现`IEnumerable<T>`接口的类的扩展方法，例如，所有内置集合类都实现了`IEnumerable<T>`接口，因此我们可以编写LINQ查询来从内置集合中检索数据。

下图显示了Enumerable类中包含的扩展方法，可以与C＃或VB.Net中的泛型集合一起使用。

![img](https://www.cainiaojc.com/static/upload/201229/1137311.png)

下图显示了Enumerable该类中所有可用的扩展方法。

![img](https://www.cainiaojc.com/static/upload/201229/1137312.png)

Enumerable 类

## 可查询(Queryable) 

Queryable类包含用于实现成员“> `IQueryable <t>`接口的类的扩展方法。该`IQueryable<T>`接口用于提供针对已知数据类型的特定数据源的查询功能，例如，Entity Framework api实现了`IQueryable<T>`针对通过底层数据库（例如MS SQL Server）支持LINQ查询。

此外，还有一些API可用于访问第三方数据。例如，LINQ to Amazon提供了将LINQ与Amazon Web服务结合使用以搜索书籍和其他物品的功能。这可以通过IQueryable为Amazon实现接口来实现。

下图显示了Queryable该类中可用的扩展方法，可以与各种本机或第三方数据提供程序一起使用。

![img](https://www.cainiaojc.com/static/upload/201229/1137313.png)

下图显示了Queryable该类中可用的扩展方法。

![img](https://www.cainiaojc.com/static/upload/201229/1137314.png)Queryable 类

##  要记住的要点

1. 使用 System.LINQ 命名空间来使用 LINQ。
2. LINQ api包括两个主要的静态类Enumerable 和 Queryable。
3. 静态Enumerable类包括用于实现`IEnumerable <T>`接口的类的扩展方法。
4. `IEnumerable <T>`集合的类型是内存中的集合，例如List，Dictionary，SortedList，Queue，HashSet，LinkedList。
5. 静态Queryable类包括用于实现`IQueryable <T>`接口的类的扩展方法。
6. 远程查询提供程序实现了例如Linq-to-SQL，LINQ-to-Amazon等。

## 查询语法

查询语法类似于数据库的SQL（结构化查询语言）。它在C＃或VB代码中定义。

LINQ查询语法：

```
from <range variable> in <IEnumerable<T> or IQueryable<T> Collection>

<Standard Query Operators> <lambda expression>

<select or groupBy operator> <result formation>
```

LINQ查询语法以from关键字开头，以select关键字结尾。下面是一个示例LINQ查询，该查询返回一个字符串集合，其中包含一个单词“ Tutorials”。

```c#
// 字符串集合
IList<string> stringList = new List<string>() { 
    "C# Tutorials",
    "VB.NET Tutorials",
    "Learn C++",
    "MVC Tutorials" ,
    "Java" 
};

// LINQ查询语法
var result = from s in stringList
            where s.Contains("Tutorials") 
            select s;
```

下图显示了LINQ查询语法的结构。

![img](https://www.cainiaojc.com/static/upload/201229/1138520.png)LINQ查询语法

查询语法以 From 子句开头，后跟 Range 变量。From 子句的结构类似于“ From rangeVariableName in i enumerablecollection”。在英语中，这意味着，从集合中的每个对象。它类似于 foreach 循环:foreach(Student s in studentList)。

在FROM子句之后，可以使用不同的标准查询运算符来过滤，分组和联接集合中的元素。LINQ中大约有50个标准查询运算符。在上图中，我们使用了“ where”运算符（又称子句），后跟一个条件。通常使用lambda表达式来表达此条件。

LINQ查询语法始终以Select或Group子句结尾。Select子句用于整形数据。您可以按原样选择整个对象，也可以仅选择某些属性。在上面的示例中，我们选择了每个结果字符串元素。

## 要记住的要点

1. 顾名思义，**查询语法**与SQL（结构查询语言）语法相同。
2. 查询语法以*from*子句开头，可以以*Select*或*GroupBy*子句结尾。
3. 使用各种其他运算符，例如过滤，联接，分组，排序运算符来构造所需的结果。
4. 隐式类型变量-var可用于保存LINQ查询的结果。

# LINQ 方法语法

在上一节中，您已经了解了LINQ查询语法。在这里，您将了解方法语法。

方法语法（也称为连贯语法）使用**Enumerable 或 Queryable静态类**中包含的扩展方法，类似于您调用任何类的扩展方法的方式。  

编译器在编译时将查询语法转换为方法语法。

以下是LINQ方法语法查询示例，该查询返回字符串集合，其中包含单词“ Tutorials”。

```c#
// 字符串集合
IList<string> stringList = new List<string>() { 
    "C# Tutorials",
    "VB.NET Tutorials",
    "Learn C++",
    "MVC Tutorials" ,
    "Java" 
};

// LINQ查询语法
var result = stringList.Where(s => s.Contains("Tutorials"));
```

下图说明了LINQ方法语法的结构。

![img](https://www.cainiaojc.com/static/upload/201229/1339390.png)LINQ方法语法结构

如上图所示，**方法语法包括扩展方法和 Lambda 表达式**。

**谓词 (`s => s.Contains("Tutorials")`)**:

- 这是一个 lambda 表达式，它定义了一个函数来检查每个字符串 `s` 是否包含 `"Tutorials"`。
- `s.Contains("Tutorials")` 是 `string` 类型的一个方法，用于检查字符串 `s` 是否包含子字符串 `"Tutorials"`。如果包含，则返回 `true`；否则返回 `false`。

**`result`**: `result` 是一个 `IEnumerable<string>` 类型的集合，包含了所有 `stringList` 中包含 `"Tutorials"` 的字符串。

在枚举(Enumerable)类中定义的扩展方法 Where ()。

如果**检查Where扩展方法的签名**，就会发现**Where方法接受一个 [predicate 委托](https://www.cainiaojc.com/csharp/csharp-predicate.html) Func<Student,bool>**。这意味着您可以传递任何接受Student对象作为**输入参数并返回布尔值的委托函数**，如下图所示。lambda表达式用作Where子句中传递的委托。在下一节学习 Lambda 表达式。

![img](https://www.cainiaojc.com/static/upload/201229/1339391.png)

Where 中的 Func 委托

## 要记住的要点

1. 顾名思义，**方法语法**就像**调用扩展方法**。
2. LINQ**方法语法**又称Fluent语法（连贯语法），因为它允许一系列扩展方法调用。
3. **隐式类型变量-var**可用于**保存LINQ查询的结果。**

# LINQ Lambda 表达式

C＃3.0（.NET 3.5）引入了lambda表达式以及 LINQ。lambda表达式是使用某些特殊语法表示匿名方法的一种较短方法。

例如，采用以下匿名方法检查学生是否为青少年：

示例：C＃中的匿名方法

```
delegate(Student s) { return s.Age > 12 && s.Age < 20; };
```

示例：VB.Net中的匿名方法

```
Dim isStudentTeenAger = Function(s As Student) As Boolean
                                    Return s.Age > 12 And s.Age < 20
                        End Function
```

可以使用C＃和VB.Net中的Lambda表达式来表示上述匿名方法，如下所示：

示例：C＃中的Lambda表达式

```
s => s.Age > 12 && s.Age < 20
```

示例：VB.Net中的Lambda表达式

```
Function(s) s.Age  > 12 And s.Age < 20
```

让我们看看lambda表达式是如何从以下匿名方法演变而来的。

示例：C＃中的匿名方法

```
delegate(Student s) { return s.Age > 12 && s.Age < 20; };
```

Lambda表达式是**通过首先删除委托关键字和参数类型**并**添加lambda运算符=>从匿名方法演变而来的**。

![img](https://www.cainiaojc.com/static/upload/201229/1400160.png)**来自匿名方法的Lambda表达式**

上面的lambda表达式绝对有效，但是如果我们只有一个返回值的语句，则不需要花括号，return和分号。因此我们可以删除它。具有多个参数的Lambda表达式

如果您需要传递多个参数，则可以将参数括在括号中，如下所示：

  示例：**在Lambda表达式C＃中指定多个参数**

```
(s, youngAge) => s.Age >= youngage;
```

如果觉得参数表达不清，您还可以给出每个参数的类型：

示例：指定参数类型

```
(Student s,int youngAge) => s.Age >= youngage;
```

示例：在Lambda表达式中指定多个参数VB.Net

```
Function(s, youngAge) s.Age >= youngAge
```

## 不带参数的Lambda表达式

lambda表达式中不必至少有一个参数。lambda表达式也可以不带任何参数来指定。

示例：不带参数的Lambda表达式

```
() => Console.WriteLine("无参数 lambda 表达式")
```

## Lambda表达主体中的多个语句

如果要在主体中包含多个语句，可以将表达式用大括号括起来：

示例：C# 多语句Lambda表达式

```
(s, youngAge) =>{
  Console.WriteLine("在主体中包含多个语句的Lambda表达式");
    
  Return s.Age >= youngAge;}
```

示例：VB.Net多语句Lambda表达式

```
Function(s , youngAge)
    
    Console.WriteLine("在主体中包含多个语句的Lambda表达式")
    
    Return s.Age >= youngAge

End Function
```

## 在Lambda表达主体中声明局部变量

您可以在表达式主体中声明一个变量，以在表达式主体中的任何地方使用它，如下所示：

示例：C＃中的Lambda表达式局部变量

```
s =>
{   int youngAge = 18;

    Console.WriteLine("在主体中包含多个语句的Lambda表达式");

    return s.Age >= youngAge;
}
```

示例：VB.Net Lambda表达式中的局部变量

```
Function(s) 
                                      
        Dim youngAge As Integer = 18
            
        Console.WriteLine("在主体中包含多个语句的Lambda表达式")
            
        Return s.Age >= youngAge
            
End Function
```

Lambda表达式也可以分配给内置委托，例如Func，Action和Predicate。

在 C# 中，直接调用 `.ToString()` 方法可能无法提供你期望的结果，特别是在处理集合类型（如 `IEnumerable<string>`）时。让我们详细解释一下原因以及为什么你需要其他方法来转换集合为字符串。

### `.ToString()` 方法的行为

1. **默认实现**:
   - 对于大多数对象，`ToString()` 方法返回对象的类型名称，而不是对象的实际内容。例如，`IEnumerable<string>` 的 `ToString()` 方法通常返回类似 `System.Linq.Enumerable+WhereEnumerableIterator` 的字符串，而不是集合中的实际元素。
2. **`IEnumerable<string>` 的 `ToString()`**:
   - `IEnumerable<string>` 是一个接口，它的 `ToString()` 方法不会将集合中的所有字符串连接成一个可读的格式。相反，它将返回接口的类型名称，这并不符合你的期望。

# LINQ 标准查询运算符

LINQ中的标准查询运算符实际上是 `IEnumerable<T> and IQueryable<T>`类型的扩展方法。它们在System.Linq.Enumerable和System.Linq.Queryable类中定义。LINQ中提供了50多个标准查询运算符，它们提供了不同的功能，例如**过滤，排序，分组，聚合，串联**等。

## 查询语法中的标准查询运算符

![img](https://www.cainiaojc.com/static/upload/201229/1138520.png)查询语法中的标准查询运算符

## 方法语法中的标准查询运算符

![img](https://www.cainiaojc.com/static/upload/201229/1443321.png)方法语法中的标准查询运算符

查询语法中的标准查询运算符在编译时转换为扩展方法。所以两者都是一样的。

可以根据标准查询运算符提供的功能对其进行分类。下表列出了标准查询运算符的所有分类：

| 类别     | 标准查询运算符                                               |
| :------- | :----------------------------------------------------------- |
| 过滤     | Where, OfType                                                |
| 排序     | OrderBy, OrderByDescending, ThenBy, ThenByDescending, Reverse |
| 分组     | GroupBy, ToLookup                                            |
| 联合     | GroupJoin, Join                                              |
| 投射     | Select, SelectMany                                           |
| 聚合     | Aggregate, Average, Count, LongCount, Max, Min, Sum          |
| 修饰     | All, Any, Contains                                           |
| 元素     | ElementAt, ElementAtOrDefault, First, FirstOrDefault, Last, LastOrDefault, Single, SingleOrDefault |
| 集合     | Distinct, Except, Intersect, Union                           |
| 分区     | Skip, SkipWhile, Take, TakeWhile                             |
| 串联     | Concat                                                       |
| 相等     | SequenceEqual                                                |
| 范围状态 | DefaultEmpty, Empty, Range, Repeat                           |
| 转换     | AsEnumerable, AsQueryable, Cast, ToArray, ToDictionary, ToList |

# LINQ 排序运算符 OrderBy和OrderByDescending

排序运算符以升序或降序排列集合的元素。LINQ包括以下排序运算符。

| 运算符            | 描述                                                     |
| :---------------- | :------------------------------------------------------- |
| OrderBy           | 根据指定的字段按升序或降序对集合中的元素进行排序。       |
| OrderByDescending | 根据指定的字段按降序对集合进行排序。仅在方法语法中有效。 |
| ThenBy            | 仅在方法语法中有效。用于按升序进行二次排序。             |
| ThenByDescending  | 仅在方法语法中有效。用于按降序进行二次排序。             |
| Reverse           | 仅在方法语法中有效。按相反顺序对集合排序。               |

# LINQ 转换运算符

LINQ中的Conversion运算符可用于转换序列（集合）中元素的类型。转换运算符分为三种：**As**运算符（AsEnumerable和AsQueryable），**To**运算符（ToArray，ToDictionary，ToList和ToLookup）和**转换**运算符（Cast和OfType）。

下表列出了所有转换运算符。

| 方法         | 描述                                                    |
| :----------- | :------------------------------------------------------ |
| AsEnumerable | 将输入序列作为 IEnumerable < T> 返回                    |
| AsQueryable  | 将IEnumerableto转换为IQueryable，以模拟远程查询提供程序 |
| Cast         | 将非泛型集合转换为泛型集合（IEnumerable到IEnumerable）  |
| OfType       | 基于指定类型筛选集合                                    |
| ToArray      | 将集合转换为数组                                        |
| ToDictionary | 根据键选择器函数将元素放入 Dictionary 中                |
| ToList       | 将集合转换为 List                                       |
| ToLookup     | 将元素分组到 Lookup<TKey,TElement>                      |

## AsEnumerable和AsQueryable方法

AsEnumerable和AsQueryable方法分别将源对象转换或转换为`IEnumerable <T>`或`IQueryable <T>`。

# LINQ 表达式树

您已在上一节中了解了表达式。现在，让我们在这里了解表达式树。

顾名思义，表达式树不过是按树状数据结构排列的表达式。表达式树中的每个节点都是一个表达式。例如，表达式树可用于表示数学公式x <y，其中x，<和y将表示为表达式，并排列在树状结构中。

表达式树是lambda表达式的内存表示形式。它保存查询的实际元素，而不是查询的结果。

表达式树使lambda表达式的结构透明和显式。您可以与表达式树中的数据进行交互，就像与其他任何数据结构一样。

例如，看以下isTeenAgerExpr表达式：

```c#
Expression<Func<Student, bool>> isTeenAgerExpr = s => s.age > 12 && s.age < 20;
```

# LINQ into 关键字

在LINQ查询中使用into关键字可组成一个组或在select子句后继续查询。

示例：LINQ中的 into 关键字

```c#
var teenAgerStudents = from s in studentList
    where s.age > 12 && s.age < 20
    select s
        into teenStudents
        where teenStudents.StudentName.StartsWith("B")
        select teenStudents;
```

在上面的查询中，“into”关键字引入了一个新的范围变量teenStudents，因此第一个范围变量s超出范围。您可以使用新的范围变量在into关键字之后编写进一步的查询。

### 基本用法

`into` 语句通常用于两个主要场景：

1. **在 `from` 子句中创建中间结果集**： `into` 可以用来将查询的结果存储在一个新的临时变量中，然后在这个变量上继续进行查询操作。
2. **在 `select` 子句中创建中间结果集**： 你可以在 `select` 子句中使用 `into`，将结果存储到一个新的集合中，然后对这个集合应用额外的查询。

### 示例

假设我们有一个 `students` 集合和一个 `courses` 集合：

```c#
public class Student
{
    public string Name { get; set; }
    public int Age { get; set; }
    public List<string> Courses { get; set; }
}

public class Course
{
    public string CourseName { get; set; }
    public string Instructor { get; set; }
}

```

我们可以使用 `into` 来简化复杂的查询。以下是一些示例：

#### 示例 1: 使用 `into` 创建中间结果集

假设我们想要先查询所有年龄大于 20 的学生，然后对这些学生进一步查询他们的课程：

```c#
var studentsWithCourses = from student in students
                          where student.Age > 20
                          into filteredStudents
                          select new
                          {
                              StudentNames = filteredStudents.Select(s => s.Name),
                              Courses = filteredStudents.SelectMany(s => s.Courses)
                          };

```

