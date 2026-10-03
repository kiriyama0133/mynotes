
## Colors
Thems负责的是主体系统，我们点开Basic文件夹里，可以看到的是Colors、ColorsDark、ColorsViolet。
![[Pasted image 20260402170708.png]]
这里看名称还是比较可以清晰明了的，基本是用Light/Dark这样的开头来阐述，还有一个文件叫做ColorsDark，里面保存的是黑暗主题的颜色代码，这两个对比的颜色就可以任意切换颜色主题。
而在Avalonia版本中，HandyControl使用了更现代的主题字典方式，在单个`Colors.axaml`文件中通过`ResourceDictionary.ThemeDictionaries`来管理不同主题的颜色 Colors.axaml:4-96 ：

```xml
<ResourceDictionary.ThemeDictionaries>

<ResourceDictionary x:Key="Default">

<!-- 明亮主题颜色 -->

</ResourceDictionary>

<ResourceDictionary x:Key="Dark">

<!-- 黑暗主题颜色 -->

</ResourceDictionary>

</ResourceDictionary.ThemeDictionaries>
```


定义的颜色资源会在Theme.xaml中被映射为各种画刷，供控件使用 Theme.xaml:838-888 。例如：
- `PrimaryColor` → `PrimaryBrush`（渐变画刷）
- `LightPrimaryColor` → `LightPrimaryBrush`（纯色画刷）
- `DarkPrimaryColor` → `DarkPrimaryBrush`（纯色画刷）

先不聊Theme，因为还有其他重要的文件需要阅读，以及Theme这个文件实在过于庞大，很多的任务都在这里面处理。除了颜色画刷以外，theme还处理了大量的矢量图形资源，用于图标和装饰元素 Theme.xaml:91-94

```xml
<Geometry o:Freeze="True" x:Key="AllGeometry">M 721.005 638.949...</Geometry>

<Geometry o:Freeze="True" x:Key="DragVerticalGeometry">M2,12 C3.1045694,12...</Geometry>

<Geometry o:Freeze="True" x:Key="DropperGeometry">M798.165333 97.834667...</Geometry>
```

还有各种控件的基础样式，通用尺寸值 Theme.xaml:100-105：

```xml
<system:Double x:Key="DefaultControlHeight">28</system:Double>

<system:Double x:Key="SmallControlHeight">20</system:Double>

<Thickness x:Key="DefaultControlPadding">10,5</Thickness>

<CornerRadius x:Key="DefaultCornerRadius">4</CornerRadius>
```

动画行为、控件模板、阴影效果、基础控件样式、特殊画刷和模板、路径样式等等，十分的杂乱，在Avalonia版本里，这些都会被拆分为不同的文件中管理，如`Geometries.axaml`、`Fonts.axaml`、`Effects.axaml`等 Theme.axaml:8-16 ，使主题系统更加模块化（真的及其恶心）。

## Behaviors

![[Pasted image 20260402172859.png]]

这个xaml文件封装了6个流体移动行为，用于实现平滑的动画效果。

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                    xmlns:interactivity="clr-namespace:HandyControl.Interactivity">

    <interactivity:FluidMoveBehavior x:Key="BehaviorXY200" x:Shared="False" AppliesTo="Children" Duration="0:0:.2">
        <interactivity:FluidMoveBehavior.EaseX>
            <PowerEase/>
        </interactivity:FluidMoveBehavior.EaseX>
        <interactivity:FluidMoveBehavior.EaseY>
            <PowerEase/>
        </interactivity:FluidMoveBehavior.EaseY>
    </interactivity:FluidMoveBehavior>

    <interactivity:FluidMoveBehavior x:Key="BehaviorX200" x:Shared="False" AppliesTo="Children" Duration="0:0:.2">
        <interactivity:FluidMoveBehavior.EaseX>
            <PowerEase/>
        </interactivity:FluidMoveBehavior.EaseX>
    </interactivity:FluidMoveBehavior>

    <interactivity:FluidMoveBehavior x:Key="BehaviorY200" x:Shared="False" AppliesTo="Children" Duration="0:0:.2">
        <interactivity:FluidMoveBehavior.EaseY>
            <PowerEase/>
        </interactivity:FluidMoveBehavior.EaseY>
    </interactivity:FluidMoveBehavior>

    <interactivity:FluidMoveBehavior x:Key="BehaviorXY400" x:Shared="False" AppliesTo="Children" Duration="0:0:.4">
        <interactivity:FluidMoveBehavior.EaseX>
            <PowerEase/>
        </interactivity:FluidMoveBehavior.EaseX>
        <interactivity:FluidMoveBehavior.EaseY>
            <PowerEase/>
        </interactivity:FluidMoveBehavior.EaseY>
    </interactivity:FluidMoveBehavior>

    <interactivity:FluidMoveBehavior x:Key="BehaviorX400" x:Shared="False" AppliesTo="Children" Duration="0:0:.4">
        <interactivity:FluidMoveBehavior.EaseX>
            <PowerEase/>
        </interactivity:FluidMoveBehavior.EaseX>
    </interactivity:FluidMoveBehavior>

    <interactivity:FluidMoveBehavior x:Key="BehaviorY400" x:Shared="False" AppliesTo="Children" Duration="0:0:.4">
        <interactivity:FluidMoveBehavior.EaseY>
            <PowerEase/>
        </interactivity:FluidMoveBehavior.EaseY>
    </interactivity:FluidMoveBehavior>

