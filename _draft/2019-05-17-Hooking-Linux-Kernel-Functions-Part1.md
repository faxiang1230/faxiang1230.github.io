---
layout:     post
title:      "Hooking Linux Kernel Functions"
subtitle: "Looking for the Perfect Solution"
date:       2019-05-17 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - 翻译
  - hook
---
# Hooking Linux Kernel Functions
>>原文:https://www.apriorit.com/dev-blog/544-hooking-linux-functions-1

我们最近在开展linux系统上安全相关的工作,在上面我们需要hook内核中重要的函数调用，例如打开文件、创建进程。我们需要这样一个项目来监控系统的活动并且阻塞可疑的进程。
实际上，最后我们采用了一种高效的方式来hook系统,借助于ftrace系统，可以通过函数名称来hook它的方法，并且执行我们的hook代码。这一系列文章有三章，现在是第一章，将会介绍在我们最终采用ftrace之前尝试过的4种方式，给出了每一种方式的有点和缺点。--Alexey Lozovsky
## Overview
有以下几种方式可以劫持linux内核的方法,重点介绍前面4中方法:
- 使用LSM(Linux Security Module)框架
- 修改系统调用表
- 使用kprobe工具
- Splice
- 使用ftrace工具
## 使用LSM(Linux Security Module)框架
## 修改系统调用表
## 使用kprobe工具
## Splice
最初，我们尝试使用Linux Security API(俗称LSM)来进行hook,这也是最好的选择。因为这个接口本来就是为了安全而设计的，在内核代码执行的关键点上埋下了hook点,安全模块可以注册这些hook方法，之后在执行过程中被回调，hook方法可以知道上下文并且也能决定是否禁止这个操作。
但是有两个非常大的缺点:
```
1.LSM需要built-in,它并不能被动态的加载(by insmod),所以我们需要重新编译整个内核才能将我们的安全模块嵌入进去
2.一个系统不能有多个LSM并存
```
内核开发者们对于多个LSM并存可能还有争议，但是他们一致同意LSM不能动态加载，这样在系统启动时安全性才不会有漏洞(从BIOS->kernel->发行版系统形成一个完整的可信计算环境)。
所以为了使用LSM框架,我们需要编译一个定制版的内核，处理和AppArmor,SELinux共存的问题，后面两者是目前流行发型版自带的LSM。
结论就是这个选择并不适合我们.

因为大部分需要监控的是来自用户程序产生的动作，所以可以在系统调用层次上想想办法。所以Linux系统调用处理都存储在`sys_call_table`位置，改变这个表的内容可以改变系统的行为。
最后我们可以通过保存原始的系统调用处理地址，并且把我们自己的hook代码地址放到系统调用表上。
优点:
- 从用户程序到内核的唯一方法之后系统调用，通过hook系统调用能够监控到所有的用户程序产生的动作。
- 更小的性能开销。只需要更新一次系统调用表就完成了大部分工作，另外的开销就是监控动作产生的负载和一次额外的函数调用(原本是直接调用原生的系统调用处理)
- 对于内核要求更小了。理论上这种方法可以在最近的几乎所有的系统上应用，因为你不需要关心内核的配置
缺点:
- 技术上非常复杂，替换掉系统调用表的值并不是很困难，有自己个额外的任务需要保证一定的质量
寻找系统调用表的位置
避开内核中系统调用表区域写权限保护
保证替换过程的安全性能
解决了这些问题意味着开发者必须花费更多的时间在实现、支持和理解整个过程。

