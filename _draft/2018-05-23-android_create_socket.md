---
layout:     post
title:      "Android中创建socket"
date:       2018-05-23 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - network
---
# Android中创建套接字的限制  
## 对于网络通信套接字的限制
Android向kernel加入了自己的patch（CONFIG_ANDROID_PARANOID_NETWORK）来增强网络访问限制,
这个patch也是延续了Android工程师的传统`直接`(和它相似的是wake lock的patch)：
**只有inet，net_raw，net_admin，root组才能进行网络通讯，除此之外的用户创建套接字将会遭到内核的拒绝。**  
Apk应用获取通信权限:普通应用是没有网络通信权限的，需要在AndroidManifest.xml中配置请求网络权限
<android.permission.INTERNET>,系统会将具有权限的应用加入到对应的网络组中，赋予应用访问网络权限。
Native程序访问网络权限:vendor通过在init.rc配置文件中将服务进程加入到`INET,INET_RAW`group中
```
service racoon /system/bin/racoon
    class main
    socket racoon stream 600 system system
    # IKE uses UDP port 500. Racoon will setuid to vpn after binding the port.
    group vpn net_admin inet
    disabled
    oneshot
```

## 对于使用套接字进行进程间通信的限制
```
root@firefly-rk3288:/ # ls /dev/socket/*                                       
/dev/socket/adbd
/dev/socket/displayd
...
/dev/socket/vold
/dev/socket/zygote
```
