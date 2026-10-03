

**XAML Behaviors** 提供了一种简单易用的方法，能以最少的代码为 **Windows UWP/WPF** 应用程序添加**常用和可重复使用的交互性**。

但是Microsoft XAML Behaviors包除了提供常用的**XAML Behaviors**之外，还提供了一些**Trigger**（触发器）以及**Trigger**对应的**Action**（动作）。

可以通过一个最简单的示例来感受一下
首先我们引用**Microsoft.Xaml.Behaviors.Wpf**包，然后添加如下的代码。

在下面的代码中，我们放置了一个按钮，然后为按钮**添加了XAML Behavior**，当按钮触发Click事件时，调用ChangPropertyAction去改变控件的背景颜色


```xml
<Window x:Class="XamlBehaviorsDemo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:XamlBehaviorsDemo"
        xmlns:i="http://schemas.microsoft.com/xaml/behaviors"
        mc:Ignorable="d"
        Title="MainWindow" Height="450" Width="800">
    <Grid>
        <Button Width="88" Height="28" Content="Click me">
            <i:Interaction.Triggers>
                <i:EventTrigger EventName="Click">
                    <i:ChangePropertyAction PropertyName="Background">
                        <i:ChangePropertyAction.Value>
                            <SolidColorBrush Color="Red"/>
                        </i:ChangePropertyAction.Value>
                    </i:ChangePropertyAction>
                </i:EventTrigger>
            </i:Interaction.Triggers>
        </Button>
    </Grid>
</Window>
```

如果没有使用XAML Behavior，我们**需要将控件命名**，然后在**控件的Click事件处理程序中，设置控件的背景颜色** 

当然，我们**可以通过样式和触发器来实现一样的功能**，但这**仅限于控件内部**，如果要设置其它控件就不行了。而XAML Behavior可以**设置目标对象是自身也可以是其它控件**。

# **Trigger和Behavior的区别**

## 触发器

包含**一个或多个动作的对象**，可根据**某些刺激调用这些动作**。一种非常常见的触发器是**针对事件触发的触发器（EventTrigger）**。其他例子可能包括在定时器上触发的触发器，或在抛出未处理异常时触发的触发器。

## **行为**

行为没有调用的概念；它是附加到元素上的东西，用于**指定应用程序应在何时做出响应**。

# **常见Trigger的用法**

## **EventTrigger** 

监听源上**指定事件并在事件触发时触发的触发器**。

在使用MVVM模式开发时，**需要将事件转换为命令绑定**，就可以使用**EventTrigger**。

用法如下：下面的代码演示了，当SelectionChanged事件触发时，将会执行Action

```xml
<ListBox>
  <i:Interaction.Triggers>
      <i:EventTrigger EventName="SelectionChanged">
         ..action...
      </i:EventTrigger>
  </i:Interaction.Triggers>
</ListBox>
```

## **PropertyChangedTrigger**

代表**当绑定数据发生变化时执行操作的触发器**。

用法如下：

下面的代码演示了，当TextBoxText属性发生更改时，将会执行Action

```xml
<TextBox Text="{Binding TextBoxText,UpdateSourceTrigger=PropertyChanged}">
        <i:Interaction.Triggers>
            <i:PropertyChangedTrigger Binding="{Binding TextBoxText}">
               ...action...
            </i:PropertyChangedTrigger>
        </i:Interaction.Triggers>
    </TextBox>
```
## **DataTrigger**

代表当**绑定数据满足指定条件时执行操作的触发器**。

用法如下：下面的代码演示了，**当CheckBox选中/未选中时**，将会**执行对应的Action**


```xml
 <CheckBox x:Name="checkBox" >
    <i:Interaction.Triggers>
        <i:DataTrigger Binding="{Binding IsChecked, ElementName=checkBox}" Value="False">
           ...选中 action...
       </i:DataTrigger>
        <i:DataTrigger Binding="{Binding IsChecked, ElementName=checkBox}" Value="True">
          ...未选中 action...
        </i:DataTrigger>
    </i:Interaction.Triggers>
  </CheckBox>
```
# **常见Action的用法**

## **ChangePropertyAction**

调用时**将指定属性更改为指定值的操作**，首先我们在界面上**放置一个Rectangle**，然后放置两个按钮，**当按钮的Click事件触发时**，更改Rectangle的**Background属性**。