一些系统调用处理是无法被简单替代的。在内核4.16版本以前，x86_64架构下的系统调用处理有一些额外的优化。这些优化将系统调用处理用汇编进行了处理。这种类型的处理很难或者不可能被用C语言编写的处理函数替代的，这就意味着不同版本的内核有不同的优化，极大地增加了这项工作的复杂度。
Some handlers can’t be replaced. In Linux kernels prior to version 4.16, system call processing for the x86_64 architecture has some additional optimizations. Some of these optimizations require the system call handler to be implemented in the assembler. These kinds of handlers are either hard or impossible to replace with custom handlers written in C. Furthermore, the fact that different kernel versions use different optimizations boosts the technical complexity of the task even more.
只有系统调用调用被hook。因为这种方式允许替代系统代调用处理，很明显限制了入口。所有额外的参数检查可以在系统调用前或之后，而我们只有系统调用的参数和返回值。结果就是有时候我们需要作一次额外的检查，在进程的访问权限和系统参数的有效性。有些情况我们需要显示地拷贝用户内存两次，这也增加了额外的消耗。例如当参数通过指针传递进来，首先我们自己的处理函数需要拷贝一次内存，原始的系统调用再来拷贝一次，这样就多了额外的一次。有时候系统调用没有对事件做任何的保证，你需要使用额外的过滤操作来摆脱这些无关的信息。
Only system calls are hooked. Since this approach allows you to replace system call handlers, it limits entry points significantly. All additional checks can be performed either immediately before or after a system call, and we only have system call arguments and their return values. As a result, sometimes we may need to double-check both access permissions of the process and the validity of system call arguments. Plus, in some cases the need to copy user process memory twice creates additional overhead charges. For instance, when the argument is passed through a pointer, there will be two copies: the one that you make for yourself and the second one made by the original handler. Sometimes, system calls also provide low granularity of events, so you may need to apply additional filters to get rid of the noise.
首先，我们尝试尽可能多地hook系统调用表里，我们也成功了。但是x86_64上总有一些特殊的feature和一些调用限制我们并不清楚。保证和进程创建相关的系统调用`clone`和`execve`可用，这对于我们来说通常极为重要。这也是为什么我们继续尝试新的解决放方法。
At first, we tried to alter the system call table so we could cover as many systems as possible, and we even implemented this approach successfully. But there are several specific features of the x86_64 architecture and a few hooked call limitations that we didn’t know about. Ensuring support for system calls related to the launch of specific new processes – clone () and execve () – turned out to be critical for us. This is why we continued to search for other solutions.
## 使用Kprobe
我们剩余的选项中有一个是kprobe,最初设计用来作内核的tracing和debugging.Kprobe允许你安装`pre-handlers`和`post-handlers`到任何的内核指令上，通常是函数的入口和返回处。处理函数访问寄存器并且操作他们。这个方式可以让我们有机会来监控和修改事件。
One of our remaining options was to use kprobes – a specific API designed for Linux kernel tracing and debugging. Kprobes allows you to install pre-handlers and post-handlers for any kernel instruction as well as for function-entry and function-return handlers. Handlers get access to registers and can alter them. This way, we could possibly get a chance to both monitor the work process and alter it.
使用Kprobe来tracing内核方法的好处:
- Kprobe本身是一个非常成熟的API.自从2002年开始实现，现在有非常好的文档，里面的坑也基本被填完了。
- 可以hook内核中的任何位置。Kprobe是通过breakpoints(x86中的int3)实现，所以可以hook到你知道的内核中的任何位置。也可以通过操作栈的返回地址实现`kretprobes`来trace任何函数的返回值,除非它本身永远不会返回。
The main benefits of using kprobes for tracing Linux kernel functions are the following:

    A mature API. Kprobes has been improving constantly since 2002. The utility has a well-documented interface and the majority of pitfalls have already been discovered and dealt with.
    The possibility to trace any point in the kernel. Kprobes is implemented via breakpoints (the int3 instruction) embedded in the executable kernel code. Thus, you can set the trace point literally in any part of any function as long as you know its location. Plus, you can implement kretprobes by switching the return address on the stack and trace any function’s return (except for ones that don’t return control at all).
Kprobe的缺点：
- 技术上过于复杂。Kprobe仅仅只是一个在内核中某个位置设置断点的工具。为了获取函数的参数或者局部变量的置，你需要知道它在栈上究竟是怎么摆放的，位于哪个寄存器中，手动将他们取出来。同时要阻塞原始的函数调用的执行，需要手动修改进程的状态，模拟的和从原始的函数返回一样。
Kprobes also has its disadvantages, however:

Technical complexity. Kprobes is only a tool for setting a breakpoint at a particular place in the kernel. To get function arguments or local variable values, you need to know where exactly on the stack and in what registers they’re located and get them out of there manually. Also, to block a function call you need to manually modify the state of the process so you can trick it into thinking that it’s already returned control from the function.
- Jprobe已经过时了。Jprobe是一个特殊的Kprobe实现的版本，它可以让tracing工作更加容易。Jprobe实际上是从寄存器或者站上解压函数的参数，并且调用你自己的处理函数，但是你的处理函数和原始的被trace的函数保持同样的接口。它很完美，唯一的问题是他过时了，将会从最新的内核版本上移除。
Jprobes is deprecated. Jprobes is a specialized kprobes version meant to make it easier to perform a Linux kernel trace. Jprobes can extract function arguments from the registers or the stack and call your handler, but the handler and the traced function should have the same signatures. The only problem is that jprobes is deprecated and has been removed from the latest kernels.
- 开销很大。尽管它只是一次处理，但是定位断点本身开销就很大。断点并不会影响函数的剩余部分，但是它本身花费的代价就很大。幸运的是，在x86_64架构下这个开销可以通过使用`jmp`优化来明显的减小。不过这个开销仍然会比修改系统调用表更大。
Nontrivial overhead. Even though it’s a one-time procedure, positioning breakpoints is quite costly. While breakpoints don’t affect the rest of the functions, their processing is also relatively expensive. Fortunately, the costs of using kprobes can be reduced significantly by using a jump-optimization implemented for the x86_64 architecture. Still, the cost of kprobes surpasses that of modifying the system call table.
- Kretprobes本身的限制。`kretprobes`通过替换返回地址的栈来完成的。为了在处理结束后返回原始的地址，`kretprobes`需要保存原始的地址。这些地址保存在一个固定大小的buffer中。如果这个buffer溢出了，比如系统hook了太多的函数，那么溢出部分的`kretprobes`将会被跳过。
Kretprobes limitations. The kretprobes feature is implemented by substituting the return address on the stack. To get back to the original address after processing is over, kretprobes needs to keep that original address somewhere. Addresses are stored in a buffer of a fixed size. If the buffer is overloaded, like when the system performs too many simultaneous calls of the traced function, kretprobes will skip some operations.
- 禁止抢占。kprobe是基于中断和操作寄存器来实现的。所以为了同步，所有的处理函数都必须在一个禁止抢占的环境中执行。所以对于处理函数有一个约束:不能有睡眠等待，意味着不能分配大块的内存，处理IO,在信号量上等待，处理定时器等等
Disabled preemption. Kprobes is based on interruptions and fiddles with the processor registers. So in order to perform synchronization, all handlers need to be executed with a disabled preemption. As a result, there are several restrictions for the handlers: you can’t wait in them, meaning you can’t allocate large amounts of memory, deal with input-output, sleep in semaphores and timers, and so on.
如果你实在需要trace某个函数内核指令，kprobe仍然是可以使用的。
Still, if all you need is to trace particular instructions inside a function, kprobes surely can be of use.
## Splicing
有一个非常传统的方法来配置内核的方法hook:通过替换函数开始地址的指令为一个为条件的跳转到你自己的处理函数。这个原始的指令挪到一个不同的位置并在返回到上一级函数时重新被调用。通过两次的跳转，可以拼接你的代码到这个过程。
There’s also a classic way to configure kernel function hooking: by replacing the instructions at the beginning of a function with an unconditional jump leading to your handler. The original instructions are moved to a different location and are executed right before jumping back to the intercepted function. Thus, with the help of only two jumps, you can splice your code into a function.
这个方法工作原理和kprobe是一样的。通过拼接的方法可以得到和使用`kprobe`一样的效果，但是代价更小，同时对整个过程有完全的控制。
This approach works the same way as the kprobes jump optimization. Using splicing, you can get the same results as you get using kprobes but with much lower expenses and with full control over the process.
优势:
- 对内核的配置要求最小。不需要内核任何特定的配置，可以在任何函数的开始位置拼接代码，你只需要直到他们的地址。
- 最小的开销。trace的方法需要执行两次无条件的跳转来获取控制并且将控制返回。而拼接跳转很容易利用处理器的流水线，代价更小。
The advantages of using splicing are pretty obvious:

    Minimum requirements for the kernel. Splicing doesn’t require any specific options in the kernel and can be implemented at the beginning of any function. All you need is the function’s address.
    Minimum overhead costs. The traced code needs to perform only two unconditional jumps to hand over control to the handler and get control back. These jumps are easy to predict for the processor and also are quite inexpensive.
但是也有一些缺点:技术上很复杂。替代函数的机器码并不是那么容易，需要完成下面几件事：
- 对称编写hook的安装和移除
- 旁路掉可执行代码区域的写保护
- 在替代完成后Invalidate CPU cache
- 反汇编被替代的指令
- 检查这个函数中是否有跳转指令
- 将原始方法的汇编指令当作整体拷贝走
However, this approach has one major disadvantage – technical complexity. Replacing the machine code in a function isn’t that easy. Here are only a few things you need to accomplish in order to use splicing:

    Synchronize the hook installation and removal (in case the function is called during the instruction replacement)
    Bypass the write protection of memory regions with executable code
    Invalidate CPU caches after instructions are replaced
    Disassemble replaced instructions in order to copy them as a whole
    Check that there are no jumps in the replaced part of the function
    Check that the replaced part of the function can be moved to a different place
当然，你可以使用`livepatch`的框架，在`kprobes`中找一些灵感，但是这个最终的解决方案仍然是非常复杂的，任何这种方式的实现都会存在一些潜在的问题。
如果你已经赚备好处理代码中坑，`splice`仍然是一个非常有用的Hook方案。但是我们不喜欢这个选择，有没有更好的选择呢?
我们看到了内核的`ftrace`,一个可以trace内核函数的框架。它虽然通常作为trace防范，但仍然可以作为jprobe的一个选项。事实证明，`ftrace`比jprobe能够更好地完成我们的工作。
`ftrace`可以使我们通过他的名称就能hook关键的内核函数，并且hook不需要重新编译内核就可以被安装。下一章将会集中介绍ftrace,怎么工作的,给出具体的事例来理解这个过程，同时给出他的优点和缺点。
