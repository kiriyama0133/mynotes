# COM 组件与 Windows 编程（加长版）

面向：**几乎没写过 Win32 / C++ 原生**，但听说过 **COM、ActiveX、DLL 注册** 的同学。本文尽量 **拓宽视野**：不只围绕某一个录屏工程，而是把 **桌面右键菜单、Office 自动化、Shell、拖拽** 等典型 COM 场景串起来；代码片段以 **示意 + 可对照 MSDN 补全** 为主。

---

## 〇、读完本篇你能做什么判断？

- 看到 **`HRESULT`**，知道要用 **`SUCCEEDED`/`FAILED`**，并能粗略读出 **Facility**（失败来自 Win32、COM 还是 RPC）。
- 理解 **`IUnknown`** 三根支柱：**QueryInterface / AddRef / Release**，以及为何 **`ComPtr`** 能救命。
- 知道 **`CoInitializeEx` / `CoCreateInstance`** 在干什么，**STA 与 MTA** 何时会踩 **`RPC_E_CHANGED_MODE`**。
- 明白 **壳扩展（Shell Extension）** 如何通过 **注册表 CLSID + Inproc DLL** 接到资源管理器右键菜单。
- 知道 **IDispatch / BSTR** 在 **脚本与 Office 自动化** 里扮演什么角色。

---

## 一、COM 要解决什么问题？

早年同一台 Windows 上要共存：**C++ 写的内核插件、VB 写的表单、脚本宿主、Office**。若每种语言都为每一种别的语言手写绑定，组合爆炸。**组件对象模型（COM）** 的核心约定可以概括成：

1. **二进制接口契约**：只规定 **vtable 布局 + 调用约定**，不关心实现语言。
2. **全局唯一名字**：接口用 **IID**，组件类用 **CLSID**，都是 **GUID**。
3. **统一生命周期**：对象何时销毁由 **引用计数** 决定；所有接口继承 **`IUnknown`**。

因此：**COM ≠「微软专属的 C++ 语法」**，而是一套 **操作系统级的组件约定**。你今天接触的 **WASAPI、DirectX、Media Foundation、Shell、Office** 大量 API 都是 COM 接口。

---

## 二、接口、vtable、`IUnknown`（务必建立心智图像）

### 2.1 虚函数表（vtable）

一个 COM 接口在 C++ 里常写成 **`struct IXxx : IUnknown { virtual HRESULT Foo() = 0; };`**。编译器会为对象生成 **vtable 指针**：对象内存layout开头往往是 **`vtable*`**，指向函数指针数组。

```
对象实例内存（示意）           vtable 指向的函数槽位
+------------------+          +------------------+
| vtable* --------+|---------> | slot0: QueryInterface
| 成员字段 ...      |          | slot1: AddRef
+------------------+          | slot2: Release
                              | slot3: Foo ...
                              +------------------+
```

**QueryInterface**：同一个物理对象，若实现了多个接口，通常有多张 vtable（多重继承），或共享 **`IUnknown`** 实现。**客户端只通过接口指针调用**，不必知道 C++ 类具体怎么继承。

### 2.2 `IUnknown` 三方法（所有 COM 接口的根）

```cpp
struct IUnknown {
    HRESULT QueryInterface(REFIID riid, void** ppvObject);
    ULONG AddRef();
    ULONG Release();
};
```

**QueryInterface**：输入 **IID**，若对象支持该接口，则 **`*ppvObject = 指向合法接口指针`**，并 **`AddRef`**（规范要求）；不支持返回 **`E_NOINTERFACE`**。

```cpp
void ExampleQuery(IAudioClient* pClient)
{
    IUnknown* pUnk = nullptr;
    if (FAILED(pClient->QueryInterface(IID_IUnknown, reinterpret_cast<void**>(&pUnk))))
        return;
    // 通常 QI 同对象的 IUnknown 会得到同一逻辑实体；调试引用计数时会用到
    pUnk->Release();
}
```

**AddRef / Release**：跨 DLL 边界传递指针时，文档常会写「接收方 AddRef」「调用完 Release」。**规则**：谁延长生命周期谁 **AddRef**；配对 **Release**。**裸指针忘记 Release** → 泄漏；**二次 Release** → 崩溃。

---

## 三、GUID、CLSID、ProgID、注册表（扫一眼地图）

| 名称 | 含义 |
|------|------|
| **IID** | 接口 ID，例如 **`IID_IUnknown`** |
| **CLSID** | 组件类 ID，**`CoCreateInstance` 第一个参数** |
| **LIBID** | 类型库 ID |
| **ProgID** | 人类可读字符串，如 **`Excel.Application`**，可用 **`CLSIDFromProgID`** 换 CLSID |

**注册表**（概念路径）：

- **`HKCR\CLSID\{xxxxxxxx-xxxx-...}`**：某个组件类的配置。
- **`<CLSID>\InprocServer32`**：进程内 DLL 路径 + **`ThreadingModel`**（`Apartment` / `Free` / `Both` / `Neutral`）。
- **`<CLSID>\LocalServer32`**：本地 EXE 服务器路径。

**`CoCreateInstance(CLSID, ..., IID, ppv)`** 做的事：查注册表 → 加载 DLL 或启动 EXE → 通过 **类对象（class object）** 创建实例 → **`QueryInterface` 成你要的接口**。

---

## 四、`HRESULT`：宁可当成「带类型的错误码」

### 4.1 判定方式（再强调）

```cpp
HRESULT hr = SomeMethod();
if (FAILED(hr)) {
    // 失败路径：可用 FormatMessage、_com_error，或调试器看 hex
}
if (SUCCEEDED(hr)) {
    // 成功路径：包含 S_OK、S_FALSE（少数 API）
}
```

### 4.2 位域粗解（入门够用）

高位 **bit31=1** 通常表示失败（**`FAILED`**）；中间 **Facility** 区分错误来源（COM、Win32、RPC 等）；低位 **Code** 是错误编号。

### 4.3 常见 HRESULT（混个脸熟）

| HRESULT | 含义（简述） |
|---------|----------------|
| **`S_OK`** | 成功 |
| **`S_FALSE`** | 成功但布尔语义为「假」 |
| **`E_NOINTERFACE`** | QueryInterface 不支持该 IID |
| **`E_POINTER`** | 空指针参数 |
| **`E_OUTOFMEMORY`** | 内存不足 |
| **`REGDB_E_CLASSNOTREG`** | CLSID 未注册 |
| **`CO_E_NOTINITIALIZED`** | 线程未 CoInitialize |
| **`RPC_E_CHANGED_MODE`** | 本线程 COM 已按另一种 apartment 初始化 |
| **`E_ACCESSDENIED`** | 权限不足 |

把 **`HRESULT`** 当成 **`int`** 随意算术运算，一般是错的。

---

## 五、线程与 Apartment（STA / MTA / Neutral）

### 5.1 为什么要有 Apartment？

早期大量 COM 对象内部 **不是线程安全的**，却又要被多线程桌面调用。**Apartment** 大致是说：对象愿意运行在哪种线程语义里；跨 apartment 调用 COM 可能插入 **proxy/stub** 做 **marshalling**（参数打包到另一线程）。

### 5.2 两种最常念叨的模式

- **STA（Single-Threaded Apartment）**：常与 **UI 线程 + 消息泵** 绑定；不少 Shell / Office 相关对象偏 STA。
- **MTA（Multi-Threaded Apartment）**：**`COINIT_MULTITHREADED`**；并发更高，但对象自己要线程安全。

### 5.3 `CoInitializeEx` 典型写法

```cpp
HRESULT hr = CoInitializeEx(nullptr, COINIT_MULTITHREADED);
if (hr == RPC_E_CHANGED_MODE) {
    // 线程已被别的库初始化成 STA：要么接受现状，要么专门新建线程做 COM
} else if (FAILED(hr)) {
    return hr;
}
// ...
CoUninitialize(); // 须与成功初始化的路径配对
```

### 5.4 「转换 / 封送」入门一句

跨 apartment、跨进程调用时，COM 可能把你的调用转到 **代理对象**，参数通过 **RPC / LRPC** 传到另一边的 **stub**。初学者不必手写这些；只要记住：**接口指针不能盲目跨线程乱丢**，要看文档是否提供 **Marshal API** 或是否在 MTA 内安全。

---

## 六、创建对象的两条路：`CoCreateInstance` 与类工厂

### 6.1 `CoCreateInstance`（90% 场景够用）