```xml
<Grid>
      <Grid.RowDefinitions>
          <RowDefinition Height="5*"/>
          <RowDefinition Height="*"/>
      </Grid.RowDefinitions>
      <Rectangle x:Name="DataTriggerRectangle" Grid.Row="0" />

      <Grid Grid.Row="1">
          <Grid.ColumnDefinitions>
              <ColumnDefinition Width="*" />
              <ColumnDefinition Width="*" />
          </Grid.ColumnDefinitions>
          <Button x:Name="YellowButton" Content="Yellow" Grid.Column="0">
              <Behaviors:Interaction.Triggers>
                  <Behaviors:EventTrigger EventName="Click" SourceObject="{Binding ElementName=YellowButton}">
                      <Behaviors:ChangePropertyAction TargetObject="{Binding ElementName=DataTriggerRectangle}" PropertyName="Fill" Value="LightYellow"/>
                  </Behaviors:EventTrigger>
              </Behaviors:Interaction.Triggers>
          </Button>
          <Button x:Name="PinkButton" Content="Pink" Grid.Column="1">
              <Behaviors:Interaction.Triggers>
                  <Behaviors:EventTrigger EventName="Click" SourceObject="{Binding ElementName=PinkButton}">
                      <Behaviors:ChangePropertyAction TargetObject="{Binding ElementName=DataTriggerRectangle}" PropertyName="Fill" Value="DeepPink"/>
                  </Behaviors:EventTrigger>
              </Behaviors:Interaction.Triggers>
          </Button>
      </Grid>

  </Grid>
```

## **ControlStoryboardAction**

调用时将改变目标Storyboard状态的操作，也就是可以控制动画状态。
首先我们放置一个Rectangle
```xml
<Rectangle x:Name="StoryboardRectangle" StrokeThickness="5" Fill="Pink" Width="100" Stroke="LightYellow" RenderTransformOrigin="0.5,0.5" >
     <Rectangle.RenderTransform>
         <ScaleTransform />
     </Rectangle.RenderTransform>
 </Rectangle>
```
然后**定义Storyboard**，目标对象就是前面定义的Rectangle：
```xml
<Window.Resources>
      <Storyboard x:Key="StoryboardSample" >
          <DoubleAnimation Duration="0:0:5" To="0.35"
                            Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleX)"
                            Storyboard.TargetName="StoryboardRectangle" d:IsOptimized="True"/>
          <DoubleAnimation Duration="0:0:5" To="0.35"
                            Storyboard.TargetProperty="(UIElement.RenderTransform).(ScaleTransform.ScaleY)"
                            Storyboard.TargetName="StoryboardRectangle" d:IsOptimized="True"/>
      </Storyboard>

  </Window.Resources>
```
再放置一个按钮，使用EventTrigger，当点击时，调用ControlStoryboardAction

```xml
<Button Content="Start storyboard" HorizontalAlignment="Stretch" Grid.Row="1" VerticalAlignment="Stretch" Margin="0,10,0,10" Width="128" Height="28">
      <i:Interaction.Triggers>
          <i:EventTrigger EventName="Click">
              <i:ControlStoryboardAction Storyboard="{StaticResource StoryboardSample}" ControlStoryboardOption="TogglePlayPause"/>
          </i:EventTrigger>
      </i:Interaction.Triggers>
  </Button>
```
## **CallMethodAction**
调用指定对象的方法。首先我们看一下**调用ViewModel中方法的示例**，添加一个按钮，如下所示：

```xml
<Button x:Name="button2" Content="调用当前DataContext类方法" HorizontalAlignment="Stretch" Margin="20,3" Grid.Column="1">
     <i:Interaction.Triggers>
         <i:EventTrigger EventName="Click" SourceObject="{Binding ElementName=button2}">
             <i:CallMethodAction TargetObject="{Binding}" MethodName="CallMe"/>
         </i:EventTrigger>
     </i:Interaction.Triggers>
 </Button>
```

然后我们在ViewModel中增加一个CallMe函数，为了方便演示，我直接将后台代码类设置为DataContext。

```cs
public partial class ActionWindow : Window
  {
      public ActionWindow()
      {
          InitializeComponent();

          //设置数据上下文（ViewModel）
          //这里只做演示，直接使用后台类作为ViewModel
          this.DataContext = this;
      }

      public void CallMe()
      {
          System.Windows.MessageBox.Show("调用了ActionWindow的方法");
      }
  }
```
## **GoToStateAction**

调用时将元素切换到指定 VisualState 的动作。

关于Visual State，可以参考前面的文章：**https://www.cnblogs.com/zhaotianff/p/13254430.html

