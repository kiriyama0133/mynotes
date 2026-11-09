---
date: 2026-11-08
tags:
  - win32
  - windows
  - system
  - services
---
服务控制管理器，负责管理和控制所有Windows服务的启动、停止和状态。

根据微软的介绍，关键的系统服务包括了smss.exe、csrss.exe、wininit.exe、logonui.exe、lsass.exe、services.exe、winlogon.exe，以及带有 RPCSS 的 System 服务，还有带有 Dcom/PnP 功能的 svchost.exe。

微软提供了一个叫做Restart Manager API的一段技术说明，教开发者如何通过代码来程序化判断某个进程是不是我们之前讨论的关键系统服务，要确定某个进程是不是关键提供服务，使用`RmRegisterResources` 函数注册该进程。

调用`RmGetList` 函数以获取 `RM_PROCESS_INFO` 结构体。返回的 `RM_PROCESS_INFO` 结构体中的 `ApplicationType` 成员包含一个 `RM_APP_TYPE` 枚举值。**对于关键系统进程，该值会被设置为 `RmCritical`。**

这就是windows提供给我们的重启管理器机制，可以用于安装程序的免打扰，当你安装或更新某个软件时，如果该软件正在运行，Windows 会通过这个 API 识别出需要结束哪些进程，并在结束后自动重启它们；还有安全判断；

## 后台机制

windows提供了多个机制来运行后台工作，下面的表可以帮助我们选择合适的场景：

| 机制                    | 最适用于                                   | Lifecycle         | 电源影响             |
| --------------------- | -------------------------------------- | ----------------- | ---------------- |
| **Windows 服务** (触发启动) | 响应系统事件（设备到达、网络更改、域加入）。 仅在需要时运行。        | 由 SCM 在触发时启动/停止。  | 低 — 触发时空闲。       |
| **Windows服务**（自动启动）   | 必须始终可用的长时间运行的守护程序（数据库引擎、Web 服务器、安全代理）。 | 从启动到关机一直运行。       | 中等 - 始终消耗资源。     |
| **任务计划程序** （维护）       | 定期维护（磁盘清理、日志轮换、更新检查）。                  | 在空闲状态下且接通交流电源时运行。 | 最小 — 推迟到空闲时段。    |
| **任务计划程序** （事件/时间触发器） | 与计划、登录或系统事件关联的一次性或定期作业。                | 根据计划创建/删除。        | 低 — 短暂运行，然后退出。   |
| **后台任务** （UWP/WinRT）  | 具有资源配额的特定于应用的后台工作（磁贴更新、推送通知、地理围栏）。     | 由系统代理管理。          | 非常低 — 严格由 OS 限制。 |