```cpp
Microsoft::WRL::ComPtr<IMessageFilter> filter; // 举例任意 COM 接口
HRESULT hr = CoCreateInstance(
    CLSID_StdMessageFilter,   // 哪个组件类
    nullptr,                   // pUnkOuter，聚合才用
    CLSCTX_INPROC_SERVER,      // 进程内 DLL；也可用 CLSCTX_LOCAL_SERVER 等
    IID_IMessageFilter,
    reinterpret_cast<void**>(filter.GetAddressOf()));
```

### 6.2 `IClassFactory`（心里要有这张图）

当系统加载你的 DLL 时，不是凭空 **`new`**，而是：

1. **`DllGetClassObject(CLSID, IID_IClassFactory, ...)`** 返回类工厂；
2. **`IClassFactory::CreateInstance`** 真正 **`new` 你的对象** 并 **`QueryInterface`** 成客户端要的接口。

示意：

```cpp
// 下面不是完整可运行代码，仅展示逻辑链条
class CFooFactory final : public IClassFactory {
public:
    HRESULT STDMETHODCALLTYPE CreateInstance(IUnknown* pOuter, REFIID riid, void** ppv) override {
        if (pOuter != nullptr && riid != IID_IUnknown)
            return CLASS_E_NOAGGREGATION;
        auto* obj = new(std::nothrow) CFoo();
        if (!obj) return E_OUTOFMEMORY;
        HRESULT hr = obj->QueryInterface(riid, ppv);
        obj->Release(); // 构造函数里通常为 1，QI 会 AddRef，这里 Release 工厂持有的初始计数—具体依实现而定
        return hr;
    }
    // LockServer / Release / QueryInterface … 省略
};
```

进程内 COM DLL 通常还要导出 **`DllRegisterServer` / `DllUnregisterServer`** 写注册表，资源管理器才能找到你的 **CLSID**。

---

## 七、`ComPtr`（WRL）与智能指针纪律

```cpp
#include <wrl/client.h>
using Microsoft::WRL::ComPtr;

ComPtr<IStream> spStream;
HRESULT hr = SHCreateStreamOnFileEx(L"C:\\temp.bin", STGM_READWRITE | STGM_CREATE,
    FILE_ATTRIBUTE_NORMAL, TRUE, nullptr, &spStream);
// spStream 析构时 Release
```

常用：

- **`Get()`**：**`T*`**
- **`GetAddressOf()`**：**`T**`**，用于输出参数
- **`Attach(p)`**：接管已有裸指针（不再自动 AddRef，除非你明确知道所有权）
- **`Detach()`**：交出裸指针并清空智能指针

---

## 八、BSTR、`VARIANT`、自动化（IDispatch）

### 8.1 `BSTR`：带长度前缀的宽字符串

COM 自动化里字符串常用 **`BSTR`**（**`SysAllocString` / `SysFreeString`**），不是普通的 **`wchar_t*`**。

```cpp
#include <comutil.h>
#pragma comment(lib, "comsuppw.lib")

BSTR bs = SysAllocString(L"Hello");
if (!bs) { /* OOM */ }
// 传给接口...
SysFreeString(bs);
```

### 8.2 `VARIANT`：「万能联合体」

脚本里 **`Variant`** 对应 **`VARIANT`**：`vt` 字段区分 **`VT_I4`、`VT_BSTR`、`VT_DISPATCH`**…

```cpp
VARIANT v{};
v.vt = VT_I4;
v.lVal = 42;

VARIANT dest{};
VariantInit(&dest);
HRESULT hr = VariantCopy(&dest, &v);
VariantClear(&dest);
VariantClear(&v);
```

### 8.3 `IDispatch`：迟绑定（脚本、VBA）

脚本调用 **`obj.SaveAs(...)`** 时，往往走 **`IDispatch::Invoke`**，通过 **DISPID** 找方法。Office **自动化**典型路径：**`CoCreateInstance(CLSID_Excel_Application,...)` → `IDispatch`**。

极简伪代码（真实项目请用 **#import** 或封装库）：

```cpp
ComPtr<IDispatch> app;
CoCreateInstance(CLSID_Excel_Application, nullptr, CLSCTX_LOCAL_SERVER,
    IID_IDispatch, reinterpret_cast<void**>(app.GetAddressOf()));

DISPID dispid = 0;
OLECHAR* name = const_cast<OLECHAR*>(L"Quit");
HRESULT hr = app->GetIDsOfNames(IID_NULL, &name, 1, LOCALE_USER_DEFAULT, &dispid);

DISPPARAMS dp{};
VARIANT ret{};
hr = app->Invoke(dispid, IID_NULL, LOCALE_USER_DEFAULT, DISPATCH_METHOD, &dp, &ret, nullptr, nullptr);
```

---

## 九、桌面右键菜单：Shell 扩展（COM 的经典秀场）

资源管理器右键菜单上的「压缩」「杀毒扫描」「Git Bash Here」等，大量通过 **Shell 扩展 DLL** 实现：**进程内 COM 服务器**，实现 **`IShellExtInit` + `IContextMenu`**（或 **`IExplorerCommand`**，Win11 新叙事）。

### 9.1 用户能看到的效果链路

1. 用户在 **`*.txt` 文件**上右键。
2. Shell 查询注册表 **`HKCR\*\shellex\ContextMenuHandlers\YourHandler`**（或指定 ProgID 下）。
3. 默认值是你的 **CLSID**。
4. Shell **`CoCreateInstance`** 加载 **`YourHandler.dll`**，拿到 **`IContextMenu`**。
5. Shell 调 **`QueryContextMenu`** 让你插入菜单项；用户点击后 **`InvokeCommand`**。

### 9.2 注册表（示意，手工改请先备份）

下面这段 **`.reg` 思想**说明「把 CLSID 接到全局文件右键」的路径（具体 GUID 请换成你自己的）：

```reg
Windows Registry Editor Version 5.00

; 假设 CLSID = {11111111-1111-1111-1111-111111111111}

[HKEY_CLASSES_ROOT\CLSID\{11111111-1111-1111-1111-111111111111}]
@="My Context Menu Handler Sample"

[HKEY_CLASSES_ROOT\CLSID\{11111111-1111-1111-1111-111111111111}\InprocServer32]
@="C:\\Path\\To\\MyShellExt.dll"
"ThreadingModel"="Apartment"

[HKEY_CLASSES_ROOT\*\shellex\ContextMenuHandlers\MySample]
@="{11111111-1111-1111-1111-111111111111}"
```

**ThreadingModel**：壳扩展历史上常用 **`Apartment`**（STA）。实现 **`IShellExtInit`** 时 Shell 会把 **`IDataObject`（选中文件）**、**PIDL** 等给你。

### 9.3 接口骨架（教育向精简版）

```cpp
#include <shlobj.h>
#include <shobjidl.h>

class __declspec(uuid("11111111-1111-1111-1111-111111111111"))
    CMyContextMenu final : public IShellExtInit, public IContextMenu {
public:
    // IUnknown
    HRESULT STDMETHODCALLTYPE QueryInterface(REFIID riid, void** ppv) override {
        if (IsEqualIID(riid, IID_IUnknown))
            *ppv = static_cast<IShellExtInit*>(this);
        else if (IsEqualIID(riid, IID_IShellExtInit))
            *ppv = static_cast<IShellExtInit*>(this);
        else if (IsEqualIID(riid, IID_IContextMenu))
            *ppv = static_cast<IContextMenu*>(this);
        else {
            *ppv = nullptr;
            return E_NOINTERFACE;
        }
        reinterpret_cast<IUnknown*>(*ppv)->AddRef();
        return S_OK;
    }
    ULONG STDMETHODCALLTYPE AddRef() override { return ++ref_; }
    ULONG STDMETHODCALLTYPE Release() override {
        ULONG c = --ref_;
        if (!c) delete this;
        return c;
    }

    // IShellExtInit
    HRESULT STDMETHODCALLTYPE Initialize(PCIDLIST_ABSOLUTE pidlFolder, IDataObject* pdtobj,
        HKEY hkeyProgID) override {
        // 保存 pdtobj 里文件列表：可以用 CF_HDROP 解析路径（Shell API）
        return S_OK;
    }

    // IContextMenu
    HRESULT STDMETHODCALLTYPE QueryContextMenu(HMENU hmenu, UINT indexMenu, UINT idCmdFirst,
        UINT idCmdLast, UINT uFlags) override {
        InsertMenuW(hmenu, indexMenu, MF_STRING | MF_BYPOSITION, idCmdFirst, L"我的示例菜单");
        return MAKE_HRESULT(SEVERITY_SUCCESS, 0, 1); // 占用 1 个命令 ID
    }

    HRESULT STDMETHODCALLTYPE InvokeCommand(LPCMINVOKECOMMANDINFO pici) override {
        MessageBoxW(nullptr, L"菜单被点击", L"COM Shell 扩展", MB_OK);
        return S_OK;
    }

    HRESULT STDMETHODCALLTYPE GetCommandString(UINT_PTR idCmd, UINT uFlags, UINT* pwReserved,
        CHAR* pszName, UINT cchMax) override {
        return E_NOTIMPL;
    }

private:
    ULONG ref_ = 1;
};
```

