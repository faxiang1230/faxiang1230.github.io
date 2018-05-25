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
Thanks for Randy'a question,I know what I don't know and learn a different view of Android.  
很多人都说Android是运行在Linux内核上的，我也知道Android底层跑的是Linux内核，我的认识仅止于此。
我对上层的Andrid框架，应用等的认识一直都没有和Linux模型结合起来，没有认真从Linux模型的角度来
审视Android系统，在我的观念中,Linux和Android是完全不同的两个东西，这个观念是完全错误的。
Android基于Linux,那么Android就一定遵循Linux的概念模型，下面从进程的角度去观察Android是如何基于Linux的。  
![image](/images/ape_fwk_all.jpg)  
我原来看这样的图看习惯了，习惯从功能上去划分，我把提供服务的所有代码统统划分到一起，所有在
用户空间操纵具体驱动的玩意划分到HAL,这样的划分当然是没有错误的，不过这只是一方面，横向功能
划分。还有另外一种划分，纵向进程划分。首先使用一个`pstree`工具来观察Android中进程的关系:
```
root@firefly-rk3288:/ # pstree     //以下结果中省略了很多重复的线程显示                              
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
     |      |                 |-{DownloadReceive}
     |      |-m.android.phone-+-{AsyncTask #1}
     |      |                 |-{CdmaInboundSmsH}
     |      |                 |-{CdmaServiceCate}
     |      |                 |-{CellBroadcastHa}
     |      |                 |-{DcHandlerThread}
     |      |                 |-{DcSwitchStateMa}
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
通过`pstree`看，和普通的Linux发行版显示的没有什么本质的区别，同样都是有`树`状表来维护进程间关系.  
**PS:一般的android是不带pstree的，可以自己编译一个带pstree的busybox**
## Android的native服务自动重启
仍然根据`pstree`显示结果来出发，在Android中有个神奇的特性，能够自动重启native服务进程，
这个其实就是通过`waitpid`来实现的。所有的native服务都是通过`init`进程创建的,当子进程退出
的时候，父进程`init`可以通过`waitpid`来获取状态信息，之后再次重启，完全就是利用Linux的机制啊。  
更详细的看[这里](https://faxiang1230.github.io/2018/05/23/android_init.rc_oneshot)  
## system_server和fork虚拟机
android中有个zygote进程，名为`受精卵`,当有新的应用要起来的时候就来zygote这里获取一个虚拟机，
然后运行自己的程序.其实里面就是`fork`，将现有的虚拟机拷贝一份，然后去在虚拟机上去运行新的应用。  
普通应用和`zygote`的关系可以从`pstree`看，或者直接`cat /proc/应用/status`中的`ppid`.  
1.zygote启动，实际上是启动`app_process`,只不过服务的名称为zygote并通过`prctl`来将自己显示
为zygote.  
启动AndroidRuntime来初始化虚拟机，虚拟机本身可以看做是一个可执行程序，类似于在ubuntu中的`java`.
虚拟机启动之后就能够执行dex格式的程序了，此时根据`/system/etc/preloaded-classes`来将通用的
类加载到虚拟机中.  
虚拟机基本的准备好之后就可以运行java程序，其中重要的是system_server和zygote的功能实现部分
`com.android.internal.os.ZygoteInit`  
2.分裂出system_server进程  
为了在应用启动之前，即在普通应用启动之前，需要先分裂出一个进程来启动system_server，它
本身运行了非常多的服务进程并向外提供服务,可以看`pstree`的显示.   
3.zygote功能
在`init`进程创建socket的时候就已经为它创建了一个socket套接字用于本地进程间通信，之后会发布给
zygote进程;zygote进程拿到socket之后就阻塞接收fork的请求，然后一直等待请求的到来然后履行zygote的使命.  
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
启动普通应用时发起fork的过程简要如下,Launcher或者其他应用向system_server发起请求，之后过程如下:
```
ActivityManagerService->startProcessLocked
  Process.start("android.app.ActivityThread", processname)
    Process.startViaZygote("android.app.ActivityThread", processname)
      Process.zygoteSendArgsAndGetResult
