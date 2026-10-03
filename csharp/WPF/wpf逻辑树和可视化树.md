​	在了解了wpf的基本的消息逻辑和渲染逻辑之后，我们可以来看看wpf的**视图层**，也就是要理解逻辑树和可视化树。

现在先来看一个简单的窗口：

```xml
<Window x:Class="SimpleWindow.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MainWindow" Height="301.316" Width="306.579">
    <StackPanel Margin="5">
        <Button Padding="5" Margin="5" Click="cmd_Click">First Button</Button>
        <Button Padding="5" Margin="5" Click="cmd_Click">Second Button</Button>
    </StackPanel>
</Window>
```

​	添加的元素分类称为逻辑树。WPF编程人员需要耗费大部分时间**构建逻辑树**，然后使用事件处理代码支持他们。实际上，到目前为止介绍的所有WPF特性(如**属性值继承、事件路由以及样式**)都是通过**逻辑树**进行工作的。

![img](263728-20200311194022112-1514806260.png)

​	然而，如果希望自定义元素，逻辑树起不到多大帮助作用。显然，可使用另一个元素替换整个元素(例如，可使用自定义的FacyButton类替换当前的Button类)，但这需要更多工作，并且可能扰乱应用程序的用户界面或代码。因此，WPF通过可视化树进入更深层次。

​	可视化树是逻辑树的扩展版本。它将**元素分为更小的部分**，换句话说，它并不查看被精心封装到一起的黑色方框，如按钮，**而是查看按钮的可视化元素**——使按钮具有阴影背景特性的边框(由ButtonChrome类表示)、内部的容器(ContentPresenter对象)以及存储按钮文本的块(由大家熟悉的TextBlock表示)。下图显示上面示例的可视化树。

![img](263728-20200311195041451-1789134477.png)

​	对于这两种树的区别，我们现在来进行一个更加细致的对比，来清晰他们的职责。

### **区别与联系的详细讲解**

#### **逻辑树 (Logical Tree) - “设计蓝图”**

- **是什么**: 它是 UI 的**高级、概念性**表示，是开发者眼中的 UI 结构。
- **作用**: WPF 主要通过逻辑树来实现一些高级功能，例如：
  - **属性继承**: `DataContext`, `FontSize` 等属性会沿着逻辑树向下传递给子元素。
  - **资源查找**: 当您使用 `{StaticResource}` 时，WPF 会沿着**逻辑树向上查找**。
  - **数据绑定**: `ElementName` 绑定是在逻辑树中进行的。



#### **可视化树 (Visual Tree) - “建筑实体”**

- **是什么**: 它是 UI 的**低级、完整的物理表示**，是 WPF 渲染引擎眼中的 UI 结构。
- **作用**: 可视化树是 WPF 完成实际工作的核心依据：
  - **渲染 (Rendering)**: 这是它最重要的工作。WPF 的渲染引擎会遍历可视化树，将每个视觉元素绘制到屏幕上。
  - **事件路由 (Event Routing)**: 鼠标、键盘等输入事件的**隧穿 (Tunneling)** 和**冒泡 (Bubbling)** 过程，是沿着可视化树进行的。
  - **命中测试 (Hit Testing)**: 当您移动或点击鼠标时，WPF 通过检查可视化树来判断您的指针具体悬停在哪一个视觉元素上。
  - **控件模板 (ControlTemplate)**: 可视化树的结构正是由控件的模板定义的。您可以通过修改模板来完全改变一个控件在可视化树中的结构，从而改变其外观。

​	**可视化样式改变可视化树中的元素。**可使用Style.TargetType熟悉选择**希望修改的特定元素**。甚至当控件属性发生变化时，可使用触发器自动完成更改。不过，某些特定的细节很难甚至无法修改。

​	**可为控件创建新模板**。对于这种情况，控件模板将被用于按期望的方式构建可视化树。

​	非常有趣的是，WPF提供了用于**浏览逻辑树和可视化树**的两个类：System.Windows.LogicalTreeHelper和System.Windows.Media.VisualTreeHelper。

　　LogicalTreeHelper类允许通过**动态加载XAML文档**在WPF应用程序中**关联事件处理程序**。LogicalTreeHelper类提供了较少的方法，下表列出了这些方法。尽管这些方法偶尔很有用，但大多数情况下回改用特定的FrameworkElement类中的方法。

表 LogicalTreeHelper类的方法

