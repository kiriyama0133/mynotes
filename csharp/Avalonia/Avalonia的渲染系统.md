​	在了解avalonia的渲染系统之前我们需要先了解一下avlaonia的组件系统，所有的组加都继承于一个叫做Control的基类，代码为：

```csharp
public class Control : InputElement, IDataTemplateHost, IVisualBrushInitialize, ISetterValue
```

这表明 `Control` 具有以下几个关键特性：

1. **继承自 `InputElement`**: 这意味着 `Control` 具备处理输入（如鼠标、键盘、触摸）的能力，这是所有可交互 UI 元素的基础。
2. **实现 `IDataTemplateHost`**: 这与数据模板（Data Templates）相关，意味着 `Control` 或其派生类可以作为数据模板的应用宿主，例如用于显示数据集合的容器控件。
3. **实现 `IVisualBrushInitialize`**: 这可能与 `VisualBrush` 的初始化过程有关，`VisualBrush` 是一种特殊的画刷，它使用另一个控件（`Visual`）的内容来绘制区域。
4. **实现 `ISetterValue`**: 这可能与样式系统（Styles）中的 `Setter` 设置值的能力有关，允许控件在样式系统中作为可设置值的目标或值的来源。

### InputElment

这个关于组件的输入处理，采用了分层架构，从硬件到高层的控件事件形成了完整的处理链条：

<img src="./assets/image-20251022225827879.png" alt="image-20251022225827879 20" style="zoom:50%;" />

这个叫做输入元素基类，而InputManager则是输入事件的调度中心。

Avalonia提供了丰富的键盘事件支持：

```csharp
// 键盘事件定义示例
public static readonly RoutedEvent<KeyEventArgs> KeyDownEvent =
    RoutedEvent.Register<InputElement, KeyEventArgs>(
        nameof(KeyDown),
        RoutingStrategies.Tunnel | RoutingStrategies.Bubble);
 
public static readonly RoutedEvent<KeyEventArgs> KeyUpEvent =
    RoutedEvent.Register<InputElement, KeyEventArgs>(
        nameof(KeyUp),
        RoutingStrategies.Tunnel | RoutingStrategies.Bubble);
 
public static readonly RoutedEvent<TextInputEventArgs> TextInputEvent =
    RoutedEvent.Register<InputElement, TextInputEventArgs>(
        nameof(TextInput),
        RoutingStrategies.Tunnel | RoutingStrategies.Bubble);
```

通过源码我们可以看到输入事件是可以支持隧道和冒泡这两个阶段的。

隧道： 事件从 UI 树的**根元素**（例如 `Window` 或最顶层的容器）开始，**向下**传播到触发事件的**目标元素**。

冒泡：事件从**目标元素**（实际获得焦点的控件）开始，**向上**传播到 UI 树的根元素。

<img src="./assets/image-20251022230234926.png" alt="image-20251022230234926 " style="zoom:50%;" />

#### 键盘事件处理示例：

```csharp
public class CustomInputControl : InputElement
{
    public CustomInputControl()
    {
        // 注册事件处理器
        AddHandler(KeyDownEvent, OnKeyDown);
        AddHandler(KeyUpEvent, OnKeyUp);
        AddHandler(TextInputEvent, OnTextInput);
    }
 
    private void OnKeyDown(object? sender, KeyEventArgs e)
    {
        // 处理按键按下事件
        if (e.Key == Key.Enter)
        {
            e.Handled = true; // 标记为已处理
            OnEnterPressed();
        }
    }
 
    private void OnKeyUp(object? sender, KeyEventArgs e)
    {
        // 处理按键释放事件
        Debug.WriteLine($"Key released: {e.Key}");
    }
 
    private void OnTextInput(object? sender, TextInputEventArgs e)
    {
        // 处理文本输入事件
        Debug.WriteLine($"Text input: {e.Text}");
    }
 
    protected override void OnKeyDown(KeyEventArgs e)
    {
        // 重写基类方法处理按键
        base.OnKeyDown(e);
    }
}
```

在Avalonia中，InputElment更加偏向于是逻辑组件，但它也继承了许多可视化组件的特性，他在UI元素层次中，专注于 交互 和 输入 处理的基类。它有焦点管理，键盘，指针事件等处理的功能，并且他也具有完整的绘制能力。

根据 Avalonia 的继承关系，你可以看到 `InputElement` 位于功能链的末端，但它**继承**了所有可视化和布局的特性：
$$
\text{Object} \to \text{AvaloniaObject} \to \text{Animatable} \to \text{StyledElement} \to \textbf{Visual} \to \text{Layoutable} \to \text{Interactive} \to \textbf{InputElement}
$$


- **`Visual` (可视化)：** 提供了**渲染、变换、命中测试**等所有与屏幕显示相关的基础能力。
- **`Layoutable` (布局)：** 提供了 `Width`、`Height`、`Margin`、`HorizontalAlignment` 等所有与布局计算相关的基础能力。
- **`Interactive` (交互)：** 引入了**路由事件系统**和一些更**高级的交互概念**。
- **`InputElement` (输入元素)：** 在继承了**所有可视化和布局能力的基础**上，**专门添加了**处理输入（键盘、鼠标、焦点）的逻辑。

需要注意一点的是，Avalonia的容器组件中需要的是一个Contrrol类，因此我们如果直接去继承InputElement的话就无法正常被包含在容器组件里，也就无法被渲染出来。

