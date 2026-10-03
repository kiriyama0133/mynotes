
在 Windows 平台上，**Media Foundation（常口头叫 WMF/MF）** 是比 FFmpeg 更「原生」的一套管线：系统 DLL + 可选显卡驱动的 MFT（Media Foundation Transform）。你可以用 **`IMFSinkWriter`** 把视频轨、音频轨mux进 MP4。

但要注意一句话：**不是**你一写 SinkWriter，系统就必然走 **NVENC / Quick Sync** 硬件编码——能不能用上硬件 H.264/AAC MFT，取决于 **SinkWriter 创建的拓扑**、**你注册的输出 subtype**、以及 **`MF_READWRITE_ENABLE_HARDWARE_TRANSFORMS`** 等属性。我们自己在工程里（见下文 **`WasapiLoopback.cpp` 里 `MediaRecorder::StartRecording`**）为了避开「某些机型驱动里硬件 MFT 抽风」，**默认把硬件变换关掉，优先软件编码保证可录**；视频侧仍尽量走 **DXGI Surface → MF** 的 GPU 路径喂帧，必要时再 **回退到 CPU Map/RGB 拷贝**。这样既保留 MF 的原生性，又不会在用户机上随机 HRESULT。

下面行文把你关心的 **WASAPI loopback → PCM → SinkWriter → AAC**、以及 **时间戳 / 软硬回退** 串起来；代码片段尽量与仓库 **`glfwExample/WasapiLoopback.cpp`**、**`media_recorder.h`**、**`main_n.cpp`** 对齐。

---

## WASAPI 录制与时间轴

录制里一等公民是 **A/V Sync**。视频帧是离散的，音频是连续的；采集时钟（声卡混音、QPC）和 MF 内部时钟都要落到同一套 **100ns** 时间轴上。

### Loopback 与格式

我们需要 **`IAudioClient` + `IAudioCaptureClient`**，并用 **`AUDCLNT_STREAMFLAGS_LOOPBACK`** 捕获「扬声器/指定播放设备上听到的」混音，而不是麦克风。

混音格式多为 **IEEE Float**（`WAVE_FORMAT_IEEE_FLOAT` / `WAVEFORMATEXTENSIBLE`），而我们在 SinkWriter 里给 AAC 配的输入一般是 **16-bit PCM**。因此在 **`main_n.cpp`** 的回调里会做一层 **float → int16**（clamp 到 ±1.0 再乘 32767），再 **`PushAudioData`**——笔记后面 **「PCM 与编码入口」** 一节会点到这条链。

使用 **`IMFSinkWriter`** 时，视频一侧塞 **`ID3D11Texture2D`**（尽量 **`MFCreateDXGISurfaceBuffer`**），音频一侧塞 **PCM 块**；SinkWriter 内部会拉起编码 MFT，把 **H.264 / AAC** 写进容器。

### PTS（Presentation Time）

MF 的时间单位是 **100 纳秒（1s = 10⁷ 单位）**。若统一用 **QPC** 做「录制起点」：

- 记 **`T₀`** 为第一次有效音视频写入时的 QPC（工程里用 **`m_startTimeQPC`**）。
- **PTS（100ns）** ≈ **`(QPC_now − T₀) × 10⁷ / QPC_frequency`**。

音频也可以按「累计采样数 / 采样率」推 PTS，但我们在工程里与视频共用 **QPC 差分**，实现简单、和 WGC 帧同源。

---

## 编写代码

### 头文件与类（与当前 `media_recorder.h` 一致）

除了 **`Audioclient.h`**、**`wrl/client.h`**，实际工程里 **`media_recorder.h`** 还拉了 MF 读写、DXGI 设备管理器、以及设备属性键（若你做枚举）。核心接口仍是：

- **`IAudioClient`**：流初始化、Start/Stop。
- **`IAudioCaptureClient`**：从驱动拉包。
- **`AUDCLNT_STREAMFLAGS_LOOPBACK`**：loopback 混音。

**`Microsoft::WRL::ComPtr`**：COM RAII，离开作用域自动 `Release`，避免漏释。

下面这段结构体/类声明请与仓库 **`media_recorder.h`** 对齐（含 **`enableAudio`**、**`audioOutputDeviceId`** 等；**`WasapiLoopback::Initialize` 可带设备 ID**）：

