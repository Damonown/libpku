1.

伍 异常控制流

学习目标

1. 了解异步异常与同步异常，以及异常控制流与平时的逻辑控制流的差异
2. 理解进程的工作机制，如何通过异常来进行进程切换
3. 理解 Linux 的进程控制机制，掌握 `fork` 的基本用法

前面提到过，进程可能是计算机系统中最伟大的抽象。进程这个概念背后，其实隐藏着一整套系统级机制，从进程切换、用户态与内核态的转换到系统实时响应各种事件，都离不开一个相当熟悉又陌生的概念——异常。在这个基础上，我们会一起来看看，操作系统到底是如何工作的，为什么可以同时执行不同的程序，具体又是通过什么机制来管理这一切的呢？这一讲我们就来看看这之中的奥秘。

3.

异常控制流

从开机到关机，处理器做的工作其实很简单，就是不断读取并执行指令，每次执行一条，整个指令执行的序列，称为处理器的控制流。

4.

到目前为止，我们已经学过了两种改变控制流的方式：

- 跳转和分支
- 调用和返回

这两个操作对应于程序的改变。但是这实际上仅仅局限于**程序本身的控制**，没有办法去应对更加复杂的情况。

5.

系统状态发生变化的时候，无论是跳转/分支还是调用/返回都是无能为力的，比如：

- 数据从磁盘或者网络适配器到达
- 指令除以了零
- 用户按下 ctrl+c
- 系统的计时器到时间

这时候就要轮到另一种更加复杂的机制登场了，称之为异常控制流(exceptional control flow)。首先需要注意的是，虽然名称里包含异常（实际上也用到了异常），但是跟代码中 try catch 所涉及的异常是不一样的。

6.

异常控制流存在于系统的每个层级，最底层的机制称为**异常(Exception)**，用以改变控制流以响应系统事件，通常是由硬件的操作系统共同实现的。更高层次的异常控制流包括**进程切换(Process Context Switch)**、**信号(Signal)**和**非本地跳转(Nonlocal Jumps)**，也可以看做是一个从硬件过渡到操作系统，再从操作系统过渡到语言库的过程。进程切换是由硬件计时器和操作系统共同实现的，而信号则只是操作系统层面的概念了，到了非本地跳转就已经是在 C 运行时库中实现的了。

接下来我们就分别来看看这四个跨域计算机不同层级的异常控制流机制。

9.

异常 Exception

这里的异常指的是把控制交给系统内核来响应某些事件（例如处理器状态的变化），其中内核是操作系统常驻内存的一部分，而这类事件包括除以零、数学运算溢出、页错误、I/O 请求完成或用户按下了 ctrl+c 等等系统级别的事件。

具体的过程可以用下图表示：

要注意的是：事件可能由指令本身触发,也可能由外部条件改变触发(中断)。

（图画在黑板上）

11.

系统在启动时维护了异常表和异常表指针(存在一个特定的寄存器中)，每种事件都有对应的唯一的异常编号。异常发生时，CPU确定异常号，并通过异常表(Exception Table)来确定跳转的位置，跳转调用对应的异常处理代码

12.

异常按来源可以分为两大类（同步异常和异步异常）

13.

异步异常（中断）

异步异常(Asynchronous Exception)称之为中断(Interrupt)，是由处理器外面发生的事情引起的。对于执行程序来说，这种“中断”的发生完全是异步的，因为不知道什么时候会发生。这种情况下：

- 需要设置处理器的中断指针(interrupt pin)
- 处理完成后会返回之前控制流中的『下一条』指令

比较常见的中断有两种：计时器中断和 I/O 中断。计时器中断是由计时器芯片每隔几毫秒触发的，内核用计时器终端来从用户程序手上拿回控制权。I/O 中断类型比较多样，比方说键盘输入了 ctrl-c，网络中一个包接收完毕，都会触发这样的中断。

14.

同步异常

同步异常(Synchronous Exception)是因为执行某条指令所导致的事件，分为陷阱(Trap)、故障(Fault)和终止(Abort)三种情况。

15.

总结一下有下表

| 类型   | 原因       | 行为       | 示例               |
| ---- | -------- | -------- | ---------------- |
| 陷阱   | 有意的异常    | 返回到下一条指令 | 系统调用，断点          |
| 故障   | 潜在可恢复的错误 | 返回到当前指令  | 页故障(page faults) |
| 终止   | 不可恢复的错误  | 终止当前程序   | 非法指令             |

