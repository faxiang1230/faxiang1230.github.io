---
layout:     post
title:      "硬盘速率测试"
date:       2017-07-26 19:50:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - benchmark
  - harddrive
---
# 测试硬盘速率
## 测量硬盘的写速率
```
使用dd命令来测量服务器的吞吐率（写速度) dd if=/dev/zero of=/tmp/test1.img bs=1G count=1 oflag=dsync  
使用dd命令测量服务器延迟 dd if=/dev/zero of=/tmp/test2.img bs=512 count=1000 oflag=dsync  

if=/dev/zero (if=/dev/input.file) ：用来设置dd命令读取的输入文件名。
of=/tmp/test1.img (of=/path/to/output.file)：dd命令将input.file写入的输出文件的名字。
bs=1G (bs=block-size) ：设置dd命令读取的块的大小。例子中为1个G。
count=1 (count=number-of-blocks)：dd命令读取的块的个数。
oflag=dsync (oflag=dsync) ：使用同步I/O。不要省略这个选项。这个选项能够帮助你去除caching的影响，以便呈现给你精准的结果。
conv=fdatasyn: 这个选项和oflag=dsync含义一样。
在下面这个例子中，一共写了1000次，每次写入512字节来获得RAID10服务器的延迟时间：
```
https://linux.cn/article-6104-1.html  
## 测量硬盘的读速率
```
 sudo sh -c "sync && echo 3 > /proc/sys/vm/drop_caches"  确保文件不是从缓存中读取出来的
 dd if=./largefile of=/dev/null bs=8k
```
