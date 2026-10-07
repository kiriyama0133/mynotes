---
date: 2026-07-10
tags:
  - win32
---
这个笔记我想要探讨一下win32平台下的异步和平台调用方法。

## 事件对象


CreateEventA是一个windows win32 api提供的事件对象，在我原先的系统设计里，我是打算将AsyncPlatform当作一个跨线程的唤醒机制，我想创建一个windows平台层抽象，其中的CreateEventA创建了一个windows内核事件对象，用来让Runtime的等待线程被唤醒。


```c
struct AsyncPlatform {
    HANDLE wake_event;
};

AsyncPlatform* async_platform_create(void) {
    AsyncPlatform* platform = calloc(1, sizeof(AsyncPlatform));
    if (!platform) return NULL;
    platform->wake_event = CreateEventA(NULL, FALSE, FALSE, NULL); // 
    if (!platform->wake_event) {
        free(platform);
        return NULL;
    }
    return platform;
}
```

CreateEventA的函数原型大致是：

```c
CreateEventA(
    _In_opt_ LPSECURITY_ATTRIBUTES lpEventAttributes,
    _In_ BOOL bManualReset,
    _In_ BOOL bInitialState,
    _In_opt_ LPCSTR lpName
    );

```

第一位是安全属性，用于决定这个event对象的安全属性和创建之后句柄是否可以被子进程继承，我们传入NULL就表示默认使用安全属性。第二位是决定手动重置还是自动重置，自动表示某个线程成功等待到了这个event之后windows就会自动把event恢复从signaled -> nosignaled，这个很适合只有一个消费线程的场景。第三个参数决定创建出来是什么状态，选用FALSE就表示创建出来就是nosignaled，这个最常用。最后的就是个event起名字，若是使用NULL就是表示这是一个匿名event，一般匿名事件对象只有在当前进程使用。


```c
void async_platform_destroy(AsyncPlatform* platform) {
    if (!platform) return;
    if (platform->wake_event) {
        CloseHandle(platform->wake_event);
        platform->wake_event = NULL;
    }
    free(platform);
}
```

这是很典型的资源生命周期成对设计，先释放platform内部持有的资源，然后再释放platform自身。

我们需要把WaitForSingleObject封装一下，async_platform_wait(platform, 1000);表示最多等待1000ms，如果1000ms内的Event被唤醒，就马上返回，如果1000ms内什么都没有发生，就超时返回。
```c
int async_platform_wait(AsyncPlatform* platform, int timeout_ms) {
    if (!platform) return -1;
    DWORD timeout;
    if (timeout_ms < 0) {
        timeout = INFINITE;
    } else {
        timeout = (DWORD)timeout_ms;
    }
    DWORD result = WaitForSingleObject(platform->wake_event, timeout);
    if (result == WAIT_OBJECT_0) {
        return 1;
    } else if (result == WAIT_TIMEOUT) {
        return 0;
    } else {
        return -1;
    }
}
void async_platform_wakeup(AsyncPlatform* platform) {
    if (!platform) return;
    SetEvent(platform->wake_event);
}

```

并且timeout_ms < 0的话，就设置timme为infinite，一直等待，不设置超时时间。所以具体来说就是让当前线程等待wake_event，最多等待timeout毫秒。SetEvent win32api会把一个event对象设置为signaled状态，这两个就是配对关系，上面的负责等待event，上面的负责通知event有信号了，可以唤醒了。
```
线程 A                         线程 B

async_platform_wait()
       │
       ▼
WaitForSingleObject()
       │
       │ 阻塞
       │
       │                    async_platform_wakeup()
       │                           │
       │                           ▼
       │                    SetEvent(event)
       │                           │
       ◄───────────────────────────┘
       │
       ▼
    被唤醒
       │
       ▼
    return 1
```

那么为了主线程可以不需要被阻塞，还可以继续运行其他同步代码，就需要制作worker线程，因为现在callback是同步执行的，因此callback如果是耗时任务，那么主线程就会被阻塞。worker的工作职责很清晰，就是wait之后取op，设置state为running，并执行callback，完成了再设置completed。

