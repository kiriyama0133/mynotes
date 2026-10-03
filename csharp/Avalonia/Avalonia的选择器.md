## 类CSS选择器

Avalonia也实现了类似于CSS的一样的选择器，可以在Classes里通过编写特定的字符来选择和匹配相关的样式，比如下：

```xaml
<Style Selector="Button.primary">
    <Setter Property="Background" Value="#4CAF50"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="12,8"/>
    <Setter Property="CornerRadius" Value="4"/>
</Style>

<!-- 使用自定义样式 -->
<Button Classes="primary" Content="确认"/>
```

### 命名空间选择

要在类型中**包含XAML命名空间**，请使用`|`字符将命名空间和类型分隔，比如下：

```xaml
<Style Selector="local|Button">
```

### 如果想通过名称来选择，则需要使用#

```xaml
<Style Selector="#myButton">
<Style Selector="Button#myButton">
```

### 如果想要通过样式来选择，需要使用.

```xaml
<Style Selector="Button.large">
<Style Selector="Button.large.red">
```

**伪类选择器！** https://docs.avaloniaui.net/zh-Hans/docs/reference/styles/pseudo-classes

```xaml
<Style Selector="Button:focus">
<Style Selector="Button.large:focus">
```

### 通用的伪类

以下伪类由InputElement定义，所有控件均可访问。

| Pseudoclass      | Description                                                  |
| :--------------- | ------------------------------------------------------------ |
| `:disabled`      | The Control is disabled and cannot be interacted with.       |
| `:pointerover`   | The Pointer is over the Control as determined by hit testing. |
| `:focus`         | The Control has focus.                                       |
| `:focus-within`  | The Control has focus or contains a descendant that has focus. |
| `:focus-visible` | The Control has focus and should show a visual indicator.    |

#### 自定义伪类

在创建自定义控件时，您可以定义自定义伪类来**暴露控件状态**。`[PseudoClasses]`特性向IDE提供了您的伪类信息。这种行为是可继承的，**因此自定义控件会自动受益于其基类控件定义和管理的伪类**（如InputElement的:pointerover）。

```csharp
[PseudoClasses(":left", ":right", ":middle")]
public class AreaButton : Button
{    
    protected override void OnPointerMoved(PointerEventArgs e)
    {
        base.OnPointerMoved(e);
        var pos = e.GetPosition(this);

        if (pos.X < Bounds.Width * 0.25)
            SetAreaPseudoclasses(true, false, false);
        else if (pos.X > Bounds.Width * 0.75)
            SetAreaPseudoclasses(false, true, false);
        else
            SetAreaPseudoclasses(false, false, true);
    }

    protected override void OnPointerExited(PointerEventArgs e)
    {
        base.OnPointerExited(e);
        SetAreaPseudoclasses(false, false, false);
    }

    private void SetAreaPseudoclasses(bool left, bool right, bool middle)
    {
        PseudoClasses.Set(":left", left);
        PseudoClasses.Set(":right", right);
        PseudoClasses.Set(":middle", middle);
    }
}
```

**派生选择器！**有趣的是，这允许您编写非常通用的**基于类别的选择器**。由于所有控件都派生自类`Control`，因此只选择样式类`margin2`的选择器可以编写如下：

```xaml
<Style Selector=":is(Control).margin2">
<Style Selector=":is(local|Control.margin2)">
```

```xaml
<Style Selector=":is(Button)">
<Style Selector=":is(local|Button)">
```

###  子操作符

```xaml
<Style Selector="StackPanel > Button">
```

