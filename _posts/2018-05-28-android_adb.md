---
layout:     post
title:      "Android adb arch"
date:       2018-05-28 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - Android
---
# Android adb arch(UNFINISHED)
Android里面存在一个非常重要的工具:adb,没了它大部分的应用开发人员几乎完全失去了调试手段。  
同时有很多通过adbd的漏洞来攻击android设备,第一个是本身存在一些漏洞，例如比较早版
本的`RageAgainstTheCage`，android应用连接上`adbd 5555端口`之后具有shell权限;只要连上adb
就可以获取、修改用户的数据,所以需要研究以下其本身的
运行原理，增加一些安全的限制  
![image](/images/adb.png)  
我们现在只focus在target端的软件结构  
1.adb底层通信既可以支持USB通信也可以支持TCP/IP网络通信方式，使用USB时依赖android的usb driver  
2.从HOST端收到的命令可以通过`JDWP`调试应用，java本身就支持`JDWP`协议调试而android在这里
也实现了这个接口;  
还可以解析成命令执行android的可执行程序  
## 使用总结
- 可以通过usb直接连接，也可以通过网络连接:`adb connect <host>[:<port>]`,默认的adbd监听
的端口
是`5555`.  
- 使得`/system`分区重新挂载为可读可写:`adb remount`  
- 使`adbd`重新以`root`身份运行，主要是在userdebug模式中使用  
- 当android设备重新启动的时候，可以使用`adb wait-for-device`阻塞等待，一旦adb连接上就
开始执行命令  
- 通过`adb reboot-bootloader`方式可以进入烧写模式，可能是正统的`fastboot`模式，也可能是
自己实现的烧写模式,例如rockchip在自己的u-boot中实现的loader模式  
- 通过adb install调用`pm install`来安装应用等，罗嗦一句，在android中有一些工具:`pm,wm,am`等java程序可以直接调用system_server中服务的接口  

使用`adb help all`来获取更多命令详解  
## android adbd
```                                            
service adbd /sbin/adbd --root_seclabel=u:r:su:s0
    class core
    socket adbd stream 660 system system
    disabled
    seclabel u:r:adbd:s0
```
系统启动时由init中fork出来的一个进程，有网络通信的权限，以`root`身份执行
## USB和tcp connect
![image](/images/adb2.bmp)  
主要的步骤:  
1.初始化ADB中的传输层，即和传输介质无关的，底层目前有两种:usb线连接或者通过网络连接；
主要就是创建一个类似于管道的socketpair，一端阻塞读，另外一端在下面的USB连接/TCP:5555端口有变化
的时候会向这端写信息  
2.初始化USB，开启线程周期性检测USB连接状态变化  
3.初始化TCP socket,监听5555端口  
4.阻塞等待  
5.当有USB或者TCP:5555端口接收到请求时，将其物理传输方式和ADB的传输层挂接起来  

USB和TCP在ADB看起来都是具体传输的方式，传输层主要是约定了在target和host中在传输层通信协议格式，
```
struct message {
    unsigned command;       /* command identifier constant      */
    unsigned arg0;          /* first argument                   */
    unsigned arg1;          /* second argument                  */
    unsigned data_length;   /* length of payload (0 is allowed) */
    unsigned data_crc32;    /* crc32 of data payload            */
    unsigned magic;         /* command ^ 0xffffffff             */
};
```
command有:CONNECT,AUTH,OPEN,READY等,下面主要简单介绍一个`adb shell`怎么搭建起来的:  
当你执行了`adb shell`，你本地的shell prompt消失，取而代之的是target设备上的shell prompt  
```
wangjx@wangjx-N000:～$ adb shell
root@b8g:/ #
```
看起来像远程的控制台直接到本地来了:  
1.打开一个`/dev/ptmx`,获取一个新的句柄fd，对应的是创建一个`/dev/pts/x`节点，可以通过读写`/dev/pts/x`来模拟终端  
2.fork一个进程，重定向标准输入、标准输出、标准出错到`/dev/pts/x`,之后执行`/system/bin/sh`，这样shell的所有输入输出都被重定向到了`/dev/pts/x`，之后可以通过`/dev/ptmx`中的句柄fd来获取shell的输入输出
3.将fd和传输层连接起来  
最终看起来的结构像这样:  
![image](/images/adb3.bmp)  
**Ref:**  
system/core/adb/protocol.txt  
system/core/adb/OVERVIEW.TXT  
## android usb gadget driver
drivers/usb/gadget/android.c
## 应用
1.限制adb访问  
- 不再响应通过usb连接或者网络连接，一种最粗暴的方式来防止debug的方式  
- 温柔一点的可以增加adb连接权限限制来拒绝普通开发的请求    
- 给普通用户使用时，新的adb连接需要用户手动确认  

2.扩展adb来增加定制命令    
