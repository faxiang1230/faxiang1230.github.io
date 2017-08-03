---
layout:     post
title:      "针对不同CPU的优化"
subtitle:   "未完成"
date:       2017-08-03 19:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - kernel
  - x86
---
# generic和特定cpu的kernel选项优化
在做开源项目时，为了系统能够在足够多的cpu上运行，一般都是配置generic的选项；如果有需求要压榨硬件的性能时，牺牲一下通用性，针对特定的平台优化时是可以取得一定的优化效果，具体的效果可以通过一些benchmark测试一下。

实践:
arch/x86/Kconfig.cpu列出了可以优化的cpu系列:
```
config M486
	bool "486"
	depends on X86_32
	---help---
	  Select this for an 486-class CPU such as AMD/Cyrix/IBM/Intel
	  486DX/DX2/DX4 or SL/SLC/SLC2/SLC3/SX/SX2 and UMC U5D or U5S.

config M586
	bool "586/K5/5x86/6x86/6x86MX"
	depends on X86_32
	---help---
	  Select this for an 586 or 686 series processor such as the AMD K5,
	  the Cyrix 5x86, 6x86 and 6x86MX.  This choice does not
	  assume the RDTSC (Read Time Stamp Counter) instruction.
```
arch/x86/Makefile_32.cpu对应的优化选项：
```
align := $(cc-option-align)
cflags-$(CONFIG_M486)		+= -march=i486
cflags-$(CONFIG_M586)		+= -march=i586
cflags-$(CONFIG_M586TSC)	+= -march=i586
cflags-$(CONFIG_M586MMX)	+= -march=pentium-mmx
```
另外的arch/x86/Makefile中也有一些配置，不过大部分属于通用型的选项。
`-march=i586`有什么用呢？GCC手册上这样描述:
```
-march=cpu-type

Generate instructions for the machine type cpu-type. In contrast to -mtune=cpu-type, which merely tunes the generated code for the specified cpu-type, -march=cpu-type allows GCC to generate code that may not run at all on processors other than the one indicated.

Specifying -march=cpu-type implies -mtune=cpu-type.
```
However,性能优化不大，相对于通用性的损失，这点优化完全可以忽略不计。
## Kernel编译时的优化选项
### qemu+gdb+Kernel
在使用qemu+gdb+kernel调试的时候，kernel的编译级别默认是`-O2`,在很多时候断不住，需要调整编译级别
```
Makefile

ifdef CONFIG_CC_OPTIMIZE_FOR_SIZE
KBUILD_CFLAGS	+= -Os $(call cc-disable-warning,maybe-uninitialized,)
else
KBUILD_CFLAGS	+= -O2
endif
```