**DllGetClassObject** 返回 **`IClassFactory`**，由 **`CreateInstance`** 生成上面的对象——完整工程还需 **`DllMain`、`DllCanUnloadNow`**、**x64 与 Shell 位数一致**（64 位资源管理器只加载 64 位扩展）。

### 9.3.1 进程内 COM DLL 必须导出的符号（示意）

```cpp
// stdafx / 模块全局
STDAPI DllGetClassObject(REFCLSID rclsid, REFIID riid, LPVOID* ppv)
{
    if (!IsEqualCLSID(rclsid, CLSID_MyContextMenu))
        return CLASS_E_CLASSNOTAVAILABLE;
    // 常见写法：实现 CClassFactory::QueryInterface(riid, ppv) 返回 IClassFactory
    *ppv = nullptr;
    return g_pClassFactory->QueryInterface(riid, ppv);
}

STDAPI DllCanUnloadNow()
{
    // 若模块内活动对象计数为 0，返回 S_OK，允许 OLE 卸载 DLL
    return g_serverLocks == 0 && g_objCount == 0 ? S_OK : S_FALSE;
}

STDAPI DllRegisterServer()
{
    // RegCreateKeyEx 写入 HKCR\CLSID\{...}\InprocServer32 …
    return S_OK;
}

STDAPI DllUnregisterServer()
{
    // 删除注册表项
    return S_OK;
}
```

实际工程要用 **`regsvr32 MyShellExt.dll`** 注册（管理员权限视路径而定），卸载 **`regsvr32 /u`**。

### 9.4 `IExplorerCommand`（Windows 10 1809+）

微软在新示例里推广 **`IExplorerCommand`**（配合 **`IInitializeWithStream`** 等），模型仍是 **COM**，但注册位置与 **`Shell.ExtensionRegistration`** manifest 有关——思路与 **`IContextMenu`** 类似：**实现接口 → 注册 CLSID → Shell 宿主 CoCreate**。

---

## 十、其它常见 COM 版图（拓宽视野）

### 10.1 拖拽与剪贴板：`IDataObject`、`IDropTarget`

OLE 拖拽核心是 **`IDataObject` + `IDropSource` / `IDropTarget`**，全是 COM。写过 **`DoDragDrop`** 的同学已经在用 COM。

### 10.2 ActiveX / 浏览器插件（历史包袱）

旧 IE **`IBrowserPlugin`** 生态建立在 COM 上；现代浏览器路径已换道，但遗留系统维护仍会碰到。

### 10.3 WMI（`IWbemServices`）

脚本 **`GetObject("winmgmts:")`** 底层也是 COM；C++ 可用 **WMI COM API** 查询进程、磁盘。

### 10.4 DirectX / WASAPI / Media Foundation

图形与音视频栈大量使用 COM：**`ID3D11Device`**、**`IAudioClient`**、**`IMFSinkWriter`**……与本笔记前文一致：**`ComPtr` + HRESULT**。

---

## 十一、调试与工具（知道名字即可）

- **`OLE/COM Object Viewer (OleView.exe)`**：浏览注册了的 CLSID、接口。
- **Process Monitor**：看 Shell 是否成功加载你的 DLL、路径是否错误。
- **日志**：对 **`HRESULT`** 打印 **十六进制**，对照 WinError.h / ntstatus。

---

## 十二、与「录屏工程」如何对上号（可选阅读）

若你同时在写一个 **WGC + WASAPI + MF** 工具：**线程里 `CoInitializeEx`**、**`CoCreateInstance(MMDeviceEnumerator)`**、**`MFCreateSinkWriterFromURL`**，全部都是本篇套路的子集；**没有出现 Shell 扩展**，不代表 COM 只能干那些事——**右键菜单只是 COM 桌面生态里最容易讲的一个故事**。

---

## 十三、推荐阅读顺序（自学）

1. Microsoft Learn：**「Introduction to COM」** → **「Managing Memory Allocation」**（引用计数）。
2. 《Inside COM》（旧书，概念仍清晰）——建立 mental model。
3. MSDN：**Shell Context Menu Handler** 官方示例（结合本文第 9 节）。
4. 需要 Office 自动化再深入 **`IDispatch` / 双重接口**。
5. Don Box《Essential COM》（绝版但扫描版仍可淘）——比官方文档「讲故事」。
6. 《Professional Visual Studio 2022》类书里 COM 章节——偏实操注册表与 ATL。

---

# 第二卷：深度扩展（可与上文穿插阅读）

下列章节独立编号，专门回答：**HRESULT 怎么拆**、**引用计数有哪些坑**、**聚合与连接点是什么**、**IDL/MIDL 流水线**、**Shell 里如何从 IDataObject 拿出文件路径**、**WinRT 与经典 COM 边界**、**ATL 帮你省了哪些样板代码**、**FAQ**。篇幅不设上限，你可按需跳着读。

---

## 十五、`HRESULT` 手工解码与 Facility 一览（进阶）

### 15.1 位域（记住这张图就够应付面试）

32 位 **`HRESULT`** 常见划分（概念）：

- **Bit 31**：严重位；**1** 通常表示失败（与 **`FAILED` 宏一致）。
- **Bits 30–16**：保留 / 客户位（部分 HRESULT 用法）。
- **Bits 15–0**：**Code**，具体错误号。
- **中间若干位**：**Facility**，区分错误「出自谁家」。

实务上不要手算：**打印十六进制 `printf("hr=0x%08X\n", hr)`**，再用文档或 **`FormatMessage`**。

### 15.2 常见 Facility（混脸熟即可）

| Facility 值（节选） | 大致含义 |
|---------------------|-----------|
| **FACILITY_NULL** | 通用 |
| **FACILITY_RPC** | RPC 调用失败 |
| **FACILITY_WIN32** | Win32 错误包装（常见 **`HRESULT_FROM_WIN32(ERROR_FILE_NOT_FOUND)`**） |
| **FACILITY_ITF** | 接口自定义错误区间 |
| **FACILITY_WINDOWS** | Shell / WinRT 相关 |

把 **`Win32` 错误** 包成 **`HRESULT`**：

```cpp
HRESULT hr = HRESULT_FROM_WIN32(GetLastError());
```

反向：**`HRESULT_CODE(hr)`**、**`HRESULT_FACILITY(hr)`**（宏定义在 **`winerror.h`**）。

### 15.3 `IErrorInfo` / `_com_error`（拿到「可读字符串」）

不少 COM 服务器在失败时会附加 **`IErrorInfo`**（支持 **`ISupportErrorInfo`**）。在包含 **`comdef.h`** 的场景可用 **`_com_error`**：

```cpp
#include <comdef.h>
try {
    // 调用 #import 生成的智能包装可能抛 _com_error
} catch (_com_error& e) {
    const wchar_t* desc = e.ErrorMessage(); // 人类可读
    HRESULT hr = e.Error();
}
```

纯 C++/WRL 手写时，失败路径往往只有 **`HRESULT`**，要学会 **`FormatMessageW(FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_FROM_HMODULE, ...)`** 或自建映射表。

---

## 十六、引用计数的「十诫」与典型翻车现场

### 16.1 十诫（口语版）

