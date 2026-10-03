	WPFUI 是一个为 WPF 应用程序提供现代化 UI 组件的框架，它旨在简化开发过程并增强用户界面体验。WPFUI 的目标是提供现代、直观且易于使用的控件和样式，帮助开发者更快地构建美观的应用程序。以下是对 WPFUI 框架的介绍：

​	**主要特点**

**现代化设计：**WPFUI 提供了一组符合现代设计规范的 UI 组件，使应用程序看起来更时尚和专业。
**易用性：**框架设计简洁，使用方便，开发者可以快速上手，并在项目中轻松集成 WPFUI 提供的控件和样式。
**丰富的控件库：**WPFUI 包含许多常用的控件，如按钮、文本框、菜单、对话框等，并且这些控件都经过精心设计，具有良好的用户体验。
**高可定制性：**开发者可以根据项目需求对 WPFUI 提供的控件进行自定义，从而实现符合特定要求的用户界面。
**响应式布局：**WPFUI 支持响应式布局，可以在不同大小的屏幕上保持良好的显示效果。
**持续更新与支持：**WPFUI 框架有活跃的开发社区和官方支持，定期发布更新和新特性，确保框架始终处于最新状态。

他实际上依靠社区完成的，因为现在只提供了WinUi继承了fluentUI，因此我们需要依靠这个社区工具完成fluent风格的ui搭建

## 导航栏

这里先给出一个大致的实现过程：

```xaml
 <ui:NavigationView
     x:Name="NavigationView"
     Padding="42,0,42,0"
     BreadcrumbBar="{Binding ElementName=BreadcrumbBar}"
     EnableDebugMessages="True"
     FooterMenuItemsSource="{Binding ViewModel.FooterMenuItems, Mode=OneWay}"
     FrameMargin="0"
     IsBackButtonVisible="Visible"
     IsPaneToggleVisible="True"
     MenuItemsSource="{Binding ViewModel.MenuItems, Mode=OneWay}"
     OpenPaneLength="310"
     PaneClosed="NavigationView_OnPaneClosed"
     PaneDisplayMode="Left"
     PaneOpened="NavigationView_OnPaneOpened"
     SelectionChanged="OnNavigationSelectionChanged"
     TitleBar="{Binding ElementName=TitleBar, Mode=OneWay}"
     Transition="FadeInWithSlide">
     <ui:NavigationView.Header>
         <StackPanel Margin="42,32,42,20">
             <ui:BreadcrumbBar x:Name="BreadcrumbBar" />
             <!--<controls:PageControlDocumentation Margin="0,10,0,0" NavigationView="{Binding ElementName=NavigationView}" />-->
         </StackPanel>
     </ui:NavigationView.Header>

     <ui:NavigationView.AutoSuggestBox>
         <ui:AutoSuggestBox x:Name="AutoSuggestBox" PlaceholderText="{i18n:StringLocalizer 'Search'}">
             <ui:AutoSuggestBox.Icon>
                 <ui:IconSourceElement>
                     <ui:SymbolIconSource Symbol="Search24" />
                 </ui:IconSourceElement>
             </ui:AutoSuggestBox.Icon>
         </ui:AutoSuggestBox>
     </ui:NavigationView.AutoSuggestBox>

     <ui:NavigationView.ContentOverlay>
         <Grid>
             <ui:SnackbarPresenter x:Name="SnackbarPresenter" />
         </Grid>
     </ui:NavigationView.ContentOverlay>

 </ui:NavigationView>
```

## NavigationView 详细解析

### 1. 基本属性配置

**Padding="42,0,42,0"**

- 设置导航视图的内边距，左右各42像素，上下为0
- 这确保了内容与边缘有适当的间距，提供更好的视觉体验

**BreadcrumbBar="{Binding ElementName=BreadcrumbBar}"**
- 绑定面包屑导航栏，用于显示当前**页面在导航层次结构中的位置**
- 通过ElementName绑定到同名的BreadcrumbBar控件

**EnableDebugMessages="True"**
- 启用调试消息，在开发阶段有助于排查问题
- 生产环境建议设置为False

### 2. 数据绑定配置

**FooterMenuItemsSource="{Binding ViewModel.FooterMenuItems, Mode=OneWay}"**
- 绑定底部菜单项数据源
- OneWay模式表示数据从ViewModel流向UI，UI变化不会影响数据源
- 通常用于设置、帮助等固定菜单项