```cpp
#pragma once
#pragma comment(lib, "mfplat.lib")
#pragma comment(lib, "mfuuid.lib")
#pragma comment(lib, "mfreadwrite.lib")

#include <windows.h>
#include <unknwn.h>
#include <wrl/client.h>
#include <Audioclient.h>

#include <mfapi.h>
#include <mfidl.h>
#include <mmdeviceapi.h>
#include <endpointvolume.h>
#include <Functiondiscoverykeys_devpkey.h>

#include <mfreadwrite.h>
#include <mfobjects.h>
#include <avrt.h>

#include <d3d11.h>

#include <functional>
#include <memory>
#include <string>
#include <thread>
#include <cstdint>
#include <atomic>

using Microsoft::WRL::ComPtr;
using AudioDataCallback = std::function<void(const uint8_t*, size_t*, uint64_t*)>;

struct RecorderConfig {
    uint32_t width;
    uint32_t height;
    uint32_t fps;
    uint32_t videoBitrate = 10000000;
    uint32_t audioSampleRate = 48000;
    uint32_t audioChannels = 2;
    /** false：仅 H.264，无 AAC，可绕过部分机型 AAC/PCM 协商失败 */
    bool enableAudio = true;
    /** 空：默认播放设备；否则为 IMMDevice::GetId() 同款的设备实例路径字符串 */
    std::wstring audioOutputDeviceId;
};

class MediaRecorder {
public:
    MediaRecorder(ComPtr<ID3D11Device> device, const RecorderConfig& config);
    ~MediaRecorder();

    uint64_t GetQPCFrequency();
    uint64_t GetCurrentQPC();

    HRESULT StartRecording(const std::wstring& outputPath, const RecorderConfig& config);
    HRESULT PushVideoFrame(ComPtr<ID3D11Texture2D> texture, uint64_t qpcTime);
    HRESULT PushAudioData(const uint8_t* data, size_t size, uint64_t qpcTime);
    void StopRecording();

    std::atomic<bool> m_isRecording{ false };
    uint64_t m_startTimeQPC = 0;
    uint64_t m_qpcFrequency = 0;

private:
    HRESULT SetupVideoStreaming(const RecorderConfig& config);
    HRESULT SetupAudioStreaming(const RecorderConfig& config);
    HRESULT PushVideoFrame_RgbMemory(ComPtr<ID3D11Texture2D> texture, LONGLONG pts, LONGLONG frameDur100ns);

    ComPtr<ID3D11Device> m_pDevice;
    ComPtr<ID3D11DeviceContext> m_pImmediateContext;
    ComPtr<ID3D11Texture2D> m_stagingReadback;
    bool m_useRgbMemoryPath = false;
    ComPtr<IMFDXGIDeviceManager> m_pDeviceManager;
    ComPtr<IMFSinkWriter> m_sinkWriter;

    RecorderConfig m_config;
    DWORD m_videoStreamIndex = 0;
    DWORD m_audioStreamIndex = 0;
    bool m_hasAudioStream = false;
};

class WasapiLoopback {
public:
    WasapiLoopback();
    ~WasapiLoopback();

    WasapiLoopback(const WasapiLoopback&) = delete;
    WasapiLoopback& operator=(const WasapiLoopback&) = delete;

    /** deviceId 为空或 nullptr：默认渲染端点（控制台）；否则 GetDevice(deviceId) */
    HRESULT Initialize(LPCWSTR deviceId = nullptr);

    void Start(AudioDataCallback callback);
    void Stop();

    WAVEFORMATEX* GetFormat() const { return m_pwfx; }

private:
    void CaptureThread();
    ComPtr<IAudioClient> m_audioClient;
    ComPtr<IAudioCaptureClient> m_captureClient;
    WAVEFORMATEX* m_pwfx = nullptr;
    AudioDataCallback m_onData;
    std::thread m_captureThread;
    std::atomic<bool> m_isRunning{ false };
    uint32_t m_frameSize = 0;
};
```

**分工简述**

- **`WasapiLoopback`**：独立线程 **`GetBuffer`**，把 loopback PCM 通过回调交出；**`qpcPosition`** 给上层对齐视频 QPC。
- **`MediaRecorder`**：创建 **`IMFSinkWriter`**，配置 **H.264 +（可选）AAC**，**`PushVideoFrame` / `PushAudioData`** 写 **`IMFSample`**。

---

### `HRESULT` 与 COM 成败判断

`HRESULT` 的位域分类（严重性、Facility、Code）你笔记里表格已经够用。实务上：**成功用 `SUCCEEDED(hr)`**，不要只写 `== S_OK`，因为有些 API 会返回 `S_FALSE` 仍算成功语义。

