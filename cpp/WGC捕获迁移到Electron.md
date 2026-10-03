
## 迁移

上个笔记里我们用了WINRT、D3D和glfw制作了一个WGC的窗口捕获程序，但是现在如果要把功能给迁移到electron应用中去的话，就得去掉glfw，这个是多余的功能，我们可以保留D3D11CreateDevice、WGC、frame pool，FrameArrived、staging、Map->CPU缓冲这些功能，或者说以后做D3D纹理共享给Chromium（当做以后得目标吧）

我们需要安装一个叫做 node-addon-api 的 C++ 库，Windows 上 vcpkg 常写成 `node-addon-api:x64-windows`。我们要在源码中引入 **`napi.h`**（node-addon-api 包装层）来使用 `Napi::Env` 以及 `Napi::Object` / `Napi::Function`。编译得到的是 DLL，要给 Node/Electron 使用，通常还要把输出名改成 `capture_addon.node`（或你在 CMake 里设的 `PREFIX ""` + `SUFFIX ".node"`）。

我们使用Imp的定义和实现分离的办法，定义一个,.h和一个.cpp文件:

```cpp
#pragma once

#include <atomic>
#include <cstdint>
#include <memory>
#include <mutex>
#include <string>
#include <vector>

struct RecorderConfig;

/// 默认显示器 WGC 捕获：D3D11 + 帧池；CPU 缓冲为 RGBA（自 BGRA 转换）。供 main / N-API 等复用。
class MonitorCapture {
    struct Impl;
    std::unique_ptr<Impl> impl_;

public:
    MonitorCapture();
    ~MonitorCapture();

    MonitorCapture(const MonitorCapture&) = delete;
    MonitorCapture& operator=(const MonitorCapture&) = delete;
    MonitorCapture(MonitorCapture&&) = delete;
    MonitorCapture& operator=(MonitorCapture&&) = delete;

    bool prepare_default_monitor();
    void begin_capture();
    void stop();

    bool is_prepared() const;
    int width() const;
    int height() const;

    std::mutex& buffer_mutex();
    std::vector<uint8_t>& pixel_buffer();
    std::atomic<bool>& frame_dirty();

    void set_preview_enabled(bool enabled);
    bool preview_enabled() const;

    void set_recording_enabled(bool enabled);
    bool recording_enabled() const;

    /// 输出 MP4（Media Foundation）。须已 prepare + begin_capture；path 为 UTF-16 本地路径。
    bool start_recording(const std::wstring& output_path, const RecorderConfig& config);
    void stop_recording();
    bool is_recording() const;
};
```

**说明（和旧笔记的差别）。** 捕获逻辑仍然是一段 `Impl` + `FrameArrived`，但实现文件已经 **`main_n.cpp`**（命名上和 GLFW 演示用的 `main.cpp` 区分开）。同一工程里后来叠了：

- **预览**：`set_preview_enabled`，帧里可选择只喂录制、仍然更新 `pixelBuffer` 给 Canvas。
- **录制**：`MediaRecorder` 写 H.264/AAC MP4；**WASAPI loopback** 抓「播放设备上听到的」混音，混音多为 float 时在回调里夹一层 **float→int16** 再送进 MF。

MSVC 下若写了 `std::max`， Windows.h 里的 **`min`/`max` 宏**会把模板函数顶坏，所以在 **`#include <Windows.h>` 之前** 建议定义 **`NOMINMAX`**（我在 `main_n.cpp` 里就是这么干的）；否则你会看到 `C2589` 一类语法错误。

```cpp
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <Windows.h>
// ...
const UINT32 ch = (std::max)(1u, m_audioChannelsFromDevice); // 括号也可防宏展开
```

完整 `MonitorCapture::Impl`（含 D3D staging、WGC、`WasapiLoopback`、`feed_wasapi_audio`）以仓库 **`main_n.cpp`** 为准，这里不再整段粘贴，避免下次又和线上漂移。


