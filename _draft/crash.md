# kdump体系
做软件开发的同学会花费大约1/3的时间来进行调试，一个软件开发人员的技能包括:coding,调试和设计，在各种环境中都提供了相当多的调试工具给开发人员，linux环境下C/C++有基于信号的core dump文件，java、python等都有自己的dump机制，而kdump是为内核开发而生的dump机制，他是kernel dump的缩写。而dump主要是抓取程序在内存中状态，可以有多种格式，而在linux上elf大行其道。相对于其他的dump机制，他们都是运行于操作系统之上，底层有内核机制的支撑，但是内核是直接运行在硬件之上，它借助不了任何其他的机制帮助它，只能自己动手才能丰衣足食。

kdump 是一种基于kexec的内核崩溃转储机制。当系统崩溃时，kdump使用kexec启动到第二个内核。第二个内核通常叫做捕获内核，以很小内存启动以捕获转储镜像。第一个内核保留了内存的一部分给第二内核启动用。由于 kdump 利用 kexec 启动捕获内核，绕过了 BIOS，所以第一个内核的内存得以保留。这是内核崩溃转储的本质。

kdump 需要两个不同目的的内核，生产内核和捕获内核。生产内核是捕获内核服务的对像。捕获内核会在生产内核崩溃时启动起来，与相应的 ramdisk 一起组建一个微环境，用以对生产内核下的内存进行收集和转存。

## kexec工具
kexec是个统称，它主要有用户态的kexec工具，系统调用,内核的kexec feature。其中在内核2.6.16时添加系统调用kexec_load，在内核3.17之后添加了kexec_file_load系统调用，相比于第一个接口，后面的接口对于应用开发人员更加友好，将部分解析工作挪到了内核中。
kexec主要是加载第二内核，而第二内核主要用来kdump功能和另一个是作为热启动。而启动一个系统除了内核镜像还需要一个根文件系统，它定义了init进程使系统从内核态过渡到用户任务。通常还需要一些cmdline，可以理解为系统的启动参数，可以提供给内核本身解析使用，也可以透传给用户进程来控制其行为。

下面是更加形象的解释:
![image](../images/kdump-design.jpg)

从图中看到，
1.需要为第二内核预留部分内存，用来加载内核镜像，initrd,elfcorehdr,backup region等
2.通过kexec-tools加载1中的内容，其中elfcorehdr是扫描当前的
## 内核中的kexec

## vmcore导出
## crash工具的分析

## 参考
1.https://www.ibm.com/developerworks/cn/linux/l-kexec/index.html
1.Documentation/kdump/kdump.txt
http://lse.sourceforge.net/kdump/documentation/ols2005-kdump-presentation.pdf
http://www.361way.com/centos-kdump/3751.html
