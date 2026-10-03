​	WPF 框架本身已经为我们封装好了 95% 以上常用功能，让我们不必关心底层的窗体句柄（HWND）、消息循环（Message Loop）等复杂细节。但是，总有一些特殊的、更底层的系统功能，**WPF 没有直接提供 C# 的接口**。这时，我们就需要“下到地基”，通过 **P/Invoke (Platform Invocation，平台调用)** 技术，直接使用 Win32 API 提供的原生能力。

### **窗口高级定制与控制 (Advanced Window Customization & Control)**

WPF 的 `Window` 类虽然强大，但并未暴露所有的 Windows 窗口样式。当您想实现一些“反常规”的窗口行为时，就需要 Win32 API。

- **获取窗口句柄 (`HWND`)**: 这是与几乎所有 Win32 窗口 API 交互的基础。

  - **用途**: 将 WPF 窗口句柄传递给其他需要 `HWND` 的原生库（例如我们之前讨论的 **WGC 屏幕采集** `GraphicsCapturePicker`，或者一些渲染引擎）。
  - **API**: `user32.dll` -> `GetWindowLong`, `SetWindowLong` 

  我们之前有试过平台调用user32.dll的方法来消除alt+tab的窗口显示，这些只有平台调用才能够实现。

除此以外，还有很多的可以实现的功能，就比如：**注册全局热键 (Global Hotkey)**:

- **用途**: 让您的应用在**最小化或不处于焦点**时，也能响应一个全局快捷键（例如，`Ctrl+Alt+P` 随时随地截图）。这是纯 WPF 无法做到的。
- **API**: `user32.dll` -> `RegisterHotKey`, `UnregisterHotKey`。

**控制窗口状态**:

- **用途**: 强制将窗口带到最前、在任务栏上闪烁提示用户。 有点像qq那个振动说是。
- **API**: `user32.dll` -> `SetForegroundWindow`, `FlashWindowEx`。

### **底层系统与硬件交互 (Low-Level System & Hardware Interaction)**

当您需要获取 WPF 没有直接封装的系统信息或与特定硬件交互时。

- **枚举显示器和获取详细信息**:
  - **用途**: 获取每个显示器的分辨率、工作区、在虚拟桌面上的位置等。我们之前在**多显示器 GDI 截图**的例子中就用到了。
  - **API**: `user32.dll` -> `EnumDisplayMonitors`, `GetMonitorInfoEx`。

**电源管理通知**:

- **用途**: 当系统进入睡眠、休眠或电源模式改变时，让您的应用能够收到通知并做出响应。
- **API**: `user32.dll` -> `RegisterPowerSettingNotification`。

**与特定设备通信**:

- **用途**: 与没有现成 .NET 库的硬件通信，例如某些特定的 USB HID 设备。
- **API**: `hid.dll`, `setupapi.dll` 中的各种函数。

### **图形与 UI 互操作 (Graphics & UI Interoperability)**

这是最高级、最复杂的领域，也是我们之前讨论最多的。

- **GDI/GDI+ 绘图**:
  - **用途**: 进行非常高速的位图操作，或者与只接受 GDI 设备上下文 (`HDC`) 的旧版代码或库进行交互。我们之前的**原生 `BitBlt` 截图**就是典型例子。
  - **API**: `gdi32.dll` -> `BitBlt`, `CreateCompatibleDC`, `GetDIBits` 等。
- **DirectX 互操作**:
  - **用途**: 在 WPF 控件 (`D3DImage`) 中承载和显示由 DirectX 11/12 渲染的内容。这是 WPF 进行高性能游戏渲染、视频播放、WGC 画面显示等的**唯一途径**。这个过程深 依赖 Win32 API 来进行设备和资源的共享。
  - **API**: `d3d9.dll`, `d3d11.dll`, `dxgi.dll` 中的各种函数。
