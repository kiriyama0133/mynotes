
Avalonia现在不只是支持linux平台，还支持Android平台开发，对于次，avalonia提供了两种标记扩展来实现区分平台的操作。
1. **`OnFormFactor`** - 根据设备类型(Desktop/Mobile/TV)提供不同的值
2. **`OnPlatform`** - 根据操作系统平台(Windows/macOS/Linux/iOS/Android/Browser)提供不同的值
### 实际使用示例

```
<DockPanel>  
    <!-- Tabs 导航栏,PC 在左边,移动在底部 -->  
    <TabControl DockPanel.Dock="{OnFormFactor Desktop=Left, Mobile=Bottom}">  
        <TabItem Header="首页" />  
        <TabItem Header="设置" />  
    </TabControl>  
      
    <!-- 主内容区域 -->  
    <ContentControl />  
</DockPanel>
```

同理，如果还想要设置横向或者竖向，也可以这样干：

```

<TabControl TabStripPlacement="{OnFormFactor Desktop=Left, Mobile=Top}"    DockPanel.Dock="{OnFormFactor Desktop=Left, Mobile=Bottom}">
			<TabItem Header="首页"></TabItem>
            <TabItem Header="我的"></TabItem>
</TabControl>

```

**几乎所有的属性都可以使用 `OnFormFactor` 来控制**。`OnFormFactor` 是一个通用的标记扩展,可以为任何属性类型提供基于设备类型的不同值。

`OnFormFactor` 在 XAML 编译时被处理,通过 `AvaloniaXamlIlOptionMarkupExtensionTransformer` 转换器: AvaloniaXamlIlOptionMarkupExtensionTransformer.cs:15-24

编译器会检查 `ShouldProvideOption` 方法来决定使用哪个平台的值,这个过程在编译时完成,因此运行时性能没有额外开销。

我们顺便也把tabs订阅点击事件来得到tabs变化进而进行导航跳转的逻辑，`TabControl` 继承自 `SelectingItemsControl`,提供了 `SelectionChanged` 事件:

```cs
private void TabControl_SelectionChanged(object? sender, SelectionChangedEventArgs e)  
{  
    if (sender is TabControl tabControl)  
    {  
        // 获取选中的 TabItem  
        if (tabControl.SelectedItem is TabItem selectedTab)  
        {  
            var header = selectedTab.Header;  
            Console.WriteLine($"Selected tab header: {header}");  
              
            // 根据 header 进行导航  
            NavigateBasedOnHeader(header);  
        }  
    }  
}  
  
private void NavigateBasedOnHeader(object? header)  
{  
    switch (header?.ToString())  
    {  
        case "首页":  
            // 导航到首页  
            break;  
        case "设置":  
            // 导航到设置页  
            break;  
    }  
}
```
这里的导航逻辑我们可以用一个Dictionary去获得对应的view实例，这样就能很方便地设置Content进行导航，当然，使用viewlocator也可以。如果要使用ViewLocator的话，就需要注意名称用法，因为默认的ViewLocator标准实现是通过名称替换的方式去创建实例的，
var name = data.GetType().FullName!.Replace("ViewModel", "View");

只不过我们日常在使用的时候这里会通过DI容器去获得实例而不是直接创建。