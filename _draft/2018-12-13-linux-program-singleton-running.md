---
layout:     post
title:      "进程单例执行的方法"
date:       2018-12-13 15:30:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - bash
---
# Linux中程序的单例执行
我们写了个程序，但是只想让它只执行一次，但是我们没有权利来限制其他人运行这个程序，有以下集中方法来保证程序的单例运行:
1.程序的运行需要特殊权限，而这个权限只有管理员才有，管理员自己来维护程序的单次运行，这个属于社会工程学范畴
2.程序自己探测是否已经有进程在运行自己
我们主要列举出第二个选项，毕竟我们是程序员嘛，让你相信自己的程序还是业余用户的计算机知识素养之间，我果断选择了自己的程序
## 文件存在锁
在程序开始的时候探测文件是否存在，不存在则创建文件开始运行，否则自行`exit`;退出的时候销毁文件。
创建文件都知道，探测文件使用`access`,退出的时候通过`at_exit`注册handler来销毁文件
但是我们通过`man atexit`，它只能处理这两种情况，属于程序主动正常退出，但是碰到关机怎么办?关机时会向每个进程发送`SIGTERM`和`SIGKILL`信号的
```
The atexit() function registers the given function to be called at normal process termination, either via exit(3) or via return from the program's main().
```
改进1:
看到`SIGTERM`信号，这个我们可以注册sianal handler啊，在信号处理中删除文件，不失为一种办法;
但是信号是不可靠的，如果没有及时投递到进程,而是直接收到了`SIGKILL`，无条件被终止了。
改进2:在tmpfs中创建文件，在重启之后文件就不存在了,这个方法非常好。
但是还有一个问题，机器没有关机但是异常退出了，文件仍然存在，再次运行程序无法启动

这个问题在于必须主动删除文件，而程序在遇到异常时可能直接就被杀死了，无法可靠完成清除工作。
改进2给我们一个提示，文件会随着系统重启而销毁，文件的生命周期和系统是紧相关的，所以我们可以根据进程的生命周期来判断是否已经运行

1.通过进程自身属性相关的，例如进程名或者套接字等，下面通过进程名来判断是否运行
```
File fp = popen("pgrep xxxx|wc -l" ,"r");
if(fp){
    memset(value ,0x00 ,sizeof(value));
    fread(value ,sizeof(char) ,sizeof(value) ,fp);
    pclose(fp);
    if (atoi(value) >=1)
        exit(0);
}  
```
2.通过文件锁
flock锁:
```
#include <stdio.h>
#include <stdlib.h>
#include <sys/file.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#define FILE_NAME "flock.test"
void main() {
	int fd = open(FILE_NAME, O_RDONLY);
	if(flock(fd, LOCK_EX|LOCK_NB))
		exit(0);
	pause();
}
```
fcntl锁就比较复杂了，参考APUE