- **承载原生 Win32 控件**:
  - **用途**: 在 WPF 应用中嵌入一个由原生 C++ 编写的、只有 `HWND` 的旧版控件。
  - **实现**: 通过继承 `HwndHost` 类。

## 消息机制

​	我觉得刚好我们这里来探讨一下wpf的消息机制。谈起“消息机制”这个词，我们都会想到**Windows的消息机制**，系统将**键盘鼠标的行为包装成一个Windows Message**，然后**系统主动将这些Windows Message派发给特定的窗口**，实际上消息是被Post到特定窗口所在线程的**消息队列**，应用程序的**消息循环**再不断的**从消息队列当中获取消息**，然后再**派发给特定窗口类的窗口过程来处理**，在窗口过程中完成一次用户交互。

​	其实，WPF的底层也是基于Win32的消息系统，那么对于WPF应用程序来说，它是如何跟Win32的消息交互，这里到底存在一个什么样的机制？接下来我会通过下面几篇博文介绍这个消息机制:

http://www.cnblogs.com/powertoolsteam/archive/2010/12/30/1921426.html

​	WPF大部分的对象都是从DispatcherObject派生的，从这里派生的对象具有一个明显的特征，那就是：修改对象时所在的线程，和创建对象时所在线程必须为同一个线程，**这就是微软所谓的线程亲缘性（Thread affinity）的最简单理解**。

​	这个规则并非 WPF 的独创，而是几乎所有 GUI 框架的共同选择。其根本原因是为了保证 UI 系统的**稳定、可预测和简化开发**。**技术上的原因主要有三点：**

1. **防止数据竞争与状态不一致 (Preventing Race Conditions)**: UI 元素（如一个 `Button`）有大量的内部状态（宽度、高度、颜色、内容、是否被按下等）。如果允许多个线程同时修改这些状态，就会产生不可预知的后果，例如一个线程正在根据宽度计算布局，另一个线程却突然改变了宽度，导致计算错误甚至程序崩溃。
2. **简化开发模型 (Simplifying the Development Model)**: 如果没有这个规则，那么开发者在修改任何一个 UI 属性时，都必须手动添加复杂的线程锁 (`lock`) 来防止多线程冲突。这将使 UI 编程变得极其困难和容易出错，性能也会大打折扣。
3. **保证用户输入的顺序性 (Guaranteeing Input Order)**: 用户的鼠标点击、键盘输入等操作，都是以消息的形式被操作系统发送到**一个**特定的线程（UI 线程）上。UI 框架必须在一个线程中按顺序处理这些消息，以保证操作的逻辑正确性。

​	然后，我们接下来聊一下STA线程和UI线程的关系。`STA 线程` 是对这个线程**底层工作模式**的一种技术描述，而 `UI 线程` 是从**应用程序功能层面**对这个线程的称呼。可以说，**STA 是 UI 线程的“必要属性”或“技术身份证”**。要理解 STA，我们需要追溯到 Windows 的一个更底层、更古老的技术：**COM (Component Object Model, 组件对象模型)**。 为什么呢？

因为**STA** 全称为 **Single-Threaded Apartment (单线程套间)**，这是 COM 的一种线程模型！

**我们可以用一个办公室的比喻来理解：**

- **STA (单线程套间)**: 就像一个**“单人办公室”**。
  - 这个办公室里所有的工具和文件（COM 对象），只能由**这一个指定的员工（STA 线程）**来使用。
  - 其他办公室的员工（其他线程）如果想用这些工具，不能直接进来拿，必须通过前台（COM **封送机制**）递交一个工作请求，由这个办公室的员工亲自来处理。
- **MTA (多线程套间)**: 就像一个**“开放式办公区”**。
  - 办公区里的所有工具（COM 对象）都可以被**任何一个员工（任何一个 MTA 线程）**随时使用。
  - 这种模式要求工具本身必须非常坚固，能应付多人同时使用（即**对象本身必须是线程安全的**），否则就会乱套。

