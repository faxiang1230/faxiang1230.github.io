---
layout:     post
title:      "从Linux角度看Android系统"
date:       2018-05-25 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - Android
---
# 从Linux角度看Android系统
很多人都说Android是运行在Linux内核上的，我也知道Android底层跑的是Linux内核，我的认识仅止于此。我对上层的Andrid框架，应用等的认识都没有和Linux模型结合起来，在我的观念中,Linux和Android是完全不同的两个东西，这个观念是完全错误的。Android基于Linux,那么Android就一定遵循Linux的概念模型，下面从几个角度去观察Android是如何基于Linux的。  
![image](/images/ape_fwk_all.jpg)  
我原来看这个图看习惯了，习惯从功能上去划分，我把提供服务的所有代码统统划分到一起，所有在用户空间操纵具体驱动的玩意划分到HAL,这样的划分当然是没有错误的，不过这只是一方面，横向功能划分。还有另外一种划分，纵向进程划分。
```
root@firefly-rk3288:/ # pstree                                   
init-+-adbd-+-sh
     |      |-sh---busybox
     |      `-5*[{adbd}]
     |-debuggerd
     |-displayd---4*[{displayd}]
     |-logd-+-{logd.auditd}
     |      |-{logd.writer}
     |      `-2*[{logd}]
     |-main-
     |      |-d.process.media-+-{Binder_1}
     |      |                 |-{Binder_2}
     |      |                 |-{Binder_3}
     |      |                 |-{DownloadReceive}
     |      |                 |-{FinalizerDaemon}
     |      |                 |-{FinalizerWatchd}
     |      |                 |-{GCDaemon}
     |      |                 |-3*[{Heap thread poo}]
     |      |                 |-{HeapTrimmerDaem}
     |      |                 |-{JDWP}
     |      |                 |-{ReferenceQueueD}
     |      |                 |-{Signal Catcher}
     |      |                 `-{thumbs thread}
     |      |-m.android.phone-+-{AsyncTask #1}
     |      |                 |-{CdmaInboundSmsH}
     |      |                 |-{CdmaServiceCate}
     |      |                 |-{CellBroadcastHa}
     |      |                 |-{DcHandlerThread}
     |      |                 |-{DcSwitchStateMa}
     |      |                 |-{FinalizerDaemon}
     |      |                 |-{FinalizerWatchd}
     |      |                 |-{GCDaemon}
     |      |                 |-{GsmCellBroadcas}
     |      |                 |-{GsmInboundSmsHa}
     |      |                 |-{RILReceiver0}
     |      |                 |-{RILSender0}
     |      |-system_server-+-{ActivityManager}
     |      |               |-{AlarmManager}
     |      |               |-{AudioService}
     |      |               |-2*[{AudioTrack}]
     |      |               |-{BatteryStats_wa}
     |      |               |-{ConnectivitySer}
     |      |               |-{CpuTracker}
     |      |               |-{DisplaydConnect}
     |      |               |-{EthernetService}
     |      |               |-{HeapTrimmerDaem}
     |      |               |-{InputDispatcher}
     |      |               |-{InputReader}
     |      |               |-{LazyTaskWriterT}
     |      |               |-{MountService}
     |      |               |-....//省略了好多服务
     |      |               |-{system_server}
     |      |               `-{watchdog}
     |      |-{FinalizerDaemon}
     |      |-{FinalizerWatchd}
     |      |-{GCDaemon}
     |      |-{HeapTrimmerDaem}
     |      `-{ReferenceQueueD}
     |-mediaserver-+-{ApmAudio}
     |             |-{ApmOutput}
     |             |-{ApmTone}
     |             |-{AudioOut_2}
     |             `-{mediaserver}
     |-servicemanager
     |-surfaceflinger-+
     |                |-4*[{mali-utility-wo}]
     |                `-4*[{surfaceflinger}]
```
通过`pstree`看，和普通的Linux发行版显示的没有什么本质的区别，同样都是通过`树`来维护进程间关系.  
## Android的native服务自动重启
仍然从`pstree`显示结果看，在Android中有个神奇的特性，能够自动重启native服务进程，这个其实就是通过`waitpid`来实现的。
所有的native服务都是通过`init`进程创建的,当子进程退出的时候，父进程`init`可以通过`waitpid`来获取状态信息，之后再次重启，完全就是Linux的应用实现啊。
更详细的看[这里](/_posts/2018-05-23-android_init.rc_oneshot.md)  
## fork出一个虚拟机
android中有个zygote进程，名为`受精卵`,当有新的应用要起来的时候就来zygote这里获取一个虚拟机，
然后运行自己的程序.其实里面就是`fork`和`exec`,只是封装的特别深.  普通应用和`zygote`的关系可
以从`pstree`看，一般的android是不带pstree的，可以自己编译一个带pstree的busybox，或者直接
`cat /proc/应用/status`中的`ppid`.  
1.zygote启动，实际上是启动`app_process`,只不过服务的名称为zygote并通过`prctl`来将自己显示
为zygote.  
启动AndroidRuntime来初始化虚拟机，虚拟机本身可以看做是一个可执行程序，类似于在ubuntu中的`java`.
虚拟机启动之后就能够执行dex格式的程序了，此时根据`/system/etc/preloaded-classes`来将通用的
类加载到虚拟机中.  
虚拟机基本的准备好之后就开始准备zygote功能的准备部分:`com.android.internal.os.ZygoteInit`
2.分裂出system_server进程
为了在应用启动之前，即在zygote分裂功能启动之前，需要先分裂处一个进程来启动system_server，它
本身运行了非常多的服务进程并向外提供服务,可以看`pstree`的显示.  
3.创建socket来准备接收fork的请求，然后一直等待请求的到来然后履行zygote的使命.  
```
cat /proc/185/environ
...ANDROID_SOCKET_zygote=11     　　　　　　　　-->init.rc中配置,init创建的
root@firefly-rk3288:/ # ll /proc/185/fd/11                                   
lrwx------ root     root              2000-01-01 08:33 11 -> socket:[8590]
root@firefly-rk3288:/ # cat /proc/net/unix |grep 8590                          
00000000: 00000002 00000000 00010000 0001 01  8590 /dev/socket/zygote
```
结合source从上面看,zygote就是通过`/dev/socket/zygote`来监听外界的请求的.
4.如何分裂出一个进程
Launcher向system_server中的ActivityManagerService.startProcessLocked
```
pid = fork()
if(pid == 0)
  Zygote.callPostForkChildHooks
  WrapperInit.execStandalone
    Zygote.execShell
else
  wite("/dev/socket/zygote", pid)
```
## 虚拟机的COW
从`fork出一个虚拟机`节看出来，总共就两步，一个是fork,一个是execv.现代的fork中有一个特性`COW`,
能够最小化创建进程的开销，减少物理内存的消耗。下面需要来考察一下execv中是否会影响COW，如果仍然
可以在父子进程间共享大部分代码段和文件映射，那么COW的特性仍然保留着。  
execv的工作:  
1.
通过对比zygote进程结果:  
![image](/images/zygote-apk-maps.jpeg)  
左边是zygote进程，右边是music的应用进程，他们两个除了栈和应用的代码之外几乎完全相同，即指向同
一份物理内存，完美利用了COW.
