### IRelayCommand和 ICommand

 这里我想要分清一下IRelayCommand和 ICommand，它们相比之下IRelayCommand提供了更多的方法。从定义上来看，`IRelayCommand` 继承自 `ICommand`。 这意味着：**所有的 `IRelayCommand` 都是 `ICommand`**，但反过来不成立。
```cs
// IRelayCommand 的定义（伪代码）
public interface IRelayCommand : ICommand
{
    // 它可以做所有 ICommand 能做的事（Execute, CanExecute）
    // 但它多了一个关键能力：
    void NotifyCanExecuteChanged(); 
}
```

- **`ICommand` (标准接口)**
    
    - **只管“听”**：它定义了一个事件 `CanExecuteChanged`。UI 控件（如 Button）订阅这个事件，当事件触发时，**Button 会重新检查自己是否该置灰**。
        
    - **痛点**：`ICommand` **没有定义“如何触发”这个事件的标准方法**。如果你手里只拿着一个 `ICommand` 接口的对象，你无法命令它：“喂，状态变了，赶紧通知 UI 刷新一下”。你通常必须把对象**强转成具体的实现类**（如 `RelayCommand`）才能**调用触发方法**。
        
- **`IRelayCommand` (工具包接口)**
    
    - **能“喊”**：它在 `ICommand` 的基础上，**额外增加**了一个 **`NotifyCanExecuteChanged()`** 方法。
        
    - **优势**：当你持有 `IRelayCommand` 接口时，你可以直接调用 `.NotifyCanExecuteChanged()`。这让你可以在 ViewModel 中**显式地手动触发按钮状态的刷新**，而**不需要关心底层的具体实现类是什么**。

并且RelayCommand针对了异步任务有了一个默认特性，当我们将它放在了一个Task方法上的时候，生成的命令是AsyncRelayCommand，这个命令有一个默认配置：**`AllowConcurrentExecutions = false`**（允许**并发执行** = 否）。

因为它不允许同时跑两个任务，所以它的 `CanExecute`（能否执行）状态会自动变成 `false`，绑定了这个**命令的 Button 收到通知**，自动把自己变成 `IsEnabled = false`（灰色）。

这个还是相当智能的。

并且，这个注解还可以在使用的时候传入CanExecute参数，来控制按钮的可用状态等。只需要在RelayCommand中指定了CanExecue参数，传入那个判断方法的名字，可以使用nameof来避免拼写错误。

以及这个command还可以被多个组件绑定和使用，并且都有刚才说的那个特性，完全不冲突！


### 开启预览语法特性

我们可以在**csproj文件**中输入
```
<LangVersion>preview</LangVersion>
```
来开启目前c#编译器中没有正式成为默认标准的预览版语言特性。

这里我们举一个CT常使用的例子，也就是上面提到的IRelayCommand和ICommand。


```cs
[ObservableProperty] public partial bool IsLoading { get; private set; }
```
它与我们过去几年“常见”的两种写法相比，解决了多个长期存在的痛点，对比旧版的生成器写法来说，有了访问权限的控制，新写法就是所见就是所得。


如果你想给生成的属性加 `[JsonIgnore]`，你必须用怪异的语法：
```cs
[ObservableProperty]
[property: JsonIgnore] // 必须告诉编译器这个特性是要加给属性的，不是字段
private bool _isLoading;
```

**新写法**：直接加，符合直觉。

```cs
[ObservableProperty]
[JsonIgnore] // 直接加，很自然
public partial bool IsLoading { get; private set; }
```

**旧写法**：你把注释写在私有字段 `_isLoading` 上，生成器会尝试把注释“搬运”到生成的 `IsLoading` 属性上。这在 IDE 提示中有时会有延迟或错位，**新写法**：注释直接写在 `public` 属性上，IDE 的智能提示（IntelliSense）极其完美，完全没有隔阂。