​	WPF 是一个现代化的框架，但它构建在 Windows 操作系统的基石之上，并且需要与大量**非线程安全**的底层系统组件进行交互。这些**组件中的很多都是以 COM 的形式存在的**，并且它们被设计为**只能在 STA 模式下安全运行**。

**操作系统组件交互 (Interaction with OS Components)**: WPF 的很多功能需要调用底层的 Windows API，例如：

- **剪贴板 (Clipboard)**
- **拖放操作 (Drag-and-Drop)**
- **文件对话框 (File Dialogs)**
- **与 Windows Shell (外壳) 的交互** 这些功能在底层都是通过非线程安全的 COM 组件实现的，它们**要求调用它们的线程必须是 STA 线程。**

​	**输入与消息处理 (Input and Message Processing)**: Windows 的用户输入系统（键盘、鼠标）是基于消息队列的。所有输入事件都会被投递到一个特定的线程消息队列中，并由这个线程按顺序处理。这个模型天然就是单线程的，与 STA 模式完美契合。所以我们总结一下就是，

> [!NOTE]
>
>  **在一个窗口中，所有的 UI 组件（成百上千个按钮、文本框、图片等）都由同一个、唯一的 UI 线程来创建、管理和更新。**
>
> ![image-20250912065523347](image-20250912065523347.png)



### 实现方式

WPF 通过一个名为 **`Dispatcher` (调度器)** 的核心组件来管理这个规则。

1. **`DispatcherObject` 的角色**: 提到的 `DispatcherObject` 是这个体系的基石。WPF 中几乎所有的 UI 相关对象（从 `Window`, `Button` 到 `Brush`）都继承自 `DispatcherObject`。它最重要的作用就是为每个对象提供一个 `Dispatcher` 属性，这个属性**永远指向创建该对象的那个 UI 线程的“调度器”**。
2. **`Dispatcher` 的角色**: `Dispatcher` 可以被看作是 UI 线程的**“任务调度中心”**或**“任务队列”**。每个 UI 线程都有且仅有一个 `Dispatcher`。它的工作就是**维护一个按优先级排序的任务队列**。

### **核心工具**

- **`CheckAccess()`**: 一个 `DispatcherObject` 的方法，用于检查“当前代码是否已经在正确的 UI 线程上运行？”。返回 `true` 或 `false`。
- **`Dispatcher.Invoke(...)`**: **同步**调用。后台线程将任务交给 `Dispatcher` 后，会**暂停并一直等待**，直到 UI 线程执行完该任务并返回结果。
- **`Dispatcher.BeginInvoke(...)`** 或 **`Dispatcher.InvokeAsync(...)`**: **异步**调用。后台线程将任务交给 `Dispatcher` 后，**不会等待**，而是立即继续执行自己的后续代码。这是更常用的方式，可以避免后台线程被阻塞。



刚才指的是UI 亲缘性，和我们之前说的CPU亲缘性不一样，差别可以用一个表格来理解：

在 WPF 的世界里，您可能更常听到“线程亲缘性”，但这通常指的是一个不同的、**框架层面**的概念。

| 方面     | **CPU 亲缘性 (OS Level)**                     | **UI 亲缘性 (WPF Framework Level)**                          |
| -------- | --------------------------------------------- | ------------------------------------------------------------ |
| **层面** | **操作系统内核**                              | **WPF 框架**                                                 |
| **目的** | 性能优化，**控制线程在哪个 CPU 核心上运行**。 | 保证 UI 的稳定和一致性，**规定哪个线程有权修改 UI 元素**。   |
| **机制** | `ProcessThread.ProcessorAffinity` (位掩码)    | `Dispatcher.Invoke` / `BeginInvoke` (消息队列)               |
| **规则** | 一个线程被绑定到特定 CPU 核心。               | 一个 UI 对象（如 `Button`）“属于”创建它的那个线程（UI 线程），其他线程不能直接访问它。 |

这两个概念都叫“亲缘性”，但解决的问题完全不同。

