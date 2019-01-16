---
layout:     post
title:      "内核hung debug"
date:       2019-01-16 16:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - debug
  - dump
---
# 内核hung debug
## 内核hung检测
`Documentation/lockup-watchdogs.txt`

两种hung的原因和检测方法:

`softlockup`:抢占被长时间关闭而导致进程无法调度:
本来有个线程可以周期性调度来进行喂狗一类的动作，但是由于锁死无法被调度到，所以相应喂狗的动作无法
完成，硬件超时产生中断来打印当前堆栈信息.原理就是优先级:进程优先级<软中断<中断,所以使用软中断和中断都可以检测这类信息。

`hardlockup`:中断被长时间关闭而导致更严重的问题
因为这时候无法产生常规中断，所以只能借助NMI(Non mask interrupt)类型的中断来强制从中断处理跳出来

hung task:
内核创建了`khungtaskd`周期性来遍历系统所有进程，检查两次时间段进程切换次数，如果没有切换，那就是hung住了
```
sysctl -w kernel.hardlockup_panic=1
sysctl -w kernel.hung_task_panic=1
sysctl -w kernel.softlockup_panic=1
sysctl -w kernel.unknown_nmi_panic=1
```
## hung之后panic
系统hung之后可能只是打印一些WARNING,在调试阶段我们希望能够更加激进地发现这些WARNING,可以设置如下参数,
在系统oops之后不一定panic,不过我们设置成panic而后下面的一些手段来保存内核信息乃至于dump整个内存.
```
sysctl -w kernel.panic=1
sysctl -w kernel.panic_on_io_nmi=1
sysctl -w kernel.panic_on_oops=1
sysctl -w kernel.panic_on_stackoverflow=1
sysctl -w kernel.panic_on_unrecovered_nmi=1
sysctl -w kernel.panic_on_warn=1
```
## 获取崩溃信息
### kdump
centos上默认开启了该选项，kdump的原理就是在内存中保留一小块来运行第二个内核，当主内核崩溃的时候,
panic流程会尝试启动第二个内核，第二个内核运行在受限范围的内存中不会影响主内核的数据，之后压缩
整个内存的内容保存到硬盘上。网上非常多好文章讲这个事，请自行百度。
### 通过netconsole来获取dmesg
内核需要如下配置，当netconsole作为module存在的时候可以通过insmod时指定参数，如果是built-in
就必须在内核启动时指定参数
```
CONFIG_NETCONSOLE
```
接收日志机器上设置:
```
netcat -l -p 6666 -u
```
目标机上设置:
```
modprobe netconsole netconsole=6665@192.168.242.134/ens37,6666@192.168.242.1/00:50:56:c0:00:02

对应格式:netconsole=src-port]@[src-ip]/[],[tgt-port]@/[tgt-macaddr]
```
### ramoops
内核需要如下配置,具体看`Documentation/admin-guide/ramoops.rst`
```
CONFIG_PSTORE=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_FTRACE=y
CONFIG_PSTORE_RAM=y
```
在内核启动时加上参数:
```
ramoops.mem_size=
ramoops.mem_address=
```
保存的dmesg通过`/sys/fs/pstore/console-ramoops`来看
### Discussion
我们有各种各样的dump手段，各有优缺点,其中ramoops后端通过pstore实现,在x86和arm上各有实现,当
你在不同的机器上使用该功能，很有可能没有被支持。  
x86上有成熟的标准`apei`来规定BIOS支持该功能，所以intel家的产品应该都会支持这个功能；
而arm上并没有统一的pstore实现，依赖于各个芯片厂商实现这个功能，很多arm board是没有支持这个
功能的(linaro就是为arm阵营开发这些公共框架来尽量统一实现,很显然他们认为这个优先级较低).
ramoops实现基本和kernel子系统无关，通过BIOS提供的接口将信息保存在NV上，它的独立性能够保证适应
于绝大多数的情景下，缺点也很明显，存储有限，仅能保留少量信息。

netconsole和minicom都有非常悠久的历史，只是底层承载设备不同，目前而言两个子系统目前都比较稳定,
不过它需要额外的一台机器和一些串口线(USB转串)和网络连接。

kdump是目前非常受欢迎的dump方式，抓取到系统的所有信息，能够更加深入地帮助调试更加复杂的问题。
有两个问题，一个是当内存非常大，会花费非常长的时间来dump整个内存和占用一些存储。
另外当文件系统被破坏时可能会拒绝保存dump信息。

|机制|优点|缺点|arch|
|---|---|---|-----|
|ramoops|适应性广|保存信息有限|x86,arm上很凌乱|
|netconsole|适应性广，可能受网络系统影响|保存信息有限|所有平台上，目前基本所有网卡都支持|
|kdump|抓取信息最全面|文件系统bug会影响该功能|所有平台|