| 名  称            | 说   明                                                      |
| ----------------- | ------------------------------------------------------------ |
| FindLogicalNode() | 根据名称查找特定元素，从指定的元素开始并**向下查找逻辑树**   |
| BringIntoView()   | 如果元素在可滚动的容器中，并且当前不可见，就将**元素滚动到试图**中。FrameworkElement.BringIntoView()方法执行相同的工作 |
| GetPrarent()      | 获取指定元素的**父元素**                                     |
| GetChildren()     | 获取指定元素的**子元素**。不同元素支持不同的内容模型。例如，面板支持多个子元素，而内容控件只支持一个子元素。然而，GetChildren()方法抽象了这一区别，并且**可使用任何类型的元素进行工作** |

​	除了抓们用来执行低级绘图操作的一些方法外，VisualTreeHelper类提供的方法与LogicalTreeHelper类提供的方法**类似**，也提供了GetChildrenCount()、GetChild()以及GetParent()方法。

​	VisualTreeHelper类还提供了一种研究应用程序中可视化树的有趣方法。**使用GetChild()方法**，可以**遍历任意窗口的可视化树**，并且为了进行分析可以将它们显示出来。这是一种非常好的学习工具，只需要使用一些递归的代码就可以实现。



​	然后接下来我们就可以来聊一下控件类的组成和继承关系了。在WPF中所有的控件都是继承**DispatcherObject类**，可以说在wpf中**DispatcherObject是所有控件类的基类**，而**DispatcherObject却继承Object**，而它所在的**程序集是在WindowsBase.dll里**。

看一张图，wpf控件继承关系图

![img](965851-20240128131845714-26958805.png)

再开始学习类的功能和逐步的继承之前，来聊一下我们之前提到的可视化树和逻辑树。

**`Visual` - 可视化树的“入场券”**

- 当继承到 `Visual` 这一层时，一个对象才开始**具备了被渲染的能力**（例如知道自己的**尺寸、位置、如何绘制等**）。
- 因此，**可视化树 (Visual Tree) 就是由所有派生自 `Visual` 的类的实例所组成的结构**。

**`FrameworkElement` - 逻辑树的“入场券”**

- `FrameworkElement` 在 `UIElement` 的基础上，引入了大量**高级的框架级概念**，如 `DataContext`, `Style`, `Margin`, `Name`, `Binding` 等。
- 这些概念大多与**应用结构、数据流和布局逻辑**相关，而不是纯粹的**视觉渲染**。
- 因此，**逻辑树 (Logical Tree) 主要是由派生自 `FrameworkElement` 的类的实例组成的**。
- 逻辑树代表了开发者在 **XAML 中声明的、富有“逻辑”意义的控件结构**。


### **Shape类**

​	形状控件是WPF一大系列控件。**WPF所有的形状控件都继承于Shape基类**。Shape是一个抽象基类，**它不能被实例化**，所以我们在使用时**只能实例化它的子类**。而**Shape的父类是FrameworkElement**，所以，**所有的Shape子类都是一个UIElement 类**，因此形状对象**可以用在面板和大多数控件中**。

![img](965851-20240128132520082-1527850763.png)

| Path路径      | Path只有一个Data属性，这个属性的类型为Geometry。而Geometry又是一个抽象类，所以我们不能直接使用它，那它肯定会有一系列可以实例化的子类。没错，Geometry表示一个几何 |
| ------------- | ------------------------------------------------------------ |
| Polygon多边形 | Polygon多边形，与Polyline类似，都有一个Points属性，只不过，Polygon会把起点和终点连接起来 |
| Polyline折线  | Polyline表示由一系列线段组合绘制而成的折线，因为它有一个Points属性，用来保存这些点的坐标。这些坐标点用于绘制Polyline图形中各线段相接处的顶点。集合中第一个元素表示起点，最后一元素表示终点 |
| Rectangle矩形 | Rectangle是一个比较简单而实用的图形控件，继承于Shape，有两个属性比较常用，即RadiusX和RadiusY，表示设置矩形的圆角。所以，通过这两个属性的设置，矩形也可以画出一个圆 |
| Ellipse椭圆形 | Ellipse继承于Shape，Shape继承于FrameworkElement，所以，它可以设置其 Width 和 Height。 使用其 Fill 属性指定用于绘制椭圆形内部的 Brush。 使用其 Stroke 属性指定用于绘制椭圆形轮廓的 Brush。 StrokeThickness 属性指定椭圆形轮廓的粗细 |
| Line线段      | Line(线段)继承于Shape，它自身只有4个属性，分别用于定义线段两端的端点坐标 |