`InputElement` 的价值在于**框架架构层面**,而不是应用开发层面。 它是 Avalonia 实现关注点分离的关键:将输入处理从视觉渲染和控件功能中分离出来。 这种设计使得框架更加模块化和可维护,即使应用开发者通常不直接继承它。

鼠标事件处理机制
鼠标事件类型
Avalonia提供了全面的鼠标/指针事件支持：

![image-20251022233527174](image-20251022233527174.png)

#### 指针事件处理流程

<img src="./assets/image-20251022233542172.png" alt="image-20251022233542172 " style="zoom:50%;" />

处理事件的示例：

```csharp
public class InteractiveControl : InputElement
{
    private bool _isDragging = false;
    private Point _dragStartPoint;
 
    public InteractiveControl()
    {
        // 注册指针事件处理器
        PointerPressed += OnPointerPressed;
        PointerReleased += OnPointerReleased;
        PointerMoved += OnPointerMoved;
        PointerEntered += OnPointerEntered;
        PointerExited += OnPointerExited;
    }
 
    private void OnPointerPressed(object? sender, PointerPressedEventArgs e)
    {
        var pointer = e.GetCurrentPoint(this);
        
        if (pointer.Properties.IsLeftButtonPressed)
        {
            _isDragging = true;
            _dragStartPoint = pointer.Position;
            e.Pointer.Capture(this); // 捕获指针
            e.Handled = true;
        }
    }
 
    private void OnPointerReleased(object? sender, PointerReleasedEventArgs e)
    {
        if (_isDragging)
        {
            _isDragging = false;
            e.Pointer.Capture(null); // 释放指针捕获
            e.Handled = true;
        }
    }
 
    private void OnPointerMoved(object? sender, PointerEventArgs e)
    {
        if (_isDragging)
        {
            var currentPoint = e.GetCurrentPoint(this);
            var delta = currentPoint.Position - _dragStartPoint;
            
            // 处理拖拽逻辑
            HandleDrag(delta);
            e.Handled = true;
        }
    }
 
    private void OnPointerEntered(object? sender, PointerEventArgs e)
    {
        // 指针进入时的视觉效果
        UpdateVisualState(isPointerOver: true);
    }
 
    private void OnPointerExited(object? sender, PointerEventArgs e)
    {
        // 指针离开时的视觉效果
        UpdateVisualState(isPointerOver: false);
    }
 
    protected override void OnPointerMoved(PointerEventArgs e)
    {
        // 重写基类方法处理指针移动
        base.OnPointerMoved(e);
    }
}
```

### IDataTemplateHost接口

```csharp
using Avalonia.Metadata;

namespace Avalonia.Controls.Templates
{
    /// <summary>
    /// Defines an element that has a <see cref="DataTemplates"/> collection.
    /// </summary>
    [NotClientImplementable]
    public interface IDataTemplateHost
    {
        /// <summary>
        /// Gets the data templates for the element.
        /// </summary>
        DataTemplates DataTemplates { get; }

        /// <summary>
        /// Gets a value indicating whether <see cref="DataTemplates"/> is initialized.
        /// </summary>
        /// <remarks>
        /// The <see cref="DataTemplates"/> property may be lazily initialized, if so this property
        /// indicates whether it has been initialized.
        /// </remarks>
        bool IsDataTemplatesInitialized { get; }
    }
}

```

**`IDataTemplateHost` 接口定义了一个可以拥有 `DataTemplates` 集合的元素。** 它提供了两个成员:

1. **`DataTemplates` 属性** - 获取元素的**数据模板集合**
2. **`IsDataTemplatesInitialized` 属性** - 指示 `DataTemplates` 集合**是否已被初始化**(因为它可能是延迟初始化的)

#### 使用场景

```
IDataTemplateHost` 的主要用途是支持**数据模板查找机制**。 当需要为某个数据对象找到合适的视觉表示时,系统会沿着逻辑树向上查找,检查每个 `IDataTemplateHost` 的 `DataTemplates
```

查找顺序如下:

1. 首先检查控件自身的 `DataTemplates`
2. 然后沿着逻辑树向上查找父元素的 `DataTemplates`
3. 最后查找全局的 `Application.DataTemplates`

#### 延迟初始化

`IsDataTemplatesInitialized` 属性的存在是为了**性能优化**。 例如在 `Control` 类中,只有在实际访问 `DataTemplates` 属性时才会创建集合: Control.cs:213 同样,`Application` 类也采用了相同的延迟初始化策略: Application.cs:127 Application.cs:168

`[NotClientImplementable]` 特性表明这个接口不应该由用户代码实现,它是框架内部使用的接口。 用户通常通过继承 `Control` 或使用 `Application` 来间接使用这个接口的功能,而不是直接实现它。

因此，我们可以说，它的作用就是提供一个数据模板的查找连，控件需要为某个数据对象找到合适的视觉表示的时候，系统就会沿着逻辑树向上查找DataTemplates集合。

### DataTemplate类

```csharp
using System;
using Avalonia.Collections;

namespace Avalonia.Controls.Templates
{
    /// <summary>
    /// A collection of <see cref="IDataTemplate"/>s.
    /// </summary>
    public class DataTemplates : AvaloniaList<IDataTemplate>, IAvaloniaListItemValidator<IDataTemplate>
    {
        /// <summary>
        /// Initializes a new instance of the <see cref="DataTemplates"/> class.
        /// </summary>
        public DataTemplates()
        {
            ResetBehavior = ResetBehavior.Remove;
            Validator = this;
        }

        void IAvaloniaListItemValidator<IDataTemplate>.Validate(IDataTemplate item)
        {
            var valid = item switch
            {
                ITypedDataTemplate typed => typed.DataType is not null,
                _ => true
            };
            
            if (!valid)
            {
                throw new InvalidOperationException("DataTemplate inside of DataTemplates must have a DataType set. Set DataType property or use ItemTemplate with single template instead.");
            }
        }
    }
}

```

