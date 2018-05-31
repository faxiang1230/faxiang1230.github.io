---
layout:     post
title:      "kernel debug"
date:       2017-07-08 17:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - kernel
  - oops
  - stability
  - debug
---
# kernel DEBUG feature
了解kernel现有的一些debug feature的配置可以免除一些额外的修改-编译-再运行时间;  
debug feature是由内核维护的，code的质量很高，而且是针对关键点来进行debug，这会给我们带来更加全面的信息;  
debug的代码也是我们切入内核的一个很好的点，帮助我们从现象到代码逻辑的理解，例如你一般都可以通过debug feature获取一些信息:
打印，信息汇总等等，拿着已有的结果来回溯代码逻辑,不妨一试;
## debug feature的CONFIG
1. 内核也针对debug的Kconfig文件进行了一些简单的归总,从文件上来看其分布:
```
arch/xxx/Kconfig.debug    arch自己独有的feature
lib/Kconfig.kgdb          kgdb很有用的工具
```
2. 运行起来看`make menuconfig`,Look here
```
Kernel hacking
  -->Memory Debugging
  -->Tracers
  -->Lock Debugging
  ...
```
3. CONFIG_DEBUG_FS
只有这个CONFIG打开之后才能看到其他依赖于Debug fs的输出

4. CONFIG_FRAME_POINTER

如果关闭这个feature，x86上将会EBP 寄存器当做通用寄存器；打开时EBP寄存器总是指向当前栈的基地址;  
当需要debug kernel的时候，打开这个功能吧，只不过会多点汇编，kernel整个size增大一点点而已.
## debug的信息
使用gdb调试应用程序时编译总会加上`gcc -g`来带上符号表信息，调试内核时经常也是需要debug信息的，例如在crash log中总会dump出来
```
crash原因及地址
寄存器 dump
call strace  这一步需要CONFIG_KALLSYMS
```
第一步最简单的使用`addr2line -e vmlinux address`查看最后死的位置，你需要vmlinux里带符号信息，你需要`CONFIG_DEBUG_INFO`;  
打开这个feature之后你可以使用这些工具`crash, kgdb, LKCD, gdb`;  
不过这个size增长的有点猛，大概增加了一倍吧，然后你可以使用`DEBUG_INFO_REDUCED`来适当减少size，也有点副作用  
```
1.需要使用完整的debug信息的工具就不能work:kgdb,systemtap
2.addr2line,objdump还可以正常work
```
`DEBUG_INFO_SPLIT`将会把调试信息单独放到一个文件`.dwo`文件，不过要利用.dwo文件需要比较新的gcc版本(4.7+)  
`CONFIG_READABLE_ASM`会让kernel的汇编代码更加可读一点
## MAGIC键
配置`CONFIG_MAGIC_SYSRQ`之后就会出现节点`/proc/sysrq-trigger`,向其中写值可以达到如下效果

|字符|效果|  
|--|--|  
|c|Performs a system crash by a NULL pointer de-reference|  
|w|Dumps tasks that are in uninterruptable (blocked) state|  
|l|Shows a stack backtrace for all active CPUs.|  
|m|Dumps current memory info to your console.|  
|q|Dumps per CPU lists of all armed hrtimers (but NOT regular timer_list timers) and detailed information about all clockevent devices.|  
|t| Dumps a list of current tasks and their information to your console|  
## printk
新手入门必备的利器，容易使用，健壮，内核级的printf，不必多说了  
有时候printk的log打印过多，一个是占用CPU，另外一个是容易淹没其他重要的消息，可以使用这个API`printk_ratelimited`限制输出的次数
```
#define printk_ratelimit() __printk_ratelimit(__func__)

这个函数应当在你认为打印一个可能会出现大量重复的消息之前调用，如果这个函数返回非零值, 继续打印你的消息, 否则跳过它。典型的调用如这样:

if (printk_ratelimit())
    printk(KERN_NOTICE "xxxx\n");
```
printk_ratelimit通过跟踪发向控制台的消息的数量和时间来工作。当输出超过一个限度, printk_ratelimit 开始返回 0 使消息被丢弃。我们可以通过修改 ：
```
/proc/sys/kern/printk_ratelimit( 可以看作一个监测周期，在这个周期内只能发出下面的控制量的信息)
/proc/sys/kernel/printk_ratelimit_burst(以上周期内的最大消息数，如果超过了printk_ratelimit()返回0)
```
来控制消息的输出.

`CONFIG_DYNAMIC_DEBUG`一个很容易被忽视的选项，比较容易少用，因为它只控制`pr_debug dev_dbg`形式的输出，其他的就不生效了

怎么打开/关闭调试的输出呢?官方文档`kernel/Documentation/dynamic-debug-howto.txt`

### 使用示例:

