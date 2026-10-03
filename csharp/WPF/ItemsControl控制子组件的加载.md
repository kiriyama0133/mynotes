
为了实现一个高性能的组件，**我们就不能依赖Item的index索引**，因为非虚拟化情况下，1000个元素就是1000个延迟加载。**我们需要实现的是用户拖动到1000个元素的时候，10个元素使瞬间被渲染出来的。**

UI虚拟化的核心思想就是**只渲染可视范围内的控件**，所以它通常会**搭配ScrollViewer控件一起使用**，**通过ScrollViewer控件中的VerticalOffset、HorizontalOffset、ViewportWidth、ViewportHeight等参数可以计算出在可视范围内应该显示的控件**，当控件**不被显示时将它从Panel中移出**，这样就可以保证同一时间只渲染了有限的控件，而不是渲染所有控件，从而达到性能提升的目的。

上面的话有一些问题，WPF其实已经提供了判断在视口的方法，也把结果直接返回了， **逻辑流**：
    
    1. 用户滚动。
        
    2. `VirtualizingStackPanel` 内部计算（判断哪些 Index 进入了视口）。
        
    3. `VirtualizingStackPanel` 发现 Index 50 进入了视口。
        
    4. VSP 调用 `PrepareContainerForItemOverride` (传入 Index 50 的容器)。
        
**所以：** 你不需要去问“它在不在视口里？”，**只要这个方法被调用了，它就一定刚被放进视口（或者刚被生成）。** 这就是为什么我们在那里做动画是绝对安全的。

引用的那段话里有一句：“**不被显示时将它从Panel中移出**”。 这在 WPF/Avalonia 的默认虚拟化行为（`VirtualizationMode.Recycling`）中是不准确的，且效率很低，真正的应该是滚出视口的没有被销毁或者移除，应该是替换数据容器。

### UI虚拟化
我们来通过scrollViewer验证视口中的元素的数量，最直接的方法机会直接利用WPF控件的生命周期事件Loaded和Unloaded来实现。


```xml
<ItemsControl Grid.Row="1" ItemsSource="{Binding MyItems}">  
    <ItemsControl.ItemsPanel>  
        <ItemsPanelTemplate>  
            <VirtualizingStackPanel IsVirtualizing="True"   
									VirtualizationMode="Recycling" />  
        </ItemsPanelTemplate>  
    </ItemsControl.ItemsPanel>  
  
    <ItemsControl.Template>  
        <ControlTemplate TargetType="ItemsControl">  
            <ScrollViewer CanContentScroll="True"   
							PanningMode="Both">  
                <ItemsPresenter />  
            </ScrollViewer>  
        </ControlTemplate>  
    </ItemsControl.Template>  
  
    <ItemsControl.ItemTemplate>  
        <DataTemplate>  
            <local:TestItemControl />  
        </DataTemplate>  
    </ItemsControl.ItemTemplate>  
</ItemsControl>
```
我们用上面的方法来开启虚拟化，我们需要关注的是`ScrollViewer`（滚动条） 和 **`VirtualizingStackPanel`（布局面板）** 之间。

```xml
<ItemsPanelTemplate>
    <VirtualizingStackPanel IsVirtualizing="True" 
                            VirtualizationMode="Recycling" />
</ItemsPanelTemplate>
```
**默认情况**：`ItemsControl` 默认使用的是普通的 `StackPanel`。`StackPanel` 很“傻”，它的逻辑是：“你有多少数据，我就画多长”。**如果你有 1000 个数据，它就计算出 50,000 像素的高度**，然后一次性渲染出来。

这里我们启用了VSP，这个面板实现了`IScrollInfo` 接口，它拥有“数学能力”，它不会傻傻地渲染所有东西，而是会问：“现在视口有多高？**那我只需要算出这部分显示哪几个 Item 就可以了**。”

