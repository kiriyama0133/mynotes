composition api 可以为我们在所绑定的属性的值发生变化的时候自动产生隐式动画，这个可以为我们节省很多的时间，并且Composition API 是在**渲染线程**上执行的，而**不是UI线程上执行**的。

Compositor类也就是我们说的合成器， 管理着UI线程和渲染线程之前的通信，动画计算和渲染在独立的渲染线程上执行而不会阻塞UI线程。在Win平台上，composition api 可以利用DirectCompositon或者WINUI composition 进行硬件加速渲染，**它使用批量提交机制，将多个属性变化打包成一个批次发送到渲染线程上**，减少了线程之间的通信次数，提高了整体性能。

光说不看代码没用用处，我们直接上源码：

```cs
using System;
using System.Diagnostics;
using System.Runtime.CompilerServices;
using Avalonia.Rendering.Composition.Animations;
using Avalonia.Rendering.Composition.Expressions;
using Avalonia.Rendering.Composition.Server;
using Avalonia.Rendering.Composition.Transport;
using Avalonia.Utilities;

namespace Avalonia.Rendering.Composition
{
    /// <summary>
    /// Base class of the composition API representing a node in the visual tree structure.
    /// Composition objects are the visual tree structure on which all other features of the composition API use and build on.
    /// The API allows developers to define and create one or many <see cref="CompositionVisual" /> objects each representing a single node in a Visual tree.
    /// </summary>
    public abstract class CompositionObject : ICompositorSerializable
    {
        /// <summary>
        /// The collection of implicit animations attached to this object.
        /// </summary>
        public ImplicitAnimationCollection? ImplicitAnimations { get; set; }

        private protected InlineDictionary<CompositionProperty, IAnimationInstance> PendingAnimations;
        internal CompositionObject(Compositor compositor, SimpleServerObject? server)
        {
            Compositor = compositor;
            Server = server;
        }
        
        /// <summary>
        /// The associated Compositor
        /// </summary>
        public Compositor Compositor { get; }

        SimpleServerObject ICompositorSerializable.TryGetServer(Compositor c)
        {
            Debug.Assert(c == Compositor);
            return Server ?? ThrowInvalidOperation();
        }

        internal SimpleServerObject? Server { get; }
        public bool IsDisposed { get; private set; }
        private bool _registeredForSerialization;

        [MethodImpl(MethodImplOptions.NoInlining)]
        private static SimpleServerObject ThrowInvalidOperation() =>
            throw new InvalidOperationException("There is no server-side counterpart for this object");

        protected internal void Dispose()
        {
            if (!IsDisposed && Server != null)
                Compositor.DisposeOnNextBatch(Server);
            IsDisposed = true;
        }

        /// <summary>
        /// Connects an animation with the specified property of the object and starts the animation.
        /// </summary>
        public void StartAnimation(string propertyName, CompositionAnimation animation)
            => StartAnimation(propertyName, animation, null);
        
        internal virtual void StartAnimation(string propertyName, CompositionAnimation animation, ExpressionVariant? finalValue)
        {
            throw new ArgumentException("Unknown property " + propertyName);
        }

        /// <summary>
        /// Starts an animation group.
        /// The StartAnimationGroup method on CompositionObject lets you start CompositionAnimationGroup.
        /// All the animations in the group will be started at the same time on the object.
        /// </summary>
        public void StartAnimationGroup(ICompositionAnimationBase grp)
        {
            if (grp is CompositionAnimation animation)
            {
                if(animation.Target == null)
                    throw new ArgumentException("Animation Target can't be null");
                StartAnimation(animation.Target, animation);
            }
            else if (grp is CompositionAnimationGroup group)
            {
                foreach (var a in group.Animations)
                {
                    if (a.Target == null)
                        throw new ArgumentException("Animation Target can't be null");
                    StartAnimation(a.Target, a);
                }
            }
        }

        bool StartAnimationGroupPart(CompositionAnimation animation, string target, ExpressionVariant finalValue)
        {
            if(animation.Target == null)
                throw new ArgumentException("Animation Target can't be null");
            if (animation.Target == target)
            {
                StartAnimation(animation.Target, animation, finalValue);
                return true;
            }
            else
            {
                StartAnimation(animation.Target, animation);
                return false;
            }
        }
        
        internal bool StartAnimationGroup(ICompositionAnimationBase grp, string target, ExpressionVariant finalValue)
        {
            if (grp is CompositionAnimation animation)
                return StartAnimationGroupPart(animation, target, finalValue);
            if (grp is CompositionAnimationGroup group)
            {
                var matched = false;
                foreach (var a in group.Animations)
                {
                    if (a.Target == null)
                        throw new ArgumentException("Animation Target can't be null");
                    if (StartAnimationGroupPart(a, target, finalValue))
                        matched = true;
                }

                return matched;
            }

            throw new ArgumentException();
        }

        protected void RegisterForSerialization()
        {
            if (Server == null)
                throw new InvalidOperationException("The object doesn't have an associated server counterpart");
            
            if(_registeredForSerialization)
                return;
            _registeredForSerialization = true;
            Compositor.RegisterForSerialization(this);
        }

        void ICompositorSerializable.SerializeChanges(Compositor c, BatchStreamWriter writer)
        {
            Debug.Assert(c == Compositor);
            _registeredForSerialization = false;
            SerializeChangesCore(writer);
        }

        private protected virtual void SerializeChangesCore(BatchStreamWriter writer)
        {
        }
    }
}

```
`ImplicitAnimations` 是 `CompositionObject` 的一个公开属性,可以直接设置。这正是我们之前使用的方式，每个 `CompositionObject` 都有一个关联的 `Server` 对象,这是**渲染线程上的对应实体**。这就是为什么 Composition API 能在渲染线程上执行动画的原因。

