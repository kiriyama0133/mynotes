我们最常使用到的是avalonia的属性动画，比如Transitions标签，我们经常在style标签里去定义Property去指定Transitions，然后在Transitions标签中去影响所有的过渡类型。

根据Avalonia的实现，Transitions集合支持以下的过渡类型：
### **DoubleTransition**
- **Property 支持**: 所有 `double` 类型的属性 XamlIlTests.cs:31-33
- **常见属性**: `Opacity`、`Width`、`Height`、`FontSize` 等 ScrollBar.xaml:74
- **影响的变换**: 数值的平滑过渡

### **TransformOperationsTransition**
- **Property 支持**: `RenderTransform` 属性 TransitionsPage.xaml:49
- **支持的变换函数**:
    - `translateX()` / `translateY()` / `translate()` - 平移
    - `scaleX()` / `scaleY()` / `scale()` - 缩放
    - `rotate()` - 旋转
    - `skewX()` / `skewY()` / `skew()` - 倾斜

### **CornerRadiusTransition**
- **Property 支持**: `CornerRadius` 属性 ScrollBar.xaml:33
- **影响的变换**: 圆角半径的平滑过渡

### **ThicknessTransition**

- **Property 支持**: `Thickness` 类型的属性
- **常见属性**: `Margin`、`Padding`、`BorderThickness`
- **影响的变换**: 边距/内边距的平滑过渡

### **BrushTransition**

- **Property 支持**: `IBrush` 类型的属性
- **常见属性**: `Background`、`Foreground`、`BorderBrush`
- **影响的变换**: 颜色/画刷的平滑过渡

### **FloatTransition**

- **Property 支持**: `float` 类型的属性
- **影响的变换**: 浮点数值的平滑过渡

### **VectorTransition**

- **Property 支持**: `Vector` 类型的属性
- **影响的变换**: 向量值的平滑过渡


### **PointTransition**

- **Property 支持**: `Point` 类型的属性
- **影响的变换**: 点坐标的平滑过渡

### **SizeTransition**

- **Property 支持**: `Size` 类型的属性
- **影响的变换**: 尺寸的平滑过渡

所有的过渡类型都是继承自TransitionBase，因此都是支持以下的属性的：
- **Duration**: 过渡持续时间 TransitionBase.cs:57-61
- **Delay**: 延迟开始时间 TransitionBase.cs:66-70
- **Easing**: 缓动函数 TransitionBase.cs:75-79
- **Property**: 要过渡的属性 TransitionBase.cs:83-87

这些过渡类型都是可以组合使用的，比如说我们常见的SchrollBar组件：
```XAML
 <Setter Property="Transitions">
      <Transitions>
        <CornerRadiusTransition Property="CornerRadius" Duration="0:0:0.1" />
        <TransformOperationsTransition Property="RenderTransform" Duration="0:0:0.1" />
      </Transitions>
    </Setter>
```

过渡类型都是实现了ITransition接口的，过渡只有在被附加到了可视树的时候才会生效。其中我们的TransformOperationsTransition是最为灵活的，可以处理所有的2D变换组合。