那么首先要思考第一个问题，加入了worker之后main thread和worker thread再pending queue上会有两个线程一起访问，这时候不能继续裸操作这些变量，需要思考线程同步了。在windows上，可以比较简单地使用CRITICAL_SECTION。

```c
static unsigned __stdcall platform_thread_entry(void* arg) {
    AsyncPlatformWorker* worker = (AsyncPlatformWorker*)arg;
    if (!worker) return 0;
    worker->entry(worker->context);
    free(worker);
    return 0;
}

int async_platform_start_worker(AsyncPlatform* platform, void* (*entry)(void*), void* context) {
    if (!platform || !entry) return -1;
    if (platform->worker) return -1;
    AsyncPlatformWorker* worker = malloc(sizeof(AsyncPlatformWorker));
    if (!worker) return -1;
    worker->entry = entry;
    worker->context = context;
    uintptr_t handler = _beginthreadex(NULL, 0, platform_thread_entry, worker, 0, NULL);
    if (handler == 0) {
        free(worker);
        return -1;
    }
    platform->worker = (HANDLE)handler;
    return 0;
}
```

通过worker来接管异步任务，和使用lock来同步线程之后，我们就不需要run和poll两个方法了，新的架构就可以变成：

```
submit()
 ↓
queue
 ↓
wakeup
 ↓
Worker
 ↓
callback
```

如果想要扩展为多个works支持的话就可以通过在runtime添加count和limit来实现，也是十分简单。我设想，如果为了实用性考虑，一般是会忽略掉runtime的创建的，因此随考虑设置全局count数量和全局的runtime，这样在第一次创建operation的适合顺便创建就可以将runtime的职责隐藏起来，可能更加符合开发者的习惯。并且我们可以使用atexit方法，给程序注册一个正常退出要执行的方法，这样退出就可以正常销毁资源了。

```c
static void async_runtime_cleanup(void)
{
    if (async_runtime_global)
    {
        async_runtime_destroy(async_runtime_global);
        async_runtime_global = NULL;
    }
}

atexit(async_runtime_cleanup);
```


之后，我们再去头文件编写一些宏，就可以方便地使用了：


```c
#define ASYNC_WORKER_LIMIT(worker_limit) \
    limit_async_worker_count(worker_limit)

#define ASYNC_CREATE(callback, context) \
    async_operation_create(callback, context)

#define ASYNC_SUBMIT(operation) \
    async_operation_submit(operation)

#define ASYNC_CANCEL(operation) \
    async_operation_cancel(operation)

#define ASYNC_AWAIT(operation) \
    async_operation_await(operation)

#define ASYNC_RESULT(operation) \
    async_operation_result(operation)
    
#include <stdio.h>
#include <windows.h>
#include "runtime.h"

static void task1(AsyncOperation* operation, void* context)
{
    (void)operation;
    (void)context;
    printf("task start\n");
    Sleep(3000);
    printf("task done\n");
}
static void task2(AsyncOperation* operation, void* context)
{
    (void)operation;
    (void)context;
    printf("task start\n");
    Sleep(3000);
    printf("task done\n");
}

int main(void)
{
    AsyncOperation* operation1 = ASYNC_CREATE(task1, NULL);
    AsyncOperation* operation2 = ASYNC_CREATE(task2, NULL);
    // ASYNC_SUBMIT(operation);
    // AsyncResult result = ASYNC_AWAIT(operation);
    // printf("state = %d\n", result.state);
    // return 0;
    printf("main continues\n");
    Sleep(4000);
    printf("state = %d\n", async_operation_state(operation1));
    printf("state = %d\n", async_operation_state(operation2));
    return 0;
}   

```

我这里不想让runtime来决定主线程的延续，所以开发里还是要开发者自行去判断程序退出的时机。

## 父子异步

父子异步的价值在于让运行时替你管理关系，如果一个逻辑任务是一颗任务树，比如加载一个页面，其中可能会面临这load_page，这样的话会可能包含了好几个fetch，请求不同资源，这样我们可以这样表达：

```
load_page(page_id)                    ← root
   ├── fetch_header(page_id)          ← child A
   ├── fetch_body(page_id)            ← child B
   ├── fetch_comments(page_id)        ← child C
   └── fetch_sidebar(page_id)         ← child D
```