### **Control类**

​    Control是许多控件的基类。比如**最常见的按钮（Button）**、**单选(RadioButton)**、**复选（CheckBox）**、**文本框（TextBox）**、ListBox、DataGrid、日期控件等等。这些控件通常用于展示程序的数据或获取用户输入的数据，我们可以将这一类型的控件称为内容控件或数据控件，它们与前面的布局控件有一定的区别，布局控件更专注于界面，而内容控件更专注于数据（业务）。

​    **Control类虽然可以实例化，但是在界面上是不会有任何显示的**。只有那些**继承了Control的子类（控件）才会在界面上显示**，而且所呈现的样子各不相同，为什么会是这样呢？

​    因为Control类**提供了一个控件模板（ControlTemplate）**，而几乎所有的子类都**对这个ControlTemplate进行了各自的实现**，所以在呈现子类时，我们才会看到Button拥有Button的样子，TextBox拥有TextBox的样子

### **ItemsControl类**

​	ItemsControl 用于**生成内容的集合**，也就是说如果要**显示大量的数据以列表的形式展示**，那么可使用集合控件，集合控件都是继承ItemsControl 类

| 控件名       | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| ItemsControl | 集合控件的基类，**本身也是一个可以实例化的控件**             |
| ListBox      | 一个列表集合控件                                             |
| ListView     | 表示用于显示数据项列表的控件，它可以有列头标题               |
| DataGrid     | 表示可自定义的网格中显示数据的控件。                         |
| ComboBox     | 表示带有**下拉列表的选择控件**，通过单击控件上的箭头可显示或隐藏下拉列表。 |
| TabControl   | 表示**包含多个共享相同的空间在屏幕上的项的控件**。           |
| TreeView     | 用树结构(其中的项可以展开和折叠)中显示分层数据的控件         |
| Menu         | 表示一个 Windows 菜单控件，该控件可用于按层次组织与命令和事件处理程序关联的元素。 |
| ContextMenu  | 表示使控件能够公开特定于控件的上下文的功能的弹出菜单。       |
| StatusBar    | 表示应用程序窗口中的**水平栏中显示项和信息的控件**。         |

**ContentControl类**

​	ContentControl是一个内容控件，它有一个Content属性，该属性的类型是object，它可以接收任意引用类型的实例。

| 控件名                  | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| Button按钮              | Button因为继承了ButtonBase，而ButtonBase又继承了ContentControl，所以，Button可以通过设置Content属性来设置要显示的内容 |
| ToggleButton基类        | ToggleButton为CheckBox（复选框）和RadioButton（单选框）的基类 |
| CheckBox复选框          | CheckBox继承于ToggleButton，而ToggleButton才继承于ButtonBase基类 |
| RadioButton单选框       | RadioButton也继承于ToggleButton，作用是单项选择，所以被称为单选框。本质上，它依然是一个按钮，一旦被选中，不会清除，除非它”旁边“的单选框被选中 |
| RepeatButton重复按钮    | RepeatButton,顾名思义，重复执行的按钮。就是当按钮被按下时，所订阅的回调函数会不断被执行 |
| Label标签               | Label控件继承于ContentControl控件，它是一个文本标签，如果您想修改它的标签内容，请设置Content属性 |
| TextBlock文字块         | TextBlock是专业处理文本显示的控件，在功能上比Label更全面     |
| TextBox文本框           | TextBox用来获取用户的键盘输入的信息，这也是一个常用的控件。它继承于TextBoxBase，而TextBoxBase又继承于Control |
| RichTextBox富文本框     | RichTextBox继承于TextBoxBase基类，所以很大程度上与TextBox控件类似，两者在某些情况下可以互相替换。但是，如果要为用户提供更强大的文档编辑功能，非RichTextBox莫属 |
| ToolTip控件（提示工具） | ToolTip控件继承于ContentControl，它不能有逻辑或视觉父级，意思是说，它不能单独存在于WPF的视觉树上（不能以控件的形式实例化），它必须依附于某个控件。因为它的功能被设计成提示信息，当鼠标移动到某个控件上方时，悬停一会儿，就会显示这个ToolTip的内容。 |
| Popup弹出窗口           | Popup类似于ToolTip，在指定的元素或窗体中弹出一个具有任意内容的窗口。Popup继承于FrameworkElement，算得上是独门独户的控件，因为大多数控件都是从Shape、Control或Panel三个类继承而来 |
| Image图像控件           | Image也算是独门独户的控件，因为它也是直接继承于FrameworkElement基类 |
| GroupBox标题容器控件    | GroupBox控件的功能是提供一个带标题的内容容器，它继承于HeaderedContentControl类，HeaderedContentControl继承于ContentControl类。通常它用来做一些局部的布局 |
| ScrollViewer控件        | **ScrollViewer控件封装了一个水平滚动条ScrollBar和一个垂直滚动条ScrollBar，ScrollViewer就是一个包含其它可视元素的可滚动区域控件**。ScrollViewer继承于ContentControl，所以它也是一个内容控件，只能在Content属性中设置一个子元素，如果要在ScrollViewer中显示多个子元素，请设置一个集合控件。 |
| ScrollBar滚动条         | **ScrollBar表示一个滚动条，该滚动条具有一个滑动 Thumb**，其位置对应于一个值。它继承于RangeBase抽象基类，RangeBase基类继承于Control基类。带滚动特质的还有两个控件，也继承于RangeBase抽象基类，它们分别是ProgressBar（进度条）和Slider（滑动条） |
| Slider滑动条            | **Slider滑动条与ScrollBar滚动条有点相似，甚至某些情况下，两者还可以互换使用**。Slider也继承于RangeBase基类，其功能是提供一个可以滑动取值的控件。 |
| ProgressBar进度条       | ProgressBar进度条通常在我们执行某个任务需要花费大量时间时使用，这时可以采用进度条显示任务或线程的执行进度，以便给用户良好的使用体验。 |
| Calendar日历控件        | Calendar提供一个日历界面，供用户选择日期，它继承于Control基类 |
| DatePicker日期控件      | DatePicker与Calender在某些属性上很相似，只是为了方便显示和操作，DatePicker将Calender进行了封装 |
| Expander折叠控件        | Expander也是一个内容控件，它有一个标题属性和内容属性         |
| MediaElement媒体播放器  | MediaElement，一个可以播放音频或视频的控件，继承于FrameworkElement基类 |