</ResourceDictionary>
```

有双轴移动效果（X、Y两个一起） 持续200ms 400ms的，也有各个单轴的效果，这些行为都是配置了共同属性x:Shared=“False”可以确保每次使用的时候都会创建新的实例。
`AppliesTo="Children"` - 仅应用于子元素， `PowerEase` - 使用PowerEase缓动函数。

重点不是动画，而是搞懂为什么要设定共同属性为什么这么设置，一个是通过预定义行为的资源，避免在每个控件中重复配置相同的动画参数，所有**行为共享**统一的配置模式，修改时只需更新一处。

然后是性能方面，每一次使用的是创建实例，可以避免行为实例在多个控件之间共享时可能产生的状态冲突 Theme.xaml:106-113 。

PowerEase是一个WPF/.NET Framework的一个缓动函数类，属于`System.Windows.Media.Animation`命名空间，它继承自EasingFunctionBase，用于创建具有加速和减速效果的动画。

我们接下来看看行为依赖的interactivity。

## Interactivity

![[Pasted image 20260402205532.png]]

它对应的是`HandyControl.Interactivity`命名空间，包含以下核心组件：

 Interaction附加属性容器，用于提供`Interaction.Behaviors`附加属性，**用于将行为附加到UI元素上**

ControlCommands命令定义，包含控件相关的命令定义，如TabControl中使用的关闭命令等。

 
我们先看看Command文件夹：具体是什么？
ControlCommand.cs是命令的总入口，统一定义了很多的RoutedCommand，以及几个自定义的ICommand实例，然后其他的：

- `CloseWindowCommand.cs`：关闭当前元素所在窗口
- `OpenLinkCommand.cs`：打开链接
- `ShutdownAppCommand.cs`：关闭应用
- `PushMainWindow2TopCommand.cs`：把主窗口显示并置顶到前台
- `StartScreenshotCommand.cs`：启动截图
- `ScreenshotCommand.cs`：内容与 `StartScreenshotCommand` 基本重复（看起来像历史遗留/重复文件）

在架构中的作用就是动作语义从控件里抽离出来，避免每个控件自己写一套点击逻辑，让XAML里面可以直接写命令来绑定，比如说Command="interactivity:ControlCommands.Prev"

结合 `Interactivity`（行为、触发器）时，能做“声明式交互”。你可以把它理解为：控件行为协议层。

写到这里，是不是显得十分的分散，接下来用一整条回路来解释它的作用。

### 命令 -> 到组件里
`Theme.xaml` 的 `Pagination` 模板，把按钮命令写成 `ControlCommands.Prev/Next/Jump`：

```xml
<ControlTemplate TargetType="hc:Pagination">
  <StackPanel Orientation="Horizontal" VerticalAlignment="Top">
    <Button x:Name="PART_ButtonLeft" Command="interactivity:ControlCommands.Prev" />
    ...
    <Button x:Name="PART_ButtonRight" Command="interactivity:ControlCommands.Next" />
    ...
    <Button ... Command="interactivity:ControlCommands.Jump" />
  </StackPanel>