>**首选触发器启动服务而不是自动启动服务。** 触发启动服务在触发事件触发之前消耗零资源。 这对于新式待机设备上的电池使用时间至关重要。 有关详细信息，请参阅 [服务触发器事件](https://learn.microsoft.com/zh-cn/windows/win32/services/service-trigger-events) 。

以下类型的程序使用 SCM 提供的函数：

|类型|Description|
|---|---|
|服务计划|为一个或多个服务提供可执行代码的程序。 服务程序使用连接到 SCM 并将状态信息发送到 SCM 的函数。|
|服务配置程序|查询或修改服务数据库的程序。 服务配置程序使用打开数据库的函数，在数据库中安装或删除服务，并查询或修改已安装服务的配置和安全参数。 服务配置程序同时管理服务和驱动程序服务。|
|服务控制程序|启动和控制服务和驱动程序服务的程序。 服务控制程序使用将请求发送到执行请求的 SCM 的函数。|
## 服务控制管理器

根据文档，SCM也就是上面提到的services.exe，是用于维护已经安装服务和驱动服务的数据库，并且提供统一且安全的控制方式，数据库包含了有关如何启动每个服务或者驱动程序服务的信息。

它还使系统管理员能够自定义每个服务的安全要求，从而控制对服务的访问。

SCM在系统启动的时候启动，它是远程过程调用RPC服务器，以便服务配置和服务控制程序可以作远程计算机上的服务。

服务函数为 SCM 执行的以下任务提供接口：

- 维护已安装服务的数据库。
- 在系统启动时或按需启动服务和驱动程序服务。
- 枚举已安装的服务和驱动程序服务。
- 维护运行服务和驱动程序服务的状态信息。
- 将控制请求传输到正在运行的服务。
- 锁定和解锁服务数据库。


SCM会维护注册表中已经安装服务的数据库，数据库由添加、修改或配置服务的 SCM 和程序使用。这个数据库的主路径是：

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services

这个路径下，每一个子健就是一个服务 or 驱动，其名称就是服务的内部名称，SCM正是通过这些子健中的信息来管理服务的，关键信息有：

- **`ImagePath`**：服务的可执行文件完整路径。
- **`Start`**：启动类型（如自动、手动、禁用）。
- **`ObjectName`**：服务运行所使用的账户身份。
- **`DependOnService`**：服务的依赖关系，确保启动顺序正确。
- **`Group`**：服务所属的加载顺序组，用于批量控制启动次序。

即便可以直接编辑注册表来达到修改服务配置但是不推荐，因为SCM在启动的时候会将数据库读入内存，直接修改注册表会导致配置不同步，更安全的办法是使用SCM提供的API或者命令行工具sc.exe来操作。

而且呢，并不是所有的后台程序都是SCM管理，windows的后台任务机制多样。内核驱动和文件系统驱动也记录在同一个注册表路径下，由SCM和IO管理器协同加载，但是他们的运行层级更深。还有用于计时或者触发式的维护任务，由任务计划程序服务管理而不是SCM控制。UWP/WINRT后台任务由应用生命周期管理，有着严格的资源配额限制。

## 安装服务

开发者或者安装程序会调用`CreateService` 函数来安装一个新服务时，SCM（服务控制管理器）会在注册表的 `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services` 路径下，创建一个以**该服务的内部名称**（即函数中的 `lpServiceName` 参数）命名的注册表子项。

查看微软的官方文档，方法和参数为：

```c
SC_HANDLE CreateServiceA(
  [in]            SC_HANDLE hSCManager,
  [in]            LPCSTR    lpServiceName,
  [in, optional]  LPCSTR    lpDisplayName,
  [in]            DWORD     dwDesiredAccess,
  [in]            DWORD     dwServiceType,
  [in]            DWORD     dwStartType,
  [in]            DWORD     dwErrorControl,
  [in, optional]  LPCSTR    lpBinaryPathName,
  [in, optional]  LPCSTR    lpLoadOrderGroup,
  [out, optional] LPDWORD   lpdwTagId,
  [in, optional]  LPCSTR    lpDependencies,
  [in, optional]  LPCSTR    lpServiceStartName,
  [in, optional]  LPCSTR    lpPassword
);
```

hSCManager是服务数据库句柄，此句柄由 [OpenSCManager](https://learn.microsoft.com/zh-cn/windows/desktop/api/winsvc/nf-winsvc-openscmanagera) 函数返回，必须具有 **SC_MANAGER_CREATE_SERVICE** 访问权限。 有关详细信息，请参阅 [服务安全性和访问权限](https://learn.microsoft.com/zh-cn/windows/desktop/Services/service-security-and-access-rights)。

lpServiceName要安装的服务的名称。 最大字符串长度为 256 个字符。 服务控制管理器数据库保留字符大小写，但服务名称比较始终不区分大小写。 正斜杠 (/) 和反斜杠 (\) 不是有效的服务名称字符。

lpDisplayName用户界面程序用于标识服务的显示名称。 此字符串的最大长度为 256 个字符。 名称在服务控制管理器中保留大小写。 显示名称比较始终不区分大小写。

具体文档请查看：https://learn.microsoft.com/zh-cn/windows/win32/api/winsvc/nf-winsvc-createservicea

权限方面，因为 `CreateService` 会在 `HKLM\SYSTEM` 这种受保护的核心注册表分支下创建子项，所以调用这个函数的程序**必须具备管理员权限**。这也是为什么我们在安装某些软件时，会弹出 UAC（用户账户控制）提示框。

UAC安全控制请看对应微软文档：https://learn.microsoft.com/zh-cn/windows/security/application-security/application-control/user-account-control/architecture

## 自动启动服务

在系统引导期间，SCM 会启动所有自动启动服务及其依赖的服务。 例如，如果自动启动服务依赖于需求启动服务，那么需求启动服务也会自动启动。

加载顺序组列表中组的顺序。 此信息会存储在以下注册表项的 **List** 值中：

1. **HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\ServiceGroupOrder**  若要指定服务的加载顺序组，请使用 _[CreateService](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-createservicea)_ [或](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-createservicea) **[ChangeServiceConfig](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-changeserviceconfiga)** 函数的 **lpLoadOrderGroup 参数。**

2. 标记顺序向量中指定的组内服务的顺序。 此信息会存储在以下注册表项中：**HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\GroupOrderList**
3.  针对每个服务列出的依赖项。

引导完成后，系统会执行由以下注册表项的 **ImagePath** 值指定的引导验证程序：**HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\BootVerificationProgram**。

### 引导验证程序

一般下，windows只要加载到登录界面，winlogon就会向系统报告启动成功，但是可能会出现一些关键的后台服务后者应用程序尚未完全就绪，对于需要确保特定服务稳定运行的高可靠的场景，需要一个更严格的成功标准，因此引导验证程序允许管理员或者软件定义个自定义程序，只有这个程序运行并确认所有关键条件都满足之后，系统才会正式认定本次的启动是成功的。

根据上面所说：在注册表路径 `HKLM\SYSTEM\CurrentControlSet\Control\BootVerificationProgram` 下，`ImagePath` 值指定了自定义验证程序的完整路径。系统启动时，服务控制管理器（SCM）会运行这个程序。

这个自定义程序会检查它预设的各种条件，例如某个关键服务是否已经启动，如果满足，程序会调用 `Bootok.exe` 来明确通知系统“启动成功”。失败的话可以不通知成功。

要启用自定义引导验证程序，必须进行一项关键修改：

必须在 `Winlogon` 的配置中，将 `ReportBootOk` 的值设置为 `0`。这会关闭系统默认的启动成功报告功能，否则你的自定义程序将无法接管。自定义验证程序与系统自带的 `Bootvrfy.exe` 工具是互斥的，不能同时运行。

这个程序的功能就很强，除了检测某个特定服务比如说数据库服务是否启动之外，还可以检测关键的网络端口是否连同，核心驱动程序是否加载，甚至特定应用的数据接口是否就绪，它的核心目的就是判断是否环境达到了业务可用的就绪态。


## 两种启动与传参方式操作

在windows的服务管理器中双击某个服务，在常规的选项卡里没有直接输入参数的地方，如果使用命令行工具sc.exe，可以使用：

`sc start <服务名> <参数1> <参数2>` 来传递参数。如果是自动启动的服务，系统在开机时会默认无参数启动它们。开发者调用 `StartService` 函数时，第二个参数（`lpServiceArgVectors`）就是用来传递字符串数组参数的。

论你用什么方式传参，这些参数都不会直接注入到进程内存里。它们的流转路径是：

1. services.msc或者你的代码将参数发送给了SCM之后；
2. SCM收到启动请求，检查服务的依赖项和状态，创建服务进程；
3. 进程创建后，服务程序内部调用`StartServiceCtrlDispatcher` 向 SCM 注册自己的 `ServiceMain` 回调函数。
4. SCM将之前传递的参数lpServiceArgVectors作为 `argc` 和 `argv` 参数，传递给服务程序的 `ServiceMain` 入口函数。

但是传递了参数不代表服务一定会用，服务程序的 `ServiceMain` 函数接收到参数后，必须有相应的代码去解析这些参数。如果服务的代码里根本没有处理 `argv` 的逻辑，你传再多参数它也会忽略。这些参数仅仅在该服务**启动的那一刻**有效。服务一旦运行起来，就不能再通过这种方式动态修改参数了（除非服务提供了特定的控制命令，通过 `ControlService` 发送）。

很多数据库服务比如mysql或者pg都支持传入特定参数比如--console可以让服务以前台控制器模型运行，方便排查启动失败的原因。如果你熟悉命令行，`sc start <服务名> <参数>` 其实就是在调用 `StartService` API，这在自动化运维脚本中非常常用。

所以我们是可以通过编写服务程序配置ImagePath实现自启动和其他的目的，但是和应用程序开机自启动有根本的不同，因为windows服务是为了后台基础设施设计的，开机自启程序是为了用户引用服务。

windows服务运行在 Session 0 中，这是一个与用户会话完全隔离的专用会话，处于安全考虑，从windows vista开始，服务无法直接显示任何用户界面比如窗口、对话框等，如果服务需要和用户沟通，就需要特定API如`WTSSendMessage`或进程间通信（IPC）将消息传递给用户会话中的另一个程序来显示。

开启自启的应用运行在交互式会话Session 1或者2之中，拥有完整的桌面环境， 它们可以像用户手动打开的程序一样，自由地显示窗口、弹出通知，与用户进行直接的图形化交互。[**开机自启程序**](https://detached.online/blog/windows%E5%BC%80%E6%9C%BA%E8%87%AA%E5%90%AF%E7%9A%84%E5%85%AD%E7%A7%8D%E6%96%B9%E5%BC%8F/#%e5%88%a0%e6%9d%a1%e7%9b%ae--%e5%8d%b8%e8%bd%bd) 。两者的默认运行身份和权限也不同，服务默认以`LocalSystem`等内置系统账户运行，拥有极高的系统级权限，可以访问绝大多数系统资源。安装和配置服务本身就需要管理员权限 ，自启程序默认以**当前登录的用户身份**运行，其权限受限于该用户的账户类型（标准用户或管理员）。如果程序需要管理员权限，每次启动都可能触发UAC（用户账户控制）弹窗，或直接在启动文件夹中运行失败。

windows服务由**服务控制管理器** 统一管理，生命周期与**系统**绑定。系统运行，服务就在；系统关闭，服务才结束。SCM还可以提供服务失败后的**自动恢复**策略，可靠性很高。

会话id我们可以通过query命令查看：

```shell
[17:28:42] PS C:\Users\20742>  query user
 USERNAME              SESSIONNAME        ID  STATE   IDLE TIME  LOGON TIME
>20742                 console             1  Active      none   11/9/2026 3:46 PM
[17:29:27] PS C:\Users\20742>
```

## 服务记录列表

从已安装服务的数据库读取每个服务条目时，SCM 将为服务创建服务记录。 服务记录包括：

- 服务名称
- 启动类型（自动启动或需求启动）
- 服务状态（请参阅 [**SERVICE_STATUS**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/ns-winsvc-service_status) 结构）
    
    类型  
    当前状态  
    可接受的控制代码  
    退出代码  
    等待提示
    
- 指向依赖项列表的指针

在安装服务时指定帐户的用户名和密码。 SCM 将用户名存储在注册表中，密码存储在本地安全机构（LSA）的安全部分中。 系统管理员可以创建密码永不过期的帐户。 或者，系统管理员可以通过定期更改密码来创建密码过期和管理帐户的帐户。

安装服务指定账户的用户名和密码是因为windows服务在某个特定的安全上下文里运行，SCM需要凭证在为这个上下文完成登录，获取一个代表其身份和权限的访问令牌。每个Windows服务都像一个独立的用户，以某个账户的身份在系统里工作。这个账户决定了服务能做什么、不能做什么。例如，以高权限的`LocalSystem`账户运行的服务，几乎可以访问系统任何资源；而以权限较低的`NetworkService`账户运行的服务，访问范围就会受到严格限制。

系统启动的时候，SCM会读取注册表`HKLM\SYSTEM\CurrentControlSet\Services`下的静态数据库。对于读到的**每一个服务条目**，SCM都会在内存中创建一条对应的 “服务记录”。众多的服务记录通过指针连接，构成了服务记录列表。

`sc.exe` 是管理服务最直接的命令行工具，它的大部分命令都直接对应 SCM 的内部功能。

- **`sc query`**：可以列出服务的**基本状态**（如 `SERVICE_NAME`、`STATE` 等）。
- **`sc queryex`**：在 `sc query` 的基础上，额外提供 **`PID`（进程ID）** 和 **`FLAGS`** 等扩展信息，信息更全面。

```shell
[17:29:27] PS C:\Users\20742>  sc query type= all state= all

SERVICE_NAME: 1394ohci
DISPLAY_NAME: 1394 OHCI Compliant Host Controller
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 1  STOPPED
        WIN32_EXIT_CODE    : 1077  (0x435)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0

SERVICE_NAME: 3ware
DISPLAY_NAME: 3ware
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 1  STOPPED
        WIN32_EXIT_CODE    : 1077  (0x435)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
....
```

SERVICE_NAME服务内部名称表示注册表里的子项名称；**`DISPLAY_NAME`** 是给用户看的名称；TYPE: 1 KERNEL_DRIVER表示啮合驱动，如果是普通的后台服务，这里通常会显示 `10 WIN32_OWN_PROCESS` 或 `20 WIN32_SHARE_PROCESS`；STATE_ 1 STOPPED，1表示已停止，如果是运行的就会是4 RUNNING。 **`WIN32_EXIT_CODE: 1077 (0x435)`**：这个错误码意思是 **“服务从未启动过”**。因为你这台电脑可能根本没有 1394 接口硬件，或者该硬件未被触发，所以系统根本没有尝试去加载它；**`SERVICE_EXIT_CODE: 0`**，如果服务启动后自己崩溃了，这里会记录它自身的退出码。

当服务使用 [**SetServiceStatus**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-setservicestatus) 函数发送服务状态通知时，SCM 会更新服务状态。 SCM 通过查询 I/O 系统而不是从服务接收状态通知来维护驱动程序服务的状态。

服务可以通过调用 [**SetServiceBits**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Lmserver/nf-lmserver-setservicebits) 函数来注册其他类型信息。 [**NetServerGetInfo**](https://learn.microsoft.com/zh-cn/windows/desktop/api/lmserver/nf-lmserver-netservergetinfo) 和 [**NetServerEnum**](https://learn.microsoft.com/zh-cn/windows/desktop/api/lmserver/nf-lmserver-netserverenum) 函数获取受支持的服务类型。

## SCM句柄

SCM 支持句柄类型，以允许访问以下对象。

- 已安装服务的数据库。
- 服务。
- 数据库锁。

SCManager 对象表示已安装服务的数据库。 它是保存服务对象的容器对象。 [**OpenSCManager**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-openscmanagera) 函数返回指定计算机上的 SCManager 对象的句柄。 此句柄用于安装、删除、打开和枚举服务以及锁定服务数据库时。

服务对象表示已安装的服务。 [**CreateService**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-createservicea) 和 [**OpenService**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-openservicea) 函数将句柄返回到已安装的服务。

[**OpenSCManager**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-openscmanagera)、[**CreateService**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-createservicea)和 [**OpenService**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-openservicea) 函数可以请求对 SCManager 和服务对象的不同类型的访问。 请求的访问权限根据调用进程的访问令牌以及与 SCManager 或服务对象关联的安全描述符授予或拒绝。

[**CloseServiceHandle**](https://learn.microsoft.com/zh-cn/windows/desktop/api/Winsvc/nf-winsvc-closeservicehandle) 函数关闭 SCManager 和服务对象的句柄。 如果不再需要这些句柄，请务必关闭它们。

句柄通过`OpenSCManager` 函数获得，可以用于代表与服务控制器数据库的连接，可以用于安装（`CreateService`）、删除服务，或者**打开、枚举**已安装的服务。

服务控制句柄通过 `OpenService`（打开现有服务）或 `CreateService`（创建新服务时）获得，仅对指定的那一个服务有效。用于查询或修改该服务的配置、启动、停止、暂停服务，以及向服务发送自定义控制码。

## 服务开发

visual studio里提供了用于 Windows 服务开发的官方模板，有个叫做Windows 服务的模板，不过是.NET Framework模板，这是经典选择。新建项目时，在“Windows”分类下可以找到。它会自动生成一个继承自 `ServiceBase` 的 `Service1` 类，并搭建好 `OnStart`、`OnStop` 等方法的骨架，省去了手动编写入口点和服务控制逻辑的麻烦。

现代上，我们推荐使用**Worker Service 模板** ，它基于 `Microsoft.Extensions.Hosting.BackgroundService` 构建，更符合现代应用架构，模板会自动集成依赖注入、日志和配置系统，是目前开发新服务的首选。

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794218953303-p91jkv7h.webp)

这里推荐使用顶级语句，省去Namespace、class Program等样板代码，让服务启动和配置的逻辑更加清晰。

我们只需要在Service里使用AddWindowService注册服务名称之后就可以使用sc命令来注册这个服务：

```cs
public static class WindowServiceExtension
{
    public static IServiceCollection AsWindowService(this IServiceCollection services)
    {
        services.AddWindowsService(o =>
        {
            o.ServiceName = "CpuMonitorService";
        });
        return services;
    }
}
```

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794231702179-ys9b1uan.webp)

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794231753360-nvpnieim.webp)

## Idle 进程

这是PID为0的进程，但是他不是一个真正的程序，它是由操作系统内核虚拟构建出来的，用于占坑。

因为CPU不能停下来，通电后，每一纳秒都必须执行一条命令，当windows系统发展以来发现，所有的浏览器、游戏、后台程序都没活干了的时候，CPU不能等待，于是操作系统就把CPU的指针指向了一个底层的命令HLT 叫做Halt停机。

这个指令就是断电休息，知道下一个硬件中断，比如你动了鼠标再叫醒。这个就是ldle进程干的事情，它没有什么代码在运行，纯纯就是让CPU省电。

以及现代操作系统的调度器有一个规定，time slice必须要分配出去。如果某个毫秒，系统发现没有任何用户可以执行的线程了，他不能直接扔掉时间片，必须把这个时间片塞给一个垃圾桶，所以ldle进程可以接受所有无人认领的时间片。

**100% 的 CPU 总时间 - 所有真实进程消耗的 CPU 时间 = Idle 进程的占用率。**