---

### `WasapiLoopback::Initialize`：设备与 LOOPBACK

旧笔记只贴了 **`GetDefaultAudioEndpoint`**。当前实现支持 **`IMMDeviceEnumerator::GetDevice(deviceId)`** 指定播放设备；随后 **共享模式 + LOOPBACK + `GetMixFormat`**，再 **`Initialize` / `GetService(IAudioCaptureClient)`**。片段摘自仓库（省略与正文重复的注释）：

```cpp
HRESULT WasapiLoopback::Initialize(LPCWSTR deviceId)
{
    // … Stop、释放旧 m_pwfx / COM 接口 …

    ComPtr<IMMDeviceEnumerator> pEnumerator;
    HRESULT hr = CoCreateInstance(__uuidof(MMDeviceEnumerator), nullptr, CLSCTX_ALL,
        __uuidof(IMMDeviceEnumerator), (void**)pEnumerator.GetAddressOf());
    if (FAILED(hr)) return hr;

    ComPtr<IMMDevice> pDevice;
    if (deviceId != nullptr && deviceId[0] != L'\0')
        hr = pEnumerator->GetDevice(deviceId, pDevice.GetAddressOf());
    else
        hr = pEnumerator->GetDefaultAudioEndpoint(eRender, eConsole, pDevice.GetAddressOf());
    if (FAILED(hr)) return hr;

    hr = pDevice->Activate(__uuidof(IAudioClient), CLSCTX_ALL, nullptr,
        (void**)m_audioClient.GetAddressOf());
    if (FAILED(hr)) return hr;

    hr = m_audioClient->GetMixFormat(&m_pwfx);
    if (FAILED(hr)) return hr;

    constexpr REFERENCE_TIME kBufferDuration = 500 * 10000; // 500ms，100ns 单位
    hr = m_audioClient->Initialize(
        AUDCLNT_SHAREMODE_SHARED,
        AUDCLNT_STREAMFLAGS_LOOPBACK,
        kBufferDuration,
        0,
        m_pwfx,
        nullptr);
    if (FAILED(hr)) return hr;

    hr = m_audioClient->GetService(__uuidof(IAudioCaptureClient),
        (void**)m_captureClient.GetAddressOf());
    if (FAILED(hr)) return hr;

    m_frameSize = m_pwfx->nBlockAlign;
    return S_OK;
}
```

要点：**LOOPBACK 必须配合共享模式 Initialize**；缓冲时长用 **`REFERENCE_TIME`**（100ns），不要和毫秒混着拍脑袋乘错数量级。

---

### 捕获线程：`AvSetMmThreadCharacteristicsW`、`SILENT` 位

采集线程里 **`CoInitializeEx(..., COINIT_MULTITHREADED)`** 仍然必要——**`IAudioClient` 指针从别的线程传来，本线程也要 COM 入场**。

MMCSS：工程里用的是 **`AvSetMmThreadCharacteristicsW(L"Pro Audio", ...)`**（宽字符串），降低被普通线程饿死导致断续的概率。

**双层循环**（外层生命周期 + 内层 **`GetNextPacketSize`** 清空积压）保留你原来的结论：高负载下只读一包就睡，容易堆满驱动缓冲。

**`AUDCLNT_BUFFERFLAGS_SILENT`**：静音包 **`pData` 内容不可信**，仓库里在回调前对 **`sizeInBytes`** 范围 **`memset(0)`**，再交给上层，避免把垃圾当信号：

```cpp
if ((flags & AUDCLNT_BUFFERFLAGS_SILENT) && pData && sizeInBytes > 0)
    std::memset(pData, 0, sizeInBytes);
m_onData(pData, &sizeInBytes, &qpcPosition);
```

析构时释放 **`IAudioClient` / `IAudioCaptureClient`**，避免线程结束仍占设备。

---

## `MediaRecorder`：两层「硬件不行就软件」——和仓库完全一致

下面这件事一定要分清，否则你会以为「关了硬件变换 = 不用显卡」。实际上我们做了 **两层回退**，对应 **`WasapiLoopback.cpp`** 里 **`MediaRecorder`** 的真实代码。

### 第一层（编码拓扑）：`MF_READWRITE_ENABLE_HARDWARE_TRANSFORMS`