**MenuItemsSource="{Binding ViewModel.MenuItems, Mode=OneWay}"**
- 绑定主导航菜单项数据源
- 这是导航视图的核心数据，包含所有可导航的页面项

### 3. 布局和显示控制

**FrameMargin="0"**
- 设置内容框架的边距为0，使内容占满整个可用空间

**IsBackButtonVisible="Visible"**
- 显示返回按钮，允许用户返回到上一级页面，应该本质上就是个历史路由，他会记录每一次你路由的信息。
- 在导航层次结构中提供后退功能

**IsPaneToggleVisible="True"**

- 显示面板切换按钮，允许用户展开/收起侧边栏
- 提供更好的空间利用和用户体验

**OpenPaneLength="310"**
- 设置侧边栏展开时的宽度为310像素，可自行调整。
- 这个宽度需要根据内容长度和设计需求来调整

**PaneDisplayMode="Left"**
- 设置侧边栏显示在左侧
- 这是最常见的布局方式，符合用户习惯

### 4. 事件处理

**PaneClosed="NavigationView_OnPaneClosed"**
- 侧边栏关闭时触发的事件处理程序
- 可以用于保存状态、更新UI等操作

**PaneOpened="NavigationView_OnPaneOpened"**
- 侧边栏打开时触发的事件处理程序
- 可以用于加载数据、更新显示等操作

**SelectionChanged="OnNavigationSelectionChanged"**
- 导航选择改变时触发的事件处理程序
- 这是导航功能的核心，用于处理页面切换逻辑

### 5. 标题栏和过渡效果

**TitleBar="{Binding ElementName=TitleBar, Mode=OneWay}"**
- 绑定自定义标题栏控件
- 允许完全自定义标题栏的外观和行为

**Transition="FadeInWithSlide"**
- 设置页面切换的过渡动画效果
- FadeInWithSlide提供淡入和滑动效果，提升用户体验

### 6. 头部区域 (Header)

```xaml
<ui:NavigationView.Header>
    <StackPanel Margin="42,32,42,20">
        <ui:BreadcrumbBar x:Name="BreadcrumbBar" />
        <!--<controls:PageControlDocumentation Margin="0,10,0,0" NavigationView="{Binding ElementName=NavigationView}" />-->
    </StackPanel>
</ui:NavigationView.Header>
```

- **StackPanel**: 垂直堆叠布局容器
- **Margin="42,32,42,20"**: 设置上下左右边距
- **BreadcrumbBar**: 面包屑导航，显示当前页面路径
- **注释部分**: 可能是文档控件，用于显示页面说明

### 7. 自动建议框 (AutoSuggestBox)

```xaml
<ui:NavigationView.AutoSuggestBox>
    <ui:AutoSuggestBox x:Name="AutoSuggestBox" PlaceholderText="{i18n:StringLocalizer 'Search'}">
        <ui:AutoSuggestBox.Icon>
            <ui:IconSourceElement>
                <ui:SymbolIconSource Symbol="Search24" />
            </ui:IconSourceElement>
        </ui:AutoSuggestBox.Icon>
    </ui:AutoSuggestBox>
</ui:NavigationView.AutoSuggestBox>
```

- **PlaceholderText**: 使用国际化字符串本地化占位符文本
- **Icon**: 设置搜索图标，使用SymbolIconSource提供统一的图标样式
- **Symbol="Search24"**: 使用24像素的搜索图标

### 8. 内容覆盖层 (ContentOverlay)

```xaml
<ui:NavigationView.ContentOverlay>
    <Grid>
        <ui:SnackbarPresenter x:Name="SnackbarPresenter" />
    </Grid>
</ui:NavigationView.ContentOverlay>
```

- **ContentOverlay**: 内容覆盖层，用于显示浮动在内容之上的UI元素
- **SnackbarPresenter**: 消息提示组件，用于显示通知、警告等信息
- 这种设计允许在不影响主内容的情况下显示临时信息

## 实现原理总结

1. **MVVM模式**: 通过数据绑定将UI与ViewModel分离，实现松耦合
2. **响应式设计**: 支持侧边栏展开/收起，适应不同屏幕尺寸
3. **导航管理**: 通过SelectionChanged事件处理页面切换逻辑
4. **用户体验**: 提供过渡动画、面包屑导航、搜索功能等现代化交互
5. **可扩展性**: 通过绑定和事件处理，支持自定义扩展和功能增强

