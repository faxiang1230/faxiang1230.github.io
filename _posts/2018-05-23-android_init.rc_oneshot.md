---
layout:     post
title:      "Android init.rc中的oneshot"
date:       2018-05-23 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - process
  - signal
---
# Android init.rc中的oneshot  
从uboot到kernel,内核在启动完之后会将主动权交给用户空间，就是`init`进程。
在Android系统中，它没有使用传统嵌入式中的`bash`程序而是自己定制了`init`的进程，和其搭配的就是init.rc文件。
`init.rc`文件就是一种格式的配置文件，还可以`include`其他的配置文件，`init`程序读取配置依次启动一系列程序为Android的启动做准备。
`init.rc`中间有个关键字`oneshot`,当服务进程启动时标记了`oneshot`之后，如果服务退出后，就不再重新启动。反之没有标记`oneshot`时就会重新启动服务进程
```
oneshot
  Do not restart the service when it exits.
```
**关键的问题:**  
```
是init进程重启服务吗?init进程怎么知道服务退出呢?
```
运行在Linux系统下的所有进程都有一个共同的祖先`init`,所有的进程都是`init`通过`fork()`来直接或间接地分裂出来的。  
在Android启动过程中，所有的native进程包括`zygote`都是`init`分裂出来的，他们的父进程都是init进程,我们`ps`一下
```
USER     PID   PPID  VSIZE  RSS     WCHAN    PC        NAME
root      1     0     732    496   c0117ba4 0002805c S /init
logd      163   1     7260   1764  ffffffff b6ec0548 S /system/bin/logd
root      164   1     1592   176   c0141bb8 00034b7c S /sbin/healthd
root      165   1     2380   1016  c0141bb8 b6ef2c8c S /system/bin/lmkd
system    166   1     1244   348   c05fcb0c b6f287b4 S /system/bin/servicemanager
root      167   1     6432   876   ffffffff b6f641a0 S /system/bin/displayd
root      168   1     6784   1856  ffffffff b6f1b1a0 S /system/bin/vold
system    169   1     66808  10672 ffffffff b6eaac8c S /system/bin/surfaceflinger
shell     174   1     1116   524   c02e849c b6ec867c S /system/bin/sh
root      175   1     10704  1060  ffffffff b6ee61a0 S /system/bin/netd
root      176   1     2040   1332  c064fb48 b6f34274 S /system/bin/debuggerd
root      177   1     5328   1028  ffffffff b6ed61a0 S /system/bin/rild
drm       178   1     9520   3240  ffffffff b6f187b4 S /system/bin/drmserver
install   180   1     1160   376   c0708f74 b6f0667c S /system/bin/installd
keystore  181   1     4324   1664  c05fcb0c b6f2b7b4 S /system/bin/keystore
root      182   1     3004   316   ffffffff b6efaa88 S /system/bin/rwcd
root      185   1     1484380 48628 ffffffff b6dcf360 S zygote
media_rw  186   1     3576   336   ffffffff b6ed967c S /system/bin/sdcard
root      187   1     4644   224   ffffffff 00027a78 R /sbin/adbd
root      191   1     1100   528   c0117ba4 b6f90360 S /system/bin/sh

```
他们之间有一个特殊的关系:
```
当子进程退出的时候，父进程可以通过`waitpid/waitid／wait4`来获取子进程退出的状态和原因
```
这也是常说的`子死父收尸`，init亲手启动的服务退出时它能够再次复活它,这让Android像没事人继续运行。  
## 实现过程代码简析
`system/core/init/init.c`中执行`signal_init_action`,之后注册信号`SIGCHLD`处理函数,
在子进程退出或停止的时候将会接收到`SIGCHLD`信号(不过在5.1中没啥鸟用).  
之后调用`waitpid`一直阻塞等待子进程退出;子进程退出后重置服务的资源，重启服务
```
signal_init_action
  signal_init
    handle_signal
      while (!wait_for_one_process(0));

wait_for_one_process{
  while ( (pid = waitpid(-1, &status, block ? 0 : WNOHANG)) == -1 && errno == EINTR );
  svc = service_find_by_pid(pid);
  if (!(svc->flags & SVC_ONESHOT) || (svc->flags & SVC_RESTART)) {
      kill(-pid, SIGKILL);
      NOTICE("process '%s' killing any children in process group\n", svc->name);
  }
  /* Execute all onrestart commands for this service. */
  list_for_each(node, &svc->onrestart.commands) {
    cmd = node_to_item(node, struct command, clist);
    cmd->func(cmd->nargs, cmd->args);
  }
}
```