```c#
namespace Avalonia.Animation  
{  
    /// <summary>  
    /// Defines how a property should be animated using a transition.    /// </summary>    public abstract class TransitionBase : AvaloniaObject, ITransition  
    {  
        /// <summary>  
        /// Defines the <see cref="Duration"/> property.        /// </summary>        public static readonly DirectProperty<TransitionBase, TimeSpan> DurationProperty =  
            AvaloniaProperty.RegisterDirect<TransitionBase, TimeSpan>(  
                nameof(Duration),  
                o => o._duration,  
                (o, v) => o._duration = v);  
        /// <summary>  
        /// Defines the <see cref="Delay"/> property.        /// </summary>        public static readonly DirectProperty<TransitionBase, TimeSpan> DelayProperty =  
            AvaloniaProperty.RegisterDirect<TransitionBase, TimeSpan>(  
                nameof(Delay),  
                o => o._delay,  
                (o, v) => o._delay = v);  
        /// <summary>  
        /// Defines the <see cref="Easing"/> property.        /// </summary>        public static readonly DirectProperty<TransitionBase, Easing> EasingProperty =  
            AvaloniaProperty.RegisterDirect<TransitionBase, Easing>(  
                nameof(Easing),  
                o => o._easing,  
                (o, v) => o._easing = v);  
        /// <summary>  
        /// Defines the <see cref="Property"/> property.        /// </summary>        public static readonly DirectProperty<TransitionBase, AvaloniaProperty?> PropertyProperty =  
            AvaloniaProperty.RegisterDirect<TransitionBase, AvaloniaProperty?>(  
                nameof(Property),  
                o => o._prop,  
                (o, v) => o._prop = v);  
  
        private TimeSpan _duration;  
        private TimeSpan _delay = TimeSpan.Zero;  
        private Easing _easing = new LinearEasing();  
        private AvaloniaProperty? _prop;  
  
        /// <summary>  
        /// Gets or sets the duration of the transition.        /// </summary>public TimeSpan Duration  
        {  
            get { return _duration; }  
            set { SetAndRaise(DurationProperty, ref _duration, value); }  
        }  
        /// <summary>  
        /// Gets or sets delay before starting the transition.        /// </summary>public TimeSpan Delay  
        {  
            get { return _delay; }  
            set { SetAndRaise(DelayProperty, ref _delay, value); }  
        }  
        /// <summary>  
        /// Gets the easing class to be used.        /// </summary>        public Easing Easing  
        {  
            get { return _easing; }  
            set { SetAndRaise(EasingProperty, ref _easing, value); }  
        }  
        /// <inheritdoc/>  
        [DisallowNull]  
        public AvaloniaProperty? Property  
        {  
            get { return _prop; }  
            set { SetAndRaise(PropertyProperty, ref _prop, value); }  
        }  
        AvaloniaProperty ITransition.Property  
        {  
            get => Property ?? throw new InvalidOperationException("Transition has no property specified.");  
            set => Property = value;  
        }  
        /// <inheritdoc/>  
        IDisposable ITransition.Apply(Animatable control, IClock clock, object? oldValue, object? newValue)  
            => Apply(control, clock, oldValue, newValue);  
        internal abstract IDisposable Apply(Animatable control, IClock clock, object? oldValue, object? newValue);  
  
        internal override void BuildDebugDisplay(StringBuilder builder, bool includeContent)  
        {            base.BuildDebugDisplay(builder, includeContent);  
            DebugDisplayHelper.AppendOptionalValue(builder, nameof(Property), Property, includeContent);  
            DebugDisplayHelper.AppendOptionalValue(builder, nameof(Duration), Duration, includeContent);  
        }    }
```

现在我们来讲一下常用的Easing效果，比较常用的有平滑过渡，线性的，二阶的、三阶的，还有可以创造出复杂动画效果的弹簧振子。
### 二次方缓动 (Quadratic)

- **QuadraticEaseIn** - 二次方加速
- **QuadraticEaseOut** - 二次方减速
- **QuadraticEaseInOut** - 二次方先加速后减速

### 三次方缓动 (Cubic)

- **CubicEaseIn** - 三次方加速
- **CubicEaseOut** - 三次方减速
- **CubicEaseInOut** - 三次方先加速后减速

### 四次方缓动 (Quartic)

- **QuarticEaseIn** - 四次方加速
- **QuarticEaseOut** - 四次方减速
- **QuarticEaseInOut** - 四次方先加速后减速

### 五次方缓动 (Quintic)

- **QuinticEaseIn** - 五次方加速
- **QuinticEaseOut** - 五次方减速
- **QuinticEaseInOut** - 五次方先加速后减速

### 指数缓动 (Exponential)

- **ExponentialEaseIn** - 指数加速
- **ExponentialEaseOut** - 指数减速
- **ExponentialEaseInOut** - 指数先加速后减速

### 圆形缓动 (Circular)

- **CircularEaseIn** - 圆形加速
- **CircularEaseOut** - 圆形减速
- **CircularEaseInOut** - 圆形先加速后减速

### 回弹缓动 (Back)

- **BackEaseIn** - 回弹加速(先后退再前进)
- **BackEaseOut** - 回弹减速(超过目标后回弹) KeySplineTests.cs:219-220
- **BackEaseInOut** - 回弹先加速后减速

### 弹性缓动 (Elastic)