</ControlTemplate>
```
`ControlCommands` 里把这些命令定义成静态 `RoutedCommand`：

```cs
public static RoutedCommand Prev { get; } = new(nameof(Prev), typeof(ControlCommands));
public static RoutedCommand Next { get; } = new(nameof(Next), typeof(ControlCommands));
public static RoutedCommand Jump { get; } = new(nameof(Jump), typeof(ControlCommands));
```
控件构造时注册 `CommandBinding`，`Pagination` 在构造函数里“接住”这些命令：
```cs
public Pagination()
{
    CommandBindings.Add(new CommandBinding(ControlCommands.Prev, ButtonPrev_OnClick));
    CommandBindings.Add(new CommandBinding(ControlCommands.Next, ButtonNext_OnClick));
    CommandBindings.Add(new CommandBinding(ControlCommands.Selected, ToggleButton_OnChecked));
    CommandBindings.Add(new CommandBinding(ControlCommands.Jump, (s, e) => PageIndex = (int)_jumpNumericUpDown.Value));
}
```
最终执行业务逻辑层面上，比如 `Prev/Next` 最后就是改页码：
```cs
private void ButtonPrev_OnClick(object sender, RoutedEventArgs e) => PageIndex--;
private void ButtonNext_OnClick(object sender, RoutedEventArgs e) => PageIndex++;
```
### 事件 -> 命令（`Interactivity` 桥接）

`Pagination` 中间那组页码 `RadioButton` 的选中，不是直接 `Command=`，而是通过触发器桥接：

```cs
<interactivity:Interaction.Triggers>
  <interactivity:RoutedEventTrigger RoutedEvent="RadioButton.Checked">
    <interactivity:EventToCommand Command="interactivity:ControlCommands.Selected" PassEventArgsToCommand="True" />
  </interactivity:RoutedEventTrigger>
</interactivity:Interaction.Triggers>
```

它的机制就是RoutedEventTrigger监听到路由事件之后，会调用EventToCommand.Invoke，当EventToCommand取到Command的时候会执行CanExecute/Execute来实现关键的实现点。
```cs
protected override void Invoke(object parameter)
{
    ...
    if (command == null || !command.CanExecute(parameter1))
        return;
    command.Execute(parameter1);
}
```


### 自定义Icommand

除了 `RoutedCommand`，`ControlCommands` 也暴露了直接可执行的 `ICommand` 实例：
```cs
public static CloseWindowCommand CloseWindow { get; } = new();
```

Demo 里直接用：
```cs
<Button Command="hc:ControlCommands.CloseWindow" CommandParameter="{Binding RelativeSource={RelativeSource Self}}" .../>
```
命令实现就是：拿 `CommandParameter` 对应元素，找到所在窗口并关闭：

```cs
public void Execute(object parameter)
{
    if (parameter is DependencyObject dependencyObject)
    {
        if (Window.GetWindow(dependencyObject) is { } window)
        {
            window.Close();
        }
    }
}
```

总结来说，ControlCommand会统一动作名册，Theme.xaml会把命令挂到可视元素，控件类CommandBindings会把命令映射到控件内部的逻辑。Interactivity会把事件桥接成命令，减少code-behind。

除开command文件，就是事件转命令的实现EventToCommand：

### EventToCommand

为什么需要这个文件？ 因为很多WPF很多事件不能直接优雅的走命令，但是组件库又想要模板层没有code-behind，实现交互统一命令化，`EventToCommand` 就是为这个缺口做的桥接器。

就比如说你有一个事件MouseMove，但是你希望执行ICommand，你想要写纯XAML来交互，不用写页面代码后置，还可以把控件启用状态跟CanExecue联动。`Button` 这种有 `Command` 属性的控件可以直接绑命令，但很多交互是“事件型”的（尤其在 `Style/ControlTemplate`、触发器场景），不总能直接命令绑定

就比如说这里：
```xml
<interactivity:Interaction.Triggers>  
    <interactivity:RoutedEventTrigger RoutedEvent="RadioButton.Checked">  
       <interactivity:EventToCommand Command="interactivity:ControlCommands.Selected" PassEventArgsToCommand="True" />  
    </interactivity:RoutedEventTrigger>  