首先我们**定义一个Button控件**，并**增加两个VisualState**。**启用和禁用状态**，启用时，颜色切换为Pink，禁用时切换为Silver
```xml
<Button x:Name="sampleStateButton" Content="示例按钮" VerticalAlignment="Stretch" Height="28" Width="88" Margin="0,0,0,0">
         <Button.Resources>
             <Style TargetType="Button">
                 <Setter Property="Template">
                     <Setter.Value>
                         <ControlTemplate TargetType="Button">
                             <Grid x:Name="BaseGrid" HorizontalAlignment="Stretch" VerticalAlignment="Stretch" Background="Pink">
                                 <Label Content="{TemplateBinding Content}" HorizontalAlignment="Center" VerticalAlignment="Center"></Label>
                                 <VisualStateManager.VisualStateGroups>
                                     <VisualStateGroup>
                                         <VisualState x:Name="Normal">
                                             <Storyboard>
                                                 <ColorAnimation Storyboard.TargetName="BaseGrid" Duration="0:0:0.3" Storyboard.TargetProperty="(Grid.Background).(SolidColorBrush.Color)" To="Pink"/>
                                             </Storyboard>
                                         </VisualState>
                                         <VisualState x:Name="Disabled">
                                             <Storyboard>
                                                 <ColorAnimation Storyboard.TargetName="BaseGrid" Duration="0:0:0.3" Storyboard.TargetProperty="(Grid.Background).(SolidColorBrush.Color)" To="Silver"/>
                                             </Storyboard>
                                         </VisualState>
                                     </VisualStateGroup>
                                 </VisualStateManager.VisualStateGroups>
                             </Grid>
                         </ControlTemplate>
                     </Setter.Value>
                 </Setter>
             </Style>
         </Button.Resources>
     </Button>
```

然后增加一个CheckBox，当选中时，切换到Disabled状态，未选中时，切换到Normal状态：

```xml
<CheckBox x:Name="checkBox" Grid.Row="1" Content="启用/禁用" HorizontalAlignment="Center" Margin="0,5">
      <i:Interaction.Triggers>
          <i:DataTrigger Binding="{Binding IsChecked, ElementName=checkBox}" Value="False">
              <i:GoToStateAction StateName="Normal" TargetObject="{Binding ElementName=sampleStateButton}"/>
          </i:DataTrigger>
          <i:DataTrigger Binding="{Binding IsChecked, ElementName=checkBox}" Value="True">
              <i:GoToStateAction StateName="Disabled" TargetObject="{Binding ElementName=sampleStateButton}"/>
          </i:DataTrigger>
      </i:Interaction.Triggers>
  </CheckBox>
```

## **LaunchUriOrFileAction**

用于**启动进程打开文件或 Uri 的操作**。对于文件，**该操作将启动默认程序 程序**。Uri 将在网络浏览器中打开。

这里我们**直接放置一个按钮**，当按钮点击 时，打开bing主页：

```xml
<Button x:Name="buttonlaunch" Content="打开Bing" HorizontalAlignment="Stretch" Width="88" Height="28">
     <i:Interaction.Triggers>
         <i:EventTrigger EventName="Click" SourceObject="{Binding ElementName=buttonlaunch}">
             <i:LaunchUriOrFileAction Path="https://www.bing.com" />
         </i:EventTrigger>
     </i:Interaction.Triggers>
 </Button>
```

## **PlaySoundAction**

**播放音频的动作**，此操作适用于不需要停止或控制的简短音效。它不支持暂停/继续等操作，仅适用于单次播放。
这是我们放置一个按钮，**并嵌入一个音频文件到程序中**，当点击按钮的时候，可以播放音频。
注意：音频没播放完成前，退出主窗口，进程不会退出，需要等到音频播放完

```xml
<Button x:Name="buttonplay" Content="播放音频" Width="88" Height="28">
    <i:Interaction.Triggers>
        <i:EventTrigger EventName="Click" SourceObject="{Binding ElementName=buttonplay}">
            <i:PlaySoundAction Source="../../../Resources/Cheer.mp3" Volume="1.0" />
        </i:EventTrigger>
    </i:Interaction.Triggers>
</Button>
```

## **InvokeCommandAction**

执行一个指定的命令。

这个**Action是我们最常用的Action，使用MVVM模式进行开发时**，需要**将控件的事件绑定到Command上，就可以使用InvokeCommandAction**。

这里我们**放置一个ListBox，并在ViewModel中处理ListBox选择项切换事件**。

使用EventTrigger处理SelectionChanged，然后使用InvokeCommandAction，**将SelectionChanged事件绑定到OnListBoxSelectionChangedCommand命令**。

