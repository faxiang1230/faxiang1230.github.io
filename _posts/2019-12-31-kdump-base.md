---
layout:     post
title:      "kdump基础"
date:       2019-11-27 23:50:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
---
# kdump基础
## 前言
做软件开发的同学会花费大约1/3的时间来进行调试，一个软件开发人员的技能包括:coding,调试和设计.一个友好的开发环境中必然提供了相当多的调试工具给开发人员，例如linux环境下C/C++有基于信号的core dump文件，java、python等都有自己的dump机制，而kdump就是为内核开发而生的dump机制，是kernel dump的缩写。而kdump比较重，对于内存较大的服务器耗时较长，所以简单的问题可以简单抓取崩溃时的dmesg就可以了，可以利用pstore，例如x86上的ramoops,嵌入式上的mtdoops，是比较轻量的方式。但是这些轻量工具对于解决某些复杂问题上不够给力，仍然需要kdump来抓取内存数据并分析。  

## dump主要用来做什么
dump主要是抓取程序在内存中的数据，你可以使用gdb直接attach某个程序并且`dump memory file start_addr end_addr`，dump做的事情就是这样，将内存中的数据抓取出来。但是每个应用的用户空间太大了，不可能都抓取出来，一般只需要根据`/proc/xxx/maps`的地址段dump就好，所以这么多dump数据还需要有格式，而在linux上elf大行其道，我们通常的core dump是elf的，他会为每个`/proc/xxx/maps`中的每个段都生成一个program tabler header,标记了虚拟地址和其在dump文件中的偏移，这样就能借助于gdb工具进行分析。  
相对于其他的dump机制，他们都是运行于操作系统之上，底层是内核提供机制来支撑，但是内核是直接运行在硬件之上，它借助不了任何其他的机制帮助它，只能自己动手才能丰衣足食，这也是它的复杂之处。  
## kdump工作原理
kdump实际上有两个内核，一个是正常运行的内核，一个我们称为捕获内核/第二内核。  
1.首先需要通过`crashkernel=xxx@xxxx`来专门为kdump功能专门预留一部分内存，用来放置第二个内核和其他数据信息的，这一部分是给捕获内核使用的。  
2.可以通过kexec系统调用，将第二个内核和initrd，vmcore信息，启动参数等传递给第一个内核。  
3.当系统崩溃时，保存当前寄存器信息，检查是否加载了捕获内核，如果存在则跳转到捕获内核，捕获内核和正常内核一样启动，但是会提供`/proc/vmcore`接口能够导出内存镜像信息，可以使用`dd`工具，高级的makedumpfile提供压缩选项。  
4.系统重启，之后就可以使用`crash/gdb`分析dump文件了  

![image](../images/kdump-overview.png)