因此我们需要把方法包装成 `Napi::Value` / `Napi::Function` 再绑到 `exports` 上。当前仓库里 **`addon.cpp`** 除了 `start / stop / getFrame / getSize`，还多了 **`startRecording` / `stopRecording` / `isRecording`**，以及枚举播放设备的 **`getAudioOutputDevices`**（给 Vue 里下拉选 loopback 用）。录制参数里可选 **`audioOutputDeviceId`**（UTF-8），对应 `RecorderConfig::audioOutputDeviceId`（宽字符串）。

下面这一段是「骨架」示意：UTF-8 路径宽字符转换、`ParseRecorderConfig`、`InitializeModule` 里多挂了哪些名字——细节仍以仓库 **`addon.cpp`** 为准。

```cpp
#include <napi.h>
#include <winrt/base.h>
#include <Windows.h>

#include <memory>
#include <mutex>
#include <string>

#include "media_recorder.h"
#include "screen_capture.h"
#include "audio_output_devices.h"

namespace {

    std::unique_ptr<MonitorCapture> g_capture;
    std::once_flag g_winrt_apartment_once;

    static void ensure_winrt_apartment()
    {
        std::call_once(g_winrt_apartment_once, []() {
            try {
                winrt::init_apartment(winrt::apartment_type::multi_threaded);
            }
            catch (winrt::hresult_error const& e) {
                const auto c = static_cast<uint32_t>(e.code());
                if (c != 0x80010106U)
                    throw;
                // RPC_E_CHANGED_MODE：Electron 已初始化 COM，沿用现有单元即可
            }
        });
    }

    static RecorderConfig ParseRecorderConfig(Napi::Object const& o)
    {
        RecorderConfig cfg{};
        cfg.width = o.Get("width").As<Napi::Number>().Uint32Value();
        cfg.height = o.Get("height").As<Napi::Number>().Uint32Value();
        cfg.fps = o.Get("fps").As<Napi::Number>().Uint32Value();
        if (o.Has("videoBitrate"))
            cfg.videoBitrate = o.Get("videoBitrate").As<Napi::Number>().Uint32Value();
        if (o.Has("audioSampleRate"))
            cfg.audioSampleRate = o.Get("audioSampleRate").As<Napi::Number>().Uint32Value();
        if (o.Has("audioChannels"))
            cfg.audioChannels = o.Get("audioChannels").As<Napi::Number>().Uint32Value();
        if (o.Has("enableAudio"))
            cfg.enableAudio = o.Get("enableAudio").As<Napi::Boolean>().Value();
        if (o.Has("audioOutputDeviceId")) {
            const std::string u8 = o.Get("audioOutputDeviceId").As<Napi::String>().Utf8Value();
            if (!u8.empty())
                cfg.audioOutputDeviceId = Utf8ToWide(u8); // MultiByteToWideChar CP_UTF8
        }
        return cfg;
    }

    // Start / GetFrame / Stop / GetSize：与旧笔记相同思路……

    /** JS: startRecording(outputPathUtf8, config) — path 用 UTF-8 */
    Napi::Value StartRecording(const Napi::CallbackInfo& info) { /* … */ }

    Napi::Value GetAudioOutputDevices(const Napi::CallbackInfo& info)
    {
        Napi::Env env = info.Env();
        const auto devices = EnumerateAudioRenderDevices();
        Napi::Array arr = Napi::Array::New(env, devices.size());
        for (size_t i = 0; i < devices.size(); ++i) {
            Napi::Object row = Napi::Object::New(env);
            row.Set("id", Napi::String::New(env, WideToUtf8(devices[i].id)));
            row.Set("name", Napi::String::New(env, WideToUtf8(devices[i].friendlyName)));
            arr.Set(static_cast<uint32_t>(i), row);
        }
        return arr;
    }

} // namespace

Napi::Object InitializeModule(Napi::Env env, Napi::Object exports)
{
    exports.Set("start", Napi::Function::New(env, Start));
    exports.Set("stop", Napi::Function::New(env, Stop));
    exports.Set("getFrame", Napi::Function::New(env, GetFrame));
    exports.Set("getSize", Napi::Function::New(env, GetSize));
    exports.Set("startRecording", Napi::Function::New(env, StartRecording));
    exports.Set("stopRecording", Napi::Function::New(env, StopRecording));
    exports.Set("isRecording", Napi::Function::New(env, IsRecording));
    exports.Set("getAudioOutputDevices", Napi::Function::New(env, GetAudioOutputDevices));
    return exports;
}

NODE_API_MODULE(capture_addon, InitializeModule)
```