`RegisterForSerialization()` 方法确保**对象的更改会被序列化并发送到渲染线程**。这是 UI 线程和渲染线程之间通信的关键机制。

`StartAnimation` 和 `StartAnimationGroup` 方法提供了显式启动动画的方式,而 `ImplicitAnimations` 则是在属性变化时自动触发。

所以我们应该推荐使用这种动画机制，下面我们使用一个简单的例子来看看，我们试着在loaded之后获取一下组件对象，然后获取组件的合成器来达到我们想要的效果。

```cs
 private void MyBorder_Loaded(object? sender, RoutedEventArgs e)
 {
     var visual = ElementComposition.GetElementVisual(MyBorder);
     if (visual == null) return; 
     var compositor = visual.Compositor;
     if (compositor == null) return;
     
     var offsetAnimation = compositor.CreateVector3KeyFrameAnimation();
     offsetAnimation.Target = "Offset";
     offsetAnimation.InsertExpressionKeyFrame(1, "this.FinalValue");
     offsetAnimation.Duration = TimeSpan.FromMilliseconds(2000);
     
     var implicaitAnimation = compositor.CreateImplicitAnimationCollection();
     implicaitAnimation["Offset"] = offsetAnimation;
     visual.ImplicitAnimations = implicaitAnimation;
     MyBorder.Loaded -= MyBorder_Loaded;
 }
```
我们把这个事件给组件的Loaded去订阅，我们得到组件的合成器，然后利用它创建一个三维动画，绑定属性到Offset上面，**为什么这里是3keyframe**，因为它创建的是`Vector3DKeyFrameAnimation`,专门用于动画化 `Vector3D` 类型的属性。

在 Composition API 的关键帧动画中,`InsertExpressionKeyFrame` 方法的第一个参数是 `normalizedProgressKey`,它是一个 **0.0 到 1.0 之间的浮点数**,表示动画时间轴上的位置:
- `0.0` (或 `0`) = 动画开始 (0%)
- `0.5` (或 `0.5f`) = 动画中间 (50%)
- `1.0` (或 `1`) = 动画结束 (100%) Generator.KeyFrameAnimation.cs:39-42
后面的不用想就是设置持续时间，之后我们通过合成器去创建隐式动画，将隐式动画里加入我们之前创建的动画，然后将它传递给visual的隐式动画里去执行。