**`VirtualizationMode="Recycling"`**：这是高级优化。它告诉面板：“当一个 Item 滚出屏幕时，**别销毁它**，把它扔进回收站。等下一个 Item 滚进来时，**别 new 新的**，去回收站拿一个旧的刷上新数据用。”（这就是观察到**内存锯齿**的原因）。


```xml
<ControlTemplate TargetType="ItemsControl">
    <ScrollViewer ...>
        <ItemsPresenter />
    </ScrollViewer>
</ControlTemplate>
```
虚拟化的前提是 **“有限的空间”**，如果没有 `ScrollViewer`，`ItemsControl` 就会试图**根据内容撑大自己，占据无限的空间**，一旦拥有无限空间，`VirtualizingStackPanel` 就会认为“视口无限大”，于是它还是会渲染所有 1000 个元素。

<ScrollViewer CanContentScroll="True" ...> 这个步骤也是十分关键的，如果不写这个地方，就会设置为False，为物理滚动，scrollViewer组件会把自己当做单纯的图片浏览器。

**`VirtualizingStackPanel`**：

- 计算：第 500 个元素对应的高度是多少？
    
- **回收**：把**刚才显示的第 0-20 个元素的容器（ListBoxItem）扔进回收站**。
    
- **复用**：从回收站**捞出 20 个容器**。
    
- **绑定**：把这 20 个容器的数据上下文（DataContext）换成第 500-520 条数据。
    
- **生成**：调用 `PrepareContainerForItemOverride`（这时候我们的动画代码介入）。
    
- **布局**：只测量和排列这 20 个容器。

**`ItemsPresenter`**：把这 20 个容器显示在屏幕上。
这就是为什么只写了几行 XAML，性能却提升了百倍的原因。一切都在这一套精密的**协商机制**中完成了。

### Template
它是整个控件的**骨架和皮肤**，在这里我们定义了模板组件。

```xml
<ItemsControl.Template>  
    <ControlTemplate TargetType="ItemsControl">  
        <ScrollViewer CanContentScroll="True"   
PanningMode="Both">  
            <ItemsPresenter />  
        </ScrollViewer>  
    </ControlTemplate>  
</ItemsControl.Template>
```
### ItemsPanel 

这里设置的是布局模式，我们的虚拟化是否开启也是在这里放置的。除此以外还可以设置垂直堆叠或者图标墙（WrapPanel），或者是九宫格(UniformGrid)。

### 继承ItemsControl来自定义组件

我们现在先注册两个依赖属性：

```cs
public static readonly DependencyProperty StaggerIntervalProperty =  
    DependencyProperty.Register(nameof(StaggerInterval), typeof(TimeSpan), typeof(StaggeredItemsControl),   
new PropertyMetadata(TimeSpan.FromMilliseconds(50)));  
  
public static readonly DependencyProperty AnimationDurationProperty =  
    DependencyProperty.Register(nameof(AnimationDuration), typeof(TimeSpan), typeof(StaggeredItemsControl),   
new PropertyMetadata(TimeSpan.FromMilliseconds(500)));
```
再写他们的CLR包装器。

紧接着去编写构造函数：

```cs
static StaggeredItemsControl()  
{  
    // 告诉 WPF 去 Themes/Generic.xaml 找默认样式  
    DefaultStyleKeyProperty.OverrideMetadata(typeof(StaggeredItemsControl),   
new FrameworkPropertyMetadata(typeof(StaggeredItemsControl)));  
}
```
他会覆盖ItemControl的默认样式，去Themes/Generic.xaml中寻找专用的样式，这里先不看xaml代码，我们继续向下看：

这里我们定义三个私有属性：

```cs
private long _lastPrepareTick = 0;  
private int _currentBatchIndex = 0;  
private const long BatchThresholdTicks = 100 * 10000; // 100ms 阈值
```
第一个是记录上一次生成容器的时间，第二个事记录当前加载到了第几个，第三个是整个算法的时间基准，因为1ms = 10000Ticks，这里是Ticks单位的阈值。
- 在 UI 交互心理学中，**0.1 秒 (100ms)** 是用户感觉“瞬间发生”和“有延迟”的界限。
    
