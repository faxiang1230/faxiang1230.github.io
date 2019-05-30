---
layout:     post
title:      "how mount sync option work"
date:       2019-05-30 17:10:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - filesystem
---
# How mount sync work
## Where to save flag:
1.mount option will convert to mount(2) mountflags `MS_SYNCHRONOUS`,will transfer to syscall

2.mountflags will save in `super_block->s_flags`
## How to use:
1.when filesystem use `file_operations->write = gerneric_file_write`,after write it will check if need sync:`IS_SYNC(inode)`,if yes,wait for write done

2.when filesystem not fill `file_operations->write`,kernel will use `do_sync_write->aio_write`,kernel implement default `generic_file_aio_write`,it will also check `IS_SYNC(inode)`,if yes,wait for write done
```
include/linux/fs.h
#define __IS_FLG(inode, flg)	((inode)->i_sb->s_flags & (flg))
#define IS_SYNC(inode)		(__IS_FLG(inode, MS_SYNCHRONOUS) || \
					((inode)->i_flags & S_SYNC))
```
3.when physical filesystem implement its `write/aio_write`,it will make sure that do sync when `MS_SYNCHRONOUS`  