### **Panel类**

​	Panel其实是一个**抽象类**，不可以实例化，**WPF所有的布局控件都从Panel继承而来**

| 控件名称    | 布局方式                                                     |
| ----------- | ------------------------------------------------------------ |
| Grid        | 网格，根据自定义行和列来设置控件的布局                       |
| StackPanel  | 栈式面板，包含的元素在竖直或水平方向排成一条直线             |
| WrapPanel   | 自动折行面板，包含的元素在排满一行后，自动换行               |
| DockPanel   | 泊靠式面板，内部的元素可以选择泊靠方向                       |
| UniformGrid | 网格,UniformGrid就是Grid的简化版，每个单元格的大小相同。     |
| Canvas      | 画布，内部元素根据像素为单位绝对坐标进行定位                 |
| Border      | 装饰的控件，此控件用于绘制边框及背景，在Border中只能有一个子控件 |

### **FrameworkElement类([FrameworkElement 类 (System.Windows) | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.frameworkelement?view=windowsdesktop-8.0))**

​	FrameworkElement类**继承于UIElement类**，继承关系是：Object->DispatcherObject->DependencyObject->Visual->UIElement->FrameworkElement，**它也是WPF控件的众多父类中最核心的基类**，从这里开始，**继承树开始分支，分别是Shape图形类、Control控件类和Panel布局类三个方向。**

FrameworkElement类本质上也是提供了**一系列属性、方法和事件**。同时又**扩展 UIElement 并添加了以下功能**：

![img](965851-20240128142410714-2003072759.png)



### **UIElement类([UIElement 类 (System.Windows) | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.uielement?view=windowsdesktop-8.0))**

​	UIElement类继承了Visual类，在WPF框架中排行老四，**它定义了大量的路由事件和大量的依赖属性**。大部分**输入和聚焦行为**也在UIElement类中定义。 这包括**键盘、鼠标和触笔输入的事件**，以及相关的状态属性。 其中许多事件是路由事件，许多与输入相关的事件既具有浮升路由版本，也具有事件的隧道版本。  

### **Visual类([Visual 类 (System.Windows.Media) | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.media.visual?view=windowsdesktop-8.0))**

​	Visual类是WPF框架中第三个父类，主要是为 WPF 中的呈现提供支持，其中包括命中测试、坐标转换和边界框计算.  Button、TextBox、CheckBox、Gird、ListBox等所有控件**都继承了Visual类，控件在绘制到界面的过程中**，涉及到转换、裁剪、边框计算等功能，都是使用了Visual父类的功能