**含义**：告诉 **`IMFSinkWriter`**：组装 **H.264 / AAC** 的 MFT 链时，**不要优先选用显卡驱动注册的硬件编码器**（NVENC、Intel QSV、AMD AMF 等在 MF 里暴露的 **硬件 MFT**）。部分驱动版本下硬件路径会在 **`BeginWriting` / `WriteSample`** 阶段给出 **`E_INVALIDARG`**、或其它 HRESULT，非常难在用户机器上复现排查。

我们在 **`StartRecording`** 里 **静态地** 设为 **`FALSE`**，等价于 **默认走系统自带的软件编码管线**（仍是 MF，只是编码器实例多为 CPU）。源码注释写得很直白：

```449:465:glfwExample/WasapiLoopback.cpp
    // 硬件编码器/DXGI RGB 路径在部分机型上 WriteSample 报 E_INVALIDARG；先用软件管线保证可写
    hr = pAttributes->SetUINT32(MF_READWRITE_ENABLE_HARDWARE_TRANSFORMS, FALSE);
    // ...
    hr = pAttributes->SetUnknown(MF_SINK_WRITER_D3D_MANAGER, m_pDeviceManager.Get());
    // ...
    hr = pAttributes->SetUINT32(MF_SINK_WRITER_DISABLE_THROTTLING, true);
    // ...
    hr = MFCreateSinkWriterFromURL(path.c_str(), nullptr, pAttributes.Get(), &m_sinkWriter);
```

**`MF_SINK_WRITER_D3D_MANAGER`** 仍然要设：它提供 **DXGI 设备管理器**，后面 **`MFCreateDXGISurfaceBuffer`** 把 **`ID3D11Texture2D`** 交给 SinkWriter 时要用——这和「视频编码在 GPU 还是 CPU」不是同一个开关。**`MF_SINK_WRITER_DISABLE_THROTTLING`** 用来减轻 SinkWriter 人为拖慢写入（按需保留）。

**若你以后要做「先试硬件再降级」**（笔记级思路，仓库当前未写自动重试）：可在 **`MFCreateSinkWriterFromURL`** 之前 **`SetUINT32(..., TRUE)`**，若 **`BeginWriting`** 或前几帧 **`WriteSample`** 连续失败，**释放 SinkWriter**，再用 **`FALSE`** 重建一次。注意：**同一 MP4 会话中途不能魔法切换拓扑**，通常是 **整段录制重新开始** 或 **接受默认软件编码**。

---

### 第二层（视频输入路径）：DXGI Surface → 失败则 RGB 内存拷贝

即使编码侧已是软件 H.264，**喂给编码器的像素**仍可优先走 **GPU 侧的 DXGI Surface buffer**（少一次大规模 CPU Map）。若驱动/MF 组合不接受当前 DXGI 封装，或 **`WriteSample` 对 DXGI buffer 返回 `E_INVALIDARG (0x80070057)`**，则 **运行时** 置 **`m_useRgbMemoryPath = true`**，之后每帧走 **`PushVideoFrame_RgbMemory`**：**`CopyResource` → `Map` staging → `memcpy` → `MFCreateMemoryBuffer`**。这是 **Pixel Delivery 回退**，CPU 带宽会涨，但能救活一大批「显卡正常但 MF 挑食」的机器。

仓库里的 **`PushVideoFrame`**（节选，与行号对应 **`WasapiLoopback.cpp`**）：

```cpp
HRESULT MediaRecorder::PushVideoFrame(ComPtr<ID3D11Texture2D> texture, uint64_t qpcTime) {
    if (!m_isRecording || !m_sinkWriter) return E_FAIL;

    D3D11_TEXTURE2D_DESC td{};
    texture->GetDesc(&td);
    if (td.Width != m_config.width || td.Height != m_config.height) {
        // … 打日志并返回 MF_E_INVALIDMEDIATYPE …
    }

    if (m_startTimeQPC == 0)
        m_startTimeQPC = qpcTime;

    const LONGLONG pts = (qpcTime - m_startTimeQPC) * 10000000 / static_cast<LONGLONG>(m_qpcFrequency);
    const UINT32 fps = (m_config.fps < 1u) ? 30u : m_config.fps;
    const LONGLONG frameDur100ns =
        (10000000LL + static_cast<LONGLONG>(fps) / 2LL) / static_cast<LONGLONG>(fps);

    if (m_useRgbMemoryPath)
        return PushVideoFrame_RgbMemory(texture, pts, frameDur100ns);

    ComPtr<IMFMediaBuffer> pBuffer;
    HRESULT hr = MFCreateDXGISurfaceBuffer(__uuidof(ID3D11Texture2D), texture.Get(), 0, TRUE, &pBuffer);
    if (FAILED(hr)) {
        if (m_stagingReadback) {
            m_useRgbMemoryPath = true;
            return PushVideoFrame_RgbMemory(texture, pts, frameDur100ns);
        }
        return hr;
    }
    // … MFCreateSample、SetSampleTime/Duration …
    hr = m_sinkWriter->WriteSample(m_videoStreamIndex, pSample.Get());
    if (FAILED(hr)) {
        constexpr HRESULT kInvalidArg = static_cast<HRESULT>(0x80070057);
        if (hr == kInvalidArg && m_stagingReadback) {
            m_useRgbMemoryPath = true;
            return PushVideoFrame_RgbMemory(texture, pts, frameDur100ns);
        }
    }
    return hr;
}
```