</interactivity:Interaction.Triggers>
```
这里会在容器上挂载触发器，来监听冒泡上来的RadioButton.Checked事件，然后执行EventToCommand，EventToCommand会调用ControlCommands.Selected.Execute(...)， `PassEventArgsToCommand="True"`：把这次事件参数（`RoutedEventArgs`）作为命令参数传入（如果没显式 `CommandParameter`）。

![[Pasted image 20260402213939.png]]

有了它的帮助，我们可以把事件触发转换为命令执行，让你在 XAML 里不用写 `Click` 代码后置，而是通过触发器把事件映射到 `ICommand`。这个文件做了四件事：
- 接收命令与参数：`Command`、`CommandParameter`
- 支持把事件参数传给命令：`PassEventArgsToCommand`、`EventArgsConverter`
- 执行前做可执行判断：`CanExecute` 不通过就不执行
- 可选联动控件启用状态：`MustToggleIsEnabled` 时自动根据 `CanExecute` 设置控件 `IsEnabled`

这里TriggerAction触发器接受的类型是DependencyObject，他可以提供数据绑定、样式/模板的Setter赋值。以及动画和属性值继承，还能实现默认值、回调、校验、coerce等机制(依赖属性被设置的时候，WPF在真正生效前，给你一次强制修正值的机会)，WPF中绝大部分UI对象都在这条继承链上。

你可以打开DependencyObject来看看究竟有哪些：
![[Pasted image 20260402214930.png]]

这里的## `[NameScopeProperty("NameScope", typeof(NameScope))]`意思就是在xaml设计器中可以得到NameScope属性，来关系x:Name命名作用域， `NameScope` 的作用是：在某个作用域内，名字唯一，并支持 `FindName`、Storyboard 目标解析等。

回到刚才的话题，它注册了几个依赖属性，- 给 `EventToCommand` 提供一个**可绑定的参数入口**（XAML/Binding 可赋值）， 这个参数会在触发时传给命令的 `Execute(parameter)` / `CanExecute(parameter)`

### MouseDragElementBehaviorEx

这个文件是反编译微软的System.Windows.Interactivity程序集得到的，并对其做了些扩展。他们做了LockX和LockY两个轴向锁属性，在位移应用处加入了轴向判断，代码里也写了“在该处对微软的类进行了修改”）：

- `MatrixTransform` 分支里，只有 `!LockX` 才改 `OffsetX`，只有 `!LockY` 才改 `OffsetY`
- `TranslateTransform` 分支同理，只在未锁定轴上累加位移

也就是把原来“自由拖拽（X/Y都动）”扩展成“可限制只横向拖或只纵向拖”。这样就得到了一个可以附加到`FrameworkElement` 的拖拽行为（Behavior），核心能力：
- 鼠标按下开始拖拽（捕获鼠标）
- 鼠标移动实时更新元素位置（通过 `RenderTransform` 平移）
- 鼠标抬起/失去捕获结束拖拽
- 可选限制在父容器边界内（`ConstrainToParentBounds`）
- 通过**依赖属性**暴露当前位置（`X`、`Y`）
- 暴露拖拽生命周期事件：`DragBegun` / `Dragging` / `DragFinished`

这是一个“可配置的 UI 拖拽引擎”：  
在微软原版 `MouseDragElementBehavior` 基础上，HandyControl 增强了锁轴拖动（LockX/LockY），更适合做窗口条、滑块、面板等只允许单轴移动的交互。


#### behavior

这里使用或者说继承的behavior其实是作者们改良过的：![[Pasted image 20260402224005.png]]

这里的AssociatedObject指的是Behavior当前附加到的哪个对象的实例，当你在 XAML 里给某个元素加 Behavior 时（比如给 `Grid`、`Button`、`FrameworkElement`）：

- `Attach(dependencyObject)` 被调用
- `_associatedObject = dependencyObject`
- 之后 `OnAttached()` 里你就可以通过 `AssociatedObject` 操作该元素（订阅事件、改属性等）

我们熟知的DragBehavior就是这样生效的！！

那么这里为什么不直接写Behavior<>里面Dep..Object来约束呢，为什么非要where来泛型约束。如果真的像前面这样来写的话，等于把类型信息丢了，子类里你还是要反复强转。`Behavior<T>` 的意义是把“宿主类型约束”前置到编译期。

这么说你就懂了， `DependencyObject` 太宽泛：Window、Button、Transform、Brush 都是它体系里的类型 你的行为通常只适用于某一类宿主（比如 `FrameworkElement`）
- 用泛型 `T` 可以：
    - 编译期拿到强类型 `AssociatedObject`
    - Attach 时自动做类型约束检查
    - 减少运行时 cast/空判断和错误


### RoutedEventTrigger

这个文件的职责很清晰，监听一个指定的 `RoutedEvent`，**一旦触发就把事件转发给触发器**动作链（比如 `EventToCommand`）。![[Pasted image 20260402225354.png]]

和我们刚才看到的
```xml
<interactivity:Interaction.Triggers>
    <interactivity:RoutedEventTrigger RoutedEvent="RadioButton.Checked">
        <interactivity:EventToCommand
            Command="interactivity:ControlCommands.Selected"
            PassEventArgsToCommand="True" />
    </interactivity:RoutedEventTrigger>