* 查看可用的有哪些动态打印输出控制
```
cat <debugfs>/dynamic_debug/control
# filename:lineno [module]function flags format
init/main.c:741 [main]initcall_blacklisted =p "initcall %s blacklisted\012"
```
* 控制其打开关闭
可以使用`filename:lineno [module]function flags format`中的几个属性混合来限定开关
```
enable the message at line 1603 of file svcsock.c
nullarbor:~ # echo -n 'file svcsock.c line 1603 +p' >
				<debugfs>/dynamic_debug/control

 disable all 12 messages in the function svc_process()
nullarbor:~ # echo -n 'func svc_process -p' >
				<debugfs>/dynamic_debug/control
```

有时候你可能遇到在console init之前内核就死掉了，那么可能需要这个来获取一些信息:
```
CONFIG_EARLY_PRINTK
  early_printk
CONFIG_EARLY_PRINTK_DIRECT=y
```

## PSTORE

pstore原意persistent store,是想要在系统重启之前能够保存之前的一些信息，重启之后能够再次获取到之前的数据,特别是panic的信息;

为什么不能保存到硬盘上或者是EMMC上呢？  
这些设备都是通过内核控制的，如果内核死掉,只会做一些紧急的操作，一般而言不会再继续操作设备的;

所以这个动作不仅是内核需要一些操作，更重要的是内核下一层需要实现该功能，在x86上`APEI(Advanced Platform Error Interface)->ERST(Error Record Serialization Table)`提供接口，ARM平台上当时没有注意,不过Android是有last_kmg功能的，这个应该也是利用pstore的功能;

CONFIG配置:
```
CONFIG_PSTORE
-->CONFIG_PSTORE_CONSOLE  all kernel message
-->CONFIG_PSTORE_PMSG     usespace  message /dev/pmsg0 /sys/fs/pstore/pmsg-ramoops-<ID>
-->CONFIG_PSTORE_RAM      panic/oops message
-->CONFIG_PSTORE_FTRACE   ftrace message
```
pstore提供的是一套可扩展的机制，ramoops利用这套机制实现了采用ram保存oops信息的功能;
在x86平台上是没有热重启，每次重启都是重新走一遍`BIOS->grub->kernel`的流程，所有设备都需要
`掉电->重新初始化`的过程，所以ramoops名称难免有点误导人;它的实现应该是利用BIOS里的NVRAM来
保存数据，每次启动之后通过APEI接口来将这块NVRAM映射到RAM中来访问;

我一般是用它来抓panic的message,使用方法:  
编辑内核参数，指定NVRAM要映射到RAM上哪个位置和size:  
```
ramoops.mem_size=    不要太大，可以使用<1M
ramoops.mem_address= 物理地址，置于RAM的物理地址范围
```
crash之后，你可以看这个文件
```
mount -t pstore pstore /sys/fs
cat /sys/fs/pstore/console-ramoops
```
ftrace的功能也挺好玩的，不过内核都说的比较清楚了，我就不做搬运工了`Documentation/ramoops.txt`
## KGDB
KGDB的功能能够让用户像调试用户空间程序一样来调试内核，不过也有一些不便:在调试真机时必须使用两台机器，一台充当HOST，一台充当target;  
两台机器需要连线通信啊,除了嵌入式板子还留有UART口之外，传统的PC现在都不再导出UART了，需要使用`usb->uart->交叉双绞线->uart->usb`来连接，那就是你需要两根线，而且两个是需要交叉的，这样两个的串口rx,tx才能和对方的tx，rx接通;  

使用`usb->uart->交叉双绞线->uart->usb`来连接两台机器，配置中都需要支持usb转串的驱动  
### target配置
内核配置:
```
CONFIG_HAVE_ARCH_KGDB=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_KGDB_KDB=y
CONFIG_USB_SERIAL=y
CONFIG_USB_SERIAL_GENERIC=y
```
关掉硬件看门狗,否则当kgdb断点时会触发看门狗重启系统
```
高通:CONFIG_MSM_WATCHDOG_V2
```
告诉KGDB使用USB设备连接:
```
内核参数:kgdboc=ttyUSB0,115200
或者
启动之后告诉内核:echo ttyUSB0 > /sys/module/kgdboc/parameters/kgdboc
```
怎么停止内核来连接上gdb:
```
启动时即等待gdb连接:kgdbwait
启动之后进入debug:sysrq-g或者等待gdb attach就好
```
### host配置
使用minicom来和target通信，主要就是设置对应设备的波特率，位控，校验等和target保持一致；  
如果内核没有等待gdb，那么就在minicom中向target发送break消息等待gdb  
```
Press Ctrl+ A
Press F
Press G
```
gdb连接target
```
gdb ./vmlinux
(gdb) set remotebaud 115200
(gdb) target remote /dev/ttyUSB0
```
开始吧，你的gdb该用起来了
## Ftrace
内核调试主要有三种方式吧:kgdb为代表的gdb类调试,以ftrace为代表的tracepoint调试，以systemtap为代表的kprobe调试  
当然ftrace中也可以通过kprobe_events利用kprobe方法动态开启一些trace点，相对于systemtap来说，缺少后期的数据处理方法；   
用过systemtap都知道，它可以根据filter的数据再次过滤并且能够生成一些history图之类的;  
而ftrace的优势是kernel upstream本身就支持，不需要开发者做什么，you just do it!  