当硬件编码器不可靠的时候，可能是驱动不兼容，可能是格式限制，导致无法直接处理显存纹理的时候，我们就用不了零拷贝了，这个时候我们会选择copy回cpu来处理：

```cpp
HRESULT hr = m_pImmediateContext->Map(m_stagingReadback.Get(), 0, D3D11_MAP_READ, 0, &map);
```

m_stagingReadback是我们指定的资源，这里使我们之前创建的暂存纹理，参数0用于指定资源内部索引，对于我们录屏使用的2D纹理，它索引为1。D3D11_MAP_RAED用于告知怎么操作这块内存，我们使用这个值告诉GPU ：cpu需要读取，gpu会确保数据同步到缓存里，供我们读取。 其他还有几个标志：
- **`D3D11_MAP_WRITE`**：我要改数据。
    
- **`D3D11_MAP_READ_WRITE`**：又读又改（性能最差）。
    
- **`D3D11_MAP_WRITE_DISCARD`**：我要写新的，旧的直接扔掉（这是上传顶点数据到 GPU 时常用的，非常快）。


```cpp
    const UINT packedPitch = m_config.width * 4u;
    const DWORD packedSize = packedPitch * m_config.height;
```

这个是图像的内存布局，是我们将数据从显卡的对其空间压缩到媒体编码器的紧凑控件的计算。第一行代码用于计算图像的一行的有效数据长度，由于一个像素包含4个通道，一个通道占用1个字节，因此乘以4。第二行是计算整帧的图像总字节数， 每一行的字节数 × 总行数（高度）。

这个值会用于创建MF的内存Buffer，来存储图像。


```cpp
    ComPtr<IMFMediaBuffer> buf;
    hr = MFCreateMemoryBuffer(packedSize, &buf);
    if (FAILED(hr)) {
        m_pImmediateContext->Unmap(m_stagingReadback.Get(), 0);
        return hr;
    }
```

我们在这里声明一个MF缓冲区的接口指针，这个是media foundation对原始内存的封装，它不止是一个void*，还带有管理内存长度、最大程度和提供lock机制的功能。

然后在系统内存中分配一块大小为packedSize的连续空间。之所以使用它而不是new BYTE()是因为mf的后续组件比如编码器只认识IMFMediaBuffer，**它是COM对象**，当你把它塞入IMFSample之后，就算函数结束了，只有编码器还没处理完就不会销毁内存。这个函数通常会返回一块经过SSE/AVX指令集优化对齐的内存，对后续的memcpy性能有好处。


```cpp
   BYTE* dst = nullptr;
   hr = buf->Lock(&dst, nullptr, nullptr);
   if (FAILED(hr)) {
       m_pImmediateContext->Unmap(m_stagingReadback.Get(), 0);
       return hr;
   }
```
dst是一个BYTE* 指针，调用Lock之后，系统会在该缓冲区区在进程地址空间中的真实起始地址赋值给dst，`Lock` 函数的标准签名是： `HRESULT Lock(BYTE ppbBuffer, DWORD *pcbMaxLength, DWORD *pcbCurrentLength);`

第一个nullptr用于查询缓冲区的最大容量，第二个是目前里面装了的量。

接下来我们会进行搬运：

```cpp
    const auto* src = static_cast<const BYTE*>(map.pData);
    for (UINT y = 0; y < m_config.height; ++y)
        std::memcpy(dst + static_cast<size_t>(y) * packedPitch, src + static_cast<size_t>(y) * map.RowPitch, packedPitch);
    buf->Unlock();
    buf->SetCurrentLength(packedSize);
    m_pImmediateContext->Unmap(m_stagingReadback.Get(), 0);
```