</interactivity:Interaction.Triggers>
```
这段代码完全相关

## Brushes文件

这个是颜色到画刷的中间层，我们之前见到的Color颜色不能直接使用，我们需要封装为可以给控件使用的Brush资源：

![[Pasted image 20260402231018.png]]

## Conveters 转换器

![[Pasted image 20260402231347.png]]

 它是“全局转换器注册表”

把 HandyControl 里常用的 `IValueConverter` / `IMultiValueConverter` 统一注册成可复用资源，供所有样式模板直接 `StaticResource` 引用。

- 布尔和可见性：`Boolean2VisibilityConverter`、`Boolean2VisibilityReConverter`
- 反转/组合逻辑：`Boolean2BooleanReConverter`、`BooleanArr2BooleanConverter`
- 布局/图形：`CornerRadiusSplitConverter`、`ThicknessSplitConverter`、`BorderClipConverter`
- 文本/数值：`Int2StringConverter`、`Number2PercentageConverter`
- 控件特定：`TreeViewItemMarginConverter`、`DataGridSelectAllButtonVisibilityConverter`

 为什么单独放一个文件

- 主题模板大量依赖 converter，集中定义可避免重复创建和重复声明
- 统一 key，样式里直接用，降低模板复杂度
- 作为主题基础资源的一部分，随主题一起合并加载


## Effect文件

这个用于主题里的阴影效果资源表


```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                    xmlns:o="http://schemas.microsoft.com/winfx/2006/xaml/presentation/options">

    <Color x:Key="EffectShadowColor">#88000000</Color>

    <DropShadowEffect x:Key="EffectShadow1" BlurRadius="5" ShadowDepth="1" Direction="270" Color="{StaticResource EffectShadowColor}" Opacity=".2" RenderingBias="Performance" o:Freeze="True" />
    <DropShadowEffect x:Key="EffectShadow2" BlurRadius="8" ShadowDepth="1.5" Direction="270" Color="{StaticResource EffectShadowColor}" Opacity=".2" RenderingBias="Performance" o:Freeze="True" />
    <DropShadowEffect x:Key="EffectShadow3" BlurRadius="14" ShadowDepth="4.5" Direction="270" Color="{StaticResource EffectShadowColor}" Opacity=".2" RenderingBias="Performance" o:Freeze="True" />
    <DropShadowEffect x:Key="EffectShadow4" BlurRadius="25" ShadowDepth="8" Direction="270" Color="{StaticResource EffectShadowColor}" Opacity=".2" RenderingBias="Performance" o:Freeze="True" />
    <DropShadowEffect x:Key="EffectShadow5" BlurRadius="35" ShadowDepth="13" Direction="270" Color="{StaticResource EffectShadowColor}" Opacity=".2" RenderingBias="Performance" o:Freeze="True" />

</ResourceDictionary>

