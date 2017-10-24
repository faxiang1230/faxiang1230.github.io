---                                                                                                                                                                                                                
layout:     post
title:      "drop_caches" 
date:       2017-10-24 11:00:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - kernel
  - memory
  - sysctl
---

# drop-caches
## 介绍
内核文档:
```
通过这个节点可以释放掉干净的缓存，dentry，inode，而脏的页需要通过sync系统调用来主动回写来释放

释放掉页缓存
	echo 1 > /proc/sys/vm/drop_caches
释放掉dentry和inode缓存
	echo 2 > /proc/sys/vm/drop_caches
释放掉页缓存和dentry,inode缓存
	echo 3 > /proc/sys/vm/drop_caches
```
## 实现原理
```
kernel/sysctl.c
{
  .procname	= "drop_caches",
  .data		= &sysctl_drop_caches,
  .proc_handler	= drop_caches_sysctl_handler,
},
int drop_caches_sysctl_handler(...)
{
		if (sysctl_drop_caches & 1)
			iterate_supers(drop_pagecache_sb, NULL);//释放掉页缓存
		if (sysctl_drop_caches & 2)
			drop_slab();                 //释放掉dentry和inode
}
```
### iterate_supers:
所有的super_block实例都挂在list：super_blocks上，每一个super_block实例都管理着自己的打开的inode列表:s_inodes上，每个inode都有自己的address_spacing,这里维护这该文件在内存中的缓存页；

然后遍历树上所有的页(除了dirty, locked, under writeback or mapped into * pagetables.)，然后释放掉。

如果是常规的页缓存，那么通过address_spacing->a_ops->releasepage来释放页;如果是buffer cache，那么通过drop_buffer来释放

page的mapping和private:

mapping:

直接从maping中删除，然后把自己放入到per_cpu_pageset中

private:`(struct buffer_head *)page_private(page)`

如果page->flags中Private置位则说明是指向头的buffer_head,然后buffer_head形成一个链表，然后遍历其中的buffer_head,如果全部都没有被使用，则说明该page可以被释放；否则则不能被释放。

不过如果page->flags中PageSwapCache置位，说明该page是被用于页交换缓存

* free之后的页放在哪呢？free_area？per_cpu_pageset?

看起来是放入到per_cpu_pageset中了，毕竟是刚被使用过，属于热页。
什么时候放入free_area呢，另开一篇写了

### drop_slab:
所有的文件系统在注册时，即填充super_block时都会通过register_shrinker注册到shrinker_list中，除了典型的super_block还有一些ashmem,lowermemkiller等;

每个super_block使用s_nr_dentry_unused来管理最近不使用的dentry,s_nr_inodes_unused管理最近不使用的inode，然后就沿着链表挨个释放dentry和inode，从链表中删除，从hash表中删除

## 应用范围:
测量硬盘的读速率
```
 sudo sh -c "sync && echo 3 > /proc/sys/vm/drop_caches"  确保文件不是从缓存中读取出来的
 dd if=./largefile of=/dev/null bs=8k
```

ref:Documentation/sysctl/vm.txt