- 如果**两个动作发生的时间差小于 100ms**，计算机会认为它们是**“连贯的”**（比如程序正在疯狂加载列表，或者用户正在高速滚动）。
    
- 如果时间差大于 100ms，计算机会认为**“断片了”**（比如用户滚动了一下，停下来看了一会儿，手指又拨了一下）。
这行代码定义了：**“多长时间的停顿算作是新的一波操作？”** 答案是：**100 毫秒**（即 1,000,000 Ticks）。


### 入场动画

```cs
private void PlayEntranceAnimation(FrameworkElement container, TimeSpan beginTime)  
{  
    // 容器可能是从回收站捡回来的，它可能还保持着 Opacity=1 或 Transform=0    // 必须强制把它打回“未显示”的原型  
    container.Opacity = 0;   
// 确保有位移变换对象  
    if (!(container.RenderTransform is TranslateTransform))  
    {        container.RenderTransform = new TranslateTransform(0, 50);  
    }    var translate = (TranslateTransform)container.RenderTransform;  
    translate.Y = 50; // 初始位置：下方 50px    // 1. 透明度动画 (0 -> 1)    var opacityAnim = new DoubleAnimation  
    {  
        To = 1,  
        Duration = AnimationDuration,  
        BeginTime = beginTime, // 注入延迟  
        EasingFunction = new CubicEase { EasingMode = EasingMode.EaseOut }  
    };  
    // 2. 位移动画 (50 -> 0)    var moveAnim = new DoubleAnimation  
    {  
        To = 0,  
        Duration = AnimationDuration,  
        BeginTime = beginTime, // 注入延迟  
        EasingFunction = new CubicEase { EasingMode = EasingMode.EaseOut }  
    };    // 使用 BeginAnimation 可以自动覆盖之前的动画，比较安全  
    container.BeginAnimation(UIElement.OpacityProperty, opacityAnim);  
    translate.BeginAnimation(TranslateTransform.YProperty, moveAnim);  
}
```
这里我们将容器的透明度都设置为0，因为容器是从回收里取回的，可能它还保留着加载后的特性，比如透明度=1或者transform=0，因此我们需要去清晰一下组件。


### 列表控件基类虚方法重载


```cs
protected override void PrepareContainerForItemOverride(DependencyObject element, object item)  
{  
    base.PrepareContainerForItemOverride(element, item);  
    // 只有 FrameworkElement 才能做动画 (ContentPresenter 或 ListBoxItem)    if (element is FrameworkElement container)  
    {        // A. 批次检测算法  
        long now = DateTime.Now.Ticks;  
        // 如果这次生成距离上次生成 < 100ms，视为同一批  
        if (now - _lastPrepareTick < BatchThresholdTicks)  
        {            _currentBatchIndex++;  
        }        else  
        {  
            // 否则视为新的一波操作（例如用户直接拖动滚动条到了新位置）  
            // 重置计数器，让这批的第一个元素立即显示  
            _currentBatchIndex = 0;  
        }        _lastPrepareTick = now;  
        // B. 计算延迟  
        var delay = TimeSpan.FromTicks(StaggerInterval.Ticks * _currentBatchIndex);  
        // C. 播放入场动画  
        PlayEntranceAnimation(container, delay);  
    }}
```

它的作用是将数据变成界面，将数据放入组件中，我们调用了`base.PrepareContainerForItemOverride(element, item)` 时。WPF会执行类似于element.DataContext = item;这样的操作，如果删掉，ui就拿不到数据。

long now = DateTime.Now.Ticks;这里会记录当前的tick，如果当前的tick和上一次生成的tick小于了100ms，那么就可以当做是同一批加载的。否则就会重置记录器，让这批容器的第一个元素立即显示， _lastPrepareTick = now; 来更新当前的tick。