我们会从src的y行开始，切下packedPitch宽度的有效像素，跳过map.RowPitch中多余的填充字节，然后紧密贴到dst的下一行。Unlock操作提交更改，使用SetCurrentLenghth更新元数据，因为MFCreateMemoryBuffer只是分了控件，而不知道有多少东西，这行代码会告诉后续的Sample这个Buffer有packedSize字节的有效视频数据。

最后使用Unmap归还GPU资源，这一步很重要，忘了写的话会导致显卡驱动认为还在占用资源导致后续copy无法执行从而导致D3D设备重置。


---

### `PushAudioData`：与仓库一致的完整实现

```cpp
HRESULT MediaRecorder::PushAudioData(const uint8_t* data, size_t size, uint64_t qpcTime) {
    if (!m_hasAudioStream)
        return S_OK;
    if (!m_isRecording || !m_sinkWriter) return E_FAIL;
    if (m_startTimeQPC == 0) {
        m_startTimeQPC = qpcTime;
    }
    LONGLONG pts = (qpcTime - m_startTimeQPC) * 10000000 / m_qpcFrequency;
    ComPtr<IMFMediaBuffer> pBuffer;
    HRESULT hr = MFCreateMemoryBuffer(static_cast<DWORD>(size), &pBuffer);
    if (FAILED(hr)) return hr;
    BYTE* pDest = nullptr;
    hr = pBuffer->Lock(&pDest, nullptr, nullptr);
    if (SUCCEEDED(hr)) {
        memcpy(pDest, data, size);
        pBuffer->Unlock();
        pBuffer->SetCurrentLength(static_cast<DWORD>(size));
    } else {
        return hr;
    }
    ComPtr<IMFSample> pSample;
    hr = MFCreateSample(&pSample);
    if (FAILED(hr)) return hr;
    hr = pSample->AddBuffer(pBuffer.Get());
    if (FAILED(hr)) return hr;
    hr = pSample->SetSampleTime(pts);
    hr = m_sinkWriter->WriteSample(m_audioStreamIndex, pSample.Get());
    return hr;
}
```

音频轨 **`SetSampleDuration`** 未设：当前 AAC MFT 仍可从 buffer 长度与 PCM 格式推断；若你以后遇到音频时钟漂移，再考虑补 duration 或按采样点数累计 PTS。

---

## 播放设备「友好名称」与稳定 ID（`audio_output_devices.cpp`）

loopback 要选「哪一块声卡/扬声器」，不能只显示 **`GetId`** 返回的长字符串——用户需要的是 **友好名称**（控制面板里那一串），程序需要的是 **`IMMDevice::GetId`** 同款 **实例路径**，传给 **`WasapiLoopback::Initialize(deviceId)`**。

### 头文件：`AudioOutputDeviceInfo`

```cpp
#pragma once

#include <string>
#include <vector>

struct AudioOutputDeviceInfo {
    std::wstring id;
    std::wstring friendlyName;
};

/** 枚举当前可用的音频播放（渲染）端点，用于 loopback 录制选择设备。须在 COM 已初始化的线程调用。 */
std::vector<AudioOutputDeviceInfo> EnumerateAudioRenderDevices();
```

- **`id`**：**`IMMDevice::GetId`**，UTF-16；与 WASAPI **`GetDevice`** / **`Initialize`** 用的是同一个 token。
- **`friendlyName`**：属性 **`PKEY_Device_FriendlyName`**，一般是人类可读名称；若属性读取失败，仓库里 **退回用 `id`** 占位，避免空白项。

### 枚举实现（与仓库一致）