这种设计模式使得NavigationView成为一个功能完整、易于维护的现代化导航组件。

然后就是xaml.cs绑定的函数实例：

```csharp
using Animation.Pages.Dashboard;
using Microsoft.Extensions.DependencyInjection;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Forms;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Shapes;
using Wpf.Ui;
using Wpf.Ui.Controls;

namespace Animation.Pages.ToolWindow;

/// <summary>
/// ToolWindow.xaml 的交互逻辑
/// </summary>
public partial class ToolWindow : IWindow
{
    public ToolWindowViewModel ViewModel { get; }
    private bool _isUserClosedPane;
    private bool _isPaneOpenedOrClosedFromCode;


    private void OnNavigationSelectionChanged(object sender, RoutedEventArgs e)
    {
        if (sender is not NavigationView navigationView)
        {
            return;
        }

        navigationView.SetCurrentValue(
            NavigationView.HeaderVisibilityProperty,
            navigationView.SelectedItem?.TargetPageType != typeof(DashboardPage)
                ? Visibility.Visible
                : Visibility.Collapsed
        );
    }

    private void NavigationView_OnPaneOpened(NavigationView sender, RoutedEventArgs args)
    {
        if (_isPaneOpenedOrClosedFromCode)
        {
            return;
        }

        _isUserClosedPane = false;
    }

    private void NavigationView_OnPaneClosed(NavigationView sender, RoutedEventArgs args)
    {
        if (_isPaneOpenedOrClosedFromCode)
        {
            return;
        }

        _isUserClosedPane = true;
    }

    public ToolWindow(ToolWindowViewModel viewModel,INavigationService navigationService,IServiceProvider serviceProvider,
        ISnackbarService snackbarService,IContentDialogService contentDialogService)
    {
        this.DataContext = this;
        ViewModel = viewModel; 

        InitializeComponent();

        snackbarService.SetSnackbarPresenter(SnackbarPresenter);
        navigationService.SetNavigationControl(NavigationView);
        contentDialogService.SetDialogHost(RootContentDialog);

        // 订阅窗口关闭事件
        this.Loaded += ToolWindow_Loaded;
        this.Closed += ToolWindow_Closed;

    }

    private void ToolWindow_Loaded(object sender, RoutedEventArgs e)
    { 
       this.NavigationView.Navigate(typeof(DashboardPage));
    }
    
    private void ToolWindow_Closed(object sender, EventArgs e)
    {
        // 当ToolWindow关闭时，显示MainWindow
        try
        {
            Dispatcher.BeginInvoke(new Action(() =>
            {
                if (App.MainWindow != null && !App.MainWindow.IsVisible)
                {
                    App.MainWindow.Show();
                    App.MainWindow.Activate(); // 激活窗口并置于前台
                }
            }), System.Windows.Threading.DispatcherPriority.Background);
        }
        catch (Exception ex)
        {
            // 处理错误，可以记录日志或显示错误消息
            Console.WriteLine($"Failed to show MainWindow: {ex.Message}");
        }
    }
}

```

​	然后是数据业务逻辑层面的ViewModel代码，里面定义了路由的测栏内容等信息。当然如果是为了方便你可以用另一种方式躲避每一次的手动注册，使用**程序集反射加载**实现了对应接口的页面，然后返回封装成列表

```csharp
using Animation.Pages.ColorCollecter;
using Animation.Pages.Dashboard;
using Animation.Res;
using CommunityToolkit.Mvvm.ComponentModel;
using Microsoft.Extensions.Localization;
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using Wpf.Ui.Controls;

namespace Animation.Pages.ToolWindow;

public partial class ToolWindowViewModel(IStringLocalizer<Translation> localizer) : ViewModel
{
    [ObservableProperty]
    private string title = localizer["toolbox"];

    [ObservableProperty]
    private string description = localizer["toolbox_description"];

    #region 定义菜单栏
    [ObservableProperty]
    private ObservableCollection<object> _menuItems = [
        new NavigationViewItem("Home",SymbolRegular.Home24,typeof(DashboardPage)),
        new NavigationViewItem("Color采集",SymbolRegular.Color24,typeof(ColorCollecterPage)),
        ];
    #endregion

}

```

## 重写组件实现自定义

