
### ContentControl

WPF的ContentControl是WPF控件的一种特殊形式，用于**存储用户输入或从任何其他数据源读取的内容**。内容控件**只能包含一个子元素**。这与包含多个子元素的布局控件（如Grid、WrapPanel和StackPanel控件）不同。

我们通过微软的资料就可以去看见它的定义和用途：
继承

[Object](https://learn.microsoft.com/zh-cn/dotnet/api/system.object?view=windowsdesktop-8.0)

[DispatcherObject](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.threading.dispatcherobject?view=windowsdesktop-8.0)

[DependencyObject](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.dependencyobject?view=windowsdesktop-8.0)

[Visual](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.media.visual?view=windowsdesktop-8.0)

[UIElement](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.uielement?view=windowsdesktop-8.0)

[FrameworkElement](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.frameworkelement?view=windowsdesktop-8.0)

[Control](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.control?view=windowsdesktop-8.0)

ContentControl

派生

[System.Activities.Presentation.View.ExpressionTextBox](https://learn.microsoft.com/zh-cn/dotnet/api/system.activities.presentation.view.expressiontextbox?view=windowsdesktop-8.0)

[System.Activities.Presentation.View.TypePresenter](https://learn.microsoft.com/zh-cn/dotnet/api/system.activities.presentation.view.typepresenter?view=windowsdesktop-8.0)

[System.Activities.Presentation.WorkflowElementDialog](https://learn.microsoft.com/zh-cn/dotnet/api/system.activities.presentation.workflowelementdialog?view=windowsdesktop-8.0)

[System.Activities.Presentation.WorkflowItemPresenter](https://learn.microsoft.com/zh-cn/dotnet/api/system.activities.presentation.workflowitempresenter?view=windowsdesktop-8.0)

[System.Activities.Presentation.WorkflowItemsPresenter](https://learn.microsoft.com/zh-cn/dotnet/api/system.activities.presentation.workflowitemspresenter?view=windowsdesktop-8.0)

[更多…](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.contentcontrol?view=windowsdesktop-8.0# "显示所有派生类")

几乎所有可以使用Content的控件都是继承于它的，比如：
- **`Button`**
    
- **`CheckBox`**
    
- **`RadioButton`**
    
- **`ToggleButton`**
    
- **`RepeatButton`**
还有**容器与窗口**类：
- **`Window`** (WPF 中的顶级窗口)
    
- **`UserControl`** (用户自定义控件)
    
- **`ScrollViewer`** (滚动视图)
    
- **`GroupBox`** (带标题的边框容器)
    
- **`Frame`** (用于导航的框架)
或者说**数据显示与修饰类**：
- **`Label`** (注意：`TextBlock` **不是** `ContentControl`，但 `Label` 是。区别在于 `Label` 可以放图片，`TextBlock` 只能放文字)
    
- **`ToolTip`** (鼠标悬停提示)
    
- **`StatusBarItem`**
列表项：
- **`ListBoxItem`**
    
- **`ComboBoxItem`**
    
- **`TabItem`**
    
- **`TreeViewItem`** (它比较特殊，既是 `ContentControl` 又是 `ItemsControl`，因为它有 Header 也有子节点)

[ContentControl](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.contentcontrol?view=windowsdesktop-8.0)可以包含任何类型的公共语言运行时对象 (，例如字符串或[DateTime](https://learn.microsoft.com/zh-cn/dotnet/api/system.datetime?view=windowsdesktop-8.0)对象) 或[UIElement](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.uielement?view=windowsdesktop-8.0)对象 (，如 [Rectangle](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.shapes.rectangle?view=windowsdesktop-8.0) 或 [Panel](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.panel?view=windowsdesktop-8.0)) 。 这使你能够将丰富内容添加到 控件，例如 [Button](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.button?view=windowsdesktop-8.0) 和 [CheckBox](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.checkbox?view=windowsdesktop-8.0)。

与他相对的就是条目控件ItemsControl，它则是可以容纳N个控件，我们可以控制子组件的加载周期，ItemsControl有一个核心机制叫做容器生成器，数据进来的时候，为每个数据包裹一个容器，例如ListBoxItems或者默认ContentPresenter，我们就可以监听这个容器的Loaded和Unloaded事件。

实现子组件加载的动画，比如说左侧滑入，我们可以修改ItemsControl的ItemContainerStyle来实现。
```xml
<ItemsControl ItemsSource="{Binding MyItems}">
    <ItemsControl.ItemContainerStyle>
        <Style TargetType="ContentPresenter">
            <Style.Triggers>
                <EventTrigger RoutedEvent="Loaded">
                    <BeginStoryboard>
                        <Storyboard>
                            <DoubleAnimation Storyboard.TargetProperty="Opacity" 
                                             From="0" To="1" Duration="0:0:0.5"/>
                        </Storyboard>
                    </BeginStoryboard>
                </EventTrigger>
            </Style.Triggers>
        </Style>
    </ItemsControl.ItemContainerStyle>
</ItemsControl>
```
这个就不细讲了，我们可以在另外的笔记里面细讲。

### 主要内容

```cs
using System.Windows;  
using System.Windows.Controls;  
using System.Windows.Media;  
using System.Windows.Media.Animation;  
  
namespace TheTestSolutionForWPF.Views;  
public class ContentWrapper : ContentControl  
{  
    public static readonly DependencyProperty IsActiveProperty = DependencyProperty.Register(  
        nameof(IsActive), typeof(bool), typeof(ContentWrapper), new PropertyMetadata(true, OnIsActiveChanged));  
        
    public bool IsActive  
    {  
        get => (bool)GetValue(IsActiveProperty);  
        set => SetValue(IsActiveProperty, value);  
    }    
    public ContentWrapper()   
    {  
        this.Background = Brushes.Transparent;  
        var group = new TransformGroup();  
        group.Children.Add(new ScaleTransform(1, 1));  
        group.Children.Add(new TranslateTransform(0, 0));  
        this.RenderTransform = group;  
        this.RenderTransformOrigin = new Point(0.5, 0.5); // 从中心缩放  
    }  
    
    private static void OnIsActiveChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)  
    {        var control = (ContentWrapper)d;  
        bool newValue = (bool)e.NewValue;  
            if (newValue)  
        {            control.AnimateIn();  
        }        else  
        {  
            control.AnimateOut();  
        }    }    private void AnimateIn()  
        {            this.Visibility = Visibility.Visible;  
            var springEase = new ElasticEase   
{   
Oscillations = 1,   
Springiness = 5   
};  
            var scaleAnim = new DoubleAnimation(0.8, 1.0, TimeSpan.FromSeconds(0.8))  
            {                EasingFunction = springEase  
            };  
            var opacityAnim = new DoubleAnimation(0, 1, TimeSpan.FromSeconds(0.3));  
            var transformGroup = this.RenderTransform as TransformGroup;  
            var scaleTransform = transformGroup.Children[0] as ScaleTransform;  
            scaleTransform.BeginAnimation(ScaleTransform.ScaleXProperty, scaleAnim);  
            scaleTransform.BeginAnimation(ScaleTransform.ScaleYProperty, scaleAnim);  
            this.BeginAnimation(UIElement.OpacityProperty, opacityAnim);  
        }        private void AnimateOut()  
        {            var smoothEase = new QuadraticEase { EasingMode = EasingMode.EaseOut };  
            var scaleAnim = new DoubleAnimation(1.0, 0.9, TimeSpan.FromSeconds(0.3))  
            {                EasingFunction = smoothEase  
            };  
            var opacityAnim = new DoubleAnimation(1, 0, TimeSpan.FromSeconds(0.2));  
            opacityAnim.Completed += (s, e) =>   
            {  
                if (!this.IsActive)   
                {  
                    this.Visibility = Visibility.Collapsed;  
                }            };            
                var transformGroup = this.RenderTransform as TransformGroup;  
            var scaleTransform = transformGroup.Children[0] as ScaleTransform;  
            scaleTransform.BeginAnimation(ScaleTransform.ScaleXProperty, scaleAnim);  
            scaleTransform.BeginAnimation(ScaleTransform.ScaleYProperty, scaleAnim);  
            this.BeginAnimation(UIElement.OpacityProperty, opacityAnim);  
        }}
```

上面的代码是行为封装在自定义控制内部的模式也就是：Logic-less XAML, Logic-rich Control
是符合WPF/Avalonia的高级开发习惯的。


```cs
public static readonly DependencyProperty IsActiveProperty = DependencyProperty.Register(  
    nameof(IsActive), typeof(bool), typeof(ContentWrapper), new PropertyMetadata(true, OnIsActiveChanged));
```
这里是我们注册依赖属性的地方，我们注册了一个叫做IsActive的属性，类型是Bool，属于一个叫做ContentWrapper的类，这个属性默认为True，最后面部分我们注册了一个回调函数，这个是属性的监听器，当IsActive从True变为了Flase的时候或者反之，他就会调用这个静态方法。

我们的AnimateIn和AnimateOut就是靠这个钩子来触发的。

```cs
public bool IsActive  
{  
    get => (bool)GetValue(IsActiveProperty);  
    set => SetValue(IsActiveProperty, value);  
}
```
这个地方是依赖属性的CLR包装器，GetValue因为返回的是object，我们必须进行强制转换使用bool。SetValue用于写入。

```cs
public ContentWrapper() {  
    this.Background = Brushes.Transparent;  
    var group = new TransformGroup();  
    group.Children.Add(new ScaleTransform(1, 1));  
    group.Children.Add(new TranslateTransform(0, 0));  
    this.RenderTransform = group;  
    this.RenderTransformOrigin = new Point(0.5, 0.5); // 从中心缩放  
}
```
`ContentControl` 的 `Background` 默认是 `null`。在 WPF/Avalonia 中，`null` 意味着**“点击穿透”**（Hit Test Invisible）。也就是说，如果你不设置背景，鼠标点在控件空白处时，事件会直接穿透下去，仿佛这个控件不存在。

下面则是创建了一个组合动画，我们创建了一个缩放和变换动画，**初始化值 (1, 1) 和 (0, 0)**：这是“无变化”状态，保证控件刚显示时是正常的 1:1 大小，位置不动。别忘记编写锚点，很多人会忘记，因为锚点的默认值是0,0 不写的话就会向着左上角收缩（如果你就想要这个效果就随意）。

终于来到了我们的回调函数

```cs
private static void OnIsActiveChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)  
{  
    var control = (ContentWrapper)d;  
    bool newValue = (bool)e.NewValue;  
    if (newValue)  
    {        
    control.AnimateIn();  
    }    
    else  
    {  
        control.AnimateOut();  
    }}
```
它是一个静态的，因为WPF的依赖属性是注册在类层面上，而不是某个具体的对象，内存里只有一份IsActiveProperty的定义，因为函数是静态的，他不知道哪个控件变了，因此WPF会把出事的控件实例通过参数d传递进来。

我们通过了强制类型转换拿到了我们想要的control变量，才能使用AnimatIn和Out。参数e是非常有用的，他有两个属性，一个是OldValue一个是NewValuw以及Property（哪个属性变了），一般我们**还会再利用OldValue实现一个防抖检查**。

在WPF/Avalonia中，面对频繁开关的正确处理不是防抖，是截断，当新的指令来了的时候就会停止手头的动画，马上开始新的目标，你可能马上就理解了，就是使用取消令牌。

```cs
private CancellationTokenSource _debounceCts;

private static void OnIsActiveChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
{
    var control = (ContentWrapper)d;
    
    // 1. 只要有新变化，先取消之前的计划
    control._debounceCts?.Cancel();
    control._debounceCts = new CancellationTokenSource();
    var token = control._debounceCts.Token;

    // 2. 延迟执行 (比如等 200ms)
    Task.Delay(200, token).ContinueWith(t => 
    {
        if (t.IsCanceled) return; // 如果期间又变了，这次就作废

        // 3. 只有等了 200ms 还没新变化，才真正执行 UI 操作
        control.Dispatcher.Invoke(() => 
        {
            if ((bool)e.NewValue) control.AnimateIn();
            else control.AnimateOut();
        });
    });
}
```




