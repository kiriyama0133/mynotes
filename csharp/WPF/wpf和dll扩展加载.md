# WPF和DLL扩展加载

## 目录
1. [DLL基础概念](#1-dll基础概念)
2. [WPF中的DLL加载机制](#2-wpf中的dll加载机制)
3. [程序集反射加载](#3-程序集反射加载)
4. [插件化架构设计](#4-插件化架构设计)
5. [动态加载与卸载](#5-动态加载与卸载)
6. [依赖注入与扩展](#6-依赖注入与扩展)
7. [实际应用案例](#7-实际应用案例)
8. [最佳实践与注意事项](#8-最佳实践与注意事项)

---

## 1. DLL基础概念

### 1.1 什么是DLL
- 动态链接库的定义
- DLL与静态库的区别
- .NET程序集的概念

### 1.2 DLL的类型
- 托管DLL（.NET程序集）
- 非托管DLL（Win32）
- 混合模式DLL

### 1.3 DLL的优势
- 代码复用
- 模块化开发
- 动态更新
- 内存优化

---

## 2. WPF中的DLL加载机制

### 2.1 程序集加载方式
```csharp
// 静态加载
Assembly assembly = Assembly.LoadFrom("path/to/dll");

// 动态加载
Assembly assembly = Assembly.LoadFile("path/to/dll");

// 反射加载
Type type = assembly.GetType("Namespace.ClassName");
```

### 2.2 AppDomain与程序集隔离
- AppDomain的概念
- 程序集隔离机制
- 跨域调用

在 .NET 框架中，**AppDomain** (应用程序域) 是一个重要的概念，它提供了一种在**单个进程内隔离应用程序或组件的方式**。你可以把它想象成一个轻量级的“沙盒”。

**程序集隔离**指的是将不同的程序集（.dll 或 .exe 文件）加载到**相互独立的 AppDomain** 中，从而实现它们之间的隔离。这种隔离带来的主要好处有：

- **错误隔离：** 一个 AppDomain 中的代码如果抛出未处理的异常，它只会影响到该 AppDomain，而不会导致整个进程崩溃。这使得你可以更健壮地运行第三方代码或插件。
- **卸载隔离：** 程序集**一旦被加载到 AppDomain 中**，就无法轻易地从内存中卸载。**但通过卸载整个 AppDomain，可以有效地释放其加载的所有程序集和资源。这对于需要动态加载和卸载插件或组件的应用程序非常有用**。
- **配置隔离：** 每个 AppDomain 都有自己的配置，例如应用程序设置、权限集等。这使得在同一个进程中运行不同配置的应用程序成为可能。
- **资源隔离：** 不同的 AppDomain **拥有各自的静态变量和资源，避免了相互干扰**。



### 2.3 程序集解析事件
```csharp
AppDomain.CurrentDomain.AssemblyResolve += OnAssemblyResolve;
```

---

## 3. 程序集反射加载

### 3.1 反射基础
- Type类的使用
- 获取类型信息
- 创建实例

### 3.2 动态类型发现
```csharp
// 扫描程序集中的所有类型
var types = assembly.GetTypes()
    .Where(t => t.GetInterfaces().Contains(typeof(IPlugin)))
    .ToList();
```

### 3.3 接口驱动的扩展
- 定义扩展接口
- 实现扩展发现
- 动态实例化

### 3.4 实际应用示例
```csharp
public class PluginManager
{
    public List<IPlugin> LoadPlugins(string directory)
    {
        var plugins = new List<IPlugin>();
        var dllFiles = Directory.GetFiles(directory, "*.dll");
        
        foreach (var dll in dllFiles)
        {
            var assembly = Assembly.LoadFrom(dll);
            var pluginTypes = assembly.GetTypes()
                .Where(t => typeof(IPlugin).IsAssignableFrom(t) && !t.IsInterface);
                
            foreach (var type in pluginTypes)
            {
                var plugin = (IPlugin)Activator.CreateInstance(type);
                plugins.Add(plugin);
            }
        }
        
        return plugins;
    }
}
```

---

## 4. 插件化架构设计

### 4.1 插件接口设计
```csharp
public interface IPlugin
{
    string Name { get; }
    string Version { get; }
    void Initialize();
    void Execute();
    void Dispose();
}
```

### 4.2 插件生命周期管理
- 插件加载
- 插件初始化
- 插件执行
- 插件卸载

### 4.3 插件通信机制
- 事件总线
- 消息传递
- 共享数据

### 4.4 插件配置管理
```csharp
public class PluginConfiguration
{
    public string Name { get; set; }
    public string AssemblyPath { get; set; }
    public bool Enabled { get; set; }
    public Dictionary<string, object> Settings { get; set; }
}
```

---

## 5. 动态加载与卸载

### 5.1 Assembly.LoadFrom vs Assembly.LoadFile
- 区别与使用场景
- 性能考虑
- 依赖解析

### 5.2 程序集卸载
```csharp
// 创建新的AppDomain
AppDomain pluginDomain = AppDomain.CreateDomain("PluginDomain");

// 在插件域中加载程序集
pluginDomain.Load(assemblyName);

// 卸载插件域
AppDomain.Unload(pluginDomain);
```

### 5.3 内存管理
- 程序集缓存
- 垃圾回收
- 内存泄漏预防

---

## 6. 依赖注入与扩展

### 6.1 服务容器集成
```csharp
// 使用Microsoft.Extensions.DependencyInjection
public void ConfigureServices(IServiceCollection services)
{
    // 扫描程序集注册服务
    var assemblies = Directory.GetFiles(pluginDirectory, "*.dll")
        .Select(Assembly.LoadFrom);
        
    foreach (var assembly in assemblies)
    {
        services.AddServicesFromAssembly(assembly);
    }
}
```

### 6.2 扩展点设计
- 扩展接口定义
- 扩展发现机制
- 扩展注册

### 6.3 热插拔支持
- 文件监控
- 动态重载
- 状态管理

---

## 7. 实际应用案例

### 7.1 主题系统
```csharp
public interface ITheme
{
    string Name { get; }
    ResourceDictionary GetResources();
}

// 动态加载主题
public void LoadTheme(string themePath)
{
    var assembly = Assembly.LoadFrom(themePath);
    var themeType = assembly.GetTypes()
        .FirstOrDefault(t => typeof(ITheme).IsAssignableFrom(t));
        
    if (themeType != null)
    {
        var theme = (ITheme)Activator.CreateInstance(themeType);
        Application.Current.Resources.MergedDictionaries.Add(theme.GetResources());
    }
}
```

### 7.2 控件扩展
- 自定义控件库
- 动态控件加载
- 样式扩展

### 7.3 业务模块化
- 功能模块分离
- 按需加载
- 模块间通信

---

## 8. 最佳实践与注意事项

### 8.1 安全性考虑
- 程序集签名验证
- 权限控制
- 沙箱隔离

### 8.2 性能优化
- 延迟加载
- 程序集缓存
- 资源管理

### 8.3 错误处理
```csharp
try
{
    var assembly = Assembly.LoadFrom(dllPath);
    // 处理加载成功
}
catch (BadImageFormatException)
{
    // 处理无效程序集
}
catch (FileLoadException)
{
    // 处理加载失败
}
catch (Exception ex)
{
    // 处理其他异常
}
```

### 8.4 调试与诊断
- 程序集加载日志
- 性能监控
- 内存分析

### 8.5 部署策略
- 程序集版本管理
- 依赖关系处理
- 更新机制

---

## 总结

WPF中的DLL扩展加载为应用程序提供了强大的模块化和可扩展性能力。通过合理的设计和实现，可以构建出灵活、可维护的应用程序架构。关键是要平衡灵活性、性能和安全性，选择合适的加载策略和架构模式。

## 实操

​	我们在wpf中需要建立一个wpf类库，然后在这个基础上引用原来的库，最后将其编译为dll就行，记得使用项目引用，可以防止再新建一个公用的类库。

首先我们需要一个加载dll的方法实现。

```csharp
private void LoadExtentions()
{
    try
    {
        string pluginPath = Path.Combine(AppContext.BaseDirectory, "Extensions");
        if (!Directory.Exists(pluginPath))
        {
            Console.WriteLine("Extensions folder not found.");
            return;
        }
        
        var dllFiles = Directory.GetFiles(pluginPath, "*.dll");
        if(!dllFiles.Any())
        {
            Console.WriteLine("No plugins found.");
            return;
        }
        
        var extensionPages = new List<(Type PageType, Type ViewModelType, string Title)>();
        
        foreach (var dllFile in dllFiles) {
            var assembly = Assembly.LoadFrom(dllFile);
            Console.WriteLine($"Loaded plugin: {assembly.GetName().Name}");

            var viewModelTypes = assembly.GetTypes().Where(t => t.IsSubclassOf(typeof(ExtentionViewModel))).ToList();
            var pageTypes = assembly.GetTypes().Where(t => t.IsSubclassOf(typeof(Page))).ToList();
            
            foreach (var pageType in pageTypes)
            {
                var viewModelType = viewModelTypes.FirstOrDefault(vm => 
                    vm.Name == pageType.Name + "ViewModel");
                
                string title = GetPageTitle(viewModelType, pageType);
                extensionPages.Add((pageType, viewModelType, title));
                
                Console.WriteLine($"Found extension: {title} - {pageType.Name}");
            }
        }
        
        ExtensionPages = extensionPages;
        Console.WriteLine($"Total extensions loaded: {ExtensionPages.Count}");
    }
    catch (Exception ex) { 
        Console.WriteLine($"Failed to load plugin: {ex.Message}");
    }
}
```

​	然后我们使用静态属性类去存储扩展页面的信息：

**public static List<(Type PageType, Type ViewModelType, string Title)> ExtensionPages { get; private set; } = new();**

​	这个方法是加载继承了ExtensionViewModel 和 Page 类 的类方法，我们需要从获得的viewmodel类中得到我们想要的Title Icon等信息。

```csharp
private string GetPageTitle(Type viewModelType, Type pageType)
{
    if (viewModelType != null)
    {
        try
        {
            var instance = Activator.CreateInstance(viewModelType);
            
            // 尝试 Title 属性
            var titleProperty = viewModelType.GetProperty("Title");
            if (titleProperty != null)
            {
                var title = titleProperty.GetValue(instance) as string;
                Console.WriteLine($"Title property value: '{title}'");
                if (!string.IsNullOrEmpty(title)) return title;
            }
            else
            {
                Console.WriteLine("Title property not found");
            }
            
            // 尝试 DisplayName 属性
            var displayNameProperty = viewModelType.GetProperty("DisplayName");
            if (displayNameProperty != null)
            {
                var displayName = displayNameProperty.GetValue(instance) as string;
                Console.WriteLine($"DisplayName property value: '{displayName}'");
                if (!string.IsNullOrEmpty(displayName)) return displayName;
            }
            
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error creating ViewModel instance: {ex.Message}");
        }
    }
    
    // 默认使用类名（去掉Page后缀）
    string className = pageType.Name;
    if (className.EndsWith("Page"))
    {
        className = className.Substring(0, className.Length - 4);
    }
    
    Console.WriteLine($"Using default class name: '{className}'");
    return className;
}
```

​	完成之后我们就可以去写依赖注入的服务注册扩展实现的方法

```csharp
    private void RegisterExtensionPages(IServiceCollection services)
    {
        foreach (var (pageType, viewModelType, title) in ExtensionPages)
        {
            services.AddSingleton(pageType);
            if (viewModelType != null)
            {
                services.AddSingleton(viewModelType);
            }
            Console.WriteLine($"Registered extension: {title}");
        }
    }
```

​	最后在导航栏一侧去注册就ok了，当然也可以通过反射程序集的方式加载导航栏，看自己的项目需求，下面是获取图标和添加导航的方法实现。

```csharp
    private void InitializeMenuItems()
    {
        // 添加默认菜单项
        _menuItems.Add(new NavigationViewItem("Home", SymbolRegular.Home24, typeof(DashboardPage)));
        _menuItems.Add(new NavigationViewItem("Color采集", SymbolRegular.Color24, typeof(ColorCollecterPage)));

        // 添加扩展页面的菜单项
        foreach (var (pageType, viewModelType, title) in App.ExtensionPages)
        {
            // 尝试从ViewModel获取图标
            var icon = GetIconFromViewModel(viewModelType);
            
            var menuItem = new NavigationViewItem(title, icon, pageType)
            {
                ToolTip = $"打开{title}扩展"
            };
            _menuItems.Add(menuItem);
        }
    }

// 获取图标
    private SymbolRegular GetIconFromViewModel(Type viewModelType)
    {
        if (viewModelType != null)
        {
            try
            {
                // 尝试从 Icon 属性获取图标
                var iconProperty = viewModelType.GetProperty("Icon");
                if (iconProperty != null)
                {
                    var instance = Activator.CreateInstance(viewModelType);
                    var icon = iconProperty.GetValue(instance);
                    if (icon is SymbolRegular symbolIcon)
                    {
                        return symbolIcon;
                    }
                }
            }
            catch
            {
                // 如果无法获取图标，使用默认图标
            }
        }
        // 默认图标
        return SymbolRegular.Home24;
    }
```

​	但是这里要注意一件事情，那就是这种加载方式下，xaml的加载不同于我们之前使用的 在 this.DataContext = this;方法 ，可以在xaml上用ViewModel.进行绑定，默认情况下，如果在插件实现的库中这样写有很大概率是找不到你的viewModel的，会出现绑定失败的，因此一般情况下，**推荐直接使用ViewModel进行 业务逻辑和数据的绑定**。
