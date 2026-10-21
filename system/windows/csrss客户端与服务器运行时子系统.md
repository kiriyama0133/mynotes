---
date: 2026-08-21
tags:
  - windows
  - csrss
  - system
---
如果你的工作是 Windows 内核架构、进程管理，甚至安全对抗（比如 EDR 绕过、Rootkit 隐藏）的笔记或系列文章，`csrss.exe` 是离不开的

Client Server Runtime Subsystem（客户端/服务器运行时子系统），是用户态最高的长官，windows的用户态分为两个部分，csrss.exe负责管理环境，内核负责的底层调度。

## 它的工作

1. **进程与线程的“接生婆”**：`CreateProcess` 表面上是你调用的，但底层实际上是内核通知 `csrss.exe` 来完成进程控制块（EPROCESS）的初始化和销毁。没有它，谁也别想启动。
2. **控制台（Console）的唯一主人**：你平时打开的 CMD 窗口，所有字符的渲染、键盘鼠标输入的捕获，底层全是 `csrss.exe` 在处理。
3.  **进程通信（IPC）与关机管理**：它维护着系统级的全局句柄，并且在系统关机时，负责通知所有进程安全退出。

它是免死的，如果有恶意软件通过内核驱动想要杀死他，系统会直接触发CRITICAL_PROCESS_DIED蓝屏。正常情况下csrss.exe的父进程是smss.exe会话管理器，很多病毒会伪造csrss.exe的子进程，或者自己伪造成csrss，甚至跨会话注入，因为安全软件默认对它有极大的信任。