```xml
<ListBox Width="300" Height="200">
       <i:Interaction.Triggers>
           <i:EventTrigger EventName="SelectionChanged">
               <i:InvokeCommandAction Command="{Binding OnListBoxSelectionChangedCommand}" CommandParameter="{Binding RelativeSource={RelativeSource Mode=FindAncestor,AncestorType=ListBox},Path=SelectedItem}"></i:InvokeCommandAction>
           </i:EventTrigger>
       </i:Interaction.Triggers>

       <ListBoxItem>1</ListBoxItem>
       <ListBoxItem>2</ListBoxItem>
       <ListBoxItem>3</ListBoxItem>
       <ListBoxItem>4</ListBoxItem>
   </ListBox>
```

在ViewModel中定义OnListBoxSelectionChangedCommand命令：

```cs
public RelayCommand OnListBoxSelectionChangedCommand { get; private set; }

      public ActionWindow()
      {
          OnListBoxSelectionChangedCommand = new RelayCommand(OnListBoxSelectionChanged);
      }

      public void OnListBoxSelectionChanged(object listBoxItem)
      {
          var item = listBoxItem as ListBoxItem;

          if (item != null)
          {
              MessageBox.Show(item.Content.ToString());
          }
      }
  }
```
这样我们在切换列表项时，就会弹窗显示选中项。

## **RemoveElementAction**

调用时**将从树中移除目标元素的操作**。

 我们**放置一个Rectangle和一个Button**，当Button按下时，**移除这个Rectangle**。
 
```xml
<StackPanel>
     <Rectangle x:Name="RectangleRemove" Fill="Pink" Width="120" Height="120"/>
     <Button x:Name="RemoveButton" Content="移除Rectangle" Width="88" Height="28">
         <i:Interaction.Triggers>
             <i:EventTrigger EventName="Click">
                 <i:RemoveElementAction TargetName="RectangleRemove" />
             </i:EventTrigger>
         </i:Interaction.Triggers>
     </Button>
 </StackPanel>
```

介绍完Trigger和Action，接下我们看一下Microsoft.Xaml.Behaviors.Wpf包自带的Behavior

# **常见Behavior的用法**

## **FluidMoveBehavior**

**观察元素（或元素集）的布局变化，并在需要时将元素平滑移动到新位置的行为**。  
此行为**不会对元素的大小或可见性进行动画调整**，只会对元素在其父容器中的偏移量进行动画调整。

例如我们在一个WrapPanel中放置一些控件，并加上FluidMoveBehavior。

这里有几个属性可以注意下：

AppliesTo：表示该行为仅适用于该元素，还是适用于该元素的所有子元素（如果该元素是面板）。

EaseY：用于移动垂直部分的 缓动动画。

EaseX：用于移动水平部分的 缓动动画。

我们看一下使用方法，**首先我们在界面上放置一个WrapPanel**，然后增加一些**子元素(Border)**，并增加**FluidMoveBehavior**。


```xml
<WrapPanel Width="300" Name="wrap">
     <i:Interaction.Behaviors>
         <i:FluidMoveBehavior Duration="00:00:01" AppliesTo="Children">
             <i:FluidMoveBehavior.EaseX>
                 <BounceEase EasingMode="EaseOut" Bounces="2" />
             </i:FluidMoveBehavior.EaseX>
         </i:FluidMoveBehavior>
     </i:Interaction.Behaviors>
     <Border Width="60" Height="60" BorderThickness="1" BorderBrush="Black" CornerRadius="5" Margin="5"></Border>
     <Border Width="60" Height="60" BorderThickness="1" BorderBrush="Black" CornerRadius="5" Margin="5"></Border>
     <Border Width="60" Height="60" BorderThickness="1" BorderBrush="Black" CornerRadius="5" Margin="5"></Border>
     <Border Width="60" Height="60" BorderThickness="1" BorderBrush="Black" CornerRadius="5" Margin="5"></Border>
     <Border Width="60" Height="60" BorderThickness="1" BorderBrush="Black" CornerRadius="5" Margin="5"></Border>
     <Border Width="60" Height="60" BorderThickness="1" BorderBrush="Black" CornerRadius="5" Margin="5"></Border>
     <Border Width="60" Height="60" BorderThickness="1" BorderBrush="Black" CornerRadius="5" Margin="5"></Border>
     <Border Width="60" Height="60" BorderThickness="1" BorderBrush="Black" CornerRadius="5" Margin="5"></Border>
 </WrapPanel>
```

