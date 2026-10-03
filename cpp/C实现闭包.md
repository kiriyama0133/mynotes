
C语言本身是没有闭包的因为C的函数栈是后进先出，函数已返回，栈帧就会销毁，局部变量就会随之消失，要实现闭包，就要手动模拟v8那套把逃逸的变量放在堆上。

闭包可以通过函数指针 + 捕获的环境。

首先先实现一个闭包的结构体，首先需要函数指针和环境数据，然后是创建闭包和销毁闭包。

```c
//闭包结构体
typedef struct { 
    void* (*fn)(void *env, void *args);
    void* env;
} Closure;
//环境结构体
typedef struct CounterEnv
{
    /* data */
    int count;
    int step;
} CounterEnv;
typedef void* (*ClosureFn)(void* env, void* args); // closurefn type

```

创建和销毁的方法：

```c
Closure* closure_new(ClosureFn fn, void* env) {
    Closure* c = malloc(sizeof(Closure));
    if (!c) return NULL;
    c -> fn = fn;
    c -> env = env;
    return c;
}
void closure_free(Closure* c, void (*free_env)(void*)) {
    if (!c) return;
    if (free_env && c-> env) free_env(c->env);
    free(c);
}
void* closure_call(Closure* closure, void* args) {
    return closure->fn(closure->env, args);
}
```

然后是计数器部分，计数器可以创建、执行、销毁：

```c
void* counter_fn(void* env, void* args) {
    CounterEnv* e = (CounterEnv*) env;
    if (args) e->step=*(int*)args;
    e->count += e->step;
    int* result = malloc(sizeof(int));
    *result=e->count;
    return result;
}
void counter_env_free(void* env) {
    free(env);
}
Closure* make_counter(int initial, int step) {
    CounterEnv* env = malloc(sizeof(CounterEnv));
    env->count=initial;
    env->step=step;
    return closure_new(counter_fn, env);
}
```

计数器是一个简单例子，用于验证闭包机制是否是正常工作的。因为count和step都是从make_counter捕获的，count在函数返回之后任然存货，多次调用间保持。每次调用count都累加，证明环境是共享的，c1和c2有独立env，互不干扰。

这里的Closure结构体就类似于v8里的函数对象，然后closure.env就是我们之前提到的 `[[Environment]]` 槽，env指向的堆数据就是Context上下文。

使用：

```c
int main() {
    Closure* c1 = make_counter(0, 1); // 0开始，每次+1
    Closure* c2 = make_counter(100, 10);
    for (int i = 0; i < 3; i++) {
        int* r = (int*)closure_call(c1, NULL);
        printf("c1: %d\n", *r);
        free(r);
    }
    for (int i = 0; i < 2; i++) {
    int* r = (int*)closure_call(c2, NULL);
    printf("c2: %d\n", *r);
    free(r);
    }

    int new_step = 5;
    int *r = (int *)closure_call(c1, &new_step);
    printf("c1 (step=5): %d\n", *r);
    free(r);

    closure_free(c1, counter_env_free);
    closure_free(c2, counter_env_free);

    return 0;
}

[18:17:17] PS E:\leetcode\CPP\build>  .\closure.exe
c1: 1
c1: 2
c1: 3
c2: 110
c2: 120
c1 (step=5): 8
[18:17:19] PS E:\leetcode\CPP\build>
```


C函数栈

这是程序运行时用于管理函数调用的一种后进先出LIFO的内存结构，由操作系统和CPU共同维护，负责保存函数调用时候返回的地址、参数、局部变量等信息。

每当一个函数被调用，系统就会在栈上分配一块内存，称为栈帧。函数执行完毕后，这块内存被自动回收。

```
高地址
  ┌─────────────┐
  │   ...       │
  ├─────────────┤
  │  main 栈帧  │
  ├─────────────┤
  │  foo 栈帧   │  ← 当前执行 foo
  ├─────────────┤
  │  bar 栈帧   │  ← bar 被 foo 调用
  ├─────────────┤
  │   ...       │
  └─────────────┘
低地址
```

一个典型的栈帧包含了返回地址，当函数执行完了之后就会回到调用者的哪条指令；参数，传递给函数的实参，部分通过寄存器传递；局部变量，**函数内部定义的变量**；保存的寄存器，**调用寄存器的值**，用于恢复现场；栈帧指针RBP，指向栈帧底部；栈指针RSP指向当前栈帧顶部。

C的线程栈通常1MB(windows)，或者8MB(linux)，栈溢出就会触发`EXCEPTION_STACK_OVERFLOW` 或 `SIGSEGV`。