​	因为默认的ItemsPanel 因此我决定重写组件实现包装WrapPanel。**ItemsControl 默认使用 StackPanel 作为 ItemsPanel**，这会覆盖我们在 XAML 中设置的 WrapPanel。**即使我们在 XAML 中设置了 WrapPanel，ItemsControl 仍然会使用默认的 StackPanel**。

下面是代码的全部：

```csharp
static ColorStackContent()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(ColorStackContent), new FrameworkPropertyMetadata(typeof(ColorStackContent)));
        
        // 设置默认的ItemsPanel为WrapPanel
        ItemsPanelProperty.OverrideMetadata(typeof(ColorStackContent), 
            new FrameworkPropertyMetadata(CreateDefaultItemsPanel()));
    }
    
    private static ItemsPanelTemplate CreateDefaultItemsPanel()
    {
        var template = new ItemsPanelTemplate();
        var factory = new FrameworkElementFactory(typeof(WrapPanel));
        template.VisualTree = factory;
        return template;
    }

    protected override void PrepareContainerForItemOverride(DependencyObject element, object item)
    {
        base.PrepareContainerForItemOverride(element, item);

        if (element is ContentPresenter presenter)
        {
            // 将字符串转换为Color
            Color color;
            if (item is string colorString)
            {
                try
                {
                    color = (Color)ColorConverter.ConvertFromString(colorString);
                }
                catch
                {
                    color = Colors.Gray; // 如果转换失败，使用默认颜色
                }
            }
            else
            {
                color = Colors.Gray; // 如果不是字符串，使用默认颜色
            }

            // 创建带有动画效果的Border
            var border = new Border
            {
                CornerRadius = new CornerRadius(8),
                Height = 50,
                Width = 50,
                Background = new SolidColorBrush(color),
                BorderBrush = Brushes.Black,
                BorderThickness = new Thickness(1),
                Margin = new Thickness(5),
                Effect = null // 初始无阴影
            };

            // 创建阴影效果
            var shadowEffect = new System.Windows.Media.Effects.DropShadowEffect
            {
                Color = Colors.Black,
                Direction = 315,
                ShadowDepth = 0,
                BlurRadius = 0,
                Opacity = 0
            };

            // 创建动画
            var shadowAnimation = new DoubleAnimation
            {
                To = 8,
                Duration = TimeSpan.FromMilliseconds(200),
                EasingFunction = new CubicEase { EasingMode = EasingMode.EaseOut }
            };

            var blurAnimation = new DoubleAnimation
            {
                To = 10,
                Duration = TimeSpan.FromMilliseconds(200),
                EasingFunction = new CubicEase { EasingMode = EasingMode.EaseOut }
            };

            var opacityAnimation = new DoubleAnimation
            {
                To = 0.5,
                Duration = TimeSpan.FromMilliseconds(200),
                EasingFunction = new CubicEase { EasingMode = EasingMode.EaseOut }
            };

            // 反向动画（鼠标离开时）
            var shadowAnimationOut = new DoubleAnimation
            {
                To = 0,
                Duration = TimeSpan.FromMilliseconds(150),
                EasingFunction = new CubicEase { EasingMode = EasingMode.EaseIn }
            };

            var blurAnimationOut = new DoubleAnimation
            {
                To = 0,
                Duration = TimeSpan.FromMilliseconds(150),
                EasingFunction = new CubicEase { EasingMode = EasingMode.EaseIn }
            };

            var opacityAnimationOut = new DoubleAnimation
            {
                To = 0,
                Duration = TimeSpan.FromMilliseconds(150),
                EasingFunction = new CubicEase { EasingMode = EasingMode.EaseIn }
            };

            // 设置阴影效果
            border.Effect = shadowEffect;

            // 鼠标进入事件
            border.MouseEnter += (s, e) =>
            {
                shadowEffect.BeginAnimation(System.Windows.Media.Effects.DropShadowEffect.ShadowDepthProperty, shadowAnimation);
                shadowEffect.BeginAnimation(System.Windows.Media.Effects.DropShadowEffect.BlurRadiusProperty, blurAnimation);
                shadowEffect.BeginAnimation(System.Windows.Media.Effects.DropShadowEffect.OpacityProperty, opacityAnimation);
            };

            // 鼠标离开事件
            border.MouseLeave += (s, e) =>
            {
                shadowEffect.BeginAnimation(System.Windows.Media.Effects.DropShadowEffect.ShadowDepthProperty, shadowAnimationOut);
                shadowEffect.BeginAnimation(System.Windows.Media.Effects.DropShadowEffect.BlurRadiusProperty, blurAnimationOut);
                shadowEffect.BeginAnimation(System.Windows.Media.Effects.DropShadowEffect.OpacityProperty, opacityAnimationOut);
            };

            // 创建包含颜色名称的StackPanel
            var stackPanel = new StackPanel
            {
                Orientation = Orientation.Horizontal,
                HorizontalAlignment = HorizontalAlignment.Left,
                VerticalAlignment = VerticalAlignment.Top,
                Margin = new Thickness(5)
            };

            stackPanel.Children.Add(border);

            var textBlock = new TextBlock
            {
                Text = item.ToString(),
                VerticalAlignment = VerticalAlignment.Center,
                Margin = new Thickness(5, 0, 0, 0),
                Foreground = Brushes.White
            };

            stackPanel.Children.Add(textBlock);

            presenter.Content = stackPanel;
        }
    }
```