```
而zygote收到请求后fork新的进程并将pid信息返回，之后启动ActivityThread类的main方法
```
pid = fork()
if(pid == 0)
  Zygote.callPostForkChildHooks
  Runtime.zygoteInit
    Runtime.applicationInit
      Runtime.invokeStaticMain
        ActivityThread.main
else
  wite("/dev/socket/zygote", pid)
```
## 虚拟机的COW
从`fork出一个虚拟机`节看出来，总共就两步，一个是fork,一个是虚拟机执行新的代码.
现代的fork中有一个特性`COW`,能够最小化创建进程的开销，减少物理内存的消耗。而在Android创建
新的应用时也实现了这样的特性。  
```
1.虚拟机的部分，即Dalvik/art的部分是完全可以COW的，不需要任何的特殊操作
2.Android设计人员应该是发现几乎所有的普通应用都会使用某些资源,包括java程序，共享库，图片，
字体等资源，他们就采用了预加载的方式先将这些资源统一load到内存中，当应用真的需要的时候它已经
存在在内存中了，避免了I/O等待;同时在物理内存中只存在一份，没有增加物理内存的负担,只是提前
增加了一些虚拟地址的维护
预加载资源的实现类似于ubuntu中的readahead机制，通过在启动过程中记录读取文件，在下一次启动
过程中一开机就将文件load到内存中，避免CPU和I/O的互相等待。
```
预加载资源是可以由vendor调整的，比如预加载类可以通过`/system/etc/preloaded-classes`增减  

其中有些参数会导致`execv`调用，不过这种方式的调用将会把旧的虚拟地址空间完全抹除，不存在COW特性的。
下面简要考察一下execv中是否会保留COW。  
**事实证明执行execv这种动作的时候是会将旧的mm_struct中的各种vma都释放掉的，执行execv就完全
是个新的进程了，和COW的机制完全没有一点关系。**  
![image](/images/do_execve.png)  
关于虚拟地址空间的替换大概过程如下:  
1.为新的mm_struct申请资源  
申请新的mm_struct,然后根据新的可执行文件的格式选择处理binfmt的处理钩子，elf格式即
`load_elf_binary`来填充mm_struct的各种vma  
2.替换掉旧的mm_struct  
准备替换mm_struct:`flush_old_exec->exec_mmap->activate_mm`  
3.释放掉旧的mm_struct  
释放掉旧的mm_struct及维护的vma:`flush_old_exec->exec_mmap->mmput`  
4.继续填充新的mm_struct资源  

通过对比zygote进程和普通应用进程的虚拟地址空间,虚拟机的部分应该是完全没有动，预加载资源也是
几乎没改变，只是将新的apk加载进来，从而实现了COW效果，加速了应用启动。  
![image](/images/zygote-apk-maps.jpeg)  
左边是zygote进程，右边是music的应用进程，他们两个除了栈和应用的代码之外几乎完全相同，即指向同
一份物理内存，我们可以推导出新的zygote进程只是进行了`fork`过程，之后新的进程只是去load新的应用
的apk和dex等  
## 手动启动新的java应用(UNFINISHED)
在android中是没有java等jdk存在的，我们如果想要执行java应用，需要两步:  
1.将java的字节码转换为dex码
```
$ cat HelloWorld.java
public class HelloWorld {
	public static void main(String[] args) {
		System.out.println("Hello World");
	}
}

$ javac HelloWorld.java
$ out/host/linux-x86/bin/dx --dex --output HelloWorld.dex HelloWorld.class
$ out/host/linux-x86/bin/dex2oat --dex-file=HelloWorld.dex --oat-file=HelloWorld.oat
```
2.通过app_process来启动java程序
```
frameworks/base/cmds/pm/pm
base=/system
export CLASSPATH=$base/framework/pm.jar
exec app_process $base/bin com.android.commands.pm.Pm "$@"
```