​	那么谁能保证线程亲缘性呢？那就是Dispacher了。从DispatcherObject派生的类型继承三个重要的成员：Dispatcher属性，CheckAccess(), VerifyAccess()方法。其中后面两个方法就是检验线程亲缘性的。按照WPF的实现，如果你自己定义了个WPF的类型，并且是DispatcherObject的子类，你就必须在public的成员定义的逻辑开始处，调用base.Dispatcher.VerifyAccess()，检验线程亲缘性。那么Dispatcher到底还做了什么事情呢？

![clip_image002](201012301025588804.jpg)

​	通过调用堆栈可以看出，蓝色的部分是启动了一个线程，VisualStudio在Host的进程当中运行当前应用程序；红色的部分是从**Application.Main函数开始执行**，经过几个函数到达**Dispatcher.Run()**，最后到达**Dispather.PushFrameInpl()**方法。那么一个Application在Run之后,为什么要调用**Dispatcher.Run()**呢，他做了些什么事情你？如果通过**Reflector仔细查看Application.Run()，**你会发现里面实际起作用的代码并不多，**最后都是Dispatcher.Run在做事情**。那么一个Application启动之后，**按照以前对Win32的消息机制的理解，当应用程序启动后，必须进入消息循环，对于WPF，也是一样的**。那么WPF应用程序是在什么地方进入消息循环呢？其实这就是Dispatcher.Run()做的事情。

​	查看上图最后一步Dispacther.PushFrameImpl()的代码，你会看到有下面的一段代码：

![clip_image004](201012301026044240.jpg)

​	很明显，橙色的部分是一个循环，看起来是不是很眼熟，跟Win32编程碰到的消息循环是否很像？对了，**这就是WPF应用程序进入了消息循环**。循环调用**GetMessage方法从当前线程的消息队列当中不停的获取消息**，取出一个msg之后，交给TranslateAndDispatchMessage方法Dispatch到不同的窗口过程去处理。这样以来，任何需要应用程序处理的消息通过这个过程，被不同的窗口处理了，应用程序就动起来了。

​	其实这一点和js的 event loop机制很像，它们都遵循了同一种基础的、非常重要的计算机科学模型——**“事件驱动的单线程并发模型”**

，它们都通过一个**任务队列来避免多线程直接操作共享资源（UI 或 DOM）所带来的混乱**。因为上面也说过了，WPF 作为一个构建在 Windows 操作系统之上的框架，它的所有窗口和用户输入都必须基于底层的 Win32 消息队列，**WPF是需要和windows的消息队列打交道**，         **WPF 的 `Dispatcher` 可以看作是对这个底层 Win32 消息队列的一个更高级、更强大、带有优先级的 .NET 封装。**



​	`DispatchMessage(&msg)` 将一个底层的 **Win32 消息**（比如 `WM_LBUTTONDOWN`，鼠标**左键按下**）发送给**对应的 WPF 窗口**。然后呢，WPF 内部的互操作层 (`HwndSource`) 接收到这个 Win32 消息。它将这个底层消息**翻译**成一个高级的 WPF 事件（比如 `Button.Click` 事件）。然后，它将这个事件的执行，作为一个任务项，**提交**到 UI 线程自己的 `Dispatcher` **优先级队列**中。消息循环继续，`Dispatcher` 在合适的时机（根据优先级）从队列中取出这个任务并**执行**它（例如，调用您在 C# 中写的 `Button_Click` 事件处理代码）。

![image-20250912070811524](image-20250912070811524.png)

### 顶级渲染事件

​	我这里再说一下顶级渲染事件的关系，`CompositionTarget.Rendering` 是 WPF 渲染引擎对外暴露的一个**“心跳”事件**。**定义**: 它是一个**静态事件**，会在 WPF **即将渲染（绘制）一帧新画面之前**被触发。

**频率**: 它的触发频率与您的**屏幕刷新率**（通常是 60Hz，即每秒约 60 次）以及系统的当前负载保持同步。