我们使用了std::call_once，来配合std::once_flag，保证传入的初始化代码在整个进程执行一次，其他线程若随后也执行到了call_once的话，会阻塞等待第一次跑完来返回结果而不会跑第二遍。无论多少次调用 `ensure_winrt_apartment()`、无论从哪个线程进来，WinRT/COM 这套「初始化公寓」逻辑至多执行一次。

COM经常是按照线程来初始化的，至于为什么叫做apartment，是因为一个线程上的COM并发模型分区： 

线程在参与COM之前必须声明线程大概在哪中apartment里，常见的有STA单线程、MTA多线程等。不同的Apartment里创建的COM对象，跨线程互相调用的时候要守一套排队、封送规则，就像对象被关在隔间。


头文件的来源可以用vcpkg，但是链接的时候必须与当前的Electron自带的Node ABI一致，vcpkg解决node-addon-api 头从哪来；**ABI 对齐仍靠 Electron 官方流程**。

我们为了适配electron版本和原生模块的关系，能够用cmake-js针对electron干净地配置并且编译+确认缓冲指向eletron以及把.node模块放在preload中，能干净地按照固定的相对路径找到位置，指定cmake里面怎么链接winrt、delay-load、win_delay_load_hook，运行时是/MD等。可以编写一个js脚本：

```js
/**
 * 必须用 cmake-js 针对 **Electron** 重新生成 CMake 缓存并链接 Electron 的 node.lib。
 * 若以前用默认 Node 编过，`build/` 里会残留 NODE_RUNTIME=node，此时复制 .node 仍会一加载就崩。
 *
 * 用法：在 capture 目录执行 `pnpm run build:addon`
 */
import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, readFileSync } from 'node:fs'
import { createRequire } from 'node:module'
import { dirname, join } from 'node:path'
import { fileURLToPath } from 'node:url'

const __dirname = dirname(fileURLToPath(import.meta.url))
const require = createRequire(import.meta.url)
const electronPkg = join(__dirname, '../node_modules/electron/package.json')
const electronVer = require(electronPkg).version
const addonRoot = join(__dirname, '../../glfwExample')

console.log('[build:addon] cmake 工程:', addonRoot)
console.log('[build:addon] Electron:', electronVer)

const npx = process.platform === 'win32' ? 'npx.cmd' : 'npx'

function run(args, label) {
  console.log('[build:addon]', label, args.join(' '))
  const r = spawnSync(npx, args, { cwd: addonRoot, stdio: 'inherit', shell: true })
  if ((r.status ?? 1) !== 0) {
    console.error('[build:addon] 失败:', label)
    process.exit(r.status ?? 1)
  }
}

  

/** 清掉旧缓存，否则会一直链接「系统 Node」的 node.lib */
run(['cmake-js', 'clean'], 'clean')
run(
  ['cmake-js', 'compile', '--runtime=electron', `--runtime-version=${electronVer}`],
  'compile(electron)'
)

  
const cacheFile = join(addonRoot, 'build/CMakeCache.txt')
if (existsSync(cacheFile)) {
  const cache = readFileSync(cacheFile, 'utf8')
  const rtLine = cache.split('\n').find((l) => l.startsWith('NODE_RUNTIME'))
  if (!rtLine || !/electron/i.test(rtLine)) {
    console.error('[build:addon] CMakeCache 未指向 electron runtime，当前行:', rtLine ?? '(无)')
    console.error('[build:addon] 请先删除文件夹:', join(addonRoot, 'build'), '再重跑本脚本')
    process.exit(1)
  }
}

const release = join(addonRoot, 'build/Release/capture_addon.node')
const debug = join(addonRoot, 'build/Debug/capture_addon.node')
const src = existsSync(release) ? release : debug
if (!existsSync(src)) {
  console.error('[build:addon] 未找到:', release, debug)
  process.exit(1)
}

const dstDir = join(__dirname, '../native/wgc-addon')
const dst = join(dstDir, 'capture_addon.node')
mkdirSync(dstDir, { recursive: true })
copyFileSync(src, dst)
console.log('[build:addon] 已复制 ->', dst)
```