用户点了取消或者切换到别的页面，那么有父子的好处就出来了，一次调用，整棵树直接自动取消。因为子任务可能是运行时动态派生的，事先不知道有几个，有多深，用列表手动管理会漏，会乱，父子链让派生这个动作本身自带归属。

因此我们在AsyncOperation的基础上新增几个字段：

```c
    AsyncOperation* parent;
    AsyncOperation* first_child;
    AsyncOperation* sibling_next;
```

来表达父亲、第一个孩子、下一个兄弟。

我们需要实现一个把operation从它父节点的first_child兄弟链里摘掉，并且清空它的parent/sibling_next。必须在子 operation 即将"不再被父的取消链需要"时调用。核心原则：

>[tip]**只要一个子 operation 已经从父的取消链中"用完"，就要摘链。否则父的 `first_child` 链里会留着已释放/已结束的节点，造成悬挂指针或误取消。**

在runtime_worker里，回调执行完，state定稿之后，需要执行一次：

```c
operation->state = ASYNC_OPERATION_RUNNING;
if (operation->callback)
    operation->callback(operation, operation->context);

if (operation->state == ASYNC_OPERATION_RUNNING) {
    operation->state = ASYNC_OPERATION_COMPLETED;
    operation->error = ASYNC_ERROR_NONE;
}

async_operation_unlink_from_parent(operation);   // <<< 这里
```

这个子已经跑完了，父以后cancel时不需要管他，如果不去掉，父亲cancel的时候还会遍历到，这样链会越来越长，且这个op之后被destory释放，父链里就留下悬挂指针，下次cancel直接崩溃。

实现大致如下：

```c
void async_operation_unlink_from_parent(AsyncOperation* operation) {
    if (!operation|| !operation->parent) return;
    AsyncOperation* parent = operation->parent;
    AsyncRuntime* runtime = operation->runtime;
    if (!runtime) {
        operation->parent = NULL;
        return;
    }

    async_platform_lock(runtime->platform);
    AsyncOperation** current = &parent->first_child;
    while(*current) {
        if (*current == operation) {
            *current = operation->sibling_next;
            break;
        }
        current = &(*current)->sibling_next;
    }
    operation->parent = NULL;
    operation->sibling_next = NULL;
    async_platform_unlock(runtime->platform);  
}
```

这里我还想说一下EnterCriticalSection 和  LeaveCriticalSection的原理是什么，我们目前为止到现在锁的实现都是依靠这些，这个是windows上最常用的用户态互斥原语。回想计算机系统的知识，同一进程内互斥，不能跨进程。同一线程可重入（递归锁）：同一线程多次enter不会死锁，但必须对应次数的leave。这个底层是 **`RTL_CRITICAL_SECTION`**  结构体，不是内核对象。

```c
typedef struct _RTL_CRITICAL_SECTION {
    PRTL_CRITICAL_SECTION_DEBUG DebugInfo;  // 调试用
    LONG LockCount;        // 锁计数（负值表示被占用，绝对值-1 = 等待线程数）
    LONG RecursionCount;   // 重入次数
    HANDLE OwningThread;   // 当前持有锁的线程 ID
    HANDLE LockSemaphore;  // 一个事件对象，用于阻塞等待（懒创建）
    ULONG_PTR SpinCount;   // 自旋次数
} RTL_CRITICAL_SECTION;
```

LockCount = -1就表示无人持有，-2表示有人持有且有1个线程在等。大多数情况下`EnterCriticalSection` 走**纯用户态**路径，他会先尝试用 InterlockedIncrement/CompareExchange 把 LockCount 从 -1 改成 0，成功了的话就设置 OwningThread = 当前线程，RecursionCount = 1，然后直接返回，完全不进入内核。

这个没有系统调用，没有上下文切换，没有内核对象，比mutex快得多，这个也是原因，因为mutex每次WaitForSingleObject都要陷入内核。