```

这里共享了一个半透明黑色：`EffectShadowColor`。 五档 `DropShadowEffect`：`EffectShadow1`～`EffectShadow5`，模糊半径和阴影深度递增，用于不同强度的卡片/浮层阴影

- 控件模板里用 `Effect="{StaticResource EffectShadow3}"` 这类写法，统一视觉、避免每个控件手写一遍 `DropShadowEffect`
- `o:Freeze="True"`：冻结为不可变对象，便于性能与共享

和 `Brushes.xaml`、`Converters.xaml` 一样，属于基础主题资源，被合并进主主题后全局复用。

## Fonts文件

这个就和名字一样，定义了各种文字的大小。![[Pasted image 20260403160604.png]]

在 `Theme.xaml` 顶部已经能看到类似资源被使用（比如 `LargeFontSize`、`HeadFontSize`、`TextFontSize` 这种 key），这些就是供控件样式统一引用的。

## Geometry

这个文件定义了可复用的矢量几何资源（`Geometry`）。在主题体系里它通常负责这类内容：

- 各种图标路径（关闭、箭头、搜索、旋转、保存等）
- 复杂矢量形状（控件内部装饰用）
- 统一 key（如 `CloseGeometry`、`LeftGeometry`），供样式和模板复用

单独放在这个文件里的原因是为了避免每个模块里重复编写长Path Data，并且图标视觉统一就可以实现低维护成本， 配合 `Path Data="{StaticResource XxxGeometry}"` **使用很干净**。

`Theme.xaml` 里已经能看到它的使用方式，比如按钮模板里常见：

- `hc:IconElement.Geometry="{StaticResource LeftGeometry}"`
- `...="{StaticResource CloseGeometry}"`

![[Pasted image 20260403161431.png]]


## Path文件夹

它的任务就是把刚才的几何层封装成可以直接使用的Path 样式， `Geometries.xaml`：只提供 `Geometry` 数据（图标“形状原料”）。

![[Pasted image 20260403161523.png]]

## Size文件

它的作用是集中定义尺寸相关的主题常量资源，给全局的样式和控件模板统一使用。在 HandyControl 这种主题体系里，它一般会包含类似这些 key：

- 控件高度（默认/小号）
- 圆角半径（默认/胶囊）
- 边距、间距、内边距
- 图标尺寸、线宽等
![[Pasted image 20260403161633.png]]

## Base文件夹

`Themes/Styles/Base` 里放的是各控件的“基础样式模板层”，可以理解成“样式骨架”。控件默认使用的是ControlTemplate，通用Setter，边框、内边距、状态触发器。以及设置了供上层使用的x:key。 这个也是组件设计的基本功，先定义可重复使用的基础外观和行为，外层样式再去轻量定制。![[Pasted image 20260403165637.png]]

就像上面的，我们看到的这个基础的样式，它最终会被作为下拉项样式来使用。具体关系是： `AutoCompleteTextBoxItemBaseStyle` 的 `TargetType` 是 `ComboBoxItem` 。
在 `AutoCompleteTextBoxBaseStyle` 里通过  
`ItemContainerStyle = {StaticResource AutoCompleteTextBoxItemBaseStyle}`  
应用给 `hc:AutoCompleteTextBox` 的每一项

我们可以直接看一下AutoCompleteTextBox在哪里，![[Pasted image 20260403174939.png]]

这里就是定义了这个组件的地方。

我们主要学习的就是别人是怎么自定义组件的，因此这段代码我们重点来研究，之后我们也会这样来完成。
```
[TemplatePart(Name = SearchTextBox, Type = typeof(System.Windows.Controls.TextBox))]
```
这个特性用于声明“这个自定义控件的模板里，预期应该有哪个命名部件（`PART_...`）以及它的类型”。
里应包含了名为SearchTextBox，通常是某个PART_xxx常量值的元素，该元素类型是TextBox。这样的模式便于开发者一个模板提醒（提醒这个模板里应该有一个名为SearchTextBox的部件，类型是TextBox，但是真正的控件实例必须在ControlTemplate里面自己写出来，比如说x:Name="PART_SearchTextBox"，刚才我们看到了）。 便于控件在 `OnApplyTemplate()` 里用 `GetTemplateChild(...)` 取到部件并绑定逻辑

- 这个特性本身不强制运行时校验
- 真正是否存在、能不能用，要看控件代码在 `OnApplyTemplate` 里怎么处理（判空/抛错等）


## 资源加载

在WPF中，资源字典可能会**重复加载导致的内存膨胀**。如果在大型项目中，不使用`SharedResourceDictionary`，每当你引用一次通用的 XAML 资源（比如 `Colors.xaml`），系统就会在内存中完整地重新创建一套这些对象的副本。


在标准的 `ResourceDictionary` 中，当你这样写：


```xml
<ResourceDictionary.MergedDictionaries>
    <ResourceDictionary Source="pack://application:,,,/HandyControl;component/Themes/SkinDefault.xaml"/>