使用ftrace来做:  
- 稳定性相关的debug:  
通过开启对象的一些trace收集历史信息，像clock/regulator/cpufreq/HW controller(I2C/SPI/USB)  
- 性能debug:  
调度器调优，延迟，irq/wakeup等  
- Power debug(这个和驱动的实现方式相关，高通的驱动一般都加入了tracepoint):  
CPU usage /cluster LPM distribution/ bus frequency  
DDR frequency/regulator  
- android systrace:  
android有个很强大的工具systrace，干过系统调优的应该知道这个工具，它就是利用ftrace中的trace_maker来收集信息  

CONFIG配置:
```
CONFIG_TRACEPOINTS
CONFIG_FUNCTION_TRACER
CONFIG_CONTEXT_SWITCH_TRACER
CONFIG_IRQSOFF_TRACER
CONFIG_PREEMPT_TRACER
CONFIG_SCHED_TRACER
CONFIG_NOP_TRACER
CONFIG_STACK_TRACER
CONFIG_FUNCTION_GRAPH_TRACER
```
使用:  
使用之前建议瞄一眼`/sys/kernel/debug/tracing/README`

ftrace中有很多的tracer，可以先选一中
```
cd /sys/kernel/debug/tracing
cat available_tracers
  blk mmiotrace function_graph wakeup_dl wakeup_rt wakeup function nop
echo function_graph > current_tracer
```
打开trace`echo 1 > tracing_on`  

获取trace
```
cat trace  将buffer中的数据dump出来并不清除
或者
cat trace_pipe  等待buffer数据并dump出来，读取之后将从buffer中清除
这个也是环形buffer，只是移动指针的位置而已
```
关闭trace`echo 0 > tracing_on`

特殊用法:从dump中获取出ftrace的数据
```
crash> extend ../crash/crash7.1.0/extensions/trace.so
../crash/crash-7.1.0/extensions/trace.so: shared object loaded
crash> trace dump -t rawtracedata
adroidbug$ trace-cmd report -l rawtracedata
```
trace-cmd:

website: http://git.kernel.org/cgit/linux/kernel/git/rostedt/trace-cmd.git/
## timer统计
CONFIG配置：
```
CONFIG_TIMER_STATS
```
通过这个feature可以统计一段时间内的timer触发情况，对于power debug是比较有用的
```
root@linux-S6:/proc# echo 1 > timer_stats
root@linux-S6:/proc# echo 0 > timer_stats
root@linux-S6:/proc# cat timer_stats
Timer Stats Version: v0.3
Sample period: 8.952 s
Collection: inactive
Number，PID，preocess name，timer callback，how to setup timer
  886,  6973 firefox          schedule_hrtimeout_range_clock (hrtimer_wakeup)
  881, 17966 plugin-containe  schedule_hrtimeout_range_clock (hrtimer_wakeup)
  452,     0 swapper/0        tick_nohz_restart (tick_sched_timer)
  249,     0 swapper/2        tick_nohz_restart (tick_sched_timer)
  667, 10995 Timer            futex_wait_queue_me (hrtimer_wakeup)
```
## list debug
CONFIG配置:
```
CONFIG_DEBUG_LIST
```
遍历链表时将会做一些额外的检查;  
新的node是否被重复添加，node是否被重复删除  
## Lock调试
CONFIG配置:
```
CONFIG_LOCKUP_DETECTOR
```
软件死锁导致内核移植运行在内核模式>20秒,其他的任务没有机会运行，就是没响应;监测到软锁之后即当前的call strack将会被打印到dmesg中;  
硬件死锁(禁止中断啊)将会导致CPU一致运行在内核模式>10秒，不会响应任何的中断；检测到之后dump call stack到dmesg中;  
这个feature的消耗很小；  
一个周期性的hrtime将会产生中断，中断callback来执行喂狗任务,而且这个任务的优先级调到最高以防止因为优先级较低无法被调度，这是检测是否有软锁死，否则喂狗任务无法被调度，产生中断;  
一个NMI中断每隔10产生一个中断来检测CPU是否响应，即硬锁；  
如果platform上没有NMI类似的中断，那么初始化一个12秒的hrtimer到每个CPU上来检查是否响应.

hrtimer和NMI事件都可以通过sysctl watchdog_thresh来设置

## hung状态
CONFIG配置:
```
CONFIG_DETECT_HUNG_TASK
CONFIG_BOOTPARAM_HUNG_TASK_PANIC_VALUE
CONFIG_DEFAULT_HUNG_TASK_TIMEOUT  /proc/sys/kernel/hung_task_timeout_secs
```
开启一个daemon程序`khungtaskd`周期性地记录任务的状态，当发现有任务处于uninterruptable状态超过2mins之后会打印进程call stack，根据是否`BOOTPARAM_HUNG_TASK_PANIC_VALUE=1`来重启系统或者只是简单地报告;  
DEFAULT_HUNG_TASK_TIMEOUT默认设置为2mins，当设置成0时就是disable HUNG task检测功能;  
## mutex
## spinlock