这里需要注意三种不同类型的处理方式，比方说陷阱和中断一样，会返回执行『下一条』指令；而故障会重新执行之前触发事件的指令；终止则是直接退出当前的程序。

16.

系统调用示例

系统调用看起来像是函数调用，但其实是走异常控制流的，在 x86-64 系统中，每个系统调用都有一个唯一的 ID，如

| 编号   | 名称       | 描述      |
| ---- | -------- | ------- |
| 0    | `read`   | 读取文件    |
| 1    | `write`  | 写入文件    |
| 2    | `open`   | 打开文件    |
| 3    | `close`  | 关闭文件    |
| 4    | `stat`   | 获取文件信息  |
| 57   | `fork`   | 创建进程    |
| 59   | `execve` | 执行一个程序  |
| 60   | `_exit`  | 关闭进程    |
| 62   | `kill`   | 向进程发送信号 |

要注意区分清楚异常号和系统调用号,系统调用的异常号是0x80。
(Linux下IA32的系统调用号和x86- 64的不同)

17.

举个例子，假设用户调用了 `open(filename, options)`，系统实际上会执行 `__open` 函数，也就是进行系统调用 `syscall`，如果返回值是负数，则是出错，汇编代码如下：

```
00000000000e5d70 <__open>:
    ...
    e5d79: b8 02 00 00 00     mov $0x2, %eax    # open 是编号 2 的系统调用
    e5d7e: 0f 05              syscall           # 调用的返回值会在 %rax 中
    e5d80: 48 3d 01 f0 ff ff  cmp $0xfffffffffffff001, %rax
    ...
    e5dfa: c3                 retq
```

18.

对应的示意图是：

可以看到，

系统调用类似函数调用:
1.都改变控制流
2.返回时执行下一条指令
3.类似的寄存器使用方式(传参,返回值)


重要区别:
1.函数代码是用户代码,而系统调用由内核处理
2.权限等级不同
3.函数地址 vs 系统调用号

19.

故障示例

这里我们以 Page Fault 为例，来说明 Fault 的机制。Page Fault 发生的条件是：

- 用户写入内存位置
- 但该位置目前还没有被载入内存中

比如：

```
int a[1000];
main()
{
    a[500] = 13;
}
```

那么系统会通过 Page Fault 把对应的部分载入到内存中，然后重新执行赋值语句：

（注意page fault的处理方式是回到该语句重新执行）

20.

但是如果代码改为这样：

```
int a[1000];
main()
{
    a[5000] = 13;
}
```

也就是引用非法地址的时候，整个流程就会变成：

具体来说在Unix下会向用户进程发送 `SIGSEGV` 信号，用户进程会以 segmentation fault 的标记退出，在Windows下则可能会结束于General Protection Error，弹出这样的窗口：

21-25.

从上面我们就可以看到异常的具体实现是依靠在用户代码和内核代码间切换而实现的，是非常底层的机制。

26.

要注意的是FPE(Floating Point Exception)

IEEE规定了浮点数除以0等各种异常值运算的标准，因此浮点数一般运算是不会报FPE的

27.

下面这个程序却会报FPE，原因是FPE不仅仅涵盖浮点数错误，实际包括溢出等各种算数错误

cat /proc/interrupt 可以看到自启动所有异常发生的次数

29.

进程

进程是计算机科学中最为重要的思想之一，进程才是程序（指令和数据）的真正运行实例。之所以重要，是因为进程给每个应用提供了两个非常关键的抽象：一是逻辑控制流，二是私有地址空间。逻辑控制流通过称为上下文切换(context switching)的内核机制让每个程序都感觉自己在独占处理器。私有地址空间则是通过称为虚拟内存(virtual memory)的机制让每个程序都感觉自己在独占内存。这样的抽象使得具体的进程不需要操心处理器和内存的相关适宜，也保证了在不同情况下运行同样的程序能得到相同的结果。

30.

计算机会同时运行多个进程，有前台应用，也后台任务，我们在终端下输入 `top`（或者更酷炫的 `htop`），就可以看到如下的进程信息

（展示htop）

31.-34.

进程切换 Process Context Switch

这么多进程，具体是如何工作的呢？我们来看看下面的示意图：

在多进程模型中，虚线部分可以认为是当前正在执行的进程，因为我们可能会切换到其他进程，所以内存中需要另一块区域来保存当前的寄存器值，以便下次执行的时候进行恢复（也就是所谓的上下文切换）。整个过程中，CPU 交替执行不同的进程，虚拟内存系统会负责管理地址空间，而没有执行的进程的寄存器值会被保存在内存中。切换到另一个进程的时候，会载入已保存的对应于将要执行的进程的寄存器值。