代码生成器里，每种类型都有对应的KeyFrameAnimation类，而这个方法就是创建这个类实例的工厂方法。

类似地,其他属性也有对应的动画类型:
- `Opacity` (float) → `CreateScalarKeyFrameAnimation()`
- `Color` → `CreateColorKeyFrameAnimation()`
- `Scale` (Vector3D) → `CreateVector3KeyFrameAnimation()`
- `RotationAngle` (float) → `CreateScalarKeyFrameAnimation()`

如果想要添加颜色动画，我们需要创建一个`CompositionSolidColorVisual` 作为子视觉元素，`Border` 的 `CompositionVisual` 只支持 `Offset`、`Opacity`、`Scale` 等通用属性,不支持 `Color`
如果需要颜色动画,必须创建 `CompositionSolidColorVisual` 作为子视觉元素，我们可以使用`ElementComposition.SetElementChildVisual()` 将子视觉元素附加到控件上。

我们可以知道的是，每个Visual都有一个ChildCompositionVisual的属性，这就是通过SetElementChildVisual()设置的子窗口元素。

ChildComositionVisual 会被添加到CompositionVisual.Children集合的最后，这就意味着他会渲染在其他的子元素之上。

一下是CompositionSolidColorVisual类可以设置的属性：
```
 <Object Name="CompositionVisual" Abstract="true">
        <Property Name="Root" Type="CompositionTarget?" Internal="true" />
        <Property Name="Parent" Type="CompositionVisual?" Internal="true"  />
        <Property Name="Visible" Type="bool" Animated="true" DefaultValue="true"/>
        <Property Name="Opacity" Type="float" Animated="true" DefaultValue="1"/>
        
        <Property Name="Clip" Type="Avalonia.Platform.IGeometryImpl?" Internal="true" />
        <Property Name="ClipToBounds" Type="bool" Animated="true" DefaultValue="true"/>
        <Property Name="Offset" Type="Vector3D" Animated="true"/>
        <Property Name="Size" Type="Vector" Animated="true"/>
        <Property Name="AnchorPoint" Type="Vector" Animated="true"/>
        <Property Name="CenterPoint" Type="Vector3D" Animated="true"/>
        <Property Name="RotationAngle" Type="float" Animated="true"/>
        <Property Name="Orientation" Type="Quaternion" DefaultValue="Quaternion.Identity" Animated="true"/>
        <Property Name="Scale" Type="Vector3D" DefaultValue="new Avalonia.Vector3D(1, 1, 1)" Animated="true"/>
        <Property Name="TransformMatrix" Type="Avalonia.Matrix" DefaultValue="Avalonia.Matrix.Identity" Animated="true" Internal="true"/>
        <Property Name="AdornedVisual" Type="CompositionVisual?" Internal="true" />
        <Property Name="AdornerIsClipped" Type="bool" Internal="true" />
        <Property Name="OpacityMaskBrush" ClientName="OpacityMaskBrushTransportField" Type="Avalonia.Media.IBrush?" Private="true"  />
        <Property Name="Effect" Type="Avalonia.Media.IImmutableEffect?" Internal="true" />
        <Property Name="RenderOptions" Type="Avalonia.Media.RenderOptions" />
        <Property Name="ShouldExtendDirtyRect" Type="bool" Internal="true" />
    </Object>
    <Object Name="CompositionContainerVisual" Inherits="CompositionVisual"/>
    <Object Name="CompositionSolidColorVisual" Inherits="CompositionContainerVisual">
        <Property Name="Color" Type="Avalonia.Media.Color" Animated="true" />
    </Object>
    <Object Name="CompositionSurfaceVisual" Inherits="CompositionContainerVisual">
        <Property Name="Surface" Type="CompositionSurface?" />
    </Object>
```

`CompositionSolidColorVisual` **不是**其他组件的基类，而是一个**具体的实现类**。继承关系如下：