var delay = TimeSpan.FromTicks(StaggerInterval.Ticks * _currentBatchIndex);则是用于计算延迟的，前面的是一个const，表示的是每个组件加载的延迟，后面则是我们的**计数器**，用来计算后面组件的延迟，然后播放入场动画。

又有一点我们需要注意的地方：DependencyObject是WPF依赖属性的基矢，它的设计很简陋，没有Opacity等属性，没有RenderTransform或者什么什么其他，这些都存在于FrameworkElement中，因此我们必须进行转型，否则就无法引用动画属性。

当代码运行的时候，这个element的类型取决于使用的控件类型，对于ItemsContrl，默认的容器是ContentPresenter，继承自FrameworkElement。 对于ListBox，默认的是ListBoxItme，继承自ContentContrl -> Contrl -> FrameworkElement，对于ComboBox，默认容器ComboBoxItem也是FrameworkElement，因此几乎所有情况下也是FrameworkElement。

cs的部分说完了，接下来我们来到style的xml部分。

### Generic的样式

Themes/Generic.xaml 我们知道，这个是WPF框架底层硬编码的一个标准路径，如果我们想要自行实现一个自定义控件的样式，我们就必须放在这个位置的文件里。

WPF去绘制我们的自定义组件 StaggeredItemsControl 的时候，回去检查在自身有无显示设置Sytle属性（<local:StaggeredItemsControl Style="{...}" />），如果有的话，就会直接使用，没有的话就会去Application:App.xaml中检查有没有针对这个类型的隐式样式，如果还是没有，就回去检查默认样式，然后由于我们编写了 DefaultStyleKeyProperty.OverrideMetadata ，他就回去DLL内部寻找样式，这个时候WPF就会去程序集里寻找一个叫做themes的文件夹，寻找这个文件里的样式。

### xaml样式，自定义组件样式

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"  
                    xmlns:local="clr-namespace:ItemControlsSolution"  
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">  
        <Style TargetType="{x:Type local:StaggeredItemsControl}">  
        <Setter Property="Background" Value="Transparent"/>  
        <Setter Property="BorderThickness" Value="0"/>  
        <Setter Property="ItemsPanel">  
            <Setter.Value>  
                <ItemsPanelTemplate>  
                    <VirtualizingStackPanel IsVirtualizing="True"   
VirtualizationMode="Recycling" />  
                </ItemsPanelTemplate>  
            </Setter.Value>  
        </Setter>  
        <Setter Property="Template">  
            <Setter.Value>  
                <ControlTemplate TargetType="{x:Type local:StaggeredItemsControl}">  
                    <Border Background="{TemplateBinding Background}"  
                            BorderBrush="{TemplateBinding BorderBrush}"  
                            BorderThickness="{TemplateBinding BorderThickness}">  
                        <ScrollViewer Focusable="False"  
                                      Padding="{TemplateBinding Padding}"  
                                      CanContentScroll="True"  
                                      PanningMode="Both">  
                            <ItemsPresenter />  
                        </ScrollViewer>  
                    </Border>  
                </ControlTemplate>  
            </Setter.Value>  
        </Setter>  
    </Style>  
    </ResourceDictionary>
```


```xml
<Style TargetType="{x:Type local:StaggeredItemsControl}">
```
这里引用了类型StaggerdItemsContrl，告诉了样式系统我们定义了一个组件样式，我们在cs代码中编写的DefaultStyleKeyProperty.OverrideMetadata正是通过这个TargetType匹配到了这里的xaml。

然后是基础的属性设置：

```xml
<Setter Property="Background" Value="Transparent"/>
<Setter Property="BorderThickness" Value="0"/>
```
如果用户在使用时不写 `<local:StaggeredItemsControl Background="Red" ...>`，那么背景默认就是透明的，边框是 0，这里是默认的属性。

然后再往下就是布局策略：

```xml
<Setter Property="ItemsPanel">
    <Setter.Value>
        <ItemsPanelTemplate>
            <VirtualizingStackPanel IsVirtualizing="True" 
                                    VirtualizationMode="Recycling" />
        </ItemsPanelTemplate>
    </Setter.Value>