35.

而现代处理器一般有多个核心，所以可以真正同时执行多个进程。这些进程会共享主存以及一部分缓存，具体的调度是由内核控制的，示意图如下：

37.再36.

切换进程时，内核会负责具体的调度，如下图所示

A、B,A、C是并发执行的,而B 、C是顺序执行的。(不知道会不会考?)

（关键：是否有时间重叠）


要注意,并发执行不等于并行(concurrent, parallel)。
你吃饭吃到一半,电话来了,你一直到吃完了以后才去接,这是顺序执行。
你吃饭吃到一半,电话来了,你停了下来接了电话,接完后继续吃饭,这是并
发。
你吃饭吃到一半,电话来了,你一边打电话一边吃饭,这是并行。
并发的关键是你有处理多个任务的能力,不一定要同时。
并行的关键是你有同时处理多个任务的能力。

进程控制 Process Control

39.

我们可以认为，进程有三个主要状态：

- 运行 Running
  - 正在被执行、正在等待执行或者最终将会被执行
- 停止 Stopped
  - 执行被挂起，在进一步通知前不会计划执行
- 终止 Terminated
  - 进程被永久停止

另外的两个状态称为新建(new)和就绪(ready)，这里不再赘述。

Running:正在执行或将要被系统执行(会被调度的进程)
Stopped:stopped by Ctrl-Z (SIGSTOP), continues when receiving SIGCONT(不会被系统调度)
Terminated:在下面三种情况时，进程会被终止：

1. 接收到一个终止信号
2. 返回到 `main`
3. 调用了 `exit` 函数

41.-42.

注意exit()一般是不返回的，同时其定义为void函数。

所有这些C函数都是借助系统调用实现的。

43.

**系统调用的错误处理**

在遇到错误的时候，Linux 系统级函数通常会返回 -1 并且设置 `errno` 这个全局变量来表示错误的原因。使用的时候记住两个规则：

1. 对于每个系统调用都应该检查返回值
2. 当然有一些系统调用的返回值为 void，在这里就不适用

44.

**创建进程**

调用 `fork` 来创造新进程。这个函数很有趣，执行一次，但是会返回两次，具体的函数原型为

```
// 对于子进程，返回 0
// 对于父进程，返回子进程的 PID
int fork(void)
```

子进程几乎和父进程一模一样，会有相同且独立的虚拟地址空间，也会得到父进程已经打开的文件描述符(file descriptor)。比较明显的不同之处就是进程 PID 了。

- 调用一次，但是会有两个返回值
- 并行执行，不能预计父进程和子进程的执行顺序
- 拥有自己独立的地址空间（也就是变量都是独立的），除此之外其他都相同
- 在父进程和子进程中 `stdout` 是一样的

进程图

进程图是一个很好的帮助我们理解进程执行的工具：

- 每个节点代表一条执行的语句
- a -> b 表示 a 在 b 前面执行
- 边可以用当前变量的值来标记
- `printf` 节点可以用输出来进行标记
- 每个图由一个入度为 0 的节点作为起始

对于进程图来说，只要满足拓扑排序，就是可能的输出。我们还是用刚才的例子来简单示意一下：

对应的进程图为

[![img](http://wdxtub.com/images/14625029984869.jpg)](http://wdxtub.com/images/14625029984869.jpg)

回收子进程

即使主进程已经终止，子进程也还在消耗系统资源，我们称之为『僵尸』。为了『打僵尸』，就可以采用『收割』(Reaping) 的方法。父进程利用 `wait` 或 `waitpid` 回收已终止的子进程，然后给系统提供相关信息，kernel 就会把 zombie child process 给删除。

如果父进程不回收子进程的话，通常来说会被 `init` 进程(pid == 1)回收，所以一般不必显式回收。但是在长期运行的进程中，就需要显式回收（例如 shell 和 server）。

waitpid的选项比较多杂,书上讲的比较详细,相关知识点只能靠记忆。



光使用Fork函数有个明显的缺陷：只能复制一份自己，想shell（bash）这样的程序怎么办？如果想在子进程载入其他的程序，就需要使用 `execve` 函数，execve函数通过系统调用读入外部程序,覆盖当前进程的代码(指令)、数据及栈帧(保留PID、打开的文件等)和并执行。

先fork再execve相当于建立子进程并执行外部程序。

52.-53.

由_start设置(7.9节7-14)

在execve加载程序以后,调用7.9节的启动代码,启动代码设置用户栈。并将控制
传递给新程序的主函数