1. **每次接口指针所有权转移**，想清楚：**要不要 AddRef**。
2. **`QueryInterface` 成功**：返回的指针 **已 AddRef**（对你来说是 +1）。
3. **`CoCreateInstance` 成功**：返回的指针 **已 AddRef**。
4. **函数输出参数 `[out]`**：文档若写 *caller must Release*，你必须 **Release**。
5. **函数 `[in]` 接口指针**：通常 **不接管所有权**（不调 Release），除非你额外 AddRef 保存。
6. **不要把栈上临时对象的接口指针异步传到别的线程**——对象死了指针悬空。
7. **`ComPtr::Attach`**：接管 **已有引用** 的裸指针；若指针已是「偷来的」要数清楚。
8. **循环引用**：父持有子、子回调父，两边 **`ComPtr`** 互相指 → **泄漏**；典型解法：**弱引用 / 显式断开 / 不要双向 ComPtr**。
9. **全局/单例 COM 对象**：进程退出顺序要和 **`CoUninitialize`** 对齐。
10. **调试**：怀疑泄漏时用 **Application Verifier** 或在 **`Release` 里打日志**（仅调试版）。

### 16.2 翻车故事 A：多一次 Release

```cpp
IUnknown* p = nullptr;
CoCreateInstance(..., (void**)&p); // 计数 = 1
p->Release(); // 0，对象销毁
p->Release(); // 💥 双重释放
```

### 16.3 翻车故事 B：QueryInterface 忘了 Release

```cpp
IAudioClient* ac = nullptr;
device->Activate(..., (void**)&ac); // 假设计数已 +1
IUnknown* unk = nullptr;
ac->QueryInterface(IID_IUnknown, (void**)&unk); // unk +1，此时同一对象计数更高
// 只用 unk，忘了 Release ac → 泄漏一种路径
```

原则：**从一个入口拿到指针后，理清「这条所有权链」**，宁可 **ComPtr** 全自动。

---

## 十七、`CLSCTX`：你到底想在哪创建对象？

**`CoCreateInstance`** 的 **`dwClsContext`** 常见组合：

```cpp
CoCreateInstance(CLSID_Foo, nullptr,
    CLSCTX_INPROC_SERVER | CLSCTX_LOCAL_SERVER, // 允许进程内 DLL 或本机 EXE
    IID_IFoo, ...);
```

| 标志 | 含义 |
|------|------|
| **`CLSCTX_INPROC_SERVER`** | DLL **进程内**（最快；右键菜单扩展多数如此） |
| **`CLSCTX_LOCAL_SERVER`** | **本地 EXE** COM 服务器（Excel 常见） |
| **`CLSCTX_REMOTE_SERVER`** | DCOM 远程（需额外安全配置） |
| **`CLSCTX_ALL`** | 「你能用的都来试试」——初学者慎用 |

**位数**：64 位 Shell 只能加载 **64 位** **`InprocServer32`**；混用是右键菜单「装了但不出现」的经典原因之一。

---

## 十八、聚合（Aggregation）：外层包装内层对象

有时你要实现 **`IX`**，内部委托 **`IY`** 给另一个 COM 对象；规范里 **`pOuter`** 只有 **`IUnknown`** 能用。**`CoCreateInstance`** 的 **`pUnkOuter`** 非空即进入聚合路径。**初学者**：除非你维护 Shell / ActiveX 宿主，先 **知道名词即可**，不要一上来手写聚合。

---

## 十九、连接点（Connection Points）：COM 里的「事件」

脚本 **`Dim WithEvents`**、浏览器 **`attachEvent`** 的老祖宗之一是：

- 源对象实现 **`IConnectionPointContainer`** → **`EnumConnectionPoints`** → **`IConnectionPoint`**。
- 客户端实现 **`IDispatch`**（或自定义接收接口）→ **`Advise`** 订阅。
- 触发事件 → **`IConnectionPoint::Invoke`** 调客户端。

示意（极度精简）：

```cpp
// 客户端伪代码
DWORD cookie = 0;
pConnectionPoint->Advise(pMySinkIUnknown, &cookie);
// ...
pConnectionPoint->Unadvise(cookie);
```

现实项目：**Office VBA 事件**、**IE WebBrowser 控件事件**、旧 **MSXML** 异步回调，底层常有连接点身影。

---

## 二十、IDL / MIDL / 类型库：从文本接口到代理桩代码

### 20.1 IDL 文件是什么？

**接口定义语言（IDL）** 描述：**UUID**、方法、参数方向 **`[in]`/`[out]`/`[in,out]`**、**`HRESULT`** 返回。 **`midl.exe`** 编译生成：

- **`xxx_h.h`**：C/C++ 声明；
- **`xxx_i.c`**：IID 定义；
- **`xxx_p.c` / dlldata.c**：代理/桩（跨进程时需要）。

### 20.2 类型库 `.tlb`

**`MIDL /tlb`** 或 **`LoadTypeLib`** 加载。**VBA / .NET** 早期常 **`#import "xxx.tlb"`**。手动浏览可用 **`OleView.exe`** 打开 **Type Library**。

### 20.3 `#import` 生成 TLH/TLI（ MSVC ）

```cpp
#import "C:\\path\\foo.dll" named_guids raw_interfaces_only
```

会生成 **智能指针包装**，也可能引入 **`_com_error`**——大型团队往往 **只 import 必要组件**，否则编译时间和命名污染可观。

---

## 二十一、结构化存储：`IStorage` / `IStream`（复合文档）

老式 **`.doc`（非 docx）**、部分安装包使用 **结构化存储**：目录树 **`IStorage`**，叶子二进制 **`IStream`**，全是 COM 接口。理解这一点有助于读 **Office 自动化**、**拖放文件格式**。现代 **OpenXML（docx/xlsx）** 是 ZIP + XML，不靠这一套，但遗留系统仍在。

---

## 二十二、Moniker：名字即绑定（入门只需印象）

**`IMoniker`** 表示「可解析的名字」：**文件路径 moniker**、**反引用 moniker**、**复合 moniker**。**`BindToObject`** 一路解析到真正的 COM 对象。脚本 **`GetObject("winmgmts:\\\\.\\root\\cimv2")`** 本质也在玩 naming/binding。初学不必深入 **ROT（Running Object Table）**，知道 **「名字→对象」这条链存在** 即可。

---

## 二十三、DCOM：跨机器 COM（为什么你现在见得少）

DCOM 依赖 **RPC**、**防火墙端口**、**身份模拟级别（Impersonation）**、**Launch/Activation 权限**。容器与微服务时代，很多团队改用 **HTTP/gRPC**，DCOM 多见于 **遗留工业软件**。若遇见：**学会用 DCOMCNFG.EXE** 看权限。

---

## 二十四、WinRT 与经典 COM：`IInspectable`、激活工厂

**Windows Runtime** 接口继承 **`IInspectable`**（不是 **`IUnknown`** 直接暴露给开发者那么简单）。**`RoInitialize` / `RoActivateInstance`**（**`Windows.Foundation.Uri`** 一类）背后仍有 **COM RCW** 思想的影子。**C++/WinRT** 生成的 **`winrt::Windows::Foo`** 类型隐藏了 **`QueryInterface`** 细节。

对照表（口语）：

| 经典 COM | WinRT 投影 |
|-----------|-------------|
| **`CoCreateInstance`** | **`RoActivateInstance` / get_activation_factory** |
| **`IUnknown`** | **`IInspectable` + 元数据（.winmd）** |
| **注册表 CLSID** | **打包清单 + 进程内激活** |

---

## 二十五、ATL 画像：为什么老牌 Shell 扩展示例都爱用它

**Active Template Library（ATL）** 提供：

- **`CComObjectRootEx`**：线程模型与 **`IUnknown`** 样板；
- **`CComCoClass`**：类工厂胶水；
- **`BEGIN_OBJECT_MAP` / `OBJECT_ENTRY_AUTO`**：自动 **`DllGetClassObject`**；
- **`IDispatchImpl`**：双接口 **`IDispatch`**。

极简注册思路（示意，不是完整工程）：

```cpp
class ATL_NO_VTABLE CMyMenu
    : public CComObjectRootEx<CComSingleThreadModel>,
      public CComCoClass<CMyMenu, &CLSID_MyMenu>,
      public IShellExtInit,
      public IContextMenu {
public:
    DECLARE_REGISTRY_RESOURCEID(IDR_MYMENU)
    BEGIN_COM_MAP(CMyMenu)
        COM_INTERFACE_ENTRY(IShellExtInit)
        COM_INTERFACE_ENTRY(IContextMenu)
    END_COM_MAP()
    // ...
};
```

**OBJECT_ENTRY_AUTO(CLSID_MyMenu, CMyMenu)** 把「CLSID → C++ 类」塞进映射表，省去手写 **`DllGetClassObject`** 分支。**初学**：可先 **抄官方 Sample**，再对比「手写 `QueryInterface`」版本，体会样板减少了什么。