```cpp
#include "audio_output_devices.h"

#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif
#ifndef NOMINMAX
#define NOMINMAX
#endif

#include <Windows.h>
#include <mmdeviceapi.h>
#include <propsys.h>
#include <Functiondiscoverykeys_devpkey.h>

#include <wrl/client.h>

#pragma comment(lib, "propsys.lib")

using Microsoft::WRL::ComPtr;

std::vector<AudioOutputDeviceInfo> EnumerateAudioRenderDevices()
{
    std::vector<AudioOutputDeviceInfo> list;

    const HRESULT coin = CoInitializeEx(nullptr, COINIT_MULTITHREADED);
    if (FAILED(coin) && coin != RPC_E_CHANGED_MODE)
        return list;
    const bool coinit_ok = SUCCEEDED(coin);

    ComPtr<IMMDeviceEnumerator> enumerator;
    HRESULT hr = CoCreateInstance(
        __uuidof(MMDeviceEnumerator),
        nullptr,
        CLSCTX_ALL,
        __uuidof(IMMDeviceEnumerator),
        enumerator.ReleaseAndGetAddressOf());
    if (FAILED(hr)) {
        if (coinit_ok)
            CoUninitialize();
        return list;
    }

    ComPtr<IMMDeviceCollection> collection;
    hr = enumerator->EnumAudioEndpoints(eRender, DEVICE_STATE_ACTIVE, collection.GetAddressOf());
    if (FAILED(hr)) {
        if (coinit_ok)
            CoUninitialize();
        return list;
    }

    UINT count = 0;
    collection->GetCount(&count);
    for (UINT i = 0; i < count; ++i) {
        ComPtr<IMMDevice> device;
        hr = collection->Item(i, device.GetAddressOf());
        if (FAILED(hr))
            continue;

        LPWSTR idRaw = nullptr;
        if (FAILED(device->GetId(&idRaw)) || !idRaw)
            continue;
        std::wstring id(idRaw);
        CoTaskMemFree(idRaw);

        ComPtr<IPropertyStore> props;
        std::wstring friendly = id;
        if (SUCCEEDED(device->OpenPropertyStore(STGM_READ, props.GetAddressOf()))) {
            PROPVARIANT pv{};
            PropVariantInit(&pv);
            if (SUCCEEDED(props->GetValue(PKEY_Device_FriendlyName, &pv)) && pv.vt == VT_LPWSTR && pv.pwszVal)
                friendly = pv.pwszVal;
            PropVariantClear(&pv);
        }

        list.push_back(AudioOutputDeviceInfo{ std::move(id), std::move(friendly) });
    }

    if (coinit_ok)
        CoUninitialize();
    return list;
}
```

由于我们必须使用CoInializeEx让当前线程初始化COM库，这里我们设置为了MTA模式，这样这个线程不仅可以调用COM对象，还可以多个线程同时访问COM对象，不需要windows来进行消息循环进行同步。

coin_ok是为了配套之后的反初始化，先不管。我们使用CoCreateInstance创建COM对象工厂，然后告诉系统创建具体的类，

CoCreateInstance成功的前提是windows音频服务必须开启，如果被禁用了那么就会返回REGDB_E_CLASSNOTREG错误。


```cpp
    ComPtr<IMMDeviceCollection> collection;
    hr = enumerator->EnumAudioEndpoints(eRender, DEVICE_STATE_ACTIVE, collection.GetAddressOf()); //colletion接收产出的“设备清单”
    if (FAILED(hr)) {
        if (coinit_ok)
            CoUninitialize();
        return list;
    }
```

我们使用EnumAudioEndpoints枚举音频端点，eRender是定义数据流向，由于我们做的是loopback回访采集，录制的是电脑发声，因此声音是流向扬声器的，所以样定位到这些渲染端点。然后开启它们的环回模式，如果我们选的是eCapture，那么就是麦克风了。

DEVICE_STATE_ACTIVE 是过滤设备状态，列出**当前已插入且可用**的设备。


```cpp
    UINT count = 0;
    collection->GetCount(&count);
    for (UINT i = 0; i < count; ++i) {
        ComPtr<IMMDevice> device;
        hr = collection->Item(i, device.GetAddressOf());
        if (FAILED(hr))
            continue;
  
        LPWSTR idRaw = nullptr;
        if (FAILED(device->GetId(&idRaw)) || !idRaw)
            continue;
        std::wstring id(idRaw);
        CoTaskMemFree(idRaw);
        ComPtr<IPropertyStore> props;
        std::wstring friendly = id;
        if (SUCCEEDED(device->OpenPropertyStore(STGM_READ, props.GetAddressOf()))) {
            PROPVARIANT pv{};
            PropVariantInit(&pv);
            if (SUCCEEDED(props->GetValue(PKEY_Device_FriendlyName, &pv)) && pv.vt == VT_LPWSTR && pv.pwszVal)
                friendly = pv.pwszVal;
            PropVariantClear(&pv);
        }  
        list.push_back(AudioOutputDeviceInfo{ std::move(id), std::move(friendly) });
    }
```

在windows的驱动模型中，设备会被抽象为一组键值对，IPropertyStore存储的是设备信息。PROPVARIANT是COM的容器，由于属性可能是字符串、整数甚至是日期，那么这个容器就必须容纳这些不同类型的数据，我们在这里通过pv.vt == VT_LPWSTR来确认他是一个宽字符字符串。PKEY_Device_FriendlyName 是索引，存放设备在控制面板里显示的名字。