这是一恶搞专门用于存储IDataTemplate对象的集合类，它继承自`AvaloniaList<IDataTemplate>` 并实现了 `IAvaloniaListItemValidator<IDataTemplate>` 接口,用于在**添加模板时进行验证**。数据模板（Data Templates）在Avalonia中为您提供了定义数据的可视化表示的强大方法。**它们允许您指定数据的展示方式和格式**，从而创建**动态和可定制的用户界面**。本文档将介绍Avalonia中的数据模板概念，并演示如何在应用程序中有效使用它们。

#### 核心功能

1. **集合类型** - `DataTemplates` 是 `AvaloniaList<IDataTemplate>` 的子类,提供了一个可观察的数据模板集合
2. **重置行为** - 构造函数中设置 `ResetBehavior = ResetBehavior.Remove`,这意味着当集合被清空时,会触发移除事件而不是重置事件
3. **验证机制** - 实现了 `IAvaloniaListItemValidator<IDataTemplate>` 接口,在添加模板时自动验证

相关的介绍可以看文档：

https://docs.avaloniaui.net/zh-Hans/docs/basics/data/data-templates

#### 将数据模板应用于ListBox

要将数据模板应用于`ListBox`，通常使用控件的`ItemTemplate`属性。

例如，如果您有一个`ListBox`，应使用定义的数据模板来显示`Item`对象的集合，可以像这样设置`ItemTemplate`属性：

```xml
<ListBox ItemsSource="{Binding Items}">
  <ListBox.ItemTemplate>
    <DataTemplate>
        <StackPanel Orientation="Horizontal">
            <TextBlock Text="{Binding Name}" />
            <Image Source="{Binding ImageSource}" />
        </StackPanel>
    </DataTemplate>
  </ListBox.ItemTemplate>
</ListBox>
```

### IVisualBrushInitialize接口

它主要用于解决一个特定的渲染问题：当一个控件（`Control`）被用作另一个控件的背景或前景（即作为 `VisualBrush` 的来源）时，**它必须被正确地初始化和准备好，才能进行绘图**。

在 Avalonia 中,`Control` 类实现了这个接口: Control.cs:27

`Control` 的实现逻辑如下: Control.cs:228-252

1. **检查是否已附加到视觉树** - 如果控件的 `VisualRoot` 为 null(即未附加到视觉树)
2. **初始化控件** - 如果控件未初始化,遍历自身及所有视觉子元素,调用 `BeginInit()` 和 `EndInit()`
3. **执行布局** - 如果布局无效,执行 `Measure` 和 `Arrange` 以确保控件**有正确的大小和位置**

这是一个内部接口(标记为 `internal`),并且带有 `[Unstable]` 特性,表明它是框架内部使用的 API,不应该由用户代码直接实现。 用户通常通过继承 `Control` 来间接获得这个功能。

### ISetterValue接口

```csharp
#nullable enable

namespace Avalonia.Styling
{
    /// <summary>
    /// Customizes the behavior of a class when added as a value to a <see cref="SetterBase"/>.
    /// </summary>
    public interface ISetterValue
    {
        /// <summary>
        /// Notifies that the object has been added as a setter value.
        /// </summary>
        void Initialize(SetterBase setter);
    }
}

```

**`ISetterValue` 接口用于自定义类在被添加为 `SetterBase` 的值时的行为。** 它只有一个方法 `Initialize(SetterBase setter)`,当对象被添加为 setter 值时会被调用。

#### 核心功能

该接口允许对象在被用作 setter 值时**执行自定义初始化逻辑**。 当一个实现了 `ISetterValue` 的对象被赋值给 `Setter.Value` 时,系统会自动调用其 `Initialize` 方法。

#### 使用场景

在 `Setter` 类中,当设置 `Value` 属性时会检查值是否实现了 `ISetterValue` 接口: Setter.cs:57

如果实现了该接口,就会调用 `Initialize` 方法并传入当前的 setter 实例。 Setter.cs:57

#### 实现示例

`Control` 类实现了 `ISetterValue` 接口: Control.cs:216-225

在 `Control` 的实现中:

- 它检查 setter 是否是为 `ContextFlyoutProperty` 设置值,如果是则允许直接使用控件而不需要包装在 `<Template>` 中 Control.cs:218-220
- 对于其他属性,会抛出异常,要求将控件包装在 `<Template>` 中 Control.cs:223-224

```csharp
 void ISetterValue.Initialize(SetterBase setter)
        {
            if (setter is Setter s && s.Property == ContextFlyoutProperty)
            {
                return; // Allow ContextFlyout to not need wrapping in <Template>
            }

            throw new InvalidOperationException(
                "Cannot use a control as a Setter value. Wrap the control in a <Template>.");
        }
```

这个设计防止了直接将控件实例用作 setter 值的常见错误。 在测试代码中可以看到这个验证: SetterTests.cs:21-26

`ISetterValue` 是样式系统中的一个重要扩展点,它允许特定类型的对象在被用作样式 setter 值时执行自定义逻辑。 最常见的用例是防止错误使用(如 `Control` 的实现)和支持特殊值类型(如绑定表达式)。 这个接口是框架内部使用的,普通应用开发者很少需要直接实现它。