我们使用了spawnSync方法，同步运行一个子进程并且执行命令，一直等到这个命令结束，函数返回。这里的stdio:'inherit'表示子进程的标准输入/错误 直接接到当前的终端，所以我们可以看到cmake的完整输出。shell:true表示系统shell里执行，windows上能够更稳找到npx.cmd。


```js
const cacheFile = join(addonRoot, 'build/CMakeCache.txt')
if (existsSync(cacheFile)) {
  const cache = readFileSync(cacheFile, 'utf8')
  const rtLine = cache.split('\n').find((l) => l.startsWith('NODE_RUNTIME'))
  if (!rtLine || !/electron/i.test(rtLine)) {
    console.error('[build:addon] CMakeCache 未指向 electron runtime，当前行:', rtLine ?? '(无)')
    console.error('[build:addon] 请先删除文件夹:', join(addonRoot, 'build'), '再重跑本脚本')
    process.exit(1)
  }
}
```

这段使用cacheFile，cmake会把本次配置写进这个文件，里面会有NODE_RUNTIME=...之类的项，然后读文件按行找：找到以NODE_RUNTIME开头的那一行， 因为cmake-js会写入当前是按哪种runtime配置的。校验：这一行里必须能匹配 `electron`（不区分大小写）。

若找不到这一行，或里面是 `node` 而不是 `electron`，说明 缓存仍不对：最常见是以前用默认 Node 编过、`cmake-js clean` 没生效、或手动改过目录但没重新配置。

失败时：打印当前读到的行（或「无」），提示你去 删掉整个 `build` 文件夹 再跑脚本，然后 `process.exit(1)`，避免后面还以为编译成功、把错的 `.node` 复制去 `capture`。

最后正常的找到.node并且复制过去。


那么我们可以在终端里初始化这个c++项目，pnpm init，然后安装一下node-addon-api，之后在运行npx node-gyp install即可

```shell
PS F:\glfwExamle\glfwExample> npx node-gyp install
Need to install the following packages:
node-gyp@12.3.0
Ok to proceed? (y) y

gyp info it worked if it ends with ok
gyp info using node-gyp@12.3.0
gyp info using node@22.16.0 | win32 | x64

gyp http GET https://nodejs.org/download/release/v22.16.0/node-v22.16.0-headers.tar.gz
gyp http 200 https://nodejs.org/download/release/v22.16.0/node-v22.16.0-headers.tar.gz
gyp http GET https://nodejs.org/download/release/v22.16.0/SHASUMS256.txt
gyp http GET https://nodejs.org/download/release/v22.16.0/win-x64/node.lib
gyp http 200 https://nodejs.org/download/release/v22.16.0/SHASUMS256.txt
gyp http 200 https://nodejs.org/download/release/v22.16.0/win-x64/node.lib
gyp info ok
PS F:\glfwExamle\glfwExample>
```

我们编写一个cmake文件来制定对应的文件和制定编译规则和包含等信息：

