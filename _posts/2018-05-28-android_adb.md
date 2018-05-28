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
通信总框架
## 使用总结
可以通过usb直接连接，也可以通过网络连接:`adb connect <host>[:<port>]`,默认的adbd监听的端口
是`5555`.  
使得`/system`分区重新挂载为可读可写:`adb remount`  
使`adbd`重新以`root`身份运行，主要是在userdebug模式中使用  
当android设备重新启动的时候，可以使用`adb wait-for-device`阻塞等待，一旦adb连接上就开始执行命令  
通过`adb reboot-bootloader`方式可以进入烧写模式，可能是正统的`fastboot`模式，也可能是自己实现的
烧写模式,例如rockchip在自己的u-boot中实现的loader模式  
更多可以使用`adb help all`来获取更多命令  
## android adbd
从init中fork出来
## USB和tcp connect

## android usb gadget driver
drivers/usb/gadget/android.c
## 应用
1.限制adb访问  
不再响应通过usb连接或者网络连接的处理入口，是一种最粗暴的方式来防止debug的方式  
温柔一点的可以增加adb连接权限限制来拒绝普通开发的请求    
2.扩展adb来响应定制命令  