- **ElasticEaseIn** - 弹性加速(震荡加速)
- **ElasticEaseOut** - 弹性减速(震荡减速) KeySplineTests.cs:223
- **ElasticEaseInOut** - 弹性先加速后减速

### 弹跳缓动 (Bounce)

- **BounceEaseIn** - 弹跳加速
- **BounceEaseOut** - 弹跳减速 BounceEaseUtils.cs:14
- **BounceEaseInOut** - 弹跳先加速后减速

### 自定义贝塞尔曲线

- **SplineEasing** - 使用自定义三次贝塞尔曲线控制点 (X1, Y1, X2, Y2) SplineEasing.cs:9


## Animation支持的数值类型

我们在avalonia里经常使用Animations，比如我们常常创建一个延迟透明度和Translate平移的组合动画，通过设定visual对象和自定义动画关键字和延迟的方式创建：

```c#
private void TranslateAnimation()    {      
ScrollViewer navigationScrollViewer = NaviScrollViewer;      
if (navigationScrollViewer.Content is StackPanel stackPanel)      
    {      
var children = stackPanel.Children;      
foreach (var child in children)      
        {      
child.RenderTransform = new TranslateTransform(-50, 0);    
child.Opacity = 0;  
        }for (int i = 0; i < children.Count; i++)      
        {      
var child = children[i];      
var delay = TimeSpan.FromMilliseconds(i * 200);      
var animation = new Animation      
{      
Duration = TimeSpan.FromMilliseconds(500),      
Delay = delay,      
FillMode = FillMode.Forward,      
Easing = new BackEaseOut(),  
                Children =      
                {      
new KeyFrame      
{      
Cue = new Cue(0),      
Setters =      
                        {      
new Setter(TranslateTransform.XProperty, -50.0),  // 关键:作为附加属性使用    
}      
                    },      
new KeyFrame      
{      
Cue = new Cue(1),      
Setters =      
                        {      
new Setter(TranslateTransform.XProperty, 0.0),    
}      
                    }      
                }      
            };  
            var animation_opacity = new Animation  
            {  
                Duration = TimeSpan.FromMilliseconds(500),  
                Delay = delay,  
                FillMode = FillMode.Forward,  
                Children =  
                {                    new KeyFrame  
                    {  
                        Cue = new Cue(0),  
                        Setters =  
                        {                            new Setter(OpacityProperty, 0.0),  
                        }  
                    },                    new KeyFrame  
                    {  
                        Cue = new Cue(1),  
                        Setters =  
                        {                            new Setter(OpacityProperty, 1.0),  
                        }  
                    }                }            };            animation_opacity.RunAsync(child);  
            animation.RunAsync(child);    
        }      
}    }
```
除了Opacity，还有很多属性也是可以这样使用的：
### **数值类型**

- **Double**: `Opacity`、`Width`、`Height`、`FontSize` 等<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Visual.cs:64-65><link to Repo AvaloniaUI/Avalonia: tests/Avalonia.Base.UnitTests/Animation/AnimatableTests.cs:461-469>
- **Float**: 浮点数属性
- **Integer**: 整数属性

### **变换属性**

- **TranslateTransform.X / Y**: 平移变换<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Media/TranslateTransform.cs:15-22><link to Repo AvaloniaUI/Avalonia: tests/Avalonia.Base.UnitTests/Animation/KeySplineTests.cs:236-242>
- **RotateTransform.Angle**: 旋转角度<link to Repo AvaloniaUI/Avalonia: samples/RenderDemo/Pages/AnimationsPage.xaml:54-62>
- **ScaleTransform.ScaleX / ScaleY**: 缩放比例<link to Repo AvaloniaUI/Avalonia: samples/RenderDemo/Pages/AnimationsPage.xaml:58-71>
- **SkewTransform.AngleX / AngleY**: 倾斜角度<link to Repo AvaloniaUI/Avalonia: samples/RenderDemo/Pages/AnimationsPage.xaml:102-107>

### **颜色和画刷**

- **Color**: 颜色值<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/composition-schema.xml:63-63>
- **Background / Foreground**: 背景和前景画刷<link to Repo AvaloniaUI/Avalonia: samples/RenderDemo/Pages/AnimationsPage.xaml:116-137>

### **布局属性**