**线程**: 它总是在**主 UI 线程**上被触发。

​	`CompositionTarget.Rendering` 事件可以被理解为在 `Dispatcher` 队列中**拥有极高优先级的事件**。我们知道，WPF 的 UI 线程由一个 `Dispatcher` 来管理，`Dispatcher` 像一个任务队列，并按优先级处理任务。

**`Dispatcher` 的优先级从高到低大致如下：**

1. `Send` (内部使用)
2. `Input` (处理键盘、鼠标输入)
3. `Loaded` (处理控件加载事件)
4. **`Render` (处理布局和渲染)** `← CompositionTarget.Rendering` 在这里触发
5. `DataBind` (处理数据绑定更新)
6. `Background` (普通后台任务)
7. `Idle` (系统空闲时)

​	`CompositionTarget.Rendering` 事件的触发，是 `Dispatcher` 在处理其**`Render` 优先级**任务时的一个核心环节。

​	这意味着，当 `CompositionTarget.Rendering` 事件触发时，UI 线程会**优先处理**您的 `CompositionTarget_Rendering` 方法。执行您的代码是渲染流程的一部分，它的优先级高于普通的数据绑定、后台任务等。

​	除了这个高优先级的事件以外，开发者还有很多经常使用的事件，

**生命周期事件 (Lifecycle Events)**，这类事件与控件或窗口的“生老病死”有关，让您可以在正确的时机执行初始化或清理代码。

**用户输入事件 (User Input Events)**这类事件是应用程序响应用户操作的核心。

**路由事件 (Routed Events)**WPF 的**大多数输入事件都是“路由事件”**，这个机制允许您在父控件上拦截和处理子控件的事件，非常强大。它们有两种传播方式：

​	**隧穿 (Tunneling)**: 事件从顶层元素（窗口）**向下**传播到鼠标所在的具体元素。这类事件通常以 `Preview` 开头，例如 `PreviewMouseLeftButtonDown`。

​	**冒泡 (Bubbling)**: 事件从鼠标所在的具体元素**向上**传播到顶层元素（窗口）。这就是常规的事件，例如 `MouseLeftButtonDown`。

**键盘事件**:

- **`KeyDown` / `KeyUp`**: 底层的键盘按下/抬起事件。可以捕获所有按键，包括功能键（如 F1, Ctrl, Shift）。
- **`TextInput`**: 更高级的文本输入事件。它只在**可显示的字符**被输入时触发，并且能正确处理输入法（IME）的输入。**对于获取文本输入，应优先使用 `TextInput`** 而不是 `KeyDown`。

​	让我们来完整地追踪一次鼠标左键点击 `Button` 的过程，看看这两个阶段是如何衔接的：**Windows 内核**: 内核根据 `(X, Y)` 坐标和当前窗口的 Z-order（层叠顺序），判断出鼠标指针**正下方的是您的 WPF 应用程序的窗口**。它知道了这个窗口的**句柄 (HWND)**。**Win32 消息队列**: Windows 系统将这个点击事件打包成一个**窗口消息**（例如 `WM_LBUTTONDOWN`），然后将这个消息“**投递**”到创建了该窗口的**线程的消息队列**中。

![image-20250912162416024](image-20250912162416024.png)

### **数据与属性变化事件 (Data & Property Change Events)**

这类事件是实现动态 UI 和 MVVM 模式的关键。

- **`SelectionChanged`**
  - **触发对象**: `ListBox`, `ComboBox`, `DataGrid`, `TabControl` 等所有继承自 `Selector` 的控件。
  - **何时触发**: 当用户选择了一个新的项目时。
  - **典型用途**: 根据用户的选择，更新界面的其他部分。例如，在一个列表中选择一个用户，右侧显示该用户的详细信息。