### **Dependency类([依赖属性概述 - WPF .NET | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/desktop/wpf/properties/dependency-properties-overview?view=netdesktop-8.0))**

​     DependencyObject 类表示参与依赖属性系统的对象。属性系统的**主要功能是计算属性的值**，并提供有关**已更改的值的系统通知**。 参与属性系统的另一个类 DependencyProperty。 DependencyProperty **允许将依赖属性注册到属性系统**，并提供有关每个依赖属性的标识和信息，**而 DependencyObject 为基类，使对象能够使用此依赖属性**。
​     **INotifyPropertyChanged 类用于通知UI刷新**，注重的仅仅是数据更新后的通知。DependencyObject 类用于给UI添加依赖和附加属性，注重数据与UI的关联。如果简单的数据通知，两者都可以实现的。

​     什么是**数据驱动模式？控件的属性不再被直接赋值，而是绑定了另一个”变量“，当这个”变量“发生改变时，控件的属性也会跟着改变**，这样的属性也被称为依赖属性

### **DispatcherObject类([DispatcherObject 类 (System.Windows.Threading) | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.threading.dispatcherobject?view=windowsdesktop-8.0))**

​     在 WPF 中， **DispatcherObject 只能由 Dispatcher 它与之关联的访问**。 例如，后台线程无法更新与 Dispatcher UI 线程上关联的内容Button。 为了使后台线程访问该 Content 属性 Button，后台线程必须将工作委托给 Dispatcher 与 UI 线程关联的工作。 这是通过使用 Invoke 或BeginInvoke。 Invoke 是同步的， BeginInvoke 是异步的。 操作将添加到指定DispatcherPriority位置的队列Dispatcher中。

​    DispatcherObject 类的主要方针路线到底是什么呢？主要有两个职责：

​    1.提供对对象所关联的当前 Dispatcher 的访问权限，意思是说谁继承了它，谁就拥有了Dispatcher。
​    2.提供方法以检查 (CheckAccess) 和验证 (VerifyAccess) 某个线程是否有权访问对象（派生于 DispatcherObject）。CheckAccess 与 VerifyAccess 的区别在于 CheckAccess 返回一个布尔值，表示当前线程是否有可以使用的对象，而 VerifyAccess 则在线程无权访问对象的情况下引发异常。

### Style绑定

#### **`DynamicResource` 和 `StaticResource` 的区别**

​	这两个都是 XAML 标记扩展，作用都是从**资源字典 (`ResourceDictionary`)** 中查找并引用一个资源（比如一个**颜色画刷**、一个**样式**等）。它们最本质的区别在于**查找资源的时机**。

**一个简单的比喻：查地址**

> - **`StaticResource` (静态资源)**: 就像您在出发前，把朋友家的地址**查好并抄写在纸上**。在整个旅途中，您都看着这张纸条上的固定地址。如果中途您的朋友搬家了，您纸条上的地址就**不会更新**，您会找不到他。
> - **`DynamicResource` (动态资源)**: 就像您在手机地图上**收藏了朋友家的位置**。您不仅得到了当前地址，还得到了一个**实时链接**。如果中途您的朋友搬家并更新了位置，您的手机地图会自动刷新，导航到新的地址。

#### **技术细节对比**

| 特性         | `StaticResource` (静态资源)                                  | `DynamicResource` (动态资源)                                 |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **查找时机** | **仅在 XAML 加载时**查找一次。                               | **在 XAML 加载时查找一次，并在运行时持续监听**。             |
| **性能**     | **更高**。因为它是一次性的查找和赋值，没有额外开销。         | **略低**。因为它需要在内部维护一个引用链，以便在资源变更时收到通知并更新属性。但在大多数应用中，这点性能差异可以忽略不计。 |
| **资源要求** | 资源**必须**在引用它之前就已经被定义和加载（例如在 `App.xaml` 或当前文件的 `Resources` 节中）。如果找不到，会直接在加载时抛出异常。 | 资源**可以**在引用它之后再定义（称为“前向引用”）。如果加载时找不到，不会报错，直到运行时资源被添加或修改，UI 才会更新。 |
| **主要用途** | 引用**固定不变**的资源，如一个固定的颜色、一个控件的基础样式、一个固定的几何图形等。 | **1. 实现动态主题切换（最核心用途）**<br>2. 引用可能会在运行时改变的资源<br>3. 引用系统资源（如 `SystemColors`），因为系统主题可能在程序运行时改变 |