</Setter>
```
我们抛弃了默认的StackPanel，并且采用了VirtualizingStackPanel，如果没有这段代码，10,000数据会一次性生成造成内存爆炸，ItemsPanel用于设置子元素的排列。

接下来是控件骨架部分，这里是控件的物理结构：

```xml
<Setter Property="Template">
    <Setter.Value>
        <ControlTemplate TargetType="{x:Type local:StaggeredItemsControl}">
            </ControlTemplate>
    </Setter.Value>
</Setter>
```
我们这里指定了一个控件模板，指定的类型也就是我们的自定义组件，我们进入控件模板看看里面定义的是什么：


```xml
<ControlTemplate TargetType="{x:Type local:StaggeredItemsControl}">  
    <Border Background="{TemplateBinding Background}"  
            BorderBrush="{TemplateBinding BorderBrush}"  
            BorderThickness="{TemplateBinding BorderThickness}">  
        <ScrollViewer Focusable="False"  
                      Padding="{TemplateBinding Padding}"  
                      CanContentScroll="True"  
                      PanningMode="Both">  
            <ItemsPresenter />  
        </ScrollViewer>  
    </Border>  
</ControlTemplate>
```
外壳Border部分绑定的是模板组件的背景色，如果用户传递了background的话，就会绑定到这个上面，下面的同理，有BorderBrush和BorderThickness。

然后就是我们的视口关键部分scrollViewer，我们开启了细节优化：**`Focusable="False"`**防止点击空白处时，焦点跑到滚动条上，导致原来的焦点丢失，**`PanningMode="Both"`**：支持触摸屏的手指拖拽滚动。
**`CanContentScroll="True"`**：**虚拟化总开关**。
- 这就是之前讨论的“逻辑滚动”。它告诉 `ScrollViewer`：“不要按像素滚，要把滚动权交给里面的 Panel（也就是上面的 `VirtualizingStackPanel`）。”


```xml
<ItemsPresenter />
```
这个就是插槽。

### 平滑滚动

现在的滚动是比较僵硬的，因为这是ScrollViewer的默认行为，它是按照像素移动的，但是为了渲染所有子元素，导致虚拟化失败。而我们开启了`CanContentScroll="True"` 后，滚动机制变成了逻辑滚动，会变成一次滚一行，产生了僵硬感。

自从WPF4.5之后，微软引入了一个叫做像素级虚拟化的功能，我们需在`Generic.xaml` 的 `ItemsPanelTemplate` 中加一行代码： **`VirtualizingPanel.ScrollUnit="Pixel"`**，这样就用了虚拟化的性能和像素级别的平滑滚动。

但是我们需要的是带有惯性的就像是滑动的那样，这个时候靠xaml是实现的不了的，最好的做法就是拦截鼠标滚动事件，取消默认行为来用动画改变VerticalOffset。


我们需要重载方法：OnApplyTemplate

```cs
public override void OnApplyTemplate()  
{  
    base.OnApplyTemplate();  
    _scrollViewer = FindVisualChild<ScrollViewer>(this);  
}
```
`OnApplyTemplate` 的目的是：**在“图纸”变成“实体”的那一刻，把手伸进控件内部，抓住我们需要操作的零件（比如 ScrollViewer）。** 代码不能写在构造函数里，因为执行构造函数的时候，此时，XAML 中的 `ControlTemplate` 还没开始工作，你的 `ScrollViewer` 此时还不存在，它只是 Generic.xaml 里的一行字。

必须在WPF找到Generic.xmal之后应用模板，**`OnApplyTemplate` 被调用**：<-- **就在这一刻！** 所有的内部控件都生成完毕了，这个重载方法保证了ItemsControl自己的初始化逻辑被正常加载的同时还会寻找到一个类型是ScrollViewer的子控件。

然后我们看看寻找子控件的方法：

```cs
private static T FindVisualChild<T>(DependencyObject parent) where T : DependencyObject  
{  
    for (int i = 0; i < VisualTreeHelper.GetChildrenCount(parent); i++)  
    {        var child = VisualTreeHelper.GetChild(parent, i);  
        if (child is T t) return t;  
        var result = FindVisualChild<T>(child);  
        if (result != null) return result;  
    }    return null;  
}
```
这个方法就是深度优先搜索DFS的实现，他会沿着WPF的可视化树一层一层挖掘，直到找到这个子组件。

最后是滚轮事件拦截的方法重载实现

```cs
protected override void OnPreviewMouseWheel(MouseWheelEventArgs e)  
{  
    if (_scrollViewer == null) return;  
    e.Handled = true;  
    double currentOffset = _scrollViewer.VerticalOffset;  
    double delta = e.Delta > 0 ? -48 : 48; // e.Delta > 0 是向上滚  
    double targetOffset = currentOffset + delta;  
    if (targetOffset < 0) targetOffset = 0;  
    if (targetOffset > _scrollViewer.ScrollableHeight) targetOffset = _scrollViewer.ScrollableHeight;  
    _scrollViewer.ScrollToVerticalOffset(targetOffset); }