```cmake
cmake_minimum_required(VERSION 3.15)
if(NOT WIN32)
  message(FATAL_ERROR "capture_addon: Windows only.")
endif()
project(capture_addon LANGUAGES CXX)
set(NODE_ADDON_API_DIR "${CMAKE_CURRENT_SOURCE_DIR}/node_modules/node-addon-api")
if(NOT EXISTS "${NODE_ADDON_API_DIR}/napi.h")
    set(NODE_ADDON_API_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../node_modules/node-addon-api")
endif()

# 必须与 ${CMAKE_JS_SRC}（win_delay_load_hook.cc）一起链接，否则 /DELAYLOAD:NODE.EXE 在 Electron 内无法正确解析，require(.node) 会崩溃。
# 参考 cmake-js README：add_library(... ${CMAKE_JS_SRC})
add_library(${PROJECT_NAME} SHARED
    "addon.cpp"
    "main_n.cpp"
    "WasapiLoopback.cpp"
    "audio_output_devices.cpp"
)

if(CMAKE_JS_SRC)
  target_sources(${PROJECT_NAME} PRIVATE ${CMAKE_JS_SRC})
endif()

  

target_include_directories(${PROJECT_NAME} PRIVATE
  "${CMAKE_CURRENT_SOURCE_DIR}"
  "${NODE_ADDON_API_DIR}"    # 提供 napi.h
  "${CMAKE_JS_INC}"         # 提供 node_api.h (由 cmake-js 自动注入)
)

target_compile_definitions(${PROJECT_NAME} PRIVATE
  WGC_NATIVE_NODE_ADDON
  NAPI_CPP_EXCEPTIONS
  FMT_HEADER_ONLY
  WIN32_LEAN_AND_MEAN
)

  
set_target_properties(${PROJECT_NAME} PROPERTIES
    CXX_STANDARD 20
    CXX_STANDARD_REQUIRED ON
    PREFIX ""
    SUFFIX ".node"
)

  

# 必须与 Node/Electron 一致使用动态 CRT（/MD）。cmake-js 默认 MultiThreaded(/MT) 时，require(.node) 常在 Electron 内立刻崩溃。
set_property(TARGET ${PROJECT_NAME} PROPERTY MSVC_RUNTIME_LIBRARY
    "$<$<CONFIG:Debug>:MultiThreadedDebugDLL>$<$<CONFIG:Release>:MultiThreadedDLL>$<$<CONFIG:RelWithDebInfo>:MultiThreadedDLL>$<$<CONFIG:MinSizeRel>:MultiThreadedDLL>")
if(MSVC)
  # /await 用于支持 C++/WinRT 的异步操作
  target_compile_options(${PROJECT_NAME} PRIVATE /await /EHsc /permissive- /utf-8)
endif()

target_link_libraries(${PROJECT_NAME} PRIVATE
    d3d11.lib
    windowsapp.lib
    delayimp.lib              # 必须链接这个库来解决 __delayLoadHelper2 错误
    mfplat.lib
    mfuuid.lib
    mfreadwrite.lib
    avrt.lib
    Mmdevapi.lib
    "${CMAKE_JS_LIB}"
)
```

这里我们链接了d3d11.lib、windowsapp.lib（因为windows runtime 、WGC、以及CPP/winrt生成的api最终都会落在这类导入上，没有它链接阶段就会缺符号）、delayimp.lib（这是延迟加载的组件，提供了__delayLoadHelper2等，配合链接器/DELAYLOAD: ... cmake-js通常会带上对NODE的delay-load，如果没哟它就会出现缺delay load辅助函数一类的链接错误）、`"${CMAKE_JS_LIB}"`（cmake-js → Electron 自带的 `node.lib`，native addon必须和当前的Eletron的ABI对应的node.lib链接，否则就会链接错版本或者加载失败）

运行的时候electron会把node（同一套v8/libuv/N-API）嵌入electron.exe中，一般不会单独给个node.lib的文件。**编译原生插件的时候，必须有一份和当前electron版本ABI一致的导入库才能在链接阶段**解析napi_create_* 、 node_* 等符号。这份东西就叫做node.lib，属于windows下的导入库，来源一般就是cmake-js / node-gyp / @electron/rebuild 按照 runtime-verson去下载和缓存到本机：例如 `~\.cmake-js\electron-x64\v39.x.x\x64\node.lib`

为什么需要delayimp.lib呢，在MSVC上，如果链接选项里使用了延迟加载（cmake-js的常见写法：/DELAYLOAD:NODE.EXE） ，意思就是一开始不把node里成千上万个符号都从磁盘解析完，而是用到哪个符号的时候再加载。
实现延迟加载需要运行时辅助函数（典型是 `__delayLoadHelper2` 等），它们写在 `delayimp.lib`（Delay Importer）里。

因此：只要你的 `.node` 是用「带 `/DELAYLOAD`」的方式链 `node.lib` 的，就必须再链 `delayimp.lib`，否则会报 「无法解析的外部符号 __delayLoadHelper2」 一类链接错误。

