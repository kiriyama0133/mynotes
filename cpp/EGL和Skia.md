
[OpenGL](https://www.opengl.org/) 是一种跨平台的图形 API，用于为 3D 图形处理硬件指定标准的软件接口。[OpenGL ES](https://www.khronos.org/opengles/) 是 OpenGL 规范的子集，适用于嵌入式设备。而EGL 是**渲染 API（如 OpenGL ES）和原生窗口系统之间的接口**.通常来说，OpenGL 是一个操作 GPU 的 API，它通过驱动向 GPU 发送相关指令，控制图形渲染管线状态机的运行状态，但是当涉及到与**本地窗口系统进行交互时**，就需要这么一个中间层，且它最好是与平台无关的,

因此 EGL 被设计出来，作为 OpenGL 和原生窗口系统之间的桥梁：![[Pasted image 20251231232401.png]]

EGL API 是**独立于 OpenGL 各版本标准的独立的一套 API**，其主要作用是**为 OpenGL 指令 创建上下文 Context 、绘制目标 Surface 、配置 FrameBuffer 属性、Swap 提交绘制结果等。

EGL提供如下机制：
- 与设备的原生窗口系统通信
- 查询绘图表面的可用类型和配置
- 创建绘图表面
- 在 OpenGL ES 和其他图形渲染 API 之间同步渲染
- 管理**纹理贴图等渲染资源**

## EGL的三大支柱

下面这张图可简要看出EGL的接口能力：![[Pasted image 20251231232442.png]]

如果我们想要在安装上使用opengl es，我们可以直接用GLSurfaceView来进行Opengl的渲染，因为GLSurfaceView内部给我们**封装好了EGL环境和渲染线程**。

![[Pasted image 20251231232801.png]]


EGL 的主要任务是**管理渲染所需的资源**，Disyplay是显示连接的功能，可以为我们提供获取显示设备的句柄（Windows上是HDC）， 初始化显示连接等。

Surface是OpenGL画图的地方，本质是一个内存缓冲区，他可以将EGL和本地窗口通过我们在GLFW拿到的HWND绑定在一起，`eglCreatePBufferSurface()`甚至可以创建一个离屏缓冲区（不显示在屏幕上）。

EGL可以利用一个渲染上下文，存储所有opengel状态的容器，比如说当前使用的着色器、纹理、顶点数据等，**EGL是创建OpenGL ES渲染上下文所必需的**。这个上下文必须连接到合适的表面才能开始渲染。
## 单线程模型

EGL是**单线程模型**的，也就说说EGL环境的创建、渲染操作、EGL环境的销毁都必须在同一个线程内完成，否则是无效的。我们可以通过共享EGL上下文来做多多线程渲染。

EGL本身其实是支持多线程的，但是他采用了一个叫做 **线程局部** 的绑定模型，这种模型确实决定了在大多数 UI 框架（如 GLFW + ANGLE 项目）中，**将 UI 逻辑与渲染逻辑保持在同一个线程是最稳妥、最高效的选择**。

>[!tip] 顺带一提，UI线程单线程话也是工业界普遍的做法，比如说安卓的UI线程，或者说Windows的消息循环。
>
>**窗口句柄的线程依赖**：底层的 **Native Window System**（如 GLFW 创建的窗口）通常具有线程亲和性。在 Windows 上，只有创建窗口的线程才能可靠地接收和处理该窗口的消息。
>**Surface 的同步**：**Surface (S)** 是连接 OpenGL 和本地窗口的纽带。如果渲染在线程 B，而窗口缩放（DPI 变化）在线程 A 处理，那么在调用 `eglSwapBuffers` 时，可能会出现缓冲区大小不匹配导致的闪烁或崩溃。

在我们使用EGL的时候，我们会去建立与GPU的连接（以便以Display的获取）

## 渲染过程

### 建立GPU连接

这是整个流程的第一步。如果使用的是 ANGLE 扩展，不能使用标准的 `eglGetDisplay`，而必须使用 `eglGetPlatformDisplayEXT`，我们可以通过 `displayAttributes` 指定 ANGLE 使用 **D3D11** 作为底层驱动。

我们一般来说，会使用pimpl设计模式，做一个渲染环境内部的封装器，将复杂的gpu状态管理隐藏在外部类外。
```cpp
    struct RenderContext::Impl {

        EGLDisplay display_ = EGL_NO_DISPLAY; // EGL display

        EGLConfig config_ = nullptr; // EGL config

        EGLContext context_ = EGL_NO_CONTEXT; // EGL context

        sk_sp<GrDirectContext> skiaContext_ = nullptr;

    };
```
display_就是代表着和物理显示器的连接在windows上，他通过ANGLE代理了与D3D11的设备的交互，config_则是代表配置好的画布属性（像素格式、位深、采样率等）。

context_是最重要的GPU的状态机，存储了所有的着色器、纹理和渲染状态，是绘图命令执行的核心环节，可以用来接收GLES的指令。

`sk_sp<GrDirectContext> skiaContext_` 是Skia 库的 GPU 上下文入口，它封装了底层的 OpenGL ES 调用，负责管理 GPU 资源（如纹理缓存）。

### ANGEL的初始化


```cpp
    bool RenderContext::Initialize() {
        auto eglGetPlatformDisplayEXT = reinterpret_cast<PFNEGLGETPLATFORMDISPLAYEXTPROC>(eglGetProcAddress("eglGetPlatformDisplayEXT"));

        const EGLint displayAttributes[] = {

            EGL_PLATFORM_ANGLE_TYPE_ANGLE, EGL_PLATFORM_ANGLE_TYPE_D3D11_ANGLE,

            EGL_PLATFORM_ANGLE_ENABLE_AUTOMATIC_TRIM_ANGLE, EGL_TRUE,

            EGL_NONE

        };
```

它通过反射和属性子带你来配置显卡的驱动的运行模式，`eglGetPlatformDisplayEXT` 并不是标准 EGL 1.4 的函数，而是一个**扩展 (Extension)**。 这组数组定义了你希望 ANGLE 如何在你的 Windows 系统上工作：
```
const EGLint displayAttributes[] = {
    EGL_PLATFORM_ANGLE_TYPE_ANGLE, EGL_PLATFORM_ANGLE_TYPE_D3D11_ANGLE,
    EGL_PLATFORM_ANGLE_ENABLE_AUTOMATIC_TRIM_ANGLE, EGL_TRUE,
    EGL_NONE
};
```
**`EGL_PLATFORM_ANGLE_TYPE_D3D11_ANGLE`**：

- **核心作用**：强制 ANGLE 使用 **Direct3D 11** 作为翻译目标。
    
- **意义**：这意味着你写的 OpenGL ES 指令最终会被**转化为 D3D11 指令运行**。在 Windows 上，D3D11 的驱动支持通常比原生 OpenGL 更好、更稳定。
**`EGL_PLATFORM_ANGLE_ENABLE_AUTOMATIC_TRIM_ANGLE`**：

- **核心作用**：开启**自动内存释放（Trim）**。
    
- **意义**：当**窗口最小化或程序进入后台时**，ANGLE 会**自动释放一些占用的 GPU 内存资源**。这对于桌面应用程序的性能优化非常重要。
**`EGL_NONE`**：

- **作用**：这是一个终止符，告诉 EGL 这个属性列表结束了。


创建显示连接：
```cpp
 impl_->display_ = eglGetPlatformDisplayEXT(EGL_PLATFORM_ANGLE_ANGLE,

            EGL_DEFAULT_DISPLAY,

            displayAttributes);
```
它接收了 `EGL_PLATFORM_ANGLE_ANGLE`（告诉它我要用 ANGLE 平台）和刚才定义的一系列属性，会去返回一个 `EGLDisplay` 句柄。这一步成功执行后，你的程序就正式与显卡驱动建立了一条**基于 D3D11 后端的渲染链路**。

EGL的握手和版本协商阶段：

```cpp
        // Initialize EGL
        EGLint major, minor;
        if (!eglInitialize(impl_->display_, &major, &minor)) {
            std::cerr << "RenderContext: Failed to initialize EGL" << std::endl;
            impl_->display_ = EGL_NO_DISPLAY;
            return false;
        }
```

如果这里系统环境存在了问题，比如说机器显卡不支持D3D11，库文件确实，或者某些沙箱环境下的权限问题（无法访问显卡硬件）。

### 配置绘图表面

然后我们去配置绘图表面
```cpp
        // Configure surface attributes
        const EGLint configAttributes[] = {
            EGL_RED_SIZE, 8, EGL_GREEN_SIZE, 8, EGL_BLUE_SIZE, 8, EGL_ALPHA_SIZE, 8,
            EGL_DEPTH_SIZE, 8, EGL_STENCIL_SIZE, 8,
            EGL_RENDERABLE_TYPE, EGL_OPENGL_ES3_BIT,
            EGL_NONE

        };
```
我们在这里定义了一组约束条件，从开始分别是RGBA8888格式的颜色通道
辅助缓冲区（**深度缓冲区，用于判断物体的前后遮挡关系（Z-buffer）。虽然 UI 通常是 2D 的，但如果你想做 3D 效果或复杂的层级叠加，它非常有用**）  
**渲染能力**（这里配置的必须要运行OpenGL ES3.0，如果只选了一个ES2.0则后面的`eglCreateContext` 就会报错。）

配置筛选好匹配环节：

```cpp
 // Choose EGL configuration

        EGLint numConfigs;
        if (!eglChooseConfig(impl_->display_, configAttributes, &impl_->config_, 1, &numConfigs) || numConfigs == 0) {
            std::cerr << "RenderContext: Failed to choose EGL config" << std::endl;
            eglTerminate(impl_->display_);
            impl_->display_ = EGL_NO_DISPLAY;
            return false;
        }
```

**`configAttributes`（查询条件）**：传入你刚才定义的 RGBA8888、Stencil 8 等硬性指标，**`&impl_->config_`（接收缓冲区）**：这是一个输出参数。如果**找到了匹配的配置**，驱动会把这个配置的**句柄**存入这里。
**`1`（最大获取数量）**：你告诉 EGL：“我只需要最匹配的那 **1** 个。” 实际上显卡可能返回几十个符合条件的配置，但通常拿第一个（排名最高、性能最稳定）就够了。
**`&numConfigs`（实际找到的数量）**：这也是一个**输出参数**。它会告诉你显卡里到底有多少个配置符合你的要求。![[Pasted image 20260101005902.png]]

下面**设置dspplay_ = EGL_NO_DISPLAY是为了清理资源**，如果不设置的话，可能外部逻辑会误以为初始化已经部分成功，会导致后续的调用失败。

### 两个上下文的配置
绑定上下文

```cpp
        // Bind OpenGL ES API

        if (!eglBindAPI(EGL_OPENGL_ES_API)) {

            std::cerr << "RenderContext: Failed to bind OpenGL ES API" << std::endl;

            eglTerminate(impl_->display_);

            impl_->display_ = EGL_NO_DISPLAY;

            return false;

        }
```
在 EGL 的流程中，`eglBindAPI` 是一个经常被开发者忽略但在**多 API 架构**中至关重要的“声明”步骤。
这一步的作用是**明确告知 EGL 状态机：接下来的所有操作（尤其是创建 Context）都是针对哪种图形 API 的**。

我们在下面创建一个EGL上下文，设置使用的opengl es版本为3.0：

```
        // Create EGL context

        const EGLint contextAttributes[] = {

            EGL_CONTEXT_CLIENT_VERSION, 3, // 使用 OpenGL ES 3.0

            EGL_NONE

        };
```

通过context_  赋值，eglCreateContext(impl_->display_, impl_->config_, EGL_NO_CONTEXT, contextAttributes); 来接受它的返回值。

如果上下文的创建也没有问题，我们就可以开始进行上下文的激活，eglMakeCurrent(display, draw_surface, read_surface, context)
![[Pasted image 20260101012056.png]]


这里我们将EGL上下文设置成了当前现场的活动上下文，注意这里使用了EGL_NO_SURFACE，因为此时没有窗口，窗口是由其他对象创建的额，而创建skia上下文就只需要激活上下文而不需要绘制表面，**我们后续在把它绑定到surface**。

有了上下文，我们需要创建GL接口，通过GL接口我们可以连接EGL和skia，这样skia就可以去调用opengl es的函数了。

 GrGLMakeAssembledInterface 的作用，这是 Skia 提供的函数，用于创建 GL 接口：

- 输入：一个函数指针获取器（lambda）

- 输出：GL 接口对象（包含所有 OpenGL 函数指针）

![[Pasted image 20260101012355.png]]

skia可以调用opengl函数之后，我们就可以去创建skia gpu渲染上下文，也就是使用之前创建的GL接口去创建skia的gpu渲染上下文，
```cpp
        impl_->skiaContext_ = GrDirectContexts::MakeGL(interface);

        if (!impl_->skiaContext_) {

            std::cerr << "RenderContext: Failed to create Skia GL context" << std::endl;

            return false;

        }
```
GrDirectContext 是 Skia 的 GPU 渲染上下文，可以管理GPU资源（纹理和缓冲区），可以执行GPU渲染命令，可以提供skia的GPU加速渲染能力。

以上的完整初始化链条：
EGL Context (OpenGL ES 3.0 上下文)
    ↓
GL Interface (OpenGL 函数指针集合)
    ↓
Skia Context (Skia GPU 渲染上下文) ← 你问的这段
    ↓
可以创建 Skia Surface 进行绘制


 数据流和关系，从 EGL 到 Skia 的完整流程
 1. EGL Display (第 41 行)
   ↓
2. EGL Context (第 95 行)
   ↓
3. 激活 Context (第 106 行)
   ↓
4. GL Interface (第 118 行) ← 提供 OpenGL 函数指针
   ↓
5. Skia Context (第 127 行) ← 你问的这段
   ↓
6. 可以使用 Skia 进行 GPU 渲染

## 画布的初始化

我们有了上下文的初始化之后，就可以去创建一个具体的画布，并通过GLFW创建的窗口来绑定，这样就可以在窗口上来绘制东西。

我们和之前的设计思路是一样的，我会在内部实现一个结构体impl：

```cpp
struct RenderSurface::Impl {

    boost::shared_ptr<RenderContext> context_;

    EGLSurface eglSurface_ = EGL_NO_SURFACE;

    sk_sp<SkSurface> skSurface_;

    int width_ = 0;

    int height_ = 0;

    bool initialized_ = false;

    SkCanvas* currentCanvas_ = nullptr;

};
```
分别代表了渲染环境的持有，context_持有了对于刚才创建的上下文的引用，因为多个窗口可能共享一个上下文，因此这里使用共享指针，只要还存在一个窗口，那么渲染环境就不会被销毁。

eglSurface是一个GPU窗口表面，这就是EGL架构图中的surface(s)，他表示了显卡驱动管理的特定窗口的前后台缓冲区。

skSurface_是skia的绘图表面，提供了高级的绘图接口，可以用于实现路径绘画、文字和图片等。用 `sk_sp`（Skia 的智能指针）来**管理生命周期**。每当窗口缩放（Resize）时，这个对象会被重新创建以匹配新的像素尺寸。

我们在画布这里需要从渲染环境那里去接受核心的GPU管理句柄：

```cpp
    // 获取 EGL 原生句柄

    auto nativeHandles = impl_->context_->GetNativeHandles();

    EGLDisplay display = static_cast<EGLDisplay>(nativeHandles.display);

    EGLConfig config = static_cast<EGLConfig>(nativeHandles.config);

    if (display == EGL_NO_DISPLAY || config == nullptr) {

        std::cerr << "RenderSurface: Invalid EGL handles from RenderContext" << std::endl;

        return false;

    }
```
我们通过了之前的方法建立了GPU通道，我们这里的Surface负责将它应用到特定的窗口上，我们通过调用
```cpp
    RenderContext::NativeHandles RenderContext::GetNativeHandles() const {
        RenderContext::NativeHandles handles;
        handles.display = nullptr;
        handles.context = nullptr;
        handles.config = nullptr;
        if (impl_) {
            handles.display = impl_->display_;
            handles.context = impl_->context_;
            handles.config = impl_->config_;
        }
        return handles;
    }
```
获取了已经初始化好了的EGLdisplay和EGLconfig。为了保持接口的通用性，可能句柄包装成了void*，我们将其还原为EGL的标准类型，所以用了类型转换static_cast。

我们接下来去创建surface，将窗口句柄给转换为EGL表面，
```cpp
    // 创建 EGLSurface（将窗口句柄转换为 EGL 表面）
    EGLint surfaceAttributes[] = {
        EGL_NONE
    };
    // 创建 EGLSurface
    impl_->eglSurface_ = eglCreateWindowSurface(display, config, windowHandle, surfaceAttributes);
    if (impl_->eglSurface_ == EGL_NO_SURFACE) {
        EGLint error = eglGetError();
        std::cerr << "RenderSurface: Failed to create EGL surface, error: 0x"
                  << std::hex << error << std::dec << std::endl;
        return false;
    }
```
display就是刚才初始化了的gpu连接，config就是刚才的画布规格，windowHandle就是我们拿到的窗口句柄，surfaceAttributes我们设置EGL_NONE,会使用默认行为。

如果使用高级用法，1可以在这里制定渲染缓冲区行为，比如指向单缓冲还是双缓冲。

再确认了物理画布之后，我们需要建立线程 ---》 上下文 ----》 表面的绑定关系，将这一套环境正式交给skia图像引擎。

```cpp
    // 使 EGL 上下文成为当前上下文
    EGLContext context = static_cast<EGLContext>(nativeHandles.context);
    if (!eglMakeCurrent(display, impl_->eglSurface_, impl_->eglSurface_, context)) {
        EGLint error = eglGetError();
        std::cerr << "RenderSurface: Failed to make EGL context current, error: 0x"
                  << std::hex << error << std::dec << std::endl;
        eglDestroySurface(display, impl_->eglSurface_);
        impl_->eglSurface_ = EGL_NO_SURFACE;
        return false;

    }

    // 创建 Skia 表面（SkSurface）
    GrDirectContext* skiaContext = impl_->context_->GetSkiaContext();
    if (!skiaContext) {
        std::cerr << "RenderSurface: Skia context is null" << std::endl;
        eglDestroySurface(display, impl_->eglSurface_);
        impl_->eglSurface_ = EGL_NO_SURFACE;
        return false;
    }
```

`GrDirectContext` 是 Skia 指令发送给 GPU 的唯一通道，如果绑定失败或 Skia 上下文无效，代码会立即销毁刚刚创建的 `eglSurface_`。如果不手动调用 `eglDestroySurface`，即使 `RenderSurface` 对象被销毁，这块显存也可能一直被驱动占用，导致程序运行时间越长，系统显存占用越高。

最后我们需要去定义硬件帧的缓冲句柄：
```cpp
GrGLFramebufferInfo framebufferInfo;
framebufferInfo.fFBOID = 0;
```
在 OpenGL/EGL 标准中，ID 为 `0` 的帧缓冲，这是一个特殊的存在，它代表“默认的后台缓冲区”**——即直接连接到操作系统的窗口表面的那块内存。

这里我们设置为0让skia可以在这里绘制，这样绘制的东西会显示在窗口上。 

我们去构建渲染目标：
```cpp
GrBackendRenderTarget backendRenderTarget = GrBackendRenderTargets::MakeGL(
    impl_->width_, impl_->height_, 0, 0, framebufferInfo
);
```
这里是skia对外部GPU资源的抽象描述符号，这里我们组合了物理像素尺寸和刚才确定的FBO信息，这样显存就可以去接受skia的指令了。


```cpp
impl_->skSurface_ = SkSurfaces::WrapBackendRenderTarget(
    recordingContext,
    backendRenderTarget,
    kBottomLeft_GrSurfaceOrigin, // 坐标系适配
    kRGBA_8888_SkColorType,      // 像素类型
    nullptr, nullptr
);
```
因为OpenGL的坐标点在左下角，skia通过这个参数就可以自动完成坐标系的镜像反转，让我们写UI逻辑更加符合直觉，这里调用的就是skia的顶级api，他不会分配新的显存，而是会接管EGL的缓冲区。 一旦skSurface_创建成功了，我们就可以拿到它的SKCanvas，从而开启高性能的绘图。

