---
layout:     post
title:      "Android logging system"
date:       2018-05-31 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - Android
---
# Android Logging System
## 概览
Android系统中构建了一个自己的日志系统，允许从普通应用到系统组件，native应用等来记录日志。这个
日志系统和Linux内核自己的日志系统是不一样的，后者通过`dmesg`或者`/proc/kmsg`,`/dev/kmsg`等访问。
不过这个日志系统也是存储在内核空间的buffer中  
![image](/images/Android-logging-kmc-kobayashi.png)  
日志系统包括下面几个组件:
```
内核驱动logger，其中申请的buffer来保存日志信息
C,C++,Java类等帮助记录日志信息的工具类
，例如liblog.so
一个单独的程序来查阅日志:logcat
```
logger驱动申请了四个不同的buffer来分别保存不同的日志信息，可以通过`/dev/log/`下的节点来访问这些buffer
```
main - the main application log
events - for system event information
radio - for radio and phone-related information
system - a log for low-level system messages and debugging
```
日志中的每一条信息都有:一个tag，一个时间戳，日志的重要级别，日志内容。  
存储格式:`event`记录的是二进制信息,这样可以节省空间，但是在读的时候需要额外的处理来解码这些信息;
其他的日志都是直接使用的文本来保存日志内容

日志系统会根据tag来自动将日志路由到不同的buffer中去  
## kernel driver
android自己写的logger驱动，里面主要是创建了四个misc设备，每个设备有一个ring buffer来管理日志
## 日志工具类
应用可以通过`android.util.log`类来记录日志信息，而native的程序可以链接`liblog`来
记录日志.  
其实通过深入`android.util.log`发现它也是通过`JNI->liblog`来间接使用`liblog`记录日志的
最有趣的是提供了一个`log`命令行工具,它属于`toolbox`仍然是android自己写的，里面的
东西大概都是仿照busybox来做的，不过功能都是经过阉割的；另外添加了一些自己的工具，
例如`netcfg`等。在最近的版本中，android重新赞助基于busybox开发，可以参考
CM的busybox提交日志。
```
USAGE: log [-p priorityChar] [-t tag] message
       priorityChar should be one of:
               v,d,i,w,e
```
另外还有一个`logwrapper`工具,在android启动过程中，我们可能会执行一些shell脚本，
此时可以将shell脚本的执行输出从标准输出重定向到android的日志系统中，之后通过logcat
查看
```
root@b8g:/dev # logwrapper                                            
Usage: logwrapper [-a] [-d] [-k] BINARY [ARGS ...]

Forks and executes BINARY ARGS, redirecting stdout and stderr to
the Android logging system. Tag is set to BINARY, priority is
always LOG_INFO.

-a: Causes logwrapper to do abbreviated logging.
    This logs up to the first 4K and last 4K of the command
    being run, and logs the output when the command exits
-d: Causes logwrapper to SIGSEGV when BINARY terminates
    fault address is set to the status of wait()
-k: Causes logwrapper to log to the kernel log instead of
    the Android system log
```
## logcat
我们一般通过logcat来查看日志信息，或者更熟悉的`adb logcat`，android eclipse的日志
都是最终通过logcat来查看日志的，具体的参数可以`logcat -h`

有两种比较常用的:  
1.过滤，通过`-b`来选择日志类别，`-v`来选择日志的级别和tag  
2.清空日志buffer:`-c`,在开始测试之前清空日志，避免干扰  
3.重定向日志到文件，日志本身是保存在内存中的，可以通过重定向到文件中来永久存储日志  
## Changes
自从4.4 kitkat到5.1 Lollipop，默认不再使用内核的logger驱动来保存日志，而是默认使用logd来管理日志信息;  
这是一个进步，没有必要修改内核，同时其他系统也可以移植logd来管理日志,通用性更好  
![image](/images/lollipop-logd.bmp)