再结合 `win_delay_load_hook.cc`：在运行期让delay-load在electron环境下指向正确模块，而不是去磁盘上的node.exe。
它在延迟加载流程里「拦截」对 `NODE.EXE` 的解析，让符号实际从 当前 Electron 进程 里解析——这是 `delayimp` + `/DELAYLOAD` + hook 这一套配合起来，Electron 里 `require(.node)` 才稳定。

>[!tip] 符号
>可能有人第一次看到，有些不熟悉这个概念，符号symbol就是编译/链接阶段用来标识某段代码和数据的名字，链接器靠他在多个.obj .lib .dll之间对错号。一般分为函数符号比如说napi_create_string_utf8 以及 **数据/全局符号**：例如**某些全局变量**。
>编译我们的addon.cpp来**生产obj的时候里面会有未解析的符号引用**，就比如我们调用的napi_get_undefine，此时还不知道地址，node.lib是导入库（里面主要都是桩信息，告诉链接器这些符号在实际运行的时候由哪个DLL/EXE提供）。链接成capture_addon.node之后，PE会记下这些要入到运行时再去找某个模块里寻找。延迟加载的思想和这个差不多，意思也是启动/刚加载.node的时候可以不先把每个符号都绑定完，用到的时候回去解析，和delayimp.lib、hook配合

最后，为什么需要一致使用动态 CRT **`/MD`**（对应 MSVC 选项 **MultiThreadedDLL**：CRT 在 `vcruntime*.dll` / `ucrtbase.dll` 等系统 DLL 里，进程内各模块共用一套堆与实现）。若误用 **`/MT`**（**MultiThreaded**：CRT 静态链进你的 `.node`，等于模块私有一套 CRT），跨 DLL 边界 `malloc`/`free`、`FILE*`、locale 等很容易变成未定义行为，Electron 里常见表现就是加载 `.node` 立刻崩或随机崩。

>[!hint] Electron / 官方 Node Windows 发行版是按 `/MD` 编的：主程序、`node.dll`、大量系统组件都在 同一套 UCRT + VCRuntime 上跑。

因为.node本质上是一个dll，加载进了进程之后，堆内存不能混用，如果一侧是A套CRT，一侧B套CRT，是未定义行为，常表现为访问冲突，N-API/V8 会在边界上分配、传递缓冲区，很容易暴毙。

>[!tldr]  CRT: C Runtime Library，即 C 语言运行时库。提供了malloc、free、new、delete 、printf、fopen、FILE* 等。 在Windows + MSVC环境下可以静态链进exe/dll （MT）。也可以跟随系统里的CRT DLL公用（/MD）。     locale是区域/本地化设置，包括字符分类，数字格式和日期格式和排序。



然后执行 `npx cmake-js compile` 来编译（或用上面的 `pnpm run build:addon` 一条龙），需要时用 **x64 Native Tools** 环境。