---

## 二十六、Shell 进阶：从 `IDataObject` 解析选中文件（`CF_HDROP`）

右键菜单扩展真正有用时，多半要拿到 **用户选中了哪些路径**。**`IShellExtInit::Initialize`** 给你 **`IDataObject*`**；常见格式 **`CF_HDROP`**。

```cpp
#include <shellapi.h>

HRESULT ExtractFirstPath(IDataObject* pdtobj, std::wstring& outPath)
{
    FORMATETC fe = { CF_HDROP, nullptr, DVASPECT_CONTENT, -1, TYMED_HGLOBAL };
    STGMEDIUM stg{};
    HRESULT hr = pdtobj->GetData(&fe, &stg);
    if (FAILED(hr))
        return hr;

    HDROP hDrop = static_cast<HDROP>(GlobalLock(stg.hGlobal));
    if (!hDrop) {
        ReleaseStgMedium(&stg);
        return E_FAIL;
    }

    wchar_t buf[MAX_PATH]{};
    UINT n = DragQueryFileW(hDrop, 0xFFFFFFFF, nullptr, 0); // 文件个数
    if (n > 0) {
        DragQueryFileW(hDrop, 0, buf, MAX_PATH);
        outPath = buf;
    }
    GlobalUnlock(stg.hGlobal);
    ReleaseStgMedium(&stg);
    return S_OK;
}
```

**解释**：

- **`FORMATETC`**：描述「我要哪一种剪贴板格式」。
- **`STGMEDIUM`**：实际介质可能是 **全局内存 `HGLOBAL`**、流、存储等。
- **`DragQueryFile`**：经典 Shell API，读 **`HDROP`**。

若要多选文件：**循环 `DragQueryFileW(hDrop, index, ...)`**。若 **`CF_HDROP` 不存在**（某些虚拟对象），可能要 **`IShellItemArray`** / **`CFSTR_SHELLIDLIST`**——那是下一档难度。

---

## 二十七、拖拽再放：`DoDragDrop` 双方在干什么？

**源侧**：实现 **`IDropSource`** + **`IDataObject`**（提供 **`CF_TEXT`/`CF_HDROP`** 等）。  
**目标侧**：实现 **`IDropTarget`**：**`DragEnter`/`DragOver`/`Drop`**。  
**`DoDragDrop`** 内部跑消息循环协调——全是 COM。写过 **桌面拖拽上传** 的同学，其实已经做过半个 COM 教程。

---

## 二十八、`SAFEARRAY`：VB/VBA 数组如何越过边界

自动化接口有时参数是 **`SAFEARRAY`**。**`SafeArrayCreate` / `SafeArrayAccessData` / `SafeArrayUnaccessData` / `SafeArrayDestroy`**。跨语言边界传数组几乎必碰。**初学者**：优先用 **封装库**；手写时注意 **`VT_ARRAY | VT_UI1`** 等与 **`VARIANT`** 的组合。

---

## 二十九、长篇 FAQ（摘录）

**问：我是不是每个线程都要 `CoInitializeEx`？**  
答：**STA/MTA 以线程为单位**。工作线程要用 COM，就在该线程初始化（或专门搞 **COM 线程** 其它线程 **`PostMessage`/`QueueUserAPC`** 转发）。

**问：`CoInitialize(nullptr)` 和 `CoInitializeEx` 区别？**  
答：**`CoInitialize`** 等价旧版默认 STA；新项目请 **显式 `CoInitializeEx`**。

**问：我能在线程池里随便 `CoCreateInstance` 吗？**  
答：**可以，但要与该线程的 COM 初始化一致**；对象是否线程安全看文档。

**问：右键菜单扩展注册了为什么不显示？**  
答：检查 **位数（x64/x86）**、**ThreadingModel**、**regsvr32 是否成功**、**HKCR 权限**、是否被 **Shell 缓存**（有时需重启 explorer）。

**问：COM 和 .NET 互操作？**  
答：**RCW**（.NET 调 COM）、**CCW**（COM 调 .NET）。**Marshal.ReleaseComObject** 坑极多——官方文档专门警告。

**问：WinRT 还要学 COM 吗？**  
答：**底层原理相通**；日常 WinRT 开发可少碰 **`IUnknown`**，但 **排查激活失败 / 权限 / ABI** 时经典 COM 知识仍值钱。

---

## 三十、术语表（扩展）

| 术语 | 一句话 |
|------|--------|
| **Proxy/Stub** | 跨 apartment / 进程时的替身接口 |
| **Marshalling** | 把参数打包交给另一边 |
| **ROT** | Running Object Table：运行中对象登记簿 |
| **APTTYPE** | Apartment 类型枚举（调试诊断） |
| **STA thread** | 消息泵线程模型下的 COM 线程 |
| **Dual interface** | 继承 **`IDispatch`** 又可直接 **`vtable` 调用的接口** |

---

## 三十一、可以自己动手的小实验（不靠作业打分）

1. 写最小 **`CoCreateInstance(CLSID_FileOpenDialog, IID_IFileOpenDialog)`** 弹出文件框（Shell COM）。
2. **OleView** 打开任意已安装 **Type Library**，截图接口树。
3. 虚拟机里注册一个 **假 CLSID** 指向 **`notepad.exe`** 作为 **`LocalServer32`**（极端实验，勿在生产机乱搞）。
4. 读 **`sdkdi`** 示例：**GitHub microsoft/Windows-classic-samples** 里的 **ShellExtensions**。

---

# 第三辑：仍可以继续往下钻（线程、注册表、工厂模板、排障剧本）

这一辑的目标是：**把前面打散的知识点收成几张「可随时翻阅」的工作表**——尤其是你要真做一个 **Inproc Shell 扩展** 或 **Office 自动化服务** 时，脑子里得有路线图。

---

## 三十三、线程模型：把它当成「并发契约」而不是玄学

### 33.1 三个常用 Apartment

| Apartment | `CoInitializeEx` | 典型宿主 |
|-----------|------------------|-----------|
| **STA** | **`COINIT_APARTMENTTHREADED`** | UI 线程、旧 Shell 扩展、`ThreadingModel=Apartment` |
| **MTA** | **`COINIT_MULTITHREADED`** | 控制台后台线程、你自己保证线程安全的对象 |
| **Neutral** | 少用直接初始化；对象标注 **`Neutral`** | 介于两者之间，避免频繁切换 |

### 33.2 「指针能不能传给别的线程？」决策树（简化）

```
手里有接口指针 p
├─ 同线程继续用？──通常 OK（仍要看对象文档）
├─ 要传给 worker 线程？
│   ├─ 对象是 apartment-neutral / free-threaded？──文档说可以就直接传
│   └─ 不确定？──CoMarshalInterThreadInterfaceInStream / CoGetInterfaceAndReleaseStream
│       （初学阶段：**干脆在同一线程 CoCreate**，别到处甩指针）
└─ 跨进程？──必须 marshal / 换 CLSCTX remoting
```

**初学者口诀**：**不确定就别跨线程裸传 COM 接口指针**；要么 **文档写明**，要么 **在同一线程创建并使用**，要么 **学 marshal API**。

### 33.3 `CoMarshalInterThreadInterfaceInStream`（只记用途）

当你 **必须把 `IAudioClient*` 从线程 A 交到线程 B**，且两边 apartment 不同，规范上要 **marshal**。音频采集线程我们自己 **`CoInitializeEx` + GetService**，就是为了 **避免把 apartment 对象到处迁移**。这也是许多开源项目「**COM 线程专门一根**」的原因。

---

## 三十四、`ThreadingModel` 注册表值到底写给谁看？

位于 **`HKCR\CLSID\{...}\InprocServer32\ThreadingModel`**：

| 值 | 含义（口语） |
|----|----------------|
| **`Apartment`** | 偏 STA；实例可能被 STA 线程创建 |
| **`Free`** | 偏 MTA（老式称呼 **free-threaded**） |
| **`Both`** | STA/MTA 都可创建；对象自己要线程安全 |
| **`Neutral`** | .NET 时代常用 |

**Shell 扩展默认 Apartment**：因为你的 DLL 跟资源管理器 STA 线程在同一个进程房间活动时更安全。**不要乱改成 Free**，除非你熟知后果并在对象内部加锁。

---

## 三十五、手写 `IClassFactory`：最小可讲故事版本

