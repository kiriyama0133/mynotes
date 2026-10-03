我们知道一点，就是：

`TemplatedControl` 是一个**无外观控件**(lookless control),其**视觉外观**由 `Template` 属性定义。 TemplatedControl.cs:14-17 它直接继承自 `Control` 类。 TemplatedControl.cs:17

#### 附加属性
`IsTemplateFocusTarget` - 附加属性**,用于指定模板中哪个元素应该接收焦点装饰器**。 TemplatedControl.cs:106-107 当控件获得焦点时,**焦点装饰器会显示在标记了此属性的元素周围,而不是控件本身**。 TemplatedControl.cs:280-284

#### 备注

`TemplatedControl` 是许多 Avalonia 控件的基类,包括之前讨论的 `ContentControl`。 它提供了模板化控件的基础架构,使得控件的逻辑和外观可以完全分离。 所有模板子元素的 `TemplatedParent` **属性都会被设置为该控件实例。因此，他很适合用来当做动画组件等使用**。

因此我们这里可以模仿sukiUi实现一个双缓冲的FadeIn FadeOut动画。

```csharp
using System;
using System.Threading;
using Avalonia;
using Avalonia.Animation;
using Avalonia.Controls;
using Avalonia.Controls.Presenters;
using Avalonia.Controls.Primitives;
using Avalonia.Interactivity;
using Avalonia.Media;
using Avalonia.Styling;
using Avalonia.Threading;

namespace SukiUI.Controls
{
    // TODO: This needs fairly significant work to make a bit more bomb proof
    // There are probably some more gains that can be made in terms of performance.
    // Unfortunately we're still bound by the arrange of controls having to happen on the main thread.
    public class SukiTransitioningContentControl : TemplatedControl
    {
        internal static readonly StyledProperty<object?> FirstBufferProperty =
            AvaloniaProperty.Register<SukiTransitioningContentControl, object?>(nameof(FirstBuffer));

        internal object? FirstBuffer
        {
            get => GetValue(FirstBufferProperty);
            set => SetValue(FirstBufferProperty, value);
        }

        internal static readonly StyledProperty<object?> SecondBufferProperty =
            AvaloniaProperty.Register<SukiTransitioningContentControl, object?>(nameof(SecondBuffer));

        internal object? SecondBuffer
        {
            get => GetValue(SecondBufferProperty);
            set => SetValue(SecondBufferProperty, value);
        }

        public static readonly StyledProperty<object?> ContentProperty = AvaloniaProperty.Register<SukiTransitioningContentControl, object?>(nameof(Content));

        public object? Content
        {
            get => GetValue(ContentProperty);
            set => SetValue(ContentProperty, value);
        }

        private bool _isFirstBufferActive;

        private ContentPresenter? _firstBuffer = null;
        private ContentPresenter? _secondBuffer = null;

        private static readonly Animation FadeIn;
        private static readonly Animation FadeOut;
        
        private ContentPresenter? To => _isFirstBufferActive ? _firstBuffer : _secondBuffer;
        private ContentPresenter? From => _isFirstBufferActive ? _secondBuffer : _firstBuffer;

        private object? _contentBeforeApplied;

        static SukiTransitioningContentControl()
        {
            FadeIn = new Animation
            {
                Duration = TimeSpan.FromMilliseconds(400),
                Children =
                {
                    new KeyFrame()
                    {
                        Setters =
                        {
                            new Setter
                            {
                                Property = OpacityProperty,
                                Value = 0d
                            }
                        },
                        Cue = new Cue(0d)
                    },
                    new KeyFrame()
                    {
                        Setters =
                        {
                            new Setter
                            {
                                Property = OpacityProperty,
                                Value = 1d
                            }
                        },
                        Cue = new Cue(1d)
                    }
                },
                FillMode = FillMode.Forward
            };
            FadeOut = new Animation
            {
                Duration = TimeSpan.FromMilliseconds(400),
                Children =
                {
                    new KeyFrame()
                    {
                        Setters =
                        {
                            new Setter
                            {
                                Property = OpacityProperty,
                                Value = 1d
                            }
                        },
                        Cue = new Cue(0d)
                    },
                    new KeyFrame()
                    {
                        Setters =
                        {
                            new Setter
                            {
                                Property = OpacityProperty,
                                Value = 0d
                            }
                        },
                        Cue = new Cue(1d)
                    }
                },
                FillMode = FillMode.Forward
            };
            FadeIn.Duration = FadeOut.Duration = TimeSpan.FromMilliseconds(250);
        }

        private CancellationTokenSource _animCancellationToken = new();
        

        protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
        {
            base.OnPropertyChanged(change);
            if(change.Property == ContentProperty)
                PushContent(change.NewValue);
        }

        protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
        {
            base.OnApplyTemplate(e);
            if (e.NameScope.Get<ContentPresenter>("PART_FirstBufferControl") is { } fBuff)
                _firstBuffer = fBuff;
            if (e.NameScope.Get<ContentPresenter>("PART_SecondBufferControl") is { } sBuff)
                _secondBuffer = sBuff;
            if (_contentBeforeApplied != null)
            {
                PushContent(_contentBeforeApplied);
                _contentBeforeApplied = null;
            }
        }

        private void PushContent(object? content)
        {
            if (To is null || From is null)
            {
                _contentBeforeApplied = content;
                return;
            }
            
            _animCancellationToken.Cancel();
            _animCancellationToken.Dispose();
            _animCancellationToken = new CancellationTokenSource();
            
            if (_isFirstBufferActive) SecondBuffer = content;
            else FirstBuffer = content;
            _isFirstBufferActive = !_isFirstBufferActive;
            try
            {
                FadeOut.RunAsync(From, _animCancellationToken.Token).ContinueWith(_ =>
                {
                    Dispatcher.UIThread.Invoke(() =>
                    {
                        From.IsHitTestVisible = false;
                        if (_isFirstBufferActive) SecondBuffer = null;
                        else FirstBuffer = null;
                    });
                });
                FadeIn.RunAsync(To, _animCancellationToken.Token).ContinueWith(_ => 
                    Dispatcher.UIThread.Invoke(() => To.IsHitTestVisible = true));
            }
            catch
            {
                // ignored
            }
        }

        protected override void OnUnloaded(RoutedEventArgs e)
        {
            base.OnUnloaded(e);
            _animCancellationToken.Dispose();
        }
    }
}
```

