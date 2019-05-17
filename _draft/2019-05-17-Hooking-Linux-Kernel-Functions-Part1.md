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

一些
Some handlers can’t be replaced. In Linux kernels prior to version 4.16, system call processing for the x86_64 architecture has some additional optimizations. Some of these optimizations require the system call handler to be implemented in the assembler. These kinds of handlers are either hard or impossible to replace with custom handlers written in C. Furthermore, the fact that different kernel versions use different optimizations boosts the technical complexity of the task even more.

Only system calls are hooked. Since this approach allows you to replace system call handlers, it limits entry points significantly. All additional checks can be performed either immediately before or after a system call, and we only have system call arguments and their return values. As a result, sometimes we may need to double-check both access permissions of the process and the validity of system call arguments. Plus, in some cases the need to copy user process memory twice creates additional overhead charges. For instance, when the argument is passed through a pointer, there will be two copies: the one that you make for yourself and the second one made by the original handler. Sometimes, system calls also provide low granularity of events, so you may need to apply additional filters to get rid of the noise.

At first, we tried to alter the system call table so we could cover as many systems as possible, and we even implemented this approach successfully. But there are several specific features of the x86_64 architecture and a few hooked call limitations that we didn’t know about. Ensuring support for system calls related to the launch of specific new processes – clone () and execve () – turned out to be critical for us. This is why we continued to search for other solutions.
Using kprobes

One of our remaining options was to use kprobes – a specific API designed for Linux kernel tracing and debugging. Kprobes allows you to install pre-handlers and post-handlers for any kernel instruction as well as for function-entry and function-return handlers. Handlers get access to registers and can alter them. This way, we could possibly get a chance to both monitor the work process and alter it.

The main benefits of using kprobes for tracing Linux kernel functions are the following:

    A mature API. Kprobes has been improving constantly since 2002. The utility has a well-documented interface and the majority of pitfalls have already been discovered and dealt with.
    The possibility to trace any point in the kernel. Kprobes is implemented via breakpoints (the int3 instruction) embedded in the executable kernel code. Thus, you can set the trace point literally in any part of any function as long as you know its location. Plus, you can implement kretprobes by switching the return address on the stack and trace any function’s return (except for ones that don’t return control at all).

Kprobes also has its disadvantages, however:

Technical complexity. Kprobes is only a tool for setting a breakpoint at a particular place in the kernel. To get function arguments or local variable values, you need to know where exactly on the stack and in what registers they’re located and get them out of there manually. Also, to block a function call you need to manually modify the state of the process so you can trick it into thinking that it’s already returned control from the function.

Jprobes is deprecated. Jprobes is a specialized kprobes version meant to make it easier to perform a Linux kernel trace. Jprobes can extract function arguments from the registers or the stack and call your handler, but the handler and the traced function should have the same signatures. The only problem is that jprobes is deprecated and has been removed from the latest kernels.

Nontrivial overhead. Even though it’s a one-time procedure, positioning breakpoints is quite costly. While breakpoints don’t affect the rest of the functions, their processing is also relatively expensive. Fortunately, the costs of using kprobes can be reduced significantly by using a jump-optimization implemented for the x86_64 architecture. Still, the cost of kprobes surpasses that of modifying the system call table.

Kretprobes limitations. The kretprobes feature is implemented by substituting the return address on the stack. To get back to the original address after processing is over, kretprobes needs to keep that original address somewhere. Addresses are stored in a buffer of a fixed size. If the buffer is overloaded, like when the system performs too many simultaneous calls of the traced function, kretprobes will skip some operations.

Disabled preemption. Kprobes is based on interruptions and fiddles with the processor registers. So in order to perform synchronization, all handlers need to be executed with a disabled preemption. As a result, there are several restrictions for the handlers: you can’t wait in them, meaning you can’t allocate large amounts of memory, deal with input-output, sleep in semaphores and timers, and so on.

Still, if all you need is to trace particular instructions inside a function, kprobes surely can be of use.
Splicing

There’s also a classic way to configure kernel function hooking: by replacing the instructions at the beginning of a function with an unconditional jump leading to your handler. The original instructions are moved to a different location and are executed right before jumping back to the intercepted function. Thus, with the help of only two jumps, you can splice your code into a function.

This approach works the same way as the kprobes jump optimization. Using splicing, you can get the same results as you get using kprobes but with much lower expenses and with full control over the process.

The advantages of using splicing are pretty obvious:

    Minimum requirements for the kernel. Splicing doesn’t require any specific options in the kernel and can be implemented at the beginning of any function. All you need is the function’s address.
    Minimum overhead costs. The traced code needs to perform only two unconditional jumps to hand over control to the handler and get control back. These jumps are easy to predict for the processor and also are quite inexpensive.

However, this approach has one major disadvantage – technical complexity. Replacing the machine code in a function isn’t that easy. Here are only a few things you need to accomplish in order to use splicing:

    Synchronize the hook installation and removal (in case the function is called during the instruction replacement)
    Bypass the write protection of memory regions with executable code
    Invalidate CPU caches after instructions are replaced
    Disassemble replaced instructions in order to copy them as a whole
    Check that there are no jumps in the replaced part of the function
    Check that the replaced part of the function can be moved to a different place

Of course, you can use the livepatch framework and look at kprobes for some hints, but the final solution still remains too complex. And every new implementation of this solution will contain too many sleeping problems.

If you’re ready to deal with these demons hiding in your code, then splicing can be a pretty useful approach for hooking Linux kernel functions. But since we didn’t like this option, we left it as an alternative in case we couldn’t find anything better.
Is there a fifth approach?

When we were researching this topic, our attention was drawn to Linux ftrace, a framework you can use to trace Linux kernel function calls. And while performing Linux kernel tracing with ftrace is common practice, this framework also can be used as an alternative to jprobes. And, as it turned out, ftrace suits our needs for tracing function calls even better than jprobes.

Ftrace allows you to hook critical Linux kernel functions by their names, and hooks can be installed without rebuilding the kernel. In the next part of our series, we’ll talk more about ftrace. What is an ftrace? How does ftrace work? We’ll answer these questions and give you a detailed ftrace example so you can understand the process better. We’ll also tell you about the main ftrace pros and cons. Wait for the second part of this series to get more details on this unusual approach.
Conclusion

There are many ways you can try to hook critical functions in the Linux kernel. We’ve described the four most common approaches for accomplishing this task and explained the benefits and drawbacks of each. In the next part of our three-part series, we’ll tell you more about the solution that our team of experts came up with in the end – hooking Linux kernel functions with ftrace.

Have any questions? You can learn more about our Linux kernel development experience here or contact us by filling out the form below.
