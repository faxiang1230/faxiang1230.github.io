---
layout:     post
title:      "How to Debug kernel via QEMU"
date:       2017-07-08 18:10:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - kernel
  - qemu
  - debug
---
# Debug Kernel via QEMU

## 准备kernel和initrd.img

我最近都是在做OPENTHOS系统(基于Android-x86),所以就直接使用OPENTHOS的内核:4.4.10版本;
### 编译内核:
```
$cp arch/x86/configs/android-x86_64_defconfig .config  
$make menuconfig  
$make
```
### 编译一个最小的initrd.img:
下载busybox 源码并编译;因为Linux运行环境当中是不带动态库的，所以必须以静态方式来编译BusyBox。修改
```
$make defconfig
$make menuconfig
Busybox Settings --->
    Build Options --->
         [*] Build BusyBox as a static binary(no shared libs)

make
make install
```
写一个最简单的启动脚本:
```
$cd $BUSYBOX/_install
创建系统运行时的必须目录，其中，/proc用于挂载proc系统，/sys用于挂载sys系统，dev用于mdev创建设备节点，etc/init.d为放置busybox启动脚本的目录
$mkdir proc sys dev etc etc/init.d
$vim $BUSYBOX/_install/etc/init.d/rcS

#!/bin/sh
#将proc文件系统挂载到/proc目录，因为很多应用程序会使用到/proc中的信息，不挂载会导致各种异常
mount -t proc none /proc
#将sys文件系统挂载到/sys目录，因为很多应用程序会使用到/sys中的信息，不挂载会导致各种异常
mount -t sysfs none /sys
#mdev是busybox自带的一个udev，用于系统启动和热插拔或动态加载驱动程序时，自动产生设备节点，这句话如果不加上则需要手动mknod来挂载设备节点
/sbin/mdev -s

$chmod +x $BUSYBOX/_install/etc/init.d/rcS
(注：为什么编辑这个文件呢？因为我们将使用busybox的init作为我们的Linux启动的第一个进程，而busybox的init所使用的启动脚本就是/etc/init.d/rcS，该路径被声明在$BUSYBOX/init/init.c当中)
```
创建initrd.img  
```
cd _install
find . | cpio -o -H newc |gzip > $KERNEL/rootfs.img
cp rootfs.img $KERNEL/
```
## 安装QEMU
最简单的安装方式:`apt-get install qemu`,这种方式采用了QEMU的默认配置；  
可能QEMU的某些默认功能没有打开，那么你需要手动从source编译qemu(略)  
创建一个软链接来减少输入字符:
```
$ln -s /usr/bin/qemu-system-x86_64 /usr/bin/qemu
```

## Qemu启动Linux
```
qemu -kernel ./arch/x86_64/boot/bzImage -initrd rootfs.img.gz -append "root=/dev/ram rdinit=sbin/init console=ttyS0" -no-reboot  -serial stdio -s -S
-S:freeze CPU at startup
-s:shorthand for -gdb tcp::1234
```
## gdb连接  
启动加上该参数-gdb tcp::1234  
```
gdb
(gdb) target remote:1234
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb) file vmlinux
A program is being debugged already.
Are you sure you want to change the file? (y or n) y
Reading symbols from vmlinux...done.
```
## 使用QEMU启动完整的镜像
有时候拿到一个完整的iso启动镜像，就需要变一下启动参数:  

### 通过legacy引导启动
```
apt-get install ovmf
qemu-system-x86_64 -enable-kvm -m 2.5G -cdrom ubuntu-15.10-desktop-amd64.iso  -vga std -serial stdio
```
### 通过UEFI引导启动
```
qemu-system-x86_64 -enable-kvm -m 2.5G  -vga std -serial stdio -bios /usr/share/ovmf/OVMF.fd -drive file=ubuntu-15.10-desktop-amd64.iso
```
## 遇到的问题:
gdb中出现：  
```
Remote 'g' packet reply is too long:
```
进入gdb设置：
```  
set architecture i386:x86-64:intel  
```
这种修改方式不起作用, 采用下面的修改gdb源码的方式证明确实是好使的;
```
if (buf_len > 2 * rsa->sizeof_g_packet) {
    rsa->sizeof_g_packet = buf_len ;
    for (i = 0; i < gdbarch_num_regs (gdbarch); i++) {
        if (rsa->regs->pnum == -1)
            continue;
        if (rsa->regs->offset >= rsa->sizeof_g_packet)
            rsa->regs->in_g_packet = 0;
        else  
            rsa->regs->in_g_packet = 1;
    }     
}
```