下面这段 **不是 ATL**，纯粹手写心理模型：**DllGetClassObject** 返回 **`CFactory`**，**`CreateInstance`** **`new` 出真正的菜单对象**。

```cpp
class CFactory : public IClassFactory {
    LONG refCount_{ 1 };

public:
    HRESULT STDMETHODCALLTYPE QueryInterface(REFIID riid, void** ppv) override {
        if (!ppv) return E_POINTER;
        if (IsEqualIID(riid, IID_IUnknown) || IsEqualIID(riid, IID_IClassFactory)) {
            *ppv = static_cast<IClassFactory*>(this);
            AddRef();
            return S_OK;
        }
        *ppv = nullptr;
        return E_NOINTERFACE;
    }
    ULONG STDMETHODCALLTYPE AddRef() override { return InterlockedIncrement(&refCount_); }
    ULONG STDMETHODCALLTYPE Release() override {
        ULONG c = InterlockedDecrement(&refCount_);
        if (!c) delete this;
        return c;
    }

    HRESULT STDMETHODCALLTYPE CreateInstance(IUnknown* pOuter, REFIID riid, void** ppv) override {
        if (pOuter)
            return CLASS_E_NOAGGREGATION;
        auto* obj = new(std::nothrow) CMyContextMenu();
        if (!obj) return E_OUTOFMEMORY;
        HRESULT hr = obj->QueryInterface(riid, ppv);
        obj->Release(); // 交给 QI 后的引用计数管理对象生命周期
        return hr;
    }

    HRESULT STDMETHODCALLTYPE LockServer(BOOL fLock) override {
        // g_serverLocks++ / -- ; DllCanUnloadNow 用
        return S_OK;
    }
};
```

**为什么 `CreateInstance` 里 `obj->Release()` 还能活？**  
因为 **`QueryInterface` 已成功 `AddRef` 了一份给 `*ppv`**；这里的 **`Release`** 对应 **`new` 出来时的初始引用**——具体计数约定要与你的 **`CMyContextMenu` 构造函数 / CComObject 风格**一致。**ATL 帮你把这些算盘打好**，手写就要特别小心。

---

## 三十六、Shell 扩展注册：你要改的键不止一条（清单）

做一个「对所有文件右键」的菜单，至少要心里有数：

1. **`HKCR\CLSID\{GUID}`**：友好名称。
2. **`HKCR\CLSID\{GUID}\InprocServer32`**：`默认 = 完整路径.dll`，**`ThreadingModel`**。
3. **`HKCR\*\shellex\ContextMenuHandlers\YourName`**：`默认 = {GUID}`。
4. （可选）**`NoRecentDocs`**、**`Applet`** 等高级键——查官方文档。
5. **卸载**：**`DllUnregisterServer`** 必须对称删除。

只想对 **`.txt`**：把 **`*`** 换成 **`.txt`** 下的 **`ShellEx`** 路径（具体布局随文档版本略有出入，以 **当前 MSDN「Shell Context Menu Handler」** 为准）。

**位数**：**纯 64 位 Windows** 上 **64 位 explorer** → **必须注册到 64 位视图**；若错误用了 **WoW64 重定向**，会出现「**regsvr32 成功但不加载**」灵异事件。

---

## 三十七、`IExplorerCommand`：新 Shell 扩展范式（概念）

Windows 10 1809 以后官方推动 **`IExplorerCommand`**：**实现接口 → 注册 AppID / 关联 `.exe` 或 **打包清单**，而不是老式纯 **`InprocServer32`**。优点是 **Package identity**、**稀疏包**、与 **资源管理器现代化命令栏** 对齐。你若看见示例工程一大坨 **XML manifest**，不要慌：**COM 仍是内核**，注册方式是外层包装进化。

---

## 三十八、Office 自动化再往深处走一步：`Workbooks.Open` 的参数数组

真实脚本几乎总要 **`VARIANT` 数组 + 命名参数**。伪代码思路：

```cpp
VARIANT args[4]{};
args[0].vt = VT_BSTR; args[0].bstrVal = SysAllocString(L"C:\\book.xlsx");
// … Row 读之类 ...
DISPPARAMS dp{};
dp.rgvarg = args;
dp.cArgs = 4;
// Invoke DISPID 对应 Open… 
```

**教训**：**谁分配 `BSTR` / `SAFEARRAY`，谁 `VariantClear`**；循环引用与大数组泄漏自动化客户端内存，是工业软件常见 bug。

---

## 三十九、WMI：COM 仍在后台等你

```cpp
IWbemLocator* pLoc = nullptr;
CoCreateInstance(CLSID_WbemLocator, nullptr, CLSCTX_INPROC_SERVER,
    IID_IWbemLocator, reinterpret_cast<void**>(&pLoc));
// IWbemServices *pSvc … ConnectServer("ROOT\\CIMV2") … ExecQuery("SELECT … FROM Win32_Process") …
```

**要点**：**`IWbemServices`** 也要 **`Release`**；**查询返回枚举器**也是 COM。**PowerShell `Get-WmiObject`** 替你包了这些。

---

## 四十、结构化存储：十几行代码摸一下 `IStorage`

```cpp
#include <objbase.h>
#pragma comment(lib, "ole32.lib")

HRESULT DemoStorage()
{
    IStorage* pStg = nullptr;
    HRESULT hr = StgOpenStorage(L"C:\\temp\\legacy.doc", nullptr,
        STGM_READWRITE | STGM_SHARE_EXCLUSIVE, nullptr, 0, &pStg);
    if (FAILED(hr)) return hr;
    // EnumElements / OpenStorage / OpenStream …
    pStg->Release();
    return S_OK;
}
```

失败不要紧——重要的是你知道：**复合文档**曾经 **COM 一统天下**。

---

## 四十一、扩展 FAQ（第二辑）

**问：`DllRegisterServer` 必须用管理员吗？**  
答：写到 **`HKLM`** 通常要；写到 **`HKCU`**（当前用户）有时不必——取决于你的安装策略。

**问：我能用 C# 写 Shell 扩展吗？**  
答：**可以但不推荐 Inproc**：CLR 会把 **整个 CLR 运行时** 载入 explorer，进程体积与版本地狱。**微软倾向：非托管 Inproc 或 Out-of-proc surrogate**。

**问：`CoInitializeSecurity` 何时调用？**  
答：**DCOM / 某些 WMI** 场景要在 **`CoInitializeEx` 后尽早**；顺序错了 HRESULT 很玄学。

**问：Smart Pointer `ComPtr` 能用在 **线程间** `std::move` 吗？**  
答：**移动语义转移所有权**，但 **底层指针仍可能不能跨 apartment**——别指望 **`move` 解决 marshal 问题**。

---

## 四十二、当你卡住时的「排障剧本」（Shell 扩展专用）

1. **ProcMon**：过滤 **`explorer.exe`** + **`PATH contains YourDll.dll`**，看是否 **LoadImage**。
2. **Dependency Walker / Dependencies.exe**：缺 **VC++ Runtime / API Set**？
3. **`regsvr32 /s Your.dll`** 静默失败：换 **`regsvr32 Your.dll`** 看弹窗。
4. **`sxstrace`**（若有并行程序集问题）。
5. **注释掉 `QueryContextMenu` 里一半菜单**，二分法定位崩溃。
6. **虚拟机快照**：注册表乱了就回到快照。

---

## 四十三、术语表（再扩一圈）

| 术语 | 一句话 |
|------|--------|
| **Surrogate** | **dllhost.exe** 代替 DLL 跑 LocalServer 的技术 |
| **ROT** | 运行对象表： **`GetObject("Excel.Application")`** 之类 |
| **APTTYPEQUALIFIER** | 细分线程公寓限定符（调试） |
| **GIT** | Global Interface Table：跨线程 marshaled cookie |
| **NA** | 「Neutral Apartment」缩写（文档常见） |

---

# 第四辑：内存、封送、GIT、安全与 CLR Interop（再加厚一层）

---

## 四十五、`IMalloc`：COM 世界里的「谁来 malloc？」

