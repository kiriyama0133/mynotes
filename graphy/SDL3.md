---
date: 2026-10-17
tags:
  - cpp
  - sdl
---
SDL即Simple DirectMedia Layer，类似DirectX，是完整的游戏、多媒体开发包，但不同的是它跨越几乎所有的平台，有各种语言的接口，多种语言的文档，而这一切都是广大志愿者完成的。目前扩展部分还没有正式的文档，以下为核心部分文档的向导部分。

## 主要架构

SDL 的全部工作就是「把平台差异挡在下面，给你一套统一函数」。所以它的架构是纵向切开的四层：

<svg width="100%" viewBox="0 0 680 438" role="img" xmlns="http://www.w3.org/2000/svg">
  <title>SDL3 分层架构</title>
  <desc>自上而下：你的应用、SDL3 公共 API 层、子系统层、后端驱动层，最后落到操作系统与硬件。</desc>
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>

  <rect x="230" y="40" width="220" height="40" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="340" y="60" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" font-weight="500" fill="#9FE1CB">你的应用（C / C++）</text>

  <line x1="340" y1="80" x2="340" y2="97" stroke="#B4B2A9" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="40" y="100" width="600" height="258" rx="20" fill="#2C2C2A" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="60" y="124" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#888780">SDL3 运行库（libSDL3 / SDL3.dll）</text>

  <rect x="60" y="142" width="560" height="48" rx="10" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="80" y="160" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" font-weight="500" fill="#9FE1CB">公共 API 层</text>
  <text x="80" y="180" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#5DCAA5">统一的 SDL_* 函数，头文件按子系统拆分</text>

  <rect x="60" y="200" width="560" height="64" rx="10" fill="#444441" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="80" y="222" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" font-weight="500" fill="#D3D1C7">子系统层</text>
  <text x="80" y="244" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#B4B2A9">Video · Audio · Events · Input · Threads · Timer · IO · GPU …</text>

  <rect x="60" y="274" width="560" height="64" rx="10" fill="#444441" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="80" y="296" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" font-weight="500" fill="#D3D1C7">后端驱动层</text>
  <text x="80" y="318" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#B4B2A9">win32 · cocoa · X11 · Wayland · KMSDRM · Android · iOS · Emscripten</text>

  <line x1="340" y1="358" x2="340" y2="375" stroke="#B4B2A9" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="230" y="378" width="220" height="40" rx="8" fill="#444441" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="340" y="398" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" fill="#D3D1C7">操作系统 / 硬件</text>
</svg>

驱动层是运行时选的，不是编译器写死，SDL_Init(SDL_INIT_VIDEO)会按照优先级探测，windows上先试d3d/dxgi再试win32；linux平台上先试wayland、x11、kmsdrm。可以通过SDL_HINT_VIDEODRIVER或者环境变量SDL_VIDEODRIVER=x11来强制制定。

子系统是可以裁剪的，静态编译的时候可以关掉不需要的模块比如不编译音频、手柄，这个是sdl可以塞进switch、ps2、机顶盒这类设备的原因。

纵向是「层次」，横向是「功能块」。SDL3 的头文件就是它的模块清单，一个头文件对应一个子系统：

<svg width="100%" viewBox="0 0 680 338" role="img" xmlns="http://www.w3.org/2000/svg">
  <title>SDL3 子系统分组</title>
  <desc>四个分组：显示与渲染、音频、输入、基础设施。高亮的是入门最先接触的模块。</desc>

  <rect x="40" y="40" width="290" height="132" rx="12" fill="#2C2C2A" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="56" y="66" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#D3D1C7">显示与渲染</text>
  <rect x="56" y="80" width="124" height="32" rx="8" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
  <text x="118" y="96" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#FAC775">Video</text>
  <rect x="190" y="80" width="124" height="32" rx="8" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
  <text x="252" y="96" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#FAC775">Renderer</text>
  <rect x="56" y="120" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="118" y="136" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">GPU</text>
  <rect x="190" y="120" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="252" y="136" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">Camera</text>

  <rect x="350" y="40" width="290" height="132" rx="12" fill="#2C2C2A" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="366" y="66" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#D3D1C7">音频</text>
  <rect x="366" y="80" width="124" height="32" rx="8" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
  <text x="428" y="96" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#FAC775">Audio Stream</text>
  <rect x="500" y="80" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="562" y="96" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">Audio Device</text>
  <rect x="366" y="120" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="428" y="136" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">WAV 加载</text>
  <rect x="500" y="120" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="562" y="136" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">格式转换</text>

  <rect x="40" y="186" width="290" height="132" rx="12" fill="#2C2C2A" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="56" y="212" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#D3D1C7">输入</text>
  <rect x="56" y="226" width="124" height="32" rx="8" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
  <text x="118" y="242" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#FAC775">Events</text>
  <rect x="190" y="226" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="252" y="242" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">Keyboard/Mouse</text>
  <rect x="56" y="266" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="118" y="282" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">Gamepad</text>
  <rect x="190" y="266" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="252" y="282" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">Haptic/Sensor</text>

  <rect x="350" y="186" width="290" height="132" rx="12" fill="#2C2C2A" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="366" y="212" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#D3D1C7">基础设施</text>
  <rect x="366" y="226" width="124" height="32" rx="8" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
  <text x="428" y="242" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#FAC775">Threads</text>
  <rect x="500" y="226" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="562" y="242" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">Timer</text>
  <rect x="366" y="266" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="428" y="282" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">IOStream</text>
  <rect x="500" y="266" width="124" height="32" rx="8" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="562" y="282" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#9FE1CB">Properties</text>