- **`TextChanged`**
  - **触发对象**: `TextBox`, `RichTextBox` 等。
  - **何时触发**: 文本框中的内容发生任何改变时。
  - **典型用途**: 实现搜索框的实时搜索/筛选功能、统计输入字数、实时验证输入内容等。
- **`INotifyPropertyChanged.PropertyChanged`**
  - **这不是一个路由事件，而是 MVVM 模式的“心脏”**。当您在 ViewModel 中实现 `INotifyPropertyChanged` 接口后，每当一个属性的值发生变化并调用 `OnPropertyChanged()` 时，就会触发 `PropertyChanged` 事件。WPF 的数据绑定系统会**订阅**这个事件，一旦接收到通知，就会自动更新绑定到该属性的 UI 界面。

我们来看一个数据与属性的变化事件的案例：

```csharp
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
```

​	这个是WPF-UI的导航栏的控制的实现。`OnNavigationSelectionChanged` 确实是一个响应“数据变化”（被选项 `SelectedItem` 变化）的事件，它属于我们昨天讨论的**“数据与属性变化事件”**大类。它在功能层面与 `ListBox` 的 `SelectionChanged` 事件非常相似

来看一下两个行参：第一个就是:它指的是**触发这个事件的对象**，在代码里就是那个 `NavigationView` 控件**实例**。

​	RoutedEventArgs e：**`e` 是一个包含了与该事件相关的所有**附带信息**的对象（例如 `e.Source` 指向**最初触发事件的元素**，`e.Handled` 用于**标记事件是否已被处理**等）。

​	然后我们来讲一下同步上下文的基本操作，提供在各种同步模型中传播同步上下文的基本功能。同步上下文的工作就是**确保调用在正确的线程上执行**。**`Current` 获取当前同步上下文**       

> [!NOTE]
>
> 当一个 UI 线程（WPF 或 Windows Forms）启动时，.NET 框架会自动在这个线程上“注册”一个与之关联的`SynchronizationContext` 实例。在 WPF 中，这个实例是 `DispatcherSynchronizationContext`。



var context = SynchronizationContext.Current;

**`Send` 一个同步消息调度到一个同步上下文。** 

```csharp
SendOrPostCallback callback = o =>
                                 {
                                     //TODO:
                                 };
context.Send(callback,null);
```

send调用后会阻塞直到调用完成。**Post 将异步消息调度到一个同步上下文。** 

```csharp
SendOrPostCallback callback = o =>
                                {
                                      //TODO:
                                };
context.Post(callback,null);
```

和`send`的调用方法一样，不过`Post`会启动一个线程来调用，不会阻塞当前线程。

**使用同步上下文来更新UI内容** 

无论`WinFroms`和`WPF`**都只能用UI线程来更新界面的内容** 常用的调用UI更新方法是`Inovke`（WinFroms）：

```csharp
private void button_Click(object sender, EventArgs e)
{
       ThreadPool.QueueUserWorkItem(BackgroudRun);
}

private void BackgroudRun2(object state)
{
            this.Invoke(new Action(() =>
                                       {
                                           label1.Text = "Hello Invoke";
                                       }));
}

```

​	使用同步上下文也可以**实现相同的效果**，WinFroms和WPF继承了`SynchronizationContext`，使**同步上下文**能够在UI线程或者`Dispatcher`线程上正确执行。顺便说一下：**由 UI 元素触发的事件处理器（如 `Button_Click`）默认是在 UI 线程上被调用的**。  调用到后台线程中去有两个比较好的办法，一个是使用await Task.Run的方式抛到后台线程，

```csharp
System.Windows.Forms. WindowsFormsSynchronizationContext
System.Windows.Threading. DispatcherSynchronizationContext
```

​	调用方法如下：这段代码，是一段非常经典的、在 `async/await` 出现之前，用于**从后台线程安全地更新 UI 线程**的示例，它的核心是利用了 .NET 中一个名为 **`SynchronizationContext` (同步上下文)** 的强大抽象机制。