不少老式接口用 **`CoGetMalloc(MEMCTX_TASK)`** 返回 **`IMalloc*`**，再用 **`Alloc`/`Free`**。**Shell PIDL 解析**、**STRRET** 转换时常碰到。**原则**：**谁家的 `IMalloc` 分配，就用谁家 Free**；不要被 **`CoTaskMemAlloc`**（任务内存分配器）与 **`SysAllocString`** 混在一起。**现代代码**多用 **`CoTaskMemFree`** 释放 **`SH*** API** 返回给你的 **`PIDL`**——文档为准。

```cpp
LPMALLOC pMalloc = nullptr;
if (SUCCEEDED(CoGetMalloc(1, &pMalloc))) {
    void* mem = pMalloc->Alloc(1024);
    // ...
    pMalloc->Free(mem);
    pMalloc->Release();
}
```

---

## 四十六、全局接口表 GIT：`IGlobalInterfaceTable`

当你 **已经把接口指针 marshal 好**，想在 **任意线程** 用 **cookie** 取回：

```cpp
IGlobalInterfaceTable* pGIT = nullptr;
CoCreateInstance(CLSID_StdGlobalInterfaceTable, nullptr, CLSCTX_INPROC_SERVER,
    IID_IGlobalInterfaceTable, reinterpret_cast<void**>(&pGIT));

DWORD cookie = 0;
pGIT->RegisterInterfaceInGlobal(pUnknownFromMarshal, IID_IFoo, &cookie);
// 另一线程：
IFoo* pFoo = nullptr;
pGIT->GetInterfaceFromGlobal(cookie, IID_IFoo, reinterpret_cast<void**>(&pFoo));
// … 用完 RevokeInterfaceFromGlobal(cookie)
```

**典型用途**：长生命周期 worker 线程与 UI 线程交换 **已经 marshal 过的** 指针。**初学者**：优先 **专用 COM 线程模型**，别一上来 GIT。

---

## 四十七、`CoMarshalInterThreadInterfaceInStream` / `CoGetInterfaceAndReleaseStream`（配对记忆）

**线程 A**：把 **`IUnknown*`** 封进 **`IStream*`**（**`CoMarshalInterThreadInterfaceInStream`**）。  
**线程 B**：**`CoGetInterfaceAndReleaseStream`** 解出 **可直接用的代理指针**。

伪代码：

```cpp
IStream* pStm = nullptr;
CoMarshalInterThreadInterfaceInStream(GetMTAIdentifier(), pSourceUnk, &pStm);
// Post to thread B with pStm
// Thread B:
IUnknown* pProxy = nullptr;
CoGetInterfaceAndReleaseStream(pStm, IID_IFoo, reinterpret_cast<void**>(&pFoo));
```

**细节**：**MTA identifier**、**STA 消息泵**、**流所有权** 极易错——**官方示例照抄**，不要凭感觉手写。

---

## 四十八、DCOM 安全：`CoInitializeSecurity` 调用顺序

若进程要用 **DCOM**（远程 WMI、远程 OPC、分布式组件），常在 **`CoInitializeEx` 之后第一时间**：

```cpp
CoInitializeSecurity(nullptr, -1, nullptr, nullptr,
    RPC_C_AUTHN_LEVEL_PKT_PRIVACY, RPC_C_IMP_LEVEL_IDENTIFY,
    nullptr, EOAC_NONE, nullptr);