内核需要一些配置才能正确使用该功能:  
第一个内核需要如下的配置:
```
CONFIG_KEXEC=y
CONFIG_SYSFS=y
CONFIG_DEBUG_INFO=Y
```
捕获内核需要配置:
```
CONFIG_CRASH_DUMP=y
CONFIG_PROC_VMCORE=y
```
## kexec
kexec是个统称，它的名称应该是kernel exec,最重要的一部分工作是从第一个内核跳转到第二个内核，和用户空间的exec系统调用作用是一样的。  
1.它主要有用户态的kexec工具，内核的kexec,其中在内核2.6.16时添加系统调用kexec_load，在内核3.17之后添加了kexec_file_load系统调用，相比于第一个接口，后面的接口对于应用开发人员更加友好，将部分解析工作挪到了内核中。用户空间的程序kexec-tools编译出一个`kexec`可执行文件，它可以做两件事:一个是加载捕获内核，当系统崩溃的时候跳转到捕获内核；还有一个是加载第二个内核并跳转过去，就像是ping-pong一样来回切换内核，这样就达到了内核热升级的目的。我们暂时关心它第一个功能。
2.启动一个系统除了内核镜像还需要一个根文件系统，它定义了init进程使系统从内核态过渡到用户态，可以做一个initramfs或者initrd然后通过`--initrd=`或者`--ramdisk=`加载到内核预留的地址空间中。这部分做的工作类似于grub启动内核的时候加载根文件系统过程，只不过现在是预先加载到地址中。如果想节省一点空间，可以使用直接使用分区当做根文件系统`--append="root=/dev/sda1"`  
3.它除了负责加载捕获内核，还需要创建vmcore相关的信息，在捕获内核启动之后需要知道要保存哪些物理内存的内容并且它需要一个文件头。它根据`/proc/iomem`为每一个连续的内存块创建一个program table header,最后创建一个elf头，组合成vmcore的头信息然后通过`elfcorehdr=`传递个第一个内核，然后在捕获内核启动之后透传给`/proc/vmcore`,这样就控制了dump内核的什么位置并且怎么管理它们。  
4.系统启动还需要一些启动参数,可以通过`--append=`显式指定参数，kexec会直接加载到预留区域。还有一部分不同arch上的kexec可能会追加它特有的参数。  
5.内核启动还需要一些启动代码，这些代码复用第一个内核的某些代码，在kexec时备份这部分启动代码到backup区域，可以在启动捕获内核的时候启动。  

下面是更加形象的解释:  
![image](../images/kdump-design.jpg)  
从图中看到，  
1.需要为第二内核预留部分内存，用来加载内核镜像，initrd,elfcorehdr,backup region等  
2.通过kexec-tools加载1中的内容，其中elfcorehdr是扫描当前的  
## 内核中的kexec
**加载过程**   
继续用户空间的kexec的过程，1.按照kexec的指示将内核，initrd，elfcorehdr，启动参数保存到预留地址区域，还需要预备好启动捕获内核信息，以备panic发生。具体看machine_kexec_prepare  

**panic过程**  
捕获内核启动过程中会使用少量内存启动，通过启动参数“memmap=exactmap”可以限制捕获内核所用的区域，可以防止破坏第一个内核的数据，kexec工具会自动在命令行末尾添加这个参数。
基本的过程就是panic过程中检查是否有crash kernel加载，如果有开始进行捕获内核的启动:
- 1.保存cpu状态到per cpu区域并且保存到PT_NOTE  
- 2.通过IPI或者其他方式停止其他的cpu活动，只保留一个cpu继续下面的活动，这个和开机启动是相似的  
- 3.最后的cpu开始跳转到启动代码，之后执行完结后到新的内核，这个过程全是arch相关的代码，每个arch差异都比较大  

## vmcore导出
捕获内核通过`/proc/vmcore`暴露接口，能够按照elfcorehdr的要求导出第一个内核的内存内容并输出为elf core dump格式的文件;普通的dump基本上就是对内存内容的拷贝，这样就会很大，所以有专门的工具makedumpfile可以不拷贝全是零的页并提供压缩选项。收集完vmcore之后基本上就会重启机器。

vmcore的格式如下:
<table>
<tr>
  <td>ELF header</td>
  <td>Program header:PT_NOTE</td>
  <td>Program header:PT_LOAD</td>
  <td>...</td>
  <td>Per cpu register state</td>
  <td>Dump image</td>
</tr>
</table>

cpu寄存器会被存放在PT_NOTE节，每个CPU一个，每个占据1k字节，这个是arch可以调节的，只是要求存放在elf pragram header中并且供后续的分析工具解析寄存器状态。
## crash工具的分析
TODO
## 缺点
某些非破坏性的并不会触发panic，所以某些环境下可能不会进行kdump
## 参考
1.https://www.ibm.com/developerworks/cn/linux/l-kexec/index.html  
1.Documentation/kdump/kdump.txt  
http://lse.sourceforge.net/kdump/documentation/ols2005-kdump-presentation.pdf  
http://www.361way.com/centos-kdump/3751.html  
