---
layout:     post
title:      "Android binder(UNFINISHED)"
date:       2018-06-25 21:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - Android
---
# Android binder

Android是基于Linux系统的，在Linux系统中进程间通信已经存在一些成熟的进程间通信方法了，为什么
android还创造出来binder呢?肯定是现存的进程间通信不能满足需求，然后才创造的新的进程间通信方式。  
android binder是基于openbinder，而后者始于BeOS,PalmOS系统的，后来被android继续开发这个
框架形成了android的binder.  
## Oerview
binder不仅仅是binder,中间还有一个重要角色IInterface,在服务端实现了接口，在应用端可以通过
proxy来和服务端通信，实现接口调用  
binder允许进程暴露接口给其他的进程，每个服务进程维护了一些线程池来服务请求以防止请求阻塞。  
binder是一个分布式架构的框架，服务可以位于不同的进程中，只需要向系统注册就好，应用可以通过
服务名称来获取服务，之后调用接口就能够调用服务例程  
binder是RPC类型的，调用远程接口而且就像调用本地例程一样  
binder注重安全性，通常会检查uid,gid  
binder非常关系引用计数，递归情况(在binder通信中如果两个服务线程产生递归调用，会有特殊方法
来处理来避免递归调用)  
binder在3.19以后版本已经合并到内核中了，正式成为了一个IPC选项，其他的ion，lowmem还在staging中呆着呢。

android binder框架包含binder driver和framework的框架，对于开发者来说封装的非常好，很容
易就能添加native的service和java形式的服务。  
service是什么?就是提供服务的代码，最终体现为一个个的接口函数，所以，Service就是实现一组函数
的对象。里面有两种形式的service,一种是system_server中启动的，另外一种通过service_manager
来注册的服务。可以在java中调用native的服务，也可以调用java服务，还可以在native应用中调用
native服务，也可以调用java服务  

android中大部分使用binder进行IPC，但是它也没有完全抛弃传统的IPC方式，例如socket等，不过在
native之间更加常用，在java层绝大多数都是使用binder来通信  
## IBinder
![image](/images/binder-interface.png)  
在binder IPC通信中，主要有两种类在起主要作用，IBinder用于标示唯一的服务，IInterface主要
用于暴露服务接口，native的主要如上图所示。  
面向binder driver通信的主要是IPCThreadState,负责具体和驱动搞事情，一个实例代表一个线程
（Binder_1,Binder_2），服务端通常是有线程池的，多线程处理可以避免多个请求到来的时候阻塞排队
## 服务的注册
![image](/images/binder-service.png)  
service通过servicemanager来注册服务，通常我们看到的形式都是这样:
```
void MediaPlayerService::instantiate() {
    defaultServiceManager()->addService(
            String16("media.player"), new MediaPlayerService());
}
```
在系统启动中servicemanager会将自己注册称为管理服务者，负责添加服务，查询服务，检查服务,其中
添加服务肯定是为普通service服务的，而查询和检查服务是为普通客户端服务的。servicemanager也是
遵循binder架构的，使用的时候也是通过`BpXXXX->IServiceManager`获取到服务编号的，不过它是
类似于DNS服务器的角色，服务编号是已知的0.  
### native服务
1.定义一个接口文件， IXXXService, 继承IInterface
2.定义BnXXX(), 继承 BnInterface<IXXXService)
3.实现一个XXXService类，继承BnXXX(), 并具体实现onTransact() 函数。

### java服务

Java较Native端实现简单很多，通过Aidl工具来实现类似功能。所以，要实现一个Java端的service，
只需要做以下几件事情  
1.写一个.aidl文件，里面用AIDL语言定义一个接口类`IXXX`  
2.在Android.mk里加入该文件，这样编译系统会自动生成一个IXXX.java, 放在`out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/src/core`下面。  
3.在服务端，写一个类，扩展IXXX.Stub，具体实现IXXX的接口函数。  

## 服务的使用
### native服务
```
sp<IServiceManager> sm = defaultServiceManager();
sp<IBinder> binder = sm->getService(String16("media.player"));

sp<IMediaPlayerService> service =
    interface_cast<IMediaPlayerService>(binder);
```
### java服务
## LinkToDeath
## 内存管理
对于进程间通信的内存，特别是块比较大的时候，一般都会减少复制次数，每次用户/内核上下文切换是有
一定的消耗的，所以尽量减少消耗次数，在binder和graphicbuffer中都是复制和buffer关联的handle
或者直接是某类地址，之后再在接收端将buffer映射到虚拟地址上，这样就只复制了一部分控制数据从而
避免真正数据的复制。
## Ref
https://www.cnblogs.com/samchen2009/p/3316001.html