```
WPF 的事件分两种，`Preview` 开头的是**隧道事件（从外向内传）**，普通的（如 `MouseWheel`）是**冒泡事件（从内向外传）**。
**为什么用 Preview？** 因为 `ScrollViewer` 内部会捕获并处理冒泡的 `MouseWheel` 事件。如果你想在 `ScrollViewer` 动手之前就把事件截胡，必须用 `OnPreviewMouseWheel`。


```cs
if (_scrollViewer == null) return; e.Handled = true; // <--- 最关键的一行！
```
**`e.Handled = true`**：这行代码就是在这个事件的传播路径断开，表示事件已被处理。

手动计算 (Manual Calculation)：

```cs
double currentOffset = _scrollViewer.VerticalOffset;
double delta = e.Delta > 0 ? -48 : 48;
```
鼠标滚轮每“咔哒”一下，通常 Delta 值是 120（正数向上，负数向下），- **`48`**：这是一个**魔法数字**。
    
    - 这里硬编码了每次滚动移动 48 像素。
        
    - 这解决了之前“按 Item 滚动”太僵硬的问题（如果一个 Item 高 100px，之前滚一下跳 100px，现在只跳 48px）。
        
- **方向**：因为屏幕坐标系 Y 轴向下是正方向，所以向上滚（Delta > 0）要减小 Offset，向下滚要增加 Offset。

边界检查 (Clamping)：
```cs
double targetOffset = currentOffset + delta;
if (targetOffset < 0) targetOffset = 0;
if (targetOffset > _scrollViewer.ScrollableHeight) targetOffset = _scrollViewer.ScrollableHeight;
```
这个是为了防止算出的位置超出范围（比如滚到了 -48px 或者滚过了最底部）。虽然 `ScrollToVerticalOffset` 内部通常有容错，但自己写一遍更安全。

_scrollViewer.ScrollToVerticalOffset(targetOffset);就是执行移动。

这就是全部实现了，但是XAML 的 `ScrollUnit="Pixel"` 是官方的原生实现，它支持**惯性（Inertia）**（手指拨动会滑行），支持平滑加速减速，我们这段只是**生硬的线性移动**。滚一下动 48px，立马停住，手感像是在操作机床，而不是现代 App。

WPF针对普通的鼠标滚轮并不支持鼠标滚轮惯性，因此想要实现像手机一样的丝滑惯性，就需要去用C#手写一个物理动画引擎来欺骗组件或者其他方案等等。。。 有使用Composition.Rendering实现也有通过二阶系统的实现。这个就是其他的笔记要讲的了，我个人是更喜欢二阶系统缓动实现的方式的，这个更贴近真实的手感。