```shell
F:\glfwExamle\glfwExample\glfwExample>npx cmake-js compile -G "Ninja"
INFO TOOL Using Ninja generator, as specified from commandline.
INFO CMD BUILD
INFO RUN [
  'cmake',
  '--build',
  'F:\\glfwExamle\\glfwExample\\glfwExample\\build',
  '--config',
  'Release'
]
ninja: error: loading 'build.ninja': The system cannot find the file specified.

INFO REP Build has been failed, trying to do a full rebuild.
INFO CMD CLEAN
INFO RUN [
  'cmake',
  '-E',
  'remove_directory',
  'F:\\glfwExamle\\glfwExample\\glfwExample\\build'
]
INFO CMD CONFIGURE
INFO RUN [
  'cmake',
  'F:\\glfwExamle\\glfwExample\\glfwExample',
  '--no-warn-unused-cli',
  '-G',
  'Ninja',
  '-DCMAKE_JS_VERSION=8.0.0',
  '-DCMAKE_BUILD_TYPE=Release',
  '-DCMAKE_RUNTIME_OUTPUT_DIRECTORY=F:\\glfwExamle\\glfwExample\\glfwExample\\build',
  '-DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded$<$<CONFIG:Debug>:Debug>',
  '-DCMAKE_JS_INC=C:\\Users\\20742\\.cmake-js\\node-x64\\v22.16.0\\include\\node',
  '-DCMAKE_JS_SRC=F:/glfwExamle/glfwExample/glfwExample/node_modules/.pnpm/cmake-js@8.0.0/node_modules/cmake-js/lib/cpp/win_delay_load_hook.cc',
  '-DNODE_RUNTIME=node',
  '-DNODE_RUNTIMEVERSION=22.16.0',
  '-DNODE_ARCH=x64',
  '-DCMAKE_JS_LIB=C:\\Users\\20742\\.cmake-js\\node-x64\\v22.16.0\\win-x64\\node.lib',
  '-DCMAKE_SHARED_LINKER_FLAGS=/DELAYLOAD:NODE.EXE'
]
Not searching for unused variables given on the command line.
-- The CXX compiler identification is MSVC 19.44.35220.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: C:/Program Files (x86)/Microsoft Visual Studio/2022/BuildTools/VC/Tools/MSVC/14.44.35207/bin/Hostx64/x64/cl.exe - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done (25.3s)
-- Generating done (2.7s)
-- Build files have been written to: F:/glfwExamle/glfwExample/glfwExample/build
INFO CMD BUILD
INFO RUN [
  'cmake',
  '--build',
  'F:\\glfwExamle\\glfwExample\\glfwExample\\build',
  '--config',
  'Release'
]
[3/3] Linking CXX shared library capture_addon.node
```

编译成功，我们得到了一个.node文件，是node.js可以直接调用的二进制模块。可以快速做一个验证，你在同级目录新建js文件：

```js
const addon = require('./build/capture_addon.node');

console.log('模块加载成功！');

try {
    // 既然对象里有 start，直接试试它
    const result = addon.start(); 
    console.log('Start 结果:', result);
    
    const size = addon.getSize();
    console.log(`显示器尺寸: ${size.width} x ${size.height}`);
} catch (err) {
    console.error('调用失败:', err);
}
```
输出：

```shell
(mmlab) PS F:\glfwExamle\glfwExample\glfwExample> node .\test.js
模块加载成功！
Start 结果: undefined
显示器尺寸: 1920 x 1080
(mmlab) PS F:\glfwExamle\glfwExample\glfwExample> 
```

## 使用

测试通过之后，我们可以可以来真正使用这个原生模块了，由于Electron的主进程在nodejs中，渲染进程在chromium中，因此我们需要通过context bridge将原生模块能力桥接给前端。

为了让canvas可以渲染，我们需要一个能把c++的pixelBuffer传递给js的办法，我们可以使用Napi::Buffer，它可以允许JS直接访问C++的内存地址。


```cpp
    Napi::Value GetFrame(const Napi::CallbackInfo& info) {
        Napi::Env env = info.Env();
        auto& buffer = g_capture->pixel_buffer();
        auto& mutex = g_capture->buffer_mutex();
        std::lock_guard<std::mutex> lock(mutex);
        return Napi::Buffer<uint8_t>::Copy(env, buffer.data(), buffer.size());
    }
```

在electron中，由于安全限制，不能直接在渲染进程require原生模块，因此我们最好的解决方法是写一个预加载脚本preload.js。