</ResourceDictionary.MergedDictionaries>
```

如果你有100个窗口或者是用户控件都会引用这个文件的话，WPF就会默认解析和加载100次，十分耗费CPU，因此我们可以使用一个静态的缓存池接管加载过程。

`public static Dictionary<Uri, ResourceDictionary> SharedDictionaries = new();`

这是一个**全局共享的字典**。它以 `Uri`（资源路径）作为 Key，已经解析好的 `ResourceDictionary` 对象作为 Value。


```cs
public class SharedResourceDictionary : ResourceDictionary
{
    public static Dictionary<Uri, ResourceDictionary> SharedDictionaries = new();

    private Uri _sourceUri;

    public new Uri Source
    {
        get => DesignerHelper.IsInDesignMode ? base.Source : _sourceUri;
        set
        {
            if (value == null) return;
            if (DesignerHelper.IsInDesignMode)
            {
                base.Source = value;
                return;
            }
            _sourceUri = value;

            if (!SharedDictionaries.ContainsKey(value))
            {
                base.Source = value;
                SharedDictionaries.Add(value, this);
            }
            else
            {
                MergedDictionaries.Add(SharedDictionaries[value]);
            }
        }
    }
}

```
他通过new关键字隐藏了基类的Source属性，可以重新定义赋值逻辑，当设置 `Source` 时，先去 `SharedDictionaries` 看看这个路径的资源是否已经加载过了。如果没加载过，调用 `base.Source = value` 进行解析，并把结果存入静态字典。如果已经加载过，**不再重新解析**，而是直接把缓存中的实例添加到当前的 `MergedDictionaries` 中。


## 主题管理器

将原本静态的 `ResourceDictionary`（资源字典）扩展成了一个功能强大的**主题管理器**就可以实现三个自动化，皮肤切换、


```cs
public class Theme : ResourceDictionary
{
    public Theme()
    {
        if (DesignerHelper.IsInDesignMode)
        {
            MergedDictionaries.Add(ResourceHelper.GetSkin(SkinType.Default));
            MergedDictionaries.Add(ResourceHelper.GetTheme());
        }
        else
        {
            InitResource();
        }
    }

    #region Skin

    private SkinType _manualSkinType;

    private SkinType _skin;

    public virtual SkinType Skin
    {
        get => _skin;
        set
        {
            if (_skin == value) return;
            _skin = value;

            UpdateSkin();
        }
    }

    public static readonly DependencyProperty SkinProperty = DependencyProperty.RegisterAttached(
        "Skin", typeof(SkinType), typeof(Theme), new PropertyMetadata(default(SkinType), OnSkinChanged));