> [!NOTE]
>
> ### 1. DefaultStyleKeyProperty.OverrideMetadata
>
> - 作用：告诉 WPF 这个控件**使用哪个样式模板**
>
> - 参数：typeof(ColorStackContent) 表示使用名为 "ColorStackContent" 的样式
>
> - 效果：WPF 会在 XAML 资源中查找对应的样式来渲染控件

### CreateDefaultItemsPanel() 方法：

#### 为什么需要这个方法？

- ItemsPanelProperty 需要的是 ItemsPanelTemplate 类型，不是直接的 WrapPanel

- ItemsPanelTemplate 是一个模板，**告诉 WPF 如何创建面板**

- FrameworkElementFactory 是创建面板实例的工厂

### PrepareContainerForItemOverride 方法

#### 这个方法的作用：

- 时机：每当 ItemsControl 需要显示一个项目时调用

- 参数：

​		element：容器元素（通常是 ContentPresenter）

​		item：数据项（在我们的例子中是颜色字符串，如 "#FF0000"）

#### 处理流程：

1. 颜色转换：将字符串（如 "#FF0000"）转换为 Color 对象

1. 创建颜色块：创建 50x50 的 Border 作为颜色显示

1. 添加动画效果：创建阴影效果和鼠标悬停动画

1. 创建布局：用 StackPanel 水平排列颜色块和文本

1. 设置内容：将整**个布局设置为 ContentPresenter 的内容**


## 新的开始 从.NET 9开始

从.net 9 开始，已经对WPF的fluentUI有了原生的支持，我们可以直接通过在App.xaml中引用这个资源。
```xml
<Application x:Class="TheTestSolutionForWPF.App"  
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"  
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"  
             xmlns:local="clr-namespace:TheTestSolutionForWPF"  
             StartupUri="MainWindow.xaml">  
    <Application.Resources>  
         <ResourceDictionary>  
             <ResourceDictionary.MergedDictionaries>  
                 <ResourceDictionary Source="pack://application:,,,/PresentationFramework.Fluent;component/Themes/Fluent.xaml" />  
             </ResourceDictionary.MergedDictionaries>  
         </ResourceDictionary>  
    </Application.Resources>  
</Application>
```
通过这样我们就可以直接使用Fluent主题了，这是版本9的一个新特性。在此之前（.NET 8 及更早版本），WPF 的默认主题还是古老的 Aero2（Windows 8 风格），想要 Windows 11 的样式必须依赖第三方库。

在 .NET 9 中，微软向 .NET Desktop Runtime 添加了一个**全新的官方程序集**，名为 `PresentationFramework.Fluent.dll`。
![[Pasted image 20251211205538.png]]

就是这个，只有679KB，还挺小的！**它只是“皮肤” (Styles/Templates)** 这个文件里面几乎没有复杂的 C# 逻辑代码。它主要包含的是编译后的 XAML (即 BAML)，定义了 `Button`、`TextBox`、`ListBox` 等标准控件在 Windows 11 下该长什么样（圆角、颜色、字体、边框粗细）

引入它的开销几乎可以忽略不计，应用启动速度不会受到影响，**不仅仅是“好看”**：微软把它做得这么小，意味着它是作为 **WPF 核心的一部分** 来设计的，而不是一个臃肿的插件。