我们注册了三个属性，其中有两个私有属性来当做缓冲器，一个Content属性来当做渲染的子组件内容。

我们重载了OnApplyTemplate，当模板应用完成了初始化之后就会自动调用，他会自动获取模板中的命名元素，建立控件与模板的连接，从 XAML 模板中获取两个 ContentPresenter 控件

- PART_FirstBufferControl 和 PART_SecondBufferControl 是在 XAML 中定义的名称

- 使用模式匹配 is { } 确保获取成功

之后他会调用调用PushContent去处理延迟的内容设置，然后再来讲一下为什么PushContent可以处理延迟：

```csharp
if (To is null || From is null)
{
    _contentBeforeApplied = content;
    return;
}
```

我们使用了前置检查，检查缓冲区是否已经初始化，而To和From是活跃和非活跃的ContentPresenter，如果缓冲区没有准备好，就会将内容暂存到_contentBeforeApplied。

```csharp
        _animationCancellationToken.Cancel();
        _animationCancellationToken.Dispose();
        _animationCancellationToken = new CancellationTokenSource();
```

这段代码是确保没有并发的动画，取消正在进行的动画，释放旧的取消令牌，然后创建新的令牌用于当前动画，这一段我们会自己讲述。

然后是缓冲区切换的逻辑：

```csharp
if (_isFirstBufferActive) SecondBuffer = content;
else FirstBuffer = content;
_isFirstBufferActive = !_isFirstBufferActive;
```

这样To和From会自动指向正确的缓冲区。接下来是我们设置的动画：

```csharp
FadeOut.RunAsync(From, _animCancellationToken.Token).ContinueWith(_ =>
{
    Dispatcher.UIThread.Invoke(() =>
    {
        From.IsHitTestVisible = false;
        if (_isFirstBufferActive) SecondBuffer = null;
        else FirstBuffer = null;
    });
});
```

这个作用是在旧内容上进行淡出动画，动画完成之后禁用旧内容的鼠标交互，清空缓冲区内容。

```csharp
FadeIn.RunAsync(To, _animCancellationToken.Token).ContinueWith(_ => 
    Dispatcher.UIThread.Invoke(() => To.IsHitTestVisible = true));
```

然后是淡入动画，动画完成就会启用鼠标交互。

新内容到达
    ↓
检查缓冲区是否就绪
    ↓
取消之前的动画
    ↓
将新内容放入非活跃缓冲区
    ↓
切换活跃缓冲区标志
    ↓
同时启动淡出和淡入动画
    ↓
淡出完成 → 清理旧缓冲区
淡入完成 → 启用新内容交互



重载了属性改变的方法：

```csharp

        protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
        {
            base.OnPropertyChanged(change);
            if(change.Property == ContentProperty)
                PushContent(change.NewValue);
        }
```

当属性改变之后，就会自动触发UI更新。

组件卸载的时候自动执行的重载方法：

```csharp
        protected override void OnUnloaded(RoutedEventArgs e)
        {
            base.OnUnloaded(e);
            _animCancellationToken.Dispose();
        }
    }
```



