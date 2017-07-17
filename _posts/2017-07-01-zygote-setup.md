---
layout:     post
title:      "zygote启动过程简述"
date:       2017-07-01 16:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - andrioid
  - zygote
---
# zygote 启动简述
从ZygoteInit的main方法开始
1. 创建一个socket:/dev/socket/zygote来监听其他服务对自己的fork请求
```
registerZygoteSocket
```
2. 预加载资源
zygote进程本身加载系统共用的资源，当fork一个进程时Linux有个特性COW(copy on write),写时拷贝的方式能够无谓的资源拷贝直到真正需要更改时。
而系统预加载的资源本身是只读的，可以简单看成一个文本段，这样fork进程后不需要再次加载系统资源，加速应用启动.
```
254    static void preload() {
255        Log.d(TAG, "begin preload");
256        preloadClasses();
257        preloadResources();
258        preloadOpenGL();
259        preloadSharedLibraries();
260        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
261        // for memory sharing purposes.
262        WebViewFactory.prepareWebViewInZygote();
263        Log.d(TAG, "end preload");
264    }
```
类资源:  
这个是在代码中拷贝到机器上`/system/etc/preloaded-classes`,在art初始化的时候通过dex2oat工具将boot.oat中的类按照preloaded-classes列表中的类抽取出来，生成boot.art然后通过patchoat来进行段的随机偏移防止hacker获取到代码段位置.
```
06-06 06:02:34.524 I/dex2oat ( 2040): /system/bin/dex2oat --image=/data/dalvik-cache/x86_64/system@framework@boot.art --dex-file=/system/framework/x86_64/boot.oat --oat-file=/data/dalvik-cache/x86_64/system@framework@boot.oat --instruction-set=x86_64 --instruction-set-features=smp,ssse3,sse4.1,sse4.2,-avx,-avx2 --base=0x6fc49000 --runtime-arg -Xms64m --runtime-arg -Xmx64m --compiler-filter=verify-at-runtime --image-classes=/system/etc/preloaded-classes --instruction-set-variant=x86_64 --instruction-set-features=default
```
后面的Resource，OpenGL，SharedLibraries逗比较简单，不像类有这么个弯弯.

3. fork了一个很重要的进程:system_server

起始就是开启了ProcessState，ServiceManager,然后开启了重要的framework服务:ActivityManager,WindowManager,MediaService,MountService等等

总结:

1.app_process作为zygote的实体开始运行，初始化art运行时环境

2.查找ZygoteInit类并调用main方法

3.创建一个套接字/dev/socket/zygote来监听fork进程的请求

4.预加载系统类，资源，OPENGL库

5.fork出一个进程，父进程监听套接字的请求，子进程准备开启system_server

6.子进程首先会开启ProcessState，这个是Binder通讯的基础，后面再分析

7.创建ServiceManager类，但是这个进程的名字是叫system_server

8.开启framework服务

system_server和servicemanager,zygote的关系

servicemanager是一个native的进程，在init.rc中被启动;我们知道android除了使用SDK开发之外很多追求效率的应用还会使用NDK开发，此时可以直接从serviemanager中直接请求服务；servicemanager是管理native的服务而设立的；

zygote也是在init.rc中被启动的，预先加载系统启动资源，类等，可以加速应用的启动；

zygote fork出了一个进程来启动最重要的system_server，这个是为管理java的服务而设立的;

system_server和servicemanager都是管理服务，而且system_server中的java服务一般都需要调用native的服务来和系统打交道，一般而言是APP->Java Framework->JNI->Native->lib->kernel