```csharp
private void button_Click(object sender, EventArgs e)
{
           var context = SynchronizationContext.Current; //捕获当前线程（UI 线程）的‘专属信使
           Debug.Assert(context != null);
           ThreadPool.QueueUserWorkItem(BackgroudRun, context); //启动一个后台任务，并将我们刚刚捕获的‘信使’作为参数传递过去。会从 .NET 的线程池中取出一个后台线程，并让它执行 BackgroudRun 这个方法。我们通过 state 参数，巧妙地将 UI 线程的“信使”(context) 对象，传递给了这个即将在后台运行的方法。
}

private void BackgroudRun(object state)
{
    var context = state as SynchronizationContext; //传入的同步上下文，后台线程现在拥有了一个可以与 UI 线程沟通的“联络工具”。
    Debug.Assert(context != null);
    SendOrPostCallback callback = o => //创建一个‘工作任务便签’ (callback)。”
                                      {
                                          label1.Text = "Hello SynchronizationContext";
                                      };
    //SendOrPostCallback 是一个委托类型。我们在这里定义了一个匿名方法，它包含了我们真正想在 UI 线程上执行的操作：label1.Text = "Hello SynchronizationContext";。
    context.Send(callback,null); //调用,后台线程通过‘信使’，同步地将任务派发给 UI 线程。后台线程把这张“便签” (callback) 交给了 UI 线程的“信使” (context)，并对他说：“请你立即把这个任务送到你的主人（UI线程）那里去执行，我在这里等着，直到你确认他做完了我才继续。”
}

```



### **`Send` vs. `Post`：同步与异步的区别**

`SynchronizationContext` 提供了两种“派发”任务的方式：

- **`Send` (同步)**:
  - 相当于 `Dispatcher.Invoke`。
  - **阻塞**调用线程，等待任务在**目标线程完成**。
- **`Post` (异步)**:
  - 相当于 `Dispatcher.BeginInvoke`。
  - **不阻塞**调用线程，把任务“投递”到目标线程的队列后就立刻返回，是“即发即忘”。



### **与 `async/await` 的关系**

​	现在看到的这个手动捕获 `context` 并调用 `Send` 或 `Post` 的过程，**正是 `async/await` 关键字在 UI 线程上为我们自动完成的工作！**

当您在一个 UI 线程的方法中 `await` 一个任务时：

1. C# 编译器会自动生成代码来捕获 `SynchronizationContext.Current`。
2. 当 `await` 的后台任务完成后，编译器会自动调用这个 `context` 的 `Post` 方法。
3. 它将 `await` 之后的所有代码打包成一个回调，安全地送回 UI 线程继续执行。

​	所以，`async/await` 是这个底层模式的一个**极其优雅的语法糖 (Syntactic Sugar)**，它让开发者无需再手动管理 `SynchronizationContext`，极大地简化了异步 UI 编程。



### Dispatcher.BeginInvoke

​	这是 WPF 线程模型中一个非常核心且常用的方法。它是在**后台线程**和 **UI 线程**之间进行**异步通信**的关键桥梁。**一句话总结**：`Dispatcher.BeginInvoke` 的作用是，将一个方法（委托）**异步地**安排到 **UI 线程的任务队列中去执行**，而**不阻塞**当前正在执行的后台线程。

这个过程就是“**异步**”和“**非阻塞**”的，我们通常称之为**“即发即忘” (Fire and Forget)** 模式。

------



### **`BeginInvoke` vs. `Invoke`：关键区别**

理解 `BeginInvoke` 最好的方式就是将它与它的“兄弟”方法 `Invoke` 进行对比。