当锁被其他线程持有的时候，EnterCriticalSection的时候会自选SpinCount次，默认是0.自选失败的话就会懒创建LockSemaphore，把 LockCount 减到更负，记录等待者数量，然后WaitForSingleObject(LockSemaphore, INFINITE) → 线程睡眠，直到唤醒之后重新尝试获取锁。

`LeaveCriticalSection` 的时候，RecursionCount--，如果还 > 0，直接返回（重入没退完），清空OwningThread，把 LockCount 加 1（释放），如果 LockCount 表明还有等待者（< -1） → SetEvent(LockSemaphore) 唤醒一个等待线程。

重入的实现：

```c
EnterCriticalSection:
  1. 检查 OwningThread == 当前线程？
  2. 是 → RecursionCount++，直接返回，不阻塞;

EnterCriticalSection(&cs);
EnterCriticalSection(&cs);   // 不会死锁
LeaveCriticalSection(&cs);   // RecursionCount 2→1
LeaveCriticalSection(&cs);   // RecursionCount 1→0，真正释放
```

ok讲完了，我们回到刚才的话题，既然这样，那我们的cancel方法也要升级为递归类型的，这样才可以匹配我们的任务：

```c
static void async_operation_cancel_locked(AsyncOperation* op)
{
    if (!op) return;

    /* 已取消的子树无需重复遍历 */
    if (op->state == ASYNC_OPERATION_CANCELLED) return;

    /* 只有 PENDING 能直接置为 CANCELLED；
     * RUNNING 的 op 只能靠 token 让回调自愿退出，这里改不了 */
    if (op->state == ASYNC_OPERATION_PENDING) {
        op->state = ASYNC_OPERATION_CANCELLED;
        op->error = ASYNC_ERROR_CANCELLED;
    }

    for (AsyncOperation* c = op->first_child; c; c = c->sibling_next)
        async_operation_cancel_locked(c);
}

void async_operation_cancel(AsyncOperation* operation)
{
    if (!operation) return;
    AsyncRuntime* runtime = operation->runtime;
    if (!runtime) return;

    async_platform_lock(runtime->platform);
    async_operation_cancel_locked(operation);
    async_platform_unlock(runtime->platform);

    /* 唤醒 worker，让它们丢弃 pending 队列里已取消的 op */
    async_platform_wakeup(runtime->platform);
}
```

这样我们便实现了。

## cancellation

我想要实现可以强制停止的异步实现，但是我们的windows平台的实现实际上是通过_beginthreadex实现的多线程，**用 `TerminateThread` 强制杀线程会破坏 C 运行时的状态**。这是 Windows 上所有强杀线程方案的坑。

所以，`_beginthreadex` 创建的线程，**绝对不能用 `TerminateThread` 杀**。TerminateThread会立即把线程标记为终止并不等待它运行到安全点，也不立即执行__finally块，SEH的__excpt、C++析构，或是栈展开。不释放任何锁、内存、句柄。不调用CRT的per-thread清理，线程的栈不回收，可能造成泄露整块栈内存。

一般，`_beginthreadex` 为每个线程分配一个 `_tiddata` 结构（线程本地存储、errno、strtok 状态、随机数种子等）。正常退出时 `_endthreadex` 会释放它。但TerminatThread不会调用，所以`_tiddata` **永久泄漏**。如果该线程正在用 CRT 函数（`malloc`、`printf`、`strtok`…），这些函数内部可能持有**全局锁**（如 `_malloc_lock`、`_io_lock`），锁永远不释放

CRT 的 `malloc` / `free` 内部有**全局堆锁**（或 per-heap 锁）。如果线程在 `malloc` 中途被 `TerminateThread` 杀掉：

```
线程 A: 进入 malloc → 拿堆锁 → 正在改空闲链表 → 被 TerminateThread 杀掉
                  ↑ 堆锁永远不释放，空闲链表处于半改状态
线程 B: 调 malloc → 拿不到锁 → 永远阻塞
       或拿锁后 → 看到半损坏的空闲链表 → 崩溃 / 内存损坏
```

目前我们的打断机制只有pedding能够取消，running打断不了，所以还需要支持一下running运行过程中的打断，要支持RUNNING取消，我们需要去加token。

继续更新...