由于BounceEase的存在，Border会模拟物体撞击地面的反弹效果`Bounces="2"` 意味着每当元素移动到新位置时，它会在水平方向（X轴）发生 2 次回弹，这在视觉上就表现为你所说的“跳跃”。
我们可以改为四阶函数的动画效果，取消掉弹簧，同时用EaseY控制Y轴的动画表现。

```xml
<i:Interaction.Behaviors>  
    <i:FluidMoveBehavior Duration="00:00:0.4" AppliesTo="Children">  
        <i:FluidMoveBehavior.EaseX>  
            <QuadraticEase EasingMode="EaseInOut" />  
        </i:FluidMoveBehavior.EaseX>  
        <i:FluidMoveBehavior.EaseY>  
            <QuadraticEase EasingMode="EaseInOut" />  
        </i:FluidMoveBehavior.EaseY>  
    </i:FluidMoveBehavior>  
</i:Interaction.Behaviors>
```



当窗口打开时，可以看到动画效果。**然后我们增加一个按钮，点击 一次面板宽度减少50**，因为**WrapPanel会根据宽度自动排列元素**，所以在宽度变化 时，我们也可以看到元素的动画效果。

```cs
<Button Content="移除元素" Width="88" Height="28" Click="Button_Click"></Button>
private void Button_Click(object sender, RoutedEventArgs e)
 {
            this.wrap.Width -= 50;
}
```

## **MouseDragElementBehavior**

这个行为可以**根据鼠标在元素上的拖动手势重新定位所附元素**。

属性：

ConstrainToParentBounds：是否限制在父容器内

X:被拖动元素相对于根元素左侧 X 位置。

Y:X:被拖动元素相对于根元素顶部 Y 位置。
例如我们在界面上放置一个Rectangle元素，然后使之可以拖动，使用方法如下：
```xml
1  <Rectangle Width="50" Height="50" Fill="LightGreen">
2      <i:Interaction.Behaviors>
3          <i:MouseDragElementBehavior></i:MouseDragElementBehavior>
4      </i:Interaction.Behaviors>
5  </Rectangle>
```

## **TranslateZoomRotateBehavior**

这个行为**和MouseDragElementBehavior类似**，如果在**触控设备**上，除了**支持移动元素外**，还支持**手指缩放**，如果**在不支持触控的设备上，和TranslateZoomRotateBehavior功能一致**。

## **DataStateBehavior**

根据**条件语句在两种状态之间切换**。它表示一种状态，可以**根据某些条件是否为真来改变对象的状态**。

这个行为也是跟VisualState有关的，这里我们定义两个State，一个Normal，一个Blue，并增加一个文本框和一个按钮，当文本框数据改变时，**改变按钮的状态为Blue**

```xml
<StackPanel>
     <TextBox Grid.Row="1" VerticalAlignment="Center" HorizontalAlignment="Stretch" Margin="5" x:Name="MyText" />
     <Button x:Name="sampleStateButton" Width="88" Height="28">
         <Button.Resources>
             <Style TargetType="Button" >
                 <Setter Property="Template">
                     <Setter.Value>
                         <ControlTemplate TargetType="Button">
                             <Grid x:Name="BaseGrid" HorizontalAlignment="Stretch" VerticalAlignment="Stretch" Background="Pink">
                                 <VisualStateManager.VisualStateGroups>
                                     <VisualStateGroup>
                                         <VisualState x:Name="Normal">
                                             <Storyboard>
                                                 <ColorAnimation Storyboard.TargetName="BaseGrid" Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)" To="Pink" From="Pink" />
                                             </Storyboard>
                                         </VisualState>
                                         <VisualState x:Name="Blue">
                                             <Storyboard>
                                                 <ColorAnimation Storyboard.TargetName="BaseGrid" Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)" To="LightBlue" From="LightBlue" Duration="Forever" />
                                             </Storyboard>
                                         </VisualState>
                                     </VisualStateGroup>
                                 </VisualStateManager.VisualStateGroups>
                                 <i:Interaction.Behaviors>
                                     <i:DataStateBehavior Binding="{Binding Text, ElementName=MyText}" Value="" TrueState="Normal" FalseState="Blue" />
                                 </i:Interaction.Behaviors>
                             </Grid>
                         </ControlTemplate>
                     </Setter.Value>
                 </Setter>
             </Style>
         </Button.Resources>
     </Button>
 </StackPanel>
```

这个也是十分实用，我们可以配合VSM和FluidMoveBehavior轻松实现布局的平滑过渡，来实现具有丝滑感的逻辑解耦的导航栏