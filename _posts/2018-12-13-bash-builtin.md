---
layout:     post
title:      "bash的builtins"
date:       2018-12-13 15:30:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - bash
---
# strace和bash遇到了builtin
在描述问题之前先简单回顾一下以下几个东西:  
- ssh登录发生了什么

```
1.ssh连接远程ssh server服务，通过认证
2.ssh server打开一个新的pts,并转接给ssh client
3.ssh client拦截本地所有的输入传送给server，回显server返回信息
```
- bash是如何执行一条命令

bash接收用户的输入，解释命令行参数，之后通过fork创建一个新的子进程，之后在新的进程中execv新的程序。
原来的bash程序通过waitpid自身阻塞，等待子进程退出

- strace的工作原理

先是通过bash fork一个新的进程执行strace，strace再fork一个进程，通过ptrace来跟踪子进程所有的系统调用。本身这种方式是会增加系统调用次数的，相当于增加了一倍数目的系统调用，会影响程序执行效率的，所以在生产环境中不经常使用strace的。
我们可以通过`pstree`来看一下他们bash和strace调用过程中进程之间的关系
```
├─gnome-terminal-─┬─2*[bash───vi───cscope]
│                 ├─bash───ssh
│                 ├─bash───strace───top
```
## 一次事故
我执行了一条命令`cd`到某个目录，但是在`security_bprm_check`中找不到`cd`的踪影.

我就想通过`strace cd`来检查是否有新的进程创建，出现下面的提示，顿时懵逼了。又通过`which cd`想找到它的位置，很可惜没有找到。
```
ubuntu 16.04
$ strace cd
strace: Can't stat 'cd': No such file or directory
```
我又在centos7上执行了上面的过程，`strace cd`过程是没有问题的，确实创建了一个新的进程,执行了
cd命令，cd程序是存在的而且是可以执行的，而且是可以抓到进程的。但是单独的`cd`命令是无法被抓到的
```
# strace cd
execve("/usr/bin/cd", ["cd"], [/* 34 vars */]) = 0
```
真实的原因是bash的builtin

1.在ubuntu中,通过which找不到cd的位置，是因为它并不是一个单独的程序而是一个内建命令。
在bash解释输入字符串的时候就被过滤了，并不会起新的进程，而是在bash进程中直接执行了

2.在centos中能够找到cd的位置，详细信息如下
```
[root@localhost linux-bfx-gfy]# file /usr/bin/cd
/usr/bin/cd: POSIX shell script, ASCII text executable
[root@localhost linux-bfx-gfy]# cat /usr/bin/cd
#!/bin/sh
builtin cd "$@"
```
再来看一下builtin里面相关信息,上面的`cd`虽然存在，但是也只是一个脚本，bash解析脚本的时候发现是
使用bash来执行，那就开始顺序执行命令，只有一行builtin命令,所以最后的时候又回到bash的builtin
函数了。而这个bash很早就存活了，所以是看不到新进程，security_bprm_check也就抓不到进程了
```
builtin shell-builtin [arguments]
       Execute the specified shell builtin, passing it arguments, and return its exit status.  This is useful when defining a  function  whose  name  is  the  same  as a shell builtin, retaining the functionality of the builtin within the function.  The cd builtin is commonly redefined this way.  The return status is false if shell-builtin is not a shell builtin command.
```
## 总结一下:
1.bash遇到了builtin不会fork进程，看不到新的进程起来
```
1.打开一个终端，通过ps看一下bash的pid1
2.打开另外一个终端，strace -p pid1
3.在前一个终端中执行cd,没有任何的进程创建，只是简单的chdir
```
2.strace遇到了builtin，strace会先fork一个进程，builtin执行的过程在执行时看到的是bash  
和不加strace时是有显著区别的，给人一个错觉是原始执行的时候也是会创建进程的，其实不会。