到这里，我们已经清楚了组件基类Controls的继承关系和作用等，接下来我们就会进去高层组件看一下高层组件的实现等。

## 高层组件

### TextBlock

```csharp
public class TextBlock : Control, IInlineHost
```

我们看到他在继承基础的Control类的同时，也继承了IInlineHost接口：

```csharp
using Avalonia.Collections;
using Avalonia.LogicalTree;

namespace Avalonia.Controls.Documents
{
    internal interface IInlineHost : ILogical
    {
        void Invalidate();

        IAvaloniaList<Visual> VisualChildren { get; }
    }
}

```

**`IInlineHost` 是一个内部接口,定义了可以承载内联元素(inline elements)的宿主容器。** 它继承自 `ILogical`,并提供了两个成员:

1. **`Invalidate()` 方法** - 用于通知宿主需要**重新测量或重新渲染**
2. **`VisualChildren` 属性** - 提供**对宿主的视觉子元素集合的访问**

在 Avalonia 中,`TextBlock` 类实现了这个接口:

- **`Invalidate()` 实现** - 调用 `InvalidateMeasure()` 来触发重新测量[link to Repo AvaloniaUI/Avalonia: src/Avalonia.Controls/TextBlock.cs:901-904]
- **`VisualChildren` 实现** - 返回 `TextBlock` 的 `VisualChildren` 集合[link to Repo AvaloniaUI/Avalonia: src/Avalonia.Controls/TextBlock.cs:906-906]

`IInlineHost` 是一个内部接口(标记为 `internal`),不应该由用户代码直接实现。 它是 Avalonia 文档系统(Documents)的一部分,用于实现富文本功能,允许在 `TextBlock` 中混合文本和控件。 这个接口确保了内联元素和宿主之间的松耦合,使得内联元素可以独立于具体的宿主实现。

在 Avalonia 的文档系统中,**内联元素**是指可以在 `TextBlock` 中使用的富文本元素。 主要包括以下几种:

#### 1. Run - 文本运行

`Run` 是最基本的内联元素,用于显示一段文本。 Run.cs:31-51 它可以设置自己的文本内容和样式属性(如字体、颜色等)。

使用示例: TextBlockTests.cs:223-225

#### 2. LineBreak - 换行符

`LineBreak` 用于强制换行。 LineBreak.cs:9-20 它会在文本中插入一个换行符。 LineBreak.cs:22-31

#### 3. Span - 内联容器

`Span` 用于分组其他内联元素。 Span.cs:9-12 它本身也是一个内联元素,但可以包含其他内联元素(包括嵌套的 `Span`)。 Span.cs:23-39

使用示例: TextBlockTests.cs:288

#### 4. InlineUIContainer - 嵌入控件

`InlineUIContainer` 是最特殊的内联元素,它允许在文本流中嵌入任意的 `Control` 控件。 InlineUIContainer.cs:9-13

例如,您可以在文本中嵌入按钮、图片等控件: TextBlockTests.cs:357-369

或者嵌入图片: TextBlockTests.cs:392-400

#### TextBlock的测量逻辑

```csharp
protected override Size MeasureOverride(Size availableSize)
        {
            var scale = LayoutHelper.GetLayoutScale(this);
            var padding = LayoutHelper.RoundLayoutThickness(Padding, scale);
            var deflatedSize = availableSize.Deflate(padding);

            if (_constraint != deflatedSize)
            {
                //Reset TextLayout when the constraint is not matching.
                _textLayout?.Dispose();
                _textLayout = null;
                _constraint = deflatedSize;

                //Force arrange so text will be properly alligned.
                InvalidateArrange();
            }
           
            var inlines = Inlines;

            if (HasComplexContent)
            {
                var textRuns = new List<TextRun>();

                foreach (var inline in inlines!)
                {
                    inline.BuildTextRun(textRuns);
                }

                _textRuns = textRuns;
            }

            //This implicitly recreated the TextLayout with a new constraint if we previously reset it.
            var textLayout = TextLayout;

            // The textWidth used here is matching that TextPresenter uses to measure the text.
            var size = LayoutHelper.RoundLayoutSizeUp(new Size(textLayout.WidthIncludingTrailingWhitespace, textLayout.Height).Inflate(padding), 1);

            return size;
        }
```

下面是这段测量逻辑的详细分解和讲解：

------



#### 1. 预处理与尺寸调整 (Padding and Scaling)

```csharp
var scale = LayoutHelper.GetLayoutScale(this);
var padding = LayoutHelper.RoundLayoutThickness(Padding, scale);
var deflatedSize = availableSize.Deflate(padding);
```

1. **获取 DPI 缩放比例 (`scale`):** 获取当前控件所处的 UI 环境的布局缩放比例（DPI Scale）。这确保了布局计算是**DIPs (设备无关像素)** 而非物理像素。
2. **处理内边距 (`padding`):** 使用 `LayoutHelper.RoundLayoutThickness` 将控件设置的 `Padding` 应用于布局缩放，并进行四舍五入以避免布局中的子像素问题。
3. **计算可用内容空间 (`deflatedSize`):** 将内边距从父容器提供的 `availableSize` 中减去（`Deflate`）。这个 `deflatedSize` 就是**实际可用于容纳文本内容**的空间。

------



#### 2. 文本布局缓存和更新 (TextLayout Caching)