```
CompositionObject (抽象基类)  
    └── CompositionVisual (抽象基类)  
            └── CompositionContainerVisual  
                    ├── CompositionSolidColorVisual (具体类)  
                    └── CompositionSurfaceVisual (具体类) 
```
和他同级的`CompositionSurfaceVisual` 是用于**显示自定义渲染内容**的视觉元素,它通过 `Surface` 属性关联一个 `CompositionSurface` 对象来显示复杂的图形内容。它只有一个特有属性 `Surface`,类型为 `CompositionSurface?`。

具体实现如下：

```cs
using Avalonia;
using Avalonia.Controls;
using Avalonia.Interactivity;
using Avalonia.Media;
using Avalonia.Rendering.Composition;
using Avalonia.VisualTree;
using System;

namespace AvaloniaTestDemo.Views;

public partial class MainView : UserControl
{
    private CompositionSolidColorVisual _colorVisual;
    public MainView()
    {
        InitializeComponent();
        MyBorder.Loaded += MyBorder_Loaded;
    }

    private void MyBorder_Loaded(object? sender, RoutedEventArgs e)
    {
        var visual = ElementComposition.GetElementVisual(MyBorder);
        if (visual == null) return;
        var compositor = visual.Compositor;
        if (compositor == null) return;

        _colorVisual = compositor.CreateSolidColorVisual();
        _colorVisual.Size = new Vector(MyBorder.Bounds.Width, MyBorder.Bounds.Height);
        _colorVisual.Color = Colors.Transparent;
        _colorVisual.ClipToBounds = true;
        ElementComposition.SetElementChildVisual(MyBorder, _colorVisual);

        // 创建 Offset 动画  
        var offsetAnimation = compositor.CreateVector3KeyFrameAnimation();
        offsetAnimation.Target = "Offset";
        offsetAnimation.InsertExpressionKeyFrame(1, "this.FinalValue");
        offsetAnimation.Duration = TimeSpan.FromMilliseconds(2000);

        // 创建 Color 动画  
        var colorAnimation = compositor.CreateColorKeyFrameAnimation();
        colorAnimation.Target = "Color";
        colorAnimation.InsertExpressionKeyFrame(1, "this.FinalValue");
        colorAnimation.Duration = TimeSpan.FromMilliseconds(2000);

        // 将两个动画添加到同一个 ImplicitAnimationCollection  
        var implicitAnimations = compositor.CreateImplicitAnimationCollection();
        implicitAnimations["Offset"] = offsetAnimation;

        // 为 visual 设置 Offset 动画  
        visual.ImplicitAnimations = implicitAnimations;

        // 为 _colorVisual 设置 Color 动画  
        var colorImplicitAnimations = compositor.CreateImplicitAnimationCollection();
        colorImplicitAnimations["Color"] = colorAnimation;
        _colorVisual.ImplicitAnimations = colorImplicitAnimations;

        MyBorder.Loaded -= MyBorder_Loaded;
    }

    private void Border_Click(object? sender, Avalonia.Input.PointerPressedEventArgs e)
    {
        if (sender is Border border)
        {
            var visual = ElementComposition.GetElementVisual(border);
            if (visual == null) return; // Check if visual is null before accessing its properties
            visual.Offset += new Vector3D(100, 100, 0); // x=100, y=100
            _colorVisual.Color = Colors.Red;
        }
    }
}
```
为什么Border的ElementVisual没有Color，根据根据 composition schema 的定义,`Border` 对应的 `CompositionVisual` 类型是 `CompositionBorderVisual`,它继承自 `CompositionDrawListVisual`: BorderVisual.cs:10-18

而 `CompositionVisual` 基类只定义了通用的可动画属性,**不包括 `Color`**: composition-schema.xml:16-37

因此,要为 `Border` 添加颜色动画,必须:
1. **创建一个 `CompositionSolidColorVisual` 作为子视觉元素**
2. **使用 `Border` 的 `Compositor` 来创建这个子视觉元素**
3. **为子视觉元素设置 `Color` 的隐式动画**
4. **通过 `ElementComposition.SetElementChildVisual()` 将其附加到 `Border` 上**
所以我也推荐给父组件visual设置裁切子元素： visual.ClipToBounds = ture;

这个颜色变化就是邪教用法了，实际上我们很少这么操作，因为组件本身已经有了属性动画了。