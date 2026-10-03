
### BoxShadow
BoxShadow是一个结构体，用于为控件添加阴影效果。他有以下几个属性：

```cs
- **OffsetX**: 水平偏移,正值向右,负值向左<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Media/BoxShadow.cs:18-23>
- **OffsetY**: 垂直偏移,正值向下,负值向上<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Media/BoxShadow.cs:25-32>
- **Blur**: 模糊半径,值越大阴影越模糊<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Media/BoxShadow.cs:34-42>
- **Spread**: 扩散半径,正值使阴影扩大,负值使阴影缩小<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Media/BoxShadow.cs:44-52>
- **Color**: 阴影颜色<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Media/BoxShadow.cs:54-57>
- **IsInset**: 是否为内阴影<link to Repo AvaloniaUI/Avalonia: src/Avalonia.Base/Media/BoxShadow.cs:59-68>
```
如果前面加上inset，那么就是给设置内部阴影，比如说这样：

```xaml
  <Border x:Name="MyBorder" Width="200" Height="50" CornerRadius="12" BoxShadow="inset -1 2 30 #CCFFFFFF, 10 5 20 #33000000" Background="#000">
      <TextBlock Foreground="#fff" Text="{Binding Greeting}" HorizontalAlignment="Center" VerticalAlignment="Center"/>
  </Border>
```