- **Width / Height**: 宽度和高度<link to Repo AvaloniaUI/Avalonia: tests/Avalonia.Base.UnitTests/Animation/AnimatableTests.cs:105-111>
- **Margin**: 外边距
- **Padding**: 内边距

### **视觉效果**

- **BoxShadow**: 阴影效果<link to Repo AvaloniaUI/Avalonia: samples/RenderDemo/Pages/AnimationsPage.xaml:147-149>
- **CornerRadius**: 圆角半径
- **BorderThickness**: 边框粗细

### **Composition API 属性**

- **Offset**: 位置偏移 (Vector3D)<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/composition-schema.xml:24-24>
- **Scale**: 缩放 (Vector3D)<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/composition-schema.xml:30-30>
- **RotationAngle**: 旋转角度<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/composition-schema.xml:28-28>
- **Visible**: 可见性<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/composition-schema.xml:19-19>

### **其他类型**

- **Boolean**: 布尔值属性<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/composition-schema.xml:62-62>
- **Vector / Vector2 / Vector3**: 向量类型<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/composition-schema.xml:64-67>
- **Quaternion**: 四元数(用于 3D 旋转)<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/composition-schema.xml:69-69>

需要题型一点的是，Children属性不是StackPanel的特权，一般来说，只要是Panel类型，那么都是有Children这个属性的，除了垂直、水平堆叠布局StackPanel以外，我们还有其他常用的Panel类型：
- **Grid** - 网格布局
- **Canvas** - 绝对定位布局
- **DockPanel** - 停靠布局
- **WrapPanel** - 自动换行布局
- **VirtualizingStackPanel** - 虚拟化堆叠布局

对于使用ItemsControl的场景，我们可以通过ItemsPresenter来获取实际的面板和子元素。
```c#
  internal Control? ContainerFromIndex(int index)
        {
            if (Panel is VirtualizingPanel v)
                return v.ContainerFromIndex(index);
            return index >= 0 && index < Panel?.Children.Count ? Panel.Children[index] : null;
        }

        internal IEnumerable<Control>? GetRealizedContainers()
        {
            if (Panel is VirtualizingPanel v)
                return v.GetRealizedContainers();
            return Panel?.Children;
        }
```
 在 Composition API 的隐式动画示例中,也使用了 `ItemsControl` 配合 `WrapPanel` CompositionPage.axaml:9-14
 ```c#
<ItemsControl x:Name="Items">
            <ItemsControl.ItemsPanel>
              <ItemsPanelTemplate>
                <WrapPanel/>
              </ItemsPanelTemplate>
            </ItemsControl.ItemsPanel>
            <ItemsControl.DataTemplates>
              <DataTemplate DataType="pages:CompositionPageColorItem">
                <Border 
                  pages:CompositionPage.EnableAnimations="True"
                  Padding="10" BorderBrush="Gray" BorderThickness="2"
                  Background="{Binding ColorBrush}" Width="100" Height="100" Margin="10">
                    <TextBlock Text="{Binding ColorHexValue}"/>
                </Border>
              </DataTemplate>
            </ItemsControl.DataTemplates>
```

### 视口判断
Avalonia可以通过`EffectiveViewportChanged` 事件来获取组件是否在视口中，利用它我们可以完成一些加载动画。

我们需要将要判断的控件放在scrollViewer中，如果没有在里面的话，那么被判定的组件的`EffectiveViewport` 通常会覆盖整个控件区域,导致 `isInViewport` 始终为 `true`

 `EffectiveViewport` 的计算会递归考虑所有父控件的 `ClipToBounds` 和 `RenderTransform` 设置<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Layout/LayoutManager.cs:404-437>



```
private void OnAttachedToVisualTree(object sender, VisualTreeAttachmentEventArgs e)
{
    MyBorder.EffectiveViewportChanged += OnEffectiveViewportChanged;
}

private void OnEffectiveViewportChanged(object? sender, EffectiveViewportChangedEventArgs e)
{
    var viewport = e.EffectiveViewport;
    var controlBounds = new Rect(MyBorder.Bounds.Size);

    // 判断是否部分可见  
    bool isInViewport = viewport.Intersects(controlBounds);

    // 判断是否完全可见  
    bool isFullyInViewport = viewport.Contains(controlBounds);

    if (DataContext is MainViewModel vm)
    {
        vm.IsViewport = isInViewport.ToString();
        vm.IsFullyviewport = isFullyInViewport.ToString();
    }
}
```
于是我们这样设置，就可以在窗口上观察到我们想要的内容了，可以正确判断是否在视口上，并且是否完整显示了，并且，我们即便是将opacity调整为了0，也是只影响控件的视觉渲染，而不会影响布局和视口检测，我们完全可以用它来**创建一些淡入的加载动画**。


