---
date: 2026-10-02
tags:
  - windows
  - dkom
  - system
---
DKOM 的全称是 **Direct Kernel Object Manipulation**（直接内核对象操作）。它是 Windows 内核级 Rootkit（木马/后门）最核心的底层技术之一

简单来说，**DKOM 就是利用内核驱动（Ring 0 权限），绕过操作系统的标准 API，直接潜入内存，对内核里的数据结构** 进行操作

## 数据类型

Uchar是Windows 内核（以及 Windows 底层开发）中特有的数据类型。它其实就是一个 **8位无符号字符（Unsigned Character）**，在底层完全等价于 `unsigned char`

`PUCHAR` 里的 **`P`** 代表 **Pointer（指针）** ，所以，`PUCHAR` 的全称是 **Pointer to Unsigned Character**，即 **“指向无符号字符的指针”**。


```C
HANDLE pid = *(HANDLE*)((PUCHAR)pCurrent + OFFSET_UNIQUE_PROCESS_ID);
```

`pCurrent` 是一个 `PEPROCESS` 类型的指针（指向 `_EPROCESS` 这个巨型结构体的开头），这里强转成无符号字节指针，然后再加上偏移值后，刚好精准地停在 `_EPROCESS` 结构体里 `UniqueProcessId` 字段的**起始位置**。这里再让编译器把这个地址当成一个**指向 8 字节句柄（HANDLE）的指针**”。最后再解引用去刚才算出来的那个 8 字节内存地址里，把真实的数据（也就是 PID 的值）掏出来。

PLIST_ENTRY 是windows内核里的双向链表的指针，p 是指针，list_entry是链表条目，是windows内核定义的一个极简结构体。它的底层定义极其简单，就两个指针：

```c
typedef struct _LIST_ENTRY {
    struct _LIST_ENTRY *Flink; // 指向下一个节点 (Forward Link)
    struct _LIST_ENTRY *Blink; // 指向上一个节点 (Backward Link)
} LIST_ENTRY, *PLIST_ENTRY;
```

我们可以利用它拿当前进程的基址（`pCurrent`）加上偏移量，找到了它里面那个叫 `ActiveProcessLinks` 的字段。

```c
// move
PLIST_ENTRY pListEntry = (PLIST_ENTRY)((PUCHAR)pCurrent +
	  OFFSET_ACTIVE_PROCESS_LINKS);
PLIST_ENTRY pNextEntry = pListEntry->Flink;

```

## 驱动入口

```c
extern "C" NTSTATUS DriverEntry(PDRIVER_OBJECT pDriverObject, PUNICODE_STRING pRegistryPath) {
    UNREFERENCED_PARAMETER(pDriverObject);
    UNREFERENCED_PARAMETER(pRegistryPath);

    DbgPrint("[DKOM] Driver loaded!\n");
    MyListProcesses();

    return STATUS_SUCCESS;
}
```

`DriverEntry` 是 **Windows NT 内核驱动在加载时的标准、固定的入口点（Entry Point）**，等同于用户态应用程序的 `main` 或 `WinMain`。

操作系统的映像加载器（Image Loader）和 I/O 管理器（I/O Manager）在将 `.sys` 文件映射到内核内存后，会硬编码查找名为 `DriverEntry` 的导出符号，并将其作为执行起点。

定义在 `ntddk.h` 中，其标准签名固定为 `NTSTATUS DriverEntry(PDRIVER_OBJECT, PUNICODE_STRING)`。

## 内核函数

RltCopyMemory，这是微软 WDK 提供的一个极其重要的内核函数（类似用户态 C 语言的 `memcpy`）。`ImageFileName` 这个 `UChar` 数组（15字节）**没有 `\0` 结束符**！如果你用 `strcpy`，它会一直往后读，读到不属于这个进程的内存，导致系统直接蓝屏。`RtlCopyMemory` 是“不管里面是什么数据，我只按指定的长度（字节数）强拷”，绝对安全。


我们使用起来类似这样：

```C
CHAR szName[16] = { 0 };
RtlCopyMemory(szName, (PUCHAR)pCurrent + OFFSET_IMAGE_FILE_NAME, 15);
```



## 读取进程

我们都知道windbg有个命令!process 0 0，这个我们经常用来查看所有会话里的进程，要实现这个功能，我们也可以使用dkom里的开发。每个windows版本的偏移量不同，所以开发前务必知道偏移量大小，才能操作链表。