| 特性         | `Dispatcher.Invoke` (同步)                                   | `Dispatcher.BeginInvoke` (异步)                              |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **执行方式** | **同步 (Synchronous)**                                       | **异步 (Asynchronous)**                                      |
| **调用线程** | **被阻塞 (Blocks)**。后台线程会暂停，直到 UI 线程执行完该任务。 | **不被阻塞 (Does not block)**。后台线程提交任务后，立即继续执行自己的代码。 |
| **返回值**   | 如果委托有返回值，`Invoke` 会返回那个**具体的值**。          | 返回一个 `DispatcherOperation` 对象，代表排队中的任务状态。  |
| **比喻**     | **打电话**。您必须在线等待对方给您答复。                     | **发邮件/短信**。您发送后就可以去做别的事，对方有空时会处理。 |
| **适用场景** | 当后台线程**需要**从 UI 元素获取一个值，或者**必须等待** UI 更新完成后才能继续下一步时。 | **最常用**。当您只需要从后台更新 UI（如更新文本、进度条），而不需要等待其完成时。 |

------



### **代码示例**

​	下面的代码完美地展示了 `BeginInvoke` 的典型用例：在后台任务中**更新进度条**。

```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }

    private void StartWorkButton_Click(object sender, RoutedEventArgs e)
    {
        // 启动一个后台任务
        Task.Run(() =>
        {
            for (int i = 1; i <= 100; i++)
            {
                // 模拟耗时的工作
                Thread.Sleep(50);

                // 使用 BeginInvoke 异步更新 UI
                // 后台线程把更新任务“扔”给 Dispatcher 后，会立刻开始下一次循环
                // 而不会等待进度条真的在屏幕上重绘完成
                this.Dispatcher.BeginInvoke(System.Windows.Threading.DispatcherPriority.Normal, 
                    new Action(() =>
                    {
                        MyProgressBar.Value = i;
                        MyTextBlock.Text = $"进度: {i}%";
                    }));
            }
            
            // 所有任务提交完毕后，再提交一个最终的完成消息
            this.Dispatcher.BeginInvoke(() => { 
                MessageBox.Show("任务完成！");
            });
        });
    }
}
```

**工作流程**:

1. **View (XAML)** 的控件属性（如 `Text="{Binding ProgressText}"`）绑定到 **ViewModel** 的一个公开属性 (`public string ProgressText { ... }`)。
2. **后台线程**去修改 ViewModel 的 `ProgressText` 属性的值。
3. ViewModel 属性的 `set` 访问器中，会调用 `OnPropertyChanged("ProgressText")` 来触发 `PropertyChanged` 事件。
4. **WPF 的数据绑定引擎**订阅了这个事件，收到“提醒”后，它会自动在 **UI 线程**上更新 `TextBlock` 的 `Text` 属性。

在这个模式中，`PropertyChanged` 事件就是您所说的那个必不可少的**“提醒”**。它是连接**数据模型 (ViewModel)** 和**视图 (View)** 的桥梁。

但是这里使用的是后台代码操作，也就是winforms的时候常见的Code-Behind模式。

**工作流程**:

1. 后台线程并不关心任何 ViewModel 或数据模型。
2. 它通过 `Dispatcher.BeginInvoke`，将一个**直接操作 UI 元素**的 `Action` 委托，排队到 UI 线程。
3. UI 线程的 `Dispatcher` 在轮到这个任务时，执行 `Action` 里的代码，也就是下面这两行：

​	为什么这里不需要 `PropertyChanged` 这样的“提醒”？答案是：在这段代码里，**我们没有通过数据模型来间接更新 UI，而是直接获得了 UI 元素本身 (`MyProgressBar`, `MyTextBlock`) 的控制权，并直接修改了它们的属性。**

​	WPF 框架的**依赖属性 (Dependency Property) 系统在内部知道**，当 `Text` 属性（或 `ProgressBar.Value` 属性）发生变化时，这个控件的外观就需要被更新。因此，它会自动将一个“**重绘**”或“**重新布局**”的任务安排到 `Dispatcher` **更高优先级的渲染队列**中。

所以，这里的“提醒”是**隐式的、内建于 WPF 属性系统**的。您对 UI 元素属性的**直接赋值动作本身**，就是**通知 WPF 去更新 UI 的方式**。