</svg>

**几块正式模块**（对应 `include/SDL3/` 下的头文件）：

| 组   | 头文件                                                                                                  | 干什么                                                       |
| --- | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| 视频  | `SDL_video.h`                                                                                        | 窗口、显示器、HiDPI、全屏、OpenGL/Vulkan/Metal/D3D 上下文               |
| 渲染  | `SDL_render.h`                                                                                       | 2D 绘制：纹理、矩形、线段、旋转、裁剪                                      |
| GPU | `SDL_gpu.h`                                                                                          | **新增**。跨后端 3D/计算，一套 API 跑 Vulkan / Metal / D3D12 / WebGPU |
| 事件  | `SDL_events.h`                                                                                       | 事件队列，是**所有输入的入口**                                         |
| 输入  | `SDL_keyboard.h` / `SDL_mouse.h` / `SDL_gamepad.h` / `SDL_joystick.h` / `SDL_pen.h` / `SDL_sensor.h` | 键鼠、手柄（统一映射）、压感笔、陀螺仪                                       |
| 音频  | `SDL_audio.h`                                                                                        | 设备枚举 + `SDL_AudioStream`（播放/录音流）                          |
| 线程  | `SDL_thread.h`                                                                                       | 线程、互斥量、信号量、原子操作                                           |
| IO  | `SDL_iostream.h` / `SDL_filesystem.h` / `SDL_asyncio.h`                                              | 替代 stdio 的统一读写；异步 IO 是新增                                  |
| 系统  | `SDL_timer.h` / `SDL_properties.h` / `SDL_log.h` / `SDL_dialog.h` / `SDL_camera.h`                   | 计时、属性系统、日志、文件对话框、摄像头（新增）                                  |

还有一批「冷门但真能用」的：`SDL_messagebox.h`（原生弹窗）、`SDL_clipboard.h`、`SDL_tray.h`（托盘图标，新增）、`SDL_power.h`（电量）、`SDL_locale.h`、`SDL_process.h`

## 入手顺序

<svg width="100%" viewBox="0 0 680 358" role="img" xmlns="http://www.w3.org/2000/svg">
  <title>SDL3 上手路径</title>
  <desc>四步递进：先搭窗口和事件循环骨架，再用 Renderer 绘制，然后接入音频，最后按需选一个方向深入。</desc>

  <rect x="40" y="40" width="600" height="64" rx="10" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <circle cx="70" cy="72" r="14" fill="#04342C" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="70" y="72" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#9FE1CB">1</text>
  <text x="96" y="64" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" font-weight="500" fill="#9FE1CB">开窗口 + 事件循环</text>
  <text x="96" y="86" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#5DCAA5">SDL_Init → SDL_CreateWindow → while (SDL_PollEvent)</text>

  <rect x="40" y="118" width="600" height="64" rx="10" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <circle cx="70" cy="150" r="14" fill="#04342C" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="70" y="150" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#9FE1CB">2</text>
  <text x="96" y="142" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" font-weight="500" fill="#9FE1CB">用 Renderer 画东西</text>
  <text x="96" y="164" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#5DCAA5">SDL_CreateRenderer → SDL_RenderTexture → SDL_RenderPresent</text>

  <rect x="40" y="196" width="600" height="64" rx="10" fill="#085041" stroke="#5DCAA5" stroke-width="0.5"/>
  <circle cx="70" cy="228" r="14" fill="#04342C" stroke="#5DCAA5" stroke-width="0.5"/>
  <text x="70" y="228" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#9FE1CB">3</text>
  <text x="96" y="220" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" font-weight="500" fill="#9FE1CB">接入音频</text>
  <text x="96" y="242" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#5DCAA5">SDL_OpenAudioDeviceStream + 回调里往流里写采样</text>

  <rect x="40" y="274" width="600" height="64" rx="10" fill="#633806" stroke="#EF9F27" stroke-width="0.5"/>
  <circle cx="70" cy="306" r="14" fill="#412402" stroke="#EF9F27" stroke-width="0.5"/>
  <text x="70" y="306" text-anchor="middle" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="13" font-weight="500" fill="#FAC775">4</text>
  <text x="96" y="298" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="14" font-weight="500" fill="#FAC775">选一个方向深入</text>
  <text x="96" y="320" dominant-baseline="central" font-family="system-ui, sans-serif" font-size="12" fill="#EF9F27">GPU API / Gamepad / Camera / 线程池 / 自定义平台后端</text>
</svg>

继续更新