通过**使用`>`字符分隔两个选择器来定义子选择器**。此选择器仅匹配**逻辑控件树**中的**直接子项**。有关逻辑控件树背后的概念，请参见[这里](https://docs.avaloniaui.net/zh-Hans/docs/concepts/control-trees)。

### 任意后代操作符

```xaml
<Style Selector="StackPanel Button">
```

当两个选择器由空格分隔时，选择器将匹配逻辑树中的任意后代。父级在左边，后代在右边。因此，将上述选择器应用于之前的XAML示例，两个按钮都将被选择。

### 按属性匹配

```xaml
<Style Selector="Button[IsDefault=true]">
```

您可以细化选择器，以包含属性的值。**属性=值对在方括号内定义**。这将匹配具有**指定属性设置为指定值的任何控件**。

```xaml
<StackPanel Orientation="Horizontal">
   <Button IsDefault="True">Save</Button>
   <Button>Cancel</Button>   
</StackPanel>
```

例如，在上面的XAML中，第一个按钮将被选择，但第二个按钮不会被选择。

注意：当您将附加属性用作属性匹配时，属性名必须用括号括起来。例如：

```xml
<Style Selector="TextBlock[(Grid.Row)=0]">
```

进一步注意：当您使用**属性匹配时，属性类型必须支持组件模型类型转换器**`TypeConverter`类。有关更多信息，请参阅[_Microsoft_文档](https://learn.microsoft.com/dotnet/api/system.componentmodel.typeconverter)。

### 按模板选择

```xaml
<Style Selector="Button /template/ ContentPresenter">
```

您可以使用上述语法在**控件模板中匹配控件**。此处列出的**所有其他选择器都适用于逻辑树**，但**此选择器可以进入模板**。

在上面的示例中，**如果按钮具有模板，则该选择器将选择模板中的内容呈现控件**（类`ContentPresenter`）。

### Not 函数

```xaml
<Style Selector="TextBlock:not(.h1)">
```

该函数否定括号中的选择。在上面的示例中，将匹配所有**没有**`h1`类的文本块控件。

### 按列表选择

```xaml
<Style Selector="TextBlock, Button">
```

您可以**选择与逗号分隔的选择器列表匹配的任何元素**。样式中的任何setter必须更改对所有项目都通用的属性。

### 按子元素位置公式选择

```xaml
<Style Selector="TextBlock:nth-child(2n+3)">
```

您可以**根据元素在相邻组内的位置进行匹配**。这与父（容器）控件的类别无关。

选择是基于样式中的简单公式`An + B`，以便 **`A`** 控制步长，**`B`** 控制从开始位置的偏移。在nth-child公式（上面）中，将 **`n`** 作为零提供给公式，以及从零开始的所有正整数，并且与子元素的基于一的位置的结果进行比较。

因此，对于上面的选择器：

| Child = 1 | Child = 2 | Child = 3 | Child = 4 |
| --------- | --------- | --------- | --------- |
| n=0, n=1  | n=0, n=1  | n=0, n=1  | n=0, n=1  |
| 3, 5      | 3, 5      | **3**, 5  | 3, 5      |
| 不匹配    | 不匹配    | 匹配      | 不匹配    |

### 单个子元素位置

您可以在XAML中省略公式中的**A**和**n**，仅指定一个位置。例如，这将仅选择第3个子元素：

```xml
<Style Selector="TextBlock:nth-child(3)">
```

### 关键字符号

您还可以在公式中使用关键字符号：`odd`或`even`。因此，以下选择器是等效的：

```xaml
<Style Selector="TextBlock:nth-child(2n)">
<Style Selector="TextBlock:nth-child(even)">
```

```xaml
<Style Selector="TextBlock:nth-child(2n+1)">
<Style Selector="TextBlock:nth-child(odd)">
```

可以通过OnPlateform来做到为不同的平台来设置不同属性：

```xaml
<Window>
    <Window.TitleBarHeight>
        <OnPlatform x:TypeArguments="double">
            <On Platform="Windows">32</On>
            <On Platform="macOS">28</On>
            <On Platform="Linux">30</On>
        </OnPlatform>
    </Window.TitleBarHeight>
</Window>
```

可以根据不同平台的资源放在对应目录来让Avalonia来自动选择和加载：

```
/Assets
  /Windows
    app_icon.png
  /Linux
    app_icon.png
  /macOS
    app_icon.png
```



## 性能分析工具

- **Avalonia Inspector**：实时查看UI层级和渲染性能，定位冗余控件
- **Visual Studio Profiler**：分析CPU和内存占用，识别瓶颈函数
- **RenderTraceListener**：监听渲染事件，统计绘制次数