```csharp
if (_constraint != deflatedSize)
{
    //Reset TextLayout when the constraint is not matching.
    _textLayout?.Dispose();
    _textLayout = null;
    _constraint = deflatedSize;

    //Force arrange so text will be properly alligned.
    InvalidateArrange();
}
```

这是提高性能的关键步骤：

1. **检查约束变化：** 比较当前的内容可用空间 `deflatedSize` **是否与上次测量使用的约束** `_constraint` 相同。
2. **重新创建 `TextLayout`：**
   - 如果约束**发生变化**（例如父容器尺寸改变），说明文本的排版（如换行）可能需要重新计算。
   - 因此，它会 **销毁 (`Dispose`)** 旧的 `_textLayout` 对象，并将其设置为 `null`。
   - 同时，将 `_constraint` 更新为新的 `deflatedSize`。
3. **强制排列 (`InvalidateArrange`):** 调用 `InvalidateArrange()` 来**强制触发**后续的 **排列阶段 (Arrange Pass)**。这是因为即使测量结果（`MeasureOverride` 的返回值）可能不变，但由于约束变了，文本的**内部对齐或渲染位置**可能需要调整。

------



#### 3. 准备文本运行 (Prepare Text Runs)

```csharp
var inlines = Inlines;

if (HasComplexContent)
{
    var textRuns = new List<TextRun>();

    foreach (var inline in inlines!)
    {
        inline.BuildTextRun(textRuns);
    }

    _textRuns = textRuns;
}
```

1. **处理复杂内容 (`HasComplexContent`):** 这段代码处理的是控件可能包含**富文本 (Rich Text)** 或 **内联元素 (Inlines)** 的情况。
2. **构建 `TextRun` 集合：** 如果是复杂内容，它会遍历所有的内联元素（如 `Bold`, `Italic`, 或其他内嵌控件），并将它们转换为 **`TextRun`** 对象的列表。`TextRun` 是渲染引擎理解的最小文本单元，包含文本内容和其格式（字体、颜色等）。
3. **缓存 `_textRuns`：** 将构建好的 `textRuns` 缓存起来，供下一步创建 `TextLayout` 使用。

------



#### 4. 获取文本布局结果 (Determine Desired Size)

```csharp
//This implicitly recreated the TextLayout with a new constraint if we previously reset it.
var textLayout = TextLayout;

// The textWidth used here is matching that TextPresenter uses to measure the text.
var size = LayoutHelper.RoundLayoutSizeUp(new Size(textLayout.WidthIncludingTrailingWhitespace, textLayout.Height).Inflate(padding), 1);

return size;
```

1. **创建/获取 `TextLayout`：** `TextLayout` 属性的 getter 会检查 `_textLayout` 是否为 `null`。如果为 `null`（在第 2 步中被重置），它会使用最新的 `_constraint` 和 `_textRuns` 来**隐式地创建**一个新的 `TextLayout` 对象。这个对象包含了文本实际排版后的尺寸信息。
2. **计算最终尺寸 (`size`):**
   - 它从 `textLayout` 中获取文本内容的**实际排版宽度** (`WidthIncludingTrailingWhitespace`) 和**高度** (`Height`)。
   - 使用 `Inflate(padding)` **加回**控件的内边距，得到**包含内边距**的控件所需总尺寸。
   - 使用 `LayoutHelper.RoundLayoutSizeUp` 对最终尺寸进行向上取整和缩放，以确保尺寸兼容 DPI 缩放且没有舍入错误。
3. **返回结果：** 将计算出的 `size` 返回。这是控件在当前约束下，为容纳其内容**所需的最小尺寸**。



#### 总结布局流程

这段代码完美体现了 Avalonia 的**文本布局**和 **两步布局 (Two-Pass Layout)** 机制：

- **Measure 阶段 (`MeasureOverride`):** 确定**需要多大**。通过 `TextLayout` 计算内容所需的尺寸，并返回。
- **Arrange 阶段 (未显示):** 确定**放在哪里，最终多大**。在 `ArrangeOverride` 中，控件将使用 `MeasureOverride` 确定的理想尺寸（或父容器分配给它的最终尺寸）来**定位**其内部的内容（文本），这就是为什么在约束变化时要调用 `InvalidateArrange()` **来确保文本对齐正确**。

#### 脏组件判断触发测量和渲染

```csharp
 protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
        {
            base.OnPropertyChanged(change);

            if (change.Property == TextProperty)
            {
                if (HasComplexContent && !_clearTextInternal)
                {
                    Inlines?.Clear();
                }
            }

            switch (change.Property.Name)
            {
                case nameof(FontSize):
                case nameof(FontWeight):
                case nameof(FontStyle):
                case nameof(FontFamily):
                case nameof(FontStretch):

                case nameof(TextWrapping):
                case nameof(TextTrimming):
                case nameof(TextAlignment):

                case nameof(FlowDirection):

                case nameof(Padding):
                case nameof(LineHeight):
                case nameof(LetterSpacing):
                case nameof(MaxLines):

                case nameof(Text):
                case nameof(TextDecorations):
                case nameof(FontFeatures):
                case nameof(Foreground):
                    {
                        InvalidateTextLayout();
                        break;
                    }
                case nameof(Inlines):
                    {
                        OnInlinesChanged(change.OldValue as InlineCollection, change.NewValue as InlineCollection);
                        InvalidateTextLayout();
                        break;
                    }
            }
        }
```