```

**失败 HRESULT** 常见：**80010119**（重复初始化安全）、**权限不够**。顺序错了会导致后续 **`CoCreateInstanceEx`** 莫名其妙失败。**教训**：读官方 **「Initializing COM Security」** 一页，按顺序抄。

---

## 四十九、.NET / CLR 互操作：RCW、CCW、`Marshal.ReleaseComObject` 为何臭名昭著

### 49.1 RCW（Runtime Callable Wrapper）

托管代码里的 **`new Excel.Application()`** 背后是 **RCW** 包了一层 **`IDispatch`**。**Finalize** 时才 **`Release`**——可能导致 **COM 对象活得比你想的长**。于是有人 **`Marshal.ReleaseComObject(excel)`** 强行 Release——若仍有别的托管引用指向同一 RCW，会 **崩**。

### 49.2 CCW（COM Callable Wrapper）

.NET **公开 `[ComVisible]` 类**给原生 **`CoCreateInstance`**。**GC** 与 **COM 引用计数** 两套生命周期叠在一起，调试地狱。

### 49.3 实务建议

- **尽早显式 `Quit()` Office**，不要只靠 **`GC`**。
- **`FinalReleaseComObject`** 之类 API **慎用**；读最新文档是否有 **`NET Core COM interop` 改进**。
- **优先「薄 RCW」**：业务逻辑仍在 native，托管只做 UI——边界清晰。

---

## 五十、`IPersistFile` / `IPersistStream`：「对象，请把自己保存起来」

某些 COM 对象支持 **持久化**：**`Save`** / **`Load`**。**快捷方式（Shell Link）**、**ActiveX 控件状态**、旧 **Office** 文档嵌入对象都涉及。**心智模型**：**COM 对象序列化成流**，而非随便 **`fwrite` C++ 对象内存**。

---

## 五十一、OLE 链接与嵌入（OLE）：不只右键菜单

**OLE** 当年等于 **复合文档 + 就地激活 + 链接嵌入**。今天 **Word 里插入 Excel 表**仍有遗产。**`IOleObject`**、**`IOleClientSite`**——全是 COM。你若维护 **旧工业软件**，仍会见到 **`OleCreate`**。

---

## 五十二、`IDataObject::SetData`：自定义剪贴板格式

拖拽 / 壳扩展里有时注册 **`RegisterClipboardFormat(L"MyAppData")`**，再在 **`IDataObject::SetData`** 塞进 **`FORMATETC`**。**接收端 `GetData`** 对称。**调试**：**InsideClipboard** 小工具看格式列表。

---

## 五十三、`CoWaitForMultipleHandles`：COM 与 Win32 等待共存

STA 线程既要 **消息泵**又要 **`WaitForSingleObject`** 时，可用 **`CoWaitForMultipleHandles`**（带 **`COWAIT_DISPATCH_WINDOWS_MESSAGES`**），避免 **死锁**。**Media Foundation / DirectShow** 某些示例线程模型会扯到这个。

---

## 五十四、VBScript / JScript 迟绑定：脚本引擎里的 `IDispatch`

老式 ASP：**`Server.CreateObject`** → **`ProgID`** → **`CoCreateInstance`** → **`IDispatch`**。**`GetIDsOfNames("Open")`** → **`Invoke`**。**为何慢**：每一步名字解析；**为何灵活**：脚本不改编译。

---

## 五十五、性能与文化：为什么有人恨 COM

1. **HRESULT 噪音**：失败路径遍布；未检查 → **随机状态污染**。  
2. **引用计数心智负担**：**泄漏 / UAF** 永不过时。  
3. **注册表 + DLL Hell**：版本冲突折磨几代程序员。  
4. **Side-by-Side / WinSxS / Manifest**：补救 DLL Hell，又引入 **manifest 地狱**。  

**但仍要说**：没有 COM，就没有现代 Windows 多媒体与 Shell 生态——**恨与爱并存**。

---

## 五十六、可选书单 / 视频（续）

7. Chris Sells《ATL Internals》（硬核）。  
8. YouTube 搜索 **「Raymond Chen COM」** 系列短文（博客 **The Old New Thing** 纸质书也可）。  
9. Microsoft Learn：**「COM threading apartments」** 官方长文（务必读完 STA 消息泵段落）。

---

## 五十七、从零做一个最小 Inproc COM 服务器：只做 checklist（长）

下面是一份 **「照着一条条勾」** 的工程清单，适用于 **教学用 DLL**，不等价于 **微软官方 Sample 的全部健壮性**。

### 57.1 工程准备

1. 新建 **DLL 工程**，平台 **x64**（若目标是 **64 位 explorer**）。
2. **C/C++ → 预编译头**：可关（减小样板）。
3. **链接器 → 模块定义文件 `.def`**：**导出 `DllGetClassObject`、`DllCanUnloadNow`、`DllRegisterServer`、`DllUnregisterServer`**（也可用 **`__declspec(dllexport)`**，团队要有统一规范）。
4. **附加依赖**：**`ole32.lib` `oleaut32.lib` `shell32.lib`**（按接口而定）。

### 57.2 定义 CLSID / IID

```cpp
// 用 guidgen.exe 或 VS「工具 → 创建 GUID」生成唯一值
static const CLSID CLSID_MyServer =
{ 0xaaaaaaaa, 0xbbbb, 0xcccc, { 0xdd,0xee,0xff,0x01,0x02,0x03,0x04,0x05 } };
```

### 57.3 实现 COM 对象与类工厂

1. **`CMyObject`**：**`QueryInterface` / `AddRef` / `Release`** + 业务接口。
2. **`CFactory`**：**`IClassFactory`**：**`CreateInstance` / `LockServer`**。
3. **全局计数**：**`g_serverLocks`**、**`g_objCount`**（调试用）。

### 57.4 `DllGetClassObject`

```cpp
STDAPI DllGetClassObject(REFCLSID rclsid, REFIID riid, LPVOID* ppv)
{
    if (!IsEqualCLSID(rclsid, CLSID_MyServer))
        return CLASS_E_CLASSNOTAVAILABLE;
    static CFactory factory;
    return factory.QueryInterface(riid, ppv);
}
```

### 57.5 `DllRegisterServer`

用 **`RegCreateKeyEx`** 写：

- **`HKCR\CLSID\{...}`**
- **`InprocServer32` = 模块完整路径**
- **`ThreadingModel`**

壳扩展还要写 **Shell 挂接键**（见前文第三十六节）。

### 57.6 `regsvr32` 与验证

1. **`regsvr32 My.dll`**  
2. **OleView** → **All Classes** → 搜你的 CLSID。  
3. **ProcMon** 过滤 **`regsvr32`** 看 **`CreateFile`** 是否指向正确 DLL。

### 57.7 卸载

**`regsvr32 /u`** + **确认注册表消失**。

---

## 五十八、HRESULT「字典页」：Facility 与 Win32 映射（实用向）

### 58.1 `HRESULT_FROM_WIN32` 速查

当你拿到 **`GetLastError() == ERROR_ACCESS_DENIED (5)`**：

```cpp
HRESULT hr = HRESULT_FROM_WIN32(ERROR_ACCESS_DENIED); // 0x80070005
```

### 58.2 `FACILITY_ITF` 自定义区间

接口作者可在 **`0x0200–0xFFFF`** 区间定义 **`IFACEMETHODIMP`** 返回值——**不要与系统 HRESULT 冲突**。看见 **`0x8004xxxx`** 多看一眼是否 **自定义接口错误**。

### 58.3 打印友好字符串（示意）

```cpp
void PrintHR(HRESULT hr)
{
    wchar_t buf[512]{};
    FormatMessageW(
        FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
        nullptr, hr, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
        buf, (DWORD)(std::size(buf)), nullptr);
    std::fwprintf(stderr, L"hr=0x%08X %s\n", hr, buf);
}
```

失败时 **`FormatMessage`** 不一定总有文本——仍以 **十六进制** 为准。

---

## 五十九、DirectShow：仍是 COM，只是图特别大

**Filter Graph**：**`IGraphBuilder`**、**`IBaseFilter`**、**`IMediaControl`**……全套 COM。**GraphEdit** 工具可视化。**现状**：微软主推 **Media Foundation**，但监控摄像头、旧采集卡驱动仍 **DirectShow**。心法与本笔记 **IUnknown + HRESULT** 完全一致，只是接口数量爆炸。

---

## 六十、Active Scripting：`IActiveScript` 与宿主

IE 时代的 **JScript/VBScript 引擎**暴露 **`IActiveScript`**，宿主 **`IActiveScriptSite`**——仍是 COM。现代 **Chakra / V8** 替代了浏览器脚本栈，但 **Windows Script Host (`wscript.exe`)** 仍在。

---

## 六十一、扩展阅读：Windows Runtime 投影（CppWinRT）里的 COM 幽灵

打开 **`winrt/base.h`**，你会看见 **`IUnknown`**、**`WINRT_IMPL_IUnknown`** 之类痕迹。**不要混淆**：日常 **`winrt::Windows::Foo`** 写法不必手写 **`QueryInterface`**，但一旦 **激活失败**（**`RoActivateInstance` HRESULT**），排查链条仍会回落到 **组件注册 / manifest / apartment**。

---

## 六十二、私货小结：什么时候该停止抄笔记去写代码？

当你能 **独立解释** 如下五个问题，就可以把本篇合上：

1. **`CoCreateInstance` 失败返回 `REGDB_E_CLASSNOTREG`，你最先检查哪三处？**（注册表 CLSID、位数、DLL 路径）
2. **`RPC_E_CHANGED_MODE` 出现时，你有几种应对策略？**（忽略继续 / 换线程 / `CoInitializeEx` 参数对齐）
3. **`QueryInterface` 成功后指针是否需要 `Release`？**（看所有权转移约定）
4. **Inproc Shell 扩展为什么要匹配 explorer 位数？**
5. **`RCW` 为什么可能导致 Excel 进程残留？**

---

# 第五辑：头文件与库、伪面试题、再续 FAQ（冲体量也用干货撑着）

---

## 六十三、常用头文件 → 对应 `.lib`（桌面原生 COM 速查）

| 头文件 | 典型接口 / API | 常见静态链接库 |
|--------|----------------|----------------|
| **`unknwn.h`** | **`IUnknown`** | （通常不需要单独 lib） |
| **`objidl.h` / `objidlbase.h`** | **`IMalloc`、`IStream`、`IStorage`** | **`ole32.lib`** |
| **`ole2.h`** | OLE 复合文档全家桶 | **`ole32.lib` `oleaut32.lib`** |
| **`combaseapi.h`** | 现代 **`Ro*`**、基础 COM | **`ole32.lib`** |
| **`shlobj.h` `shobjidl.h`** | Shell、**`IShellItem`** | **`shell32.lib`** |
| **`mmdeviceapi.h`** | MMDevice | **`ole32.lib`**（间接） |
| **`audioclient.h`** | WASAPI | **`ole32.lib`** + **`Mmdevapi`** |
| **`mfreadwrite.h`** | **`IMFSinkWriter`** | **`mfplat` `mfuuid` `mfreadwrite`** |
| **`d3d11.h`** | **`ID3D11Device`** | **`d3d11.lib`** |

**原则**：**链接错误「unresolved external `__imp_...`」** → 先少一个 **`.lib`** 或 **调用约定** 不对（**`__stdcall`**）。

---

## 六十四、伪面试题（口试向，自己对着镜子练）

**Q1**：**`QueryInterface` 和 C++ `dynamic_cast` 有什么本质不同？**  
**A1**：**`dynamic_cast` 是编译期/RTTI 概念，跨模块 DLL 未必安全**；**`QI` 是 COM 约定，跨语言、跨版本接口扩展都靠它。**

**Q2**：**为什么 Inproc 服务器要导 `DllCanUnloadNow`？**  
**A2**：**OLE 要判断能否卸载 DLL 释放内存**；返回 **`S_OK`** 才安全卸载。

**Q3**：**STA 线程没有消息泵会怎样？**  
**A3**：**跨 apartment 封送可能死锁**；**UI 线程**必须 **GetMessage 循环**（或现代消息调度）。

**Q4**：**`CLSCTX_INPROC_SERVER` 和 `INPROC_HANDLER` 区别？**  
**A4**：**Handler** 与 **In-place activation** 有关（OLE 文档服务器）；多数桌面扩展接触 **`INPROC_SERVER`** 即可。

---

## 六十五、FAQ（第三辑·絮叨版）

**问：我用 CMake，`.def` 导出怎么写？**  
答：**`target_sources PRIVATE exports.def` + `LINK_FLAGS /DEF:`**，或用 **`__declspec(dllexport)`** 导出 **`DllGetClassObject`**。

**问：`regsvr32` 提示「加载失败找不到模块」？**  
答：**依赖 DLL 缺失**（VC runtime、UCRT）、或 **路径含奇怪字符**。

**问：`DllRegisterServer` 返回成功但注册表没键？**  
答：**重定向**：32 位 **`regsvr32`** 写 **`WOW6432Node`**；你用 **64 位 OleView** 却去看 **32 位视图**，会对不上。

**问：`ZeroMemory` 接口指针可以吗？**  
答：**绝对不行**，除非你确认它不是活的 COM 指针；忘了 **`Release`** 就用 **`ZeroMemory`** ——那是自欺欺人。

**问：我能混用 `CoInitialize` 和 `CoInitializeEx` 吗？**  
答：**同一线程不要混**；遗留代码另说。

**问：`IUnknown*` 能放进 `std::vector` 吗？**  
答：**可以，但要明确所有权**：用 **`ComPtr`** 包一层更安全。

**问：Rust / Go 调 COM？**  
答：**Rust `windows-rs`**、**Go `syscall` + COM ABI** 都有人做；本质仍是 **vtable + HRESULT**。

---