使用`CoTaskMemFree(idRaw)` 去COM堆上分配一块内存并返回地址作为调用者，我们需要手动调用CoTaskMemFree来归还内存，如果不写的话就会导致内存泄漏。

最核心的就是这段代码：

```cpp
        if (SUCCEEDED(device->OpenPropertyStore(STGM_READ, props.GetAddressOf()))) {
            PROPVARIANT pv{};
            PropVariantInit(&pv);
            if (SUCCEEDED(props->GetValue(PKEY_Device_FriendlyName, &pv)) && pv.vt == VT_LPWSTR && pv.pwszVal)
                friendly = pv.pwszVal;
            PropVariantClear(&pv);
        }
```

用只读模式打开OpenPropertyStore的时候，我们用PKEY_Device_FriendlyName取值放在pv里，然后赋值完后就清理。
### 暴露给前端（Electron / N-API，简述）

**`addon.cpp`** 里把 **`id` / `friendlyName`** 用 **`WideCharToMultiByte(CP_UTF8, …)`** 填进 **`{ id, name }`** 数组；Vue 里 **`audioOutputDeviceId`** 再 UTF-8 传回 **`startRecording`**，底层 **`Utf8ToWide`** 后交给 **`Initialize`**。链条与 **`WGC捕获迁移到Electron.md`** 一致，此处不重复贴 JS。

---

## PCM 格式与 AAC 入口（和 loopback 对上）

- **`SetupAudioStreaming`**：输出 **`MFAudioFormat_AAC`**，输入 **`WAVE_FORMAT_PCM` 16-bit**（**`MFInitMediaTypeFromWaveFormatEx`**）。
- Loopback **`GetMixFormat`** 常为 float：**必须在进入 `PushAudioData` 之前** 转成 **PCM16**，并与 **`config.audioChannels` / `audioSampleRate`** 对齐（若 **`main_n.cpp`** 里用 **`GetMixFormat`** 覆盖了配置，应以设备为准）。

---

## 注册流与 `BeginWriting`（结构摘录）

**`SetupVideoStreaming`**：输出 **H.264**，输入 **RGB32（BGRA）**，设置 **bitrate、MF_MT_FRAME_SIZE、帧率、stride、PAR、nominal range** 等，然后 **`AddStream` + `SetInputMediaType`**。

**`SetupAudioStreaming`**：输出 **AAC**，输入 **PCM**；**`MF_MT_AUDIO_AVG_BYTES_PER_SECOND`** 在仓库里用 **AAC 比特率换算成的字节/秒**（**`128000/8`**），勿填 PCM 的 **`nAvgBytesPerSec`**。

**`StartRecording`**（顺序与 **`WasapiLoopback.cpp`** 一致）：**`m_useRgbMemoryPath = false`**、查 **`QPCFrequency`** → **`MFCreateAttributes`** →（上文）**`MFCreateSinkWriterFromURL`** → **`SetupVideoStreaming`** → 若 **`enableAudio`** 则 **`SetupAudioStreaming`** → **`BeginWriting`** → 创建 **staging** 纹理 → **`m_isRecording = true`**。**`StopRecording`** 会 **`Finalize`**、`Reset` SinkWriter，并把 **`m_useRgbMemoryPath`** 清回 **`false`**，下次录制重新尝试 DXGI 路径。

---

## MF 模块与历史表（略作校正）

- **`mfplat.dll`**：Buffer、Sample、属性袋等底座。
- **`mfreadwrite.dll`**：**`IMFSinkWriter` / `IMFSourceReader`**，简化拓扑拼装。
- **MFT**：编码/解码/处理可由系统或显卡驱动注册；**硬件 MFT** 是否参与仍受 **`MF_READWRITE_ENABLE_HARDWARE_TRANSFORMS`** 与拓扑影响。

操作系统支持表你原文可保留：**Vista 引入，Win7 强化 H.264 + SinkWriter，Win10/11 现状标准**。

---

## 与 WGC / WinRT 的配合（按需）

**`winrt::com_ptr`** 与 **`WRL::ComPtr`**：WGC 帧常落在 **`winrt::com_ptr<ID3D11Texture2D>`**，交给 MF 时 **`texture.get()`** 取裸指针即可；不必强行纠结「现代化」与否，边界清晰就好。

---