```js

function loadAddon() {
  if (addonTried) return addon
  addonTried = true
  if (process.env.SKIP_NATIVE === '1') return null
  const p = join(__dirname, '../../native/wgc-addon/capture_addon.node')
  if (!existsSync(p)) {
    console.error('[preload] 未找到', p)
    return null
  }
  try {
    addon = require(p)
    return addon
  } catch (e) {
    console.error('[preload] require .node 失败', e)
    return null
  }
}

function getAddon() {
  return loadAddon()
}

const captureAPI = {
  start: () => {
    if (process.env.SKIP_NATIVE === '1') return Promise.resolve()
    const a = getAddon()
    if (!a) throw new Error('native addon not loaded')
    return Promise.resolve(a.start())
  },
  stop: () => {
    if (process.env.SKIP_NATIVE === '1') return undefined
    const a = getAddon()
    if (!a) return undefined
    return a.stop()
  },
  getSize: () => {
    if (process.env.SKIP_NATIVE === '1') return { width: 800, height: 600 }
    const a = getAddon()
    if (!a) return { width: 0, height: 0 }
    return a.getSize()
  },

  getFrame: () => {
    if (process.env.SKIP_NATIVE === '1') return null
    const a = getAddon()
    if (!a) return null
    return a.getFrame()
  },
  /** outputPath：UTF-8，如 C:\\\\out\\\\cap.mp4 */
  startRecording: (outputPath, config) => {
    if (process.env.SKIP_NATIVE === '1') return Promise.resolve()
    const a = getAddon()
    if (!a) throw new Error('native addon not loaded')
    return Promise.resolve(a.startRecording(outputPath, config))
  },
  stopRecording: () => {
    if (process.env.SKIP_NATIVE === '1') return undefined
    const a = getAddon()
    if (!a) return undefined
    return a.stopRecording()
  },
  isRecording: () => {
    if (process.env.SKIP_NATIVE === '1') return false
    const a = getAddon()
    if (!a) return false
    return a.isRecording()
  },
  /** 枚举播放设备；loopback 录制里可把返回的 id 填进 config.audioOutputDeviceId */
  getAudioOutputDevices: () => {
    if (process.env.SKIP_NATIVE === '1') return Promise.resolve([])
    const a = getAddon()
    if (!a) return Promise.resolve([])
    return Promise.resolve(a.getAudioOutputDevices())
  }
}
```

```js
if (process.contextIsolated) {
  try {
    contextBridge.exposeInMainWorld('captureAPI', captureAPI)
    contextBridge.exposeInMainWorld('electron', electronAPI)
    contextBridge.exposeInMainWorld('api', api)
  } catch (error) {
    console.error('[preload] expose failed:', error)
  }
} else {
  window.captureAPI = captureAPI
  window.electron = electronAPI
  window.api = api
}
```

我们使用了exposeInMainWorld，渲染进程的网页拿不到node，也不能访问preload里面的变量，必须用contextBridge.exposeInMainWorld把一组经过允许的函数/对象 代理到window上，才能让页面使用window.captureAPI等。


```vue
<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
const screenCanvas = ref(null)
let timerId = null

  
onMounted(async () => {
  const canvas = screenCanvas.value
  const ctx = canvas.getContext('2d')
  try {
    await window.captureAPI.start()
  } catch (e) {
    console.error('capture start failed', e)
    return
  }

  const size = await window.captureAPI.getSize()
  if (!size || size.width === 0) {
    console.error('无法获取显示器尺寸')
    return
  }
  canvas.width = size.width
  canvas.height = size.height
  const imgData = ctx.createImageData(size.width, size.height)

  // 原生已输出 RGBA；preload 调 getFrame。先约 15fps，后续可降采样 / SharedArrayBuffer
  const tick = async () => {
    try {
      const buffer = await window.captureAPI.getFrame()
      if (buffer) {
        imgData.data.set(new Uint8Array(buffer))
        ctx.putImageData(imgData, 0, 0)
      }
    } catch (e) {
      console.error('getFrame', e)
    }
  }
  timerId = window.setInterval(tick, 66)
})

  
onUnmounted(async () => {
  if (timerId != null) window.clearInterval(timerId)
  try {
    await window.captureAPI.stop()
  } catch (e) {
    console.error('stop', e)
  }
})
</script>

  

<template>
  <div class="capture-container">
    <canvas ref="screenCanvas" class="screen-render"></canvas>
  </div>
</template>

  

<style scoped>
.capture-container {
  width: 100%;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #000;
}
.screen-render {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}
</style>
```


之后就可以正确显示了！当前仓库里的 **`App.vue`** 在同一份思路上又加了：录制路径、`startRecording`/`stopRecording`、`getAudioOutputDevices` 下拉选播放设备、`audioOutputDeviceId` 传给原生等——仍以 **`capture/src/renderer/src/App.vue`** 为准，笔记里就不重复贴一整套表单了。

---
