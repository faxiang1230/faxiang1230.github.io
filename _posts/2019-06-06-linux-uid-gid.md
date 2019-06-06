---
layout:     post
title:      "linux下的各种uid,gid,sid"
date:       2019-06-06 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
---
# linux下的各种uid,gid,sid
uid:用户id,一般用在两个地方:文件的拥有者,启动该进程的用户
gid:用户组id,一般用在两个地方:文件的拥有者,启动该进程的用户
euid:这个只有进程拥有的属性,进程当前的有效用户。一般情况下是和进程的uid相同的，但是当可执行程序带有`s`标志时，euid就变成了该可执行程序拥有者的uid
suid(set save-user-id):当该程序被执行时，它是是euid的一份copy
suid(set-user-id):
 Linux通常都不建议用户使用root权限去进行一般的处理，但是普通用户在做很多很多services相关的操作的时候，可能需要一个特殊的权限。为了满足这样的要求，许多services相关的executable有一个标志，这就是set-user-id bit。当这个ser-user-id bit=ON的时候，这个executable被用exec启动之后的进程的effective user id就是这个executable的owner id，而并非parent process real user id。如果set-user-id bit=OFF的时候，这个被exec起来的进程的effective user id应该是等于进程的user id的。
 那这样我们就清楚了，设定一个effective user id的意义在于，它可能和real user id不同。那么为什么要设置一个saved set-user-id呢？它的意义是，它相当于是一个buffer，在exec启动进程之后，它会从effective user id位拷贝信息到自己。对于非root用户，可以在未来使用setuid()来将effective user id设置成为real user id和saved set-user-id中的任何一个。但是非root用户是不允许用setuid()把effective user id设置成为任何第三个user id。
对于root来说，就没有那么大的意义了。因为root调用setuid()的时候，将会设置所有的这三个user id位。所以可以综合说，这三个位的设置为为了让unprivilege user可以获得两种不同的permission而设置的。

- saved set-user-id是无法取出来的，是kernel来控制的。

- 注意saved set-user-id是进程中的id，而set-user-id bit则是文件上的权限。我用了比较长的时间才分清楚。

进程凭证:

参考:
- https://blog.csdn.net/rex_nie/article/details/20314397
- man 2 chmod