```cs
using Avalonia;
using Avalonia.Controls;
using Avalonia.Interactivity;
using Avalonia.Layout;
using Avalonia.LogicalTree;
using Avalonia.Media;
using Avalonia.Rendering.Composition;
using Avalonia.VisualTree;
using AvaloniaTestDemo.ViewModels;
using System;

namespace AvaloniaTestDemo.Views;

public partial class MainView : UserControl
{
    public MainView()
    {
        InitializeComponent();
        MyBorder.Loaded += MyBorder_Loaded;
        MyBorder.AttachedToVisualTree += OnAttachedToVisualTree;
        MyBorder.AttachedToLogicalTree += WrapPanelLoadAnimation;
    }

    private void OnAttachedToVisualTree(object sender, VisualTreeAttachmentEventArgs e)
    {
        MyBorder.EffectiveViewportChanged += OnEffectiveViewportChanged;
    }

    private void WrapPanelLoadAnimation(object? sender, LogicalTreeAttachmentEventArgs e)
    {
        if (ContentContainer is WrapPanel wrapPanel && ContentContainer is not null)
        {
            var childrens = wrapPanel.Children;
            foreach (var child in childrens)
            {
                child.Opacity = 0;
                child.EffectiveViewportChanged += EffectiveViewportChangedAnimation;
            }
        }
    }

    private void EffectiveViewportChangedAnimation(object? sender, EffectiveViewportChangedEventArgs e)
    {
        var viewport = e.EffectiveViewport;
        if (sender is Border border)
        {
            var controlBounds = new Rect(border.Bounds.Size); 
            bool isInViewport = viewport.Intersects(controlBounds);

            if (isInViewport)
            {
                border.Opacity = 1;
                border.EffectiveViewportChanged -= EffectiveViewportChangedAnimation;
            }
        }
    }

    private void OnEffectiveViewportChanged(object? sender, EffectiveViewportChangedEventArgs e)
    {
        var viewport = e.EffectiveViewport;
        var controlBounds = new Rect(MyBorder.Bounds.Size);

        // 判断是否部分可见  
        bool isInViewport = viewport.Intersects(controlBounds);

        // 判断是否完全可见  
        bool isFullyInViewport = viewport.Contains(controlBounds);

        if (DataContext is MainViewModel vm)
        {
            vm.IsViewport = isInViewport.ToString();
            vm.IsFullyviewport = isFullyInViewport.ToString();
        }
    }

    private void MyBorder_Loaded(object? sender, RoutedEventArgs e)
    {
        var visual = ElementComposition.GetElementVisual(MyBorder);
        if (visual == null) return;
        var compositor = visual.Compositor;
        if (compositor == null) return;


        // 创建 Offset 动画  
        var offsetAnimation = compositor.CreateVector3KeyFrameAnimation();
        offsetAnimation.Target = "Offset";
        offsetAnimation.InsertExpressionKeyFrame(1, "this.FinalValue");
        offsetAnimation.Duration = TimeSpan.FromMilliseconds(2000);
        var implicitAnimations = compositor.CreateImplicitAnimationCollection();
        implicitAnimations["Offset"] = offsetAnimation;

        // 为 visual 设置 Offset 动画  
        visual.ImplicitAnimations = implicitAnimations;
        MyBorder.Loaded -= MyBorder_Loaded;
    }

    private void Border_Click(object? sender, Avalonia.Input.PointerPressedEventArgs e)
    {
        if (sender is Border border)
        {
            var visual = ElementComposition.GetElementVisual(border);
            if (visual == null) return; // Check if visual is null before accessing its properties
            visual.ClipToBounds = true;
            visual.Offset += new Vector3D(100, 100, 0); // x=100, y=100
        }
    }
}
```
就这样，我们就可以实现当组件出现在视口的时候进行透明度变化加载（前提是要把属性动画设置好）。