这段代码的目的是在控件的**依赖属性 (`StyledProperty`) 发生变化**时做出反应，特别是针对那些影响文本渲染和布局的关键属性。它通过调用 `InvalidateTextLayout()` 来标记控件的文本布局为“脏”（Dirty），从而触发重新的测量和渲染。

当任何依赖属性（如 `FontSize`, `Text`, `Padding` 等）的值通过绑定、样式或直接代码设置而改变时，这个 `OnPropertyChanged` 方法就会被调用。

**布局/渲染管线：** Avalonia 的**调度器**会在下一个主循环中，按照顺序执行**全局**的测量、排列和渲染阶段，只处理那些被标记为“脏”的控件。

### ContentControl

这是一个模块化的控件，用于根据数据末班来显示内容，它继承自 `TemplatedControl` 并实现了两个接口: ContentControl.cs:20

#### 实现的接口

1. `IContentControl`

    

   \- 定义了使用模板显示内容的控件的契约。

    

   该接口要求实现:

   - `Content` 属性(要显示的内容) IContentControl.cs:16
   - `ContentTemplate` 属性(用于渲染的数据模板) IContentControl.cs:21
   - `HorizontalContentAlignment` 和 `VerticalContentAlignment` 属性(内容对齐方式) IContentControl.cs:26-31

2. `IContentPresenterHost`

    

   \- 定义了承载ContentPresenter的控件的接口。

    

   该接口负责:

   - 通过 `LogicalChildren` 属性管理逻辑子元素 IContentPresenterHost.cs:22
   - 通过 `RegisterContentPresenter()` 方法注册 `ContentPresenter` 实例 IContentPresenterHost.cs:32

#### 核心功能

该类使用名为 `"PART_ContentPresenter"` 的模板部件来**渲染其内容**。 ContentControl.cs:19 默认模板会创建一个 `ContentPresenter`,并将**其属性绑定到控件的相应属性**。 ContentControl.cs:48-61

当 `ContentPresenter` 注册自己时,如果它的名称正确,控件会将其存储在 `Presenter` 属性中。 ContentControl.cs:136-139

#### 主要属性

- `Content` - 要显示的**内容对象** ContentControl.cs:69-72
- `ContentTemplate` - 用于显示内容的**数据模板** ContentControl.cs:78-81
- `Presenter` - 从控件模板中获取的**内容呈现器** ContentControl.cs:87-91
- `HorizontalContentAlignment` 和 `VerticalContentAlignment` - 控制内容在控件内的对齐方式 ContentControl.cs:96-108

### TemplatedControl

`TemplatedControl` 是一个**无外观控件**(lookless control),其**视觉外观**由 `Template` 属性定义。 TemplatedControl.cs:14-17 它直接继承自 `Control` 类。 TemplatedControl.cs:17

#### 外观属性

- `Background` - 控件的背景画刷 TemplatedControl.cs:22-23
- `BorderBrush` - 边框画刷 TemplatedControl.cs:34-35
- `BorderThickness` - 边框粗细 TemplatedControl.cs:40-41
- `CornerRadius` - 圆角半径 TemplatedControl.cs:46-47
- `Padding` - 内边距 TemplatedControl.cs:94-95

#### 文本属性

- `FontFamily`、`FontSize`、`FontStyle`、`FontWeight`、`FontStretch` - 字体相关属性 TemplatedControl.cs:52-83
- `Foreground` - **前景色画刷** TemplatedControl.cs:88-89

#### 模板属性

- `Template` - **定义控件外观的控件模板** TemplatedControl.cs:100-101

#### 附加属性

`IsTemplateFocusTarget` - 附加属性**,用于指定模板中哪个元素应该接收焦点装饰器**。 TemplatedControl.cs:106-107 当控件获得焦点时,**焦点装饰器会显示在标记了此属性的元素周围,而不是控件本身**。 TemplatedControl.cs:280-284

#### 备注

`TemplatedControl` 是许多 Avalonia 控件的基类,包括之前讨论的 `ContentControl`。 它提供了模板化控件的基础架构,使得控件的逻辑和外观可以完全分离。 所有模板子元素的 `TemplatedParent` 属性都会被设置为该控件实例。 TemplatedControlTests.cs:94-120

### IContentControl

`IContentControl` 是一个内部接口,定义了根据 `FuncDataTemplate` **显示内容的控件的契约**。 IContentControl.cs:7-10

#### 1. Content 属性

用于**获取或设置要显示的内容对象**。 IContentControl.cs:13-16 这个属性可以是任何对象类型(`object?`),允许显示各种类型的内容。

#### 2. ContentTemplate 属性

用于**获取或设置显示控件内容的数据模板**。 IContentControl.cs:18-21 该属性类型为 `IDataTemplate?`,用于定义如何将 `Content` 渲染为可视化元素。

#### 3. HorizontalContentAlignment 属性

用于**获取或设置内容在控件内的水平对齐方式**。 IContentControl.cs:23-26 使用 `HorizontalAlignment` 枚举类型。

#### 4. VerticalContentAlignment 属性

用于**获取或设置内容在控件内的垂直对齐方式**。 IContentControl.cs:28-31 使用 `VerticalAlignment` 枚举类型。

#### 实现类

`ContentControl` 类是该接口的主要实现者。 ContentControl.cs:20 它将这些接口属性实现为样式化属性(StyledProperty),**并提供了完整的功能实现**:

- `ContentProperty` - 定义为 `StyledProperty<object?>` ContentControl.cs:25-26
- `ContentTemplateProperty` - 定义为 `StyledProperty<IDataTemplate?>` ContentControl.cs:31-32
- `HorizontalContentAlignmentProperty` - 定义为 `StyledProperty<HorizontalAlignment>` ContentControl.cs:37-38
- `VerticalContentAlignmentProperty` - 定义为 `StyledProperty<VerticalAlignment>` ContentControl.cs:43-44

### ICommandSource

`ICommandSource` 是一个公共接口,**定义了知道如何调用命令的类的契约**。 ICommandSource.cs:5-7 这个接口是 Avalonia **命令模式实现的核心部分**,允许控件与 `ICommand` 对象进行交互。 ICommandSource.cs:1-2

#### 接口成员

#### 1. Command 属性

获取当类被"调用"时将执行的命令。 ICommandSource.cs:10-15 实现此接口的类应该根据命令的 `CanExecute` 返回值来启用或禁用自身。 ICommandSource.cs:12 **该属性可以根据需要实现为读写属性**。 ICommandSource.cs:13

#### 2. CommandParameter 属性

获取**执行命令时将传递给命令的参数**。 ICommandSource.cs:17-21 该属性也**可以根据需要实现为读写属性**。 ICommandSource.cs:19

#### 3. CanExecuteChanged 方法

当检测到变化时为 `CanExecuteChanged` 事件调用。 ICommandSource.cs:23-28 该方法**接受事件发送者和事件参数作为参数**。 ICommandSource.cs:26-27

#### 4. IsEffectivelyEnabled 属性

获取一个值,**指示此控件及其所有父控件是否已启用**。 ICommandSource.cs:30-33

#### 实现类

#### Button

`Button` 类是该接口的主要实现者之一。 Button.cs:35 它实现了完整的命令模式功能:

- 定义了 `CommandProperty` 和 `CommandParameterProperty` 样式化属性 Button.cs:51-64
- 在命令的 `CanExecuteChanged` 事件触发时更新控件的启用状态 Button.cs:590-610
- 在附加到逻辑树时订阅命令的 `CanExecuteChanged` 事件 Button.cs:477-492

#### MenuItem

`MenuItem` 类也实现了此接口。 MenuItem.cs:25 它的实现包括:

- 定义了 `CommandProperty` 和 `CommandParameterProperty` MenuItem.cs:33-46
- 实现了性能优化:仅在菜单打开时才调用 `CanExecute` MenuItem.cs:662-667
- 显式实现了 `ICommandSource.CanExecuteChanged` 方法 MenuItem.cs:885

#### SplitButton

`SplitButton` 也实现了该接口。 SplitButton.cs:134 它的实现方式与 `Button` 类似,在命令状态改变时更新控件的启用状态。 SplitButton.cs:137-151

#### NativeMenuItem

`NativeMenuItem` 类也使用了命令模式。 NativeMenuItem.cs:133-138 它定义了 `CommandProperty` 和 `CommandParameterProperty`,并在 `CanExecuteChanged` 方法中更新 `IsEnabled` 属性。 NativeMenuItem.cs:166-172

#### 与热键管理器的集成

`HotKeyManager` 使用 `ICommandSource` 接口来处理热键绑定。 HotkeyManager.cs:27-29 当控件实现了 `ICommandSource` 接口时,热键管理器会:

- 检查命令是否可以执行 HotkeyManager.cs:43-46
- 使用 `CommandParameter` 执行命令 HotkeyManager.cs:60-62
- 验证控件是否实现了 `ICommandSource` 或 `IClickableControl` HotkeyManager.cs:152

```csharp
    [PseudoClasses(pcFlyoutOpen, pcPressed)]
    public class Button : ContentControl, ICommandSource, IClickableControl
    {
        private const string pcPressed = ":pressed";
        private const string pcFlyoutOpen = ":flyout-open";
```

`[PseudoClasses(pcFlyoutOpen, pcPressed)]` 是一个特性(Attribute),用于**声明该控件支持的伪类**。 Button.cs:34 这个特性定义在 `PseudoClassesAttribute` 类中,主要**用于 IDE 的代码补全功能**。

这两个常量**定义了伪类**的字符串名称: Button.cs:37-38

- `pcPressed = ":pressed"` - 按钮按下状态
- `pcFlyoutOpen = ":flyout-open"` - 浮出菜单打开状态

## 渲染逻辑

Avalonia 使用**双线程架构**,将 UI 线程和渲染线程分离: Compositor.cs:19-22 `Compositor` 类**管理 UI 线程**和**渲染线程**之间的通信。 Compositor.cs:25-27

### UI线程

它负责处理布局、动画和用户交互。然后通过MediaContext来调度渲染，需要重新绘制的时候会调`InvalidateVisual()` 标记为脏: Visual.cs:377-380 。

UI线程的更改会被打包成`CompositionBatch` 发送到渲染线程

```csharp
 private void RenderCore()
    {
        var now = _time.Elapsed;
        if (!_animationsAreWaitingForComposition)
            _clock.Pulse(now);

        // Since new animations could be started during the layout and can affect layout/render
        // We are doing several iterations when it happens
        for (var c = 0; c < 10; c++)
        {
            FireInvokeOnRenderCallbacks();
            
            if (_clock.HasNewSubscriptions)
            {
                _clock.PulseNewSubscriptions();
                continue;
            }

            break;
        }
        
        if (_requestedCommits.Count > 0 || _clock.HasSubscriptions)
        {
            _animationsAreWaitingForComposition = CommitCompositorsWithThrottling();
            if (!_animationsAreWaitingForComposition && _clock.HasSubscriptions) 
                _animationsTimer.Start();
        }
    }
```

