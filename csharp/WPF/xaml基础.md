
#### **`x:Name` (取名字)**

- **用途**：在 XAML 里给组件编号，好让你在 C# 后端代码里直接通过变量名访问它。
    
- **例子**：`<Border x:Name="MyClock" />` -> 在 C# 里你就可以写 `MyClock.Child = ...`

#### **`x:Key` (贴标签)**

- **用途**：这是**资源字典（Resources）**里的唯一 ID。
    
- **例子**：你在 `XmlDataProvider` 里写的 `x:Key="numbers"`。没有它，`StaticResource` 就找不到目标。

#### **`x:Type` (指明类型)**

- **用途**：告诉编译器“这是一个类”，而不是一个字符串。
    
- **例子**：`<x:Array Type="{x:Type sys:Int32}">`。这让编译器知道数组里装的是整数，而不是文本。

#### **`x:Class` (连接后端的桥梁)**

- **用途**：定义这个 XAML 文件对应的 C# 类是谁。
    
- **例子**：`<Window x:Class="ClockForWPF.MainWindow">`。这把 `.xaml` 和 `.xaml.cs` 缝合在了一起。