    private static void OnSkinChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is not FrameworkElement element)
        {
            return;
        }

        var skin = (SkinType) e.NewValue;

        var themes = new List<Theme>();
        GetAllThemes(element.Resources, ref themes);

        if (themes.Count > 0)
        {
            foreach (var theme in themes)
            {
                theme.Skin = skin;
            }
        }
        else
        {
            element.Resources.MergedDictionaries.Add(new Theme
            {
                Skin = skin
            });
        }
    }

    private static void GetAllThemes(ResourceDictionary resourceDictionary, ref List<Theme> themes)
    {
        if (resourceDictionary is Theme theme)
        {
            themes.Add(theme);
        }

        // we must consider it's MergedDictionaries
        foreach (var dictionaryMergedDictionary in resourceDictionary.MergedDictionaries)
        {
            GetAllThemes(dictionaryMergedDictionary, ref themes);
        }
    }

    public static void SetSkin(DependencyObject element, SkinType value)
        => element.SetValue(SkinProperty, value);

    public static SkinType GetSkin(DependencyObject element)
        => (SkinType) element.GetValue(SkinProperty);

    #endregion

    #region SyncWithSystem

    private bool _syncWithSystem;

    public bool SyncWithSystem
    {
        get => _syncWithSystem;
        set
        {
            _syncWithSystem = value;

            if (value)
            {
                _manualSkinType = _skin;
                SyncWithSystemTheme();

                SystemEvents.UserPreferenceChanged += SystemEvents_UserPreferenceChanged;
            }
            else
            {
                SystemEvents.UserPreferenceChanged -= SystemEvents_UserPreferenceChanged;

                _skin = _manualSkinType;
                UpdateSkin();
            }
        }
    }

    private void SystemEvents_UserPreferenceChanged(object sender, UserPreferenceChangedEventArgs e)
    {
        if (e.Category == UserPreferenceCategory.General)
        {
            SyncWithSystemTheme();
        }
    }

    private void SyncWithSystemTheme()
    {
        _skin = SystemHelper.DetermineIfInLightThemeMode() ? SkinType.Default : SkinType.Dark;
        UpdateSkin();
    }

    #endregion

    #region AccentColor

    private Color? _accentColor;

    public Color? AccentColor
    {
        get => _accentColor;
        set
        {
            _accentColor = value;

            if (value == null)
            {
                _precSkin = null;
            }

            UpdateSkin();
        }
    }

    #endregion

    #region Source

    private Uri _source;

    public new Uri Source
    {
        get => DesignerHelper.IsInDesignMode ? null : _source;
        set => _source = value;
    }

    #endregion

    public string Name { get; set; }

    private SkinType _prevSkinType;

    private ResourceDictionary _precSkin;

    public virtual ResourceDictionary GetSkin(SkinType skinType)
    {
        if (_precSkin == null || _prevSkinType != skinType)
        {
            _precSkin = ResourceHelper.GetSkin(skinType);
            _prevSkinType = skinType;
        }

        if (!SyncWithSystem)
        {
            if (AccentColor != null)
            {
                UpdateAccentColor(AccentColor.Value);
            }
        }
        else
        {
            InteropMethods.DwmGetColorizationColor(out var color, out _);
            UpdateAccentColor(ColorHelper.ToColor(color));
        }

        return _precSkin;
    }

    public static Theme GetTheme(string name, ResourceDictionary resourceDictionary)
    {
        if (string.IsNullOrEmpty(name) || resourceDictionary == null)
        {
            return null;
        }

        return resourceDictionary.MergedDictionaries.OfType<Theme>().FirstOrDefault(item => Equals(item.Name, name));
    }

    public virtual ResourceDictionary GetTheme() => ResourceHelper.GetTheme();

    private void InitResource()
    {
        if (DesignerHelper.IsInDesignMode)
        {
            return;
        }

        MergedDictionaries.Clear();
        MergedDictionaries.Add(GetSkin(Skin));
        MergedDictionaries.Add(GetTheme());
    }

    private void UpdateAccentColor(Color color)
    {
        _precSkin[ResourceToken.PrimaryColor] = color;
        _precSkin[ResourceToken.DarkPrimaryColor] = color;
        _precSkin[ResourceToken.TitleColor] = color;
        _precSkin[ResourceToken.SecondaryTitleColor] = color;
    }

    private void UpdateSkin() => MergedDictionaries[0] = GetSkin(Skin);
}

public class StandaloneTheme : Theme
{
    public override ResourceDictionary GetTheme() => ResourceHelper.GetStandaloneTheme();
}

```

这里和上一个文件不同的地方在于，这里主要是主题的相关策略，就比如说切换和控制之类的。

## 运行时汇总

![[Pasted image 20260406174534.png]]

Theme.xaml是一份极为庞大的文件，`Themes/Styles` 是“源码拆分层”，`Theme.xaml` 是“运行时汇总产物层”。

这是为了同时兼具方便人维护和方便运行时加载的功能，运行时一般只加载：

- `pack://application:,,,/HandyControl;component/Themes/Theme.xaml`
这点在 `ResourceHelper.GetTheme()` 里就能看到。

- `Theme_GE45.txt` / `Theme_40.txt` 明确列出了要合并的文件清单（包含 `Basic/*`、`Styles/*`、`Styles/Base/*`）
- `Themes/XamlCombine.exe` 存在，说明有“把拆分样式合并成 Theme.xaml”的构建流程
- `Theme.xaml` 顶部也写了 generated 注释（自动生成）