渲染线程通过 `ServerCompositor` 处理批次并**执行实际渲染**:

```csharp
      private void RenderCore(bool catchExceptions)
        {
            UpdateServerTime();
            ApplyPendingBatches();
            NotifyBatchesProcessed();

            Animations.Process();


            ApplyEnqueuedRenderResourceChanges();
            
            try
            {
                if(!RenderInterface.IsReady)
                    return;
                RenderInterface.EnsureValidBackendContext();
                ExecuteServerJobs(_receivedJobQueue);
                foreach (var t in _activeTargets)
                    t.Render();
                ExecuteServerJobs(_receivedPostTargetJobQueue);
            }
            catch (Exception e) when(RT_OnContextLostExceptionFilterObserver(e) && catchExceptions)
            {
                Logger.TryGet(LogEventLevel.Error, LogArea.Visual)?.Log(this, "Exception when rendering: {Error}", e);
            }
        }
```

渲染流程包括:

1. 应用待处理的批次(`ApplyPendingBatches`)
2. 处理动画(`Animations.Process`)
3. 遍历**所有活动的渲染目标并渲染**(`t.Render()`)

每个可视化组件通过 `ServerCompositionVisual` 在**渲染线程上执行绘制**: ServerCompositionVisual.cs:61-94

绘制操作最终通过 `IDrawingContextImpl` 接口**调用底层渲染后端**: IDrawingContextImpl.cs:14-25

```csharp
    /// Defines the interface through which drawing occurs.
    /// </summary>
    [Unstable]
    public interface IDrawingContextImpl : IDisposable
    {
        /// <summary>
        /// Gets or sets the current transform of the drawing context.
        /// </summary>
        Matrix Transform { get; set; }

        /// <summary>
        /// Clears the render target to the specified color.
        /// </summary>
        /// <param name="color">The color.</param>
        void Clear(Color color);
```

### 线程模式选项

Avalonia 支持两种渲染模式:

### 后台渲染线程(默认)

大多数平台使用独立的渲染线程,通过 `IRenderLoop` 驱动。 Compositor.cs:25-26

### UI 线程渲染

某些场景下可以**配置在 UI 线程上渲染**: Win32PlatformOptions.cs:145-150

```csharp
    /// <summary>
    /// Render directly on the UI thread instead of using a dedicated render thread.
    /// Only applicable if <see cref="CompositionMode"/> is set to <see cref="Win32CompositionMode.RedirectionSurface"/>.
    /// This setting is only recommended for interop with systems that must render on the UI thread, such as WPF.
    /// This setting is false by default.
    /// </summary>
    public bool ShouldRenderOnUIThread { get; set; }
```

这种模式下,**渲染会在 UI 线程上同步执行**: ServerCompositor.cs:172-191

```csharp
       public void Render(bool catchExceptions)
        {
            if (Dispatcher.UIThread.CheckAccess())
            {
                if (_uiThreadIsInsideRender)
                    throw new InvalidOperationException("Reentrancy is not supported");
                _uiThreadIsInsideRender = true;
                try
                {
                    using (Dispatcher.UIThread.DisableProcessing()) 
                        RenderReentrancySafe(catchExceptions);
                }
                finally
                {
                    _uiThreadIsInsideRender = false;
                }
            }
            else
                RenderReentrancySafe(catchExceptions);
        }
```

并且Avalonia也支持在浏览器环境中渲染，可使用Web Worker作为渲染线程。

**不是所有组件直接绑定到渲染线程**。实际上,组件在 UI 线程上进行布局和状态更新,然后将渲染指令批量发送到渲染线程。渲染线程独立执行绘制操作,这种设计避免了 UI 线程阻塞,提高了性能和响应性。同步等待机制(`SyncWaitCompositorBatch`)用于需要立即渲染的场景,如窗口调整大小。 MediaContext.Compositor.cs:98-114

在Linux平台，有一个 `ShouldRenderOnUIThread` 属性 X11Platform.cs:325-329 。其注释明确指出，**启用此选项可以在UI线程上直接渲染**，这在设备**没有多核处理器时可能有用** X11Platform.cs:325-329 。默认情况下，此设置为 `false` X11Platform.cs:325-329 。

虽然但是，直接在UI线程上进行渲染的话这个会导致UI线程阻塞，进而导致无法交互的作用产生。Avalonia框架默认通过将**渲染任务卸载到单独的渲染线程**来避免这种情况 ServerCompositor.cs:14-19 。只有在特定平台选项中明确启用 `ShouldRenderOnUIThread` 时，才会发生UI线程渲染，并且通常不推荐这样做，除非有特定的互操作性需求 Win32PlatformOptions.cs:145-150 。

`Dispatcher.UIThread` 是Avalonia中用于**访问UI线程的单例** Dispatcher.cs:47-54 。`CheckAccess()` 方法**用于判断当前线程是否是UI线程** Dispatcher.cs:76 。`InvokeAsync` 和 `Invoke` 方法允许在UI线程上执行操作 Dispatcher.Invoke.cs:61-63 Dispatcher.Invoke.cs:65-71 。这些方法**在渲染线程分离的场景下**，可以用于**将UI更新安全地调度回UI线程**。

```csharp
// 在UI线程上  
compositor.PostServerJob(() =>  
{  
    // 这段代码将在渲染线程上执行  
    Console.WriteLine("Hello from render thread!");  
});
```

