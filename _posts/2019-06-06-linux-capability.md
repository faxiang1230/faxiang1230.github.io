---
layout:     post
title:      "Linux能力(capability)机制的继承"
date:       2019-06-06 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
---
# Linux能力(capability)机制的继承
>转发https://www.cnblogs.com/gordon0918/p/5328436.html
## 1、Linux能力机制概述

在以往的UNIX系统上，为了做进程的权限检查，把进程分为两类：特权进程(有效用户ID是0)和非特权进程(有效用户ID是非0)。特权进程可以通过内核所有的权限检查，而非特权进程的检查则是基于进程的身份(有效ID，有效组及补充组信息)进行。

从linux内核2.2开始，Linux把超级用户不同单元的权限分开，可以单独的开启和禁止，称为能力(capability)。可以将能力赋给普通的进程，使其可以做root用户可以做的事情。

此时内核在检查进程是否具有某项权限的时候，不再检查该进程的是特权进程还是非特权进程，而是检查该进程是否具有其进行该操作的能力。例如当进程设置系统时间，内核会检查该进程是否有设置系统时间(CAP_SYS_TIME)的能力，而不是检查进程的ID是否为0；

当前Linux系统中共有37项特权，可在/usr/include/linux/capability.h文件中查看

## 2、Linux能力机制的实现

一个完整的能力机制需要满足以下三个条件:

1、对进程的所有特权操作，linux内核必须检查该进程该操作的特权位是否使能。

2、Linux内核必须提供系统调用，允许进程能力的修改与恢复。

3、文件系统必须支持能力机制可以附加到一个可执行文件上，但文件运行时，将其能力附加到进程当中。

到linux内核版本2.6.24为止，上述条件的1、2可以满足。从linux内核2.6.24开始，上述3个条件可以都可以满足

每个进程包括三个能力集，含义如下：

Permitted: 它是effective capabilities和Inheritable capability的超集。如果一个进程在Permitted集合中丢失一个能力，它无论如何不能再次获取该能力(除非特权用户再次赋予它)

Inheritable: 它是表明该进程可以通过execve继承给新进程的能力。

Effecitive: Linux内核真正检查的能力集。

从2.6.24开始，Linux内核可以给可执行文件赋予能力，可执行文件的三个能力集含义如下：

Permitted:该能力当可执行文件执行时自动附加到进程中，忽略Inhertiable capability。

Inheritable:它与进程的Inheritable集合做与操作，决定执行execve后新进程的Permitted集合。

Effective: 文件的Effective不是一个集合，而是一个单独的位，用来决定进程成的Effective集合。

有上述描述可知，Linux系统中的能力分为两部分，一部分是进程能力，一部分是文件能力，而Linux内核最终检查的是进程能力中的Effective。而文件能力和进程能力中的其他部分用来完整能力继承、限制等方面的内容。

## 3、Linux能力机制的继承

在linux终端查看capabilities的man手册，其中有继承关系公式如下
```
     P'(permitted) = (P(inheritable) & F(inheritable)) |
                          (F(permitted) & cap_bset)              //新进程的permitted有老进程的和新进程的inheritable和可执行文件的permitted及cap_bset运算得到.
     P'(effective) = F(effective) ? P'(permitted) : 0            //新进程的effective依赖可执行文件的effective位，使能：和新进程的permitted一样，负责为空
     P'(inheritable) = P(inheritable)    [i.e., unchanged]       //新进程的inheritable直接继承老进程的Inheritable

     说明:

     P   在执行execve函数前，进程的能力

     P'  在执行execve函数后，进程的能力
     F   可执行文件的能力
     cap_bset 系统能力的边界值，在此处默认全为1
```

有测试程序如下:

father.c
```
#include <stdlib.h>
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/capability.h>
#include <errno.h>

void list_capability()
{
    struct __user_cap_header_struct cap_header_data;
    cap_user_header_t cap_header = &cap_header_data;

    struct __user_cap_data_struct cap_data_data;
    cap_user_data_t cap_data = &cap_data_data;

    cap_header->pid = getpid();
    cap_header->version = _LINUX_CAPABILITY_VERSION_1;

    if (capget(cap_header, cap_data) < 0) {
        perror("Failed capget");
        exit(1);
    }   
    printf("Cap data permitted: 0x%x,  effective: 0x%x,  inheritable:0x%x\n",
        cap_data->permitted, cap_data->effective,cap_data->inheritable);
}

int main(void)
{
   cap_t caps = cap_init();
   cap_value_t capList[2] = {CAP_DAC_OVERRIDE, CAP_SYS_TIME};
   unsigned num_caps = 2;
   //cap_set_flag(caps, CAP_EFFECTIVE, num_caps, capList, CAP_SET);
   cap_set_flag(caps, CAP_INHERITABLE, num_caps, capList, CAP_SET);
   cap_set_flag(caps, CAP_PERMITTED, num_caps, capList, CAP_SET);

   if(cap_set_proc(caps))
   {   
       perror("cap_set_proc");
   }   

   list_capability();

   execl("/home/xlzh/code/capability/child", NULL);

   sleep(1000);
}
```
child.c

```
#include <stdlib.h>
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <linux/capability.h>
#include <errno.h>

void list_capability()
{
    struct __user_cap_header_struct cap_header_data;
    cap_user_header_t cap_header = &cap_header_data;

    struct __user_cap_data_struct cap_data_data;
    cap_user_data_t cap_data = &cap_data_data;

    cap_header->pid = getpid();
    cap_header->version = _LINUX_CAPABILITY_VERSION_1;

    if (capget(cap_header, cap_data) < 0) {
        perror("Failed capget");
        exit(1);
    }
    printf("child Cap data permitted: 0x%x, effective: 0x%x, inheritable:0x%x\n", cap_data->permitted, cap_data->effective,cap_data->inheritable);

}

int main(void)
{
   list_capability();
   sleep(1000);
}
```
执行结果分析

```
xlzh@cmos:~/code/capability$ gcc child.c -o child
xlzh@cmos:~/code/capability$ gcc father.c -o father -lcap
xlzh@cmos:~/code/capability$ sudo setcap cap_dac_override,cap_sys_time+ei child
xlzh@cmos:~/code/capability$ sudo setcap cap_dac_override,cap_sys_time+ip father
/* 单独执行，child文件有E(effective)I(inheritable)的能力，执行child的终端没有任何能力, 套用公式(cap_bset默认全1)
 * P'(permitted) = (P(inheritable) & F(inheritable)) | (F(permitted) & cap_bset)  // P'(permitted) = (0x0 & 0x2000002) | (0x0 & 全1)，结果为0

 * P'(effective) = F(effective) ? P'(permitted) : 0                               // P'(effective) = 1 ? P'(permitted) : 0， 结果为P'(permitted)，即0

 * P'(inheritable) = P(inheritable)                                               // P'(inheritable) = 0
```
执行结果如下所示
```
xlzh@cmos:~/code/capability$ ./child
child Cap data permitted: 0x0, effective: 0x0, inheritable 0x0

/* 单独执行，child文件有E(effective)I(inheritable)的能力，执行child的father文件有E(inheritable)和P(permitted)能力, 套用公式
 * P'(permitted) = (P(inheritable) & F(inheritable)) | (F(permitted) & cap_bset)  // P'(permitted) = (0x2000002 & 0x2000002) | (0x2000002 & 全1)，结果为0
 * P'(effective) = F(effective) ? P'(permitted) : 0                               // P'(effective) = 1 ? P'(permitted) : 0, 结果为P'(permitted)，即0x2000002

 * P'(inheritable) = P(inheritable)                                               // P'(inheritable) =
 ```
执行结果如下所示
```
xlzh@cmos:~/code/capability$ ./father
father Cap data permitted: 0x2000002, effective: 0x0, inheritable: 0x2000002
child Cap data permitted: 0x2000002, effective: 0x2000002, inheritable 0x2000002
```

上述单独运行child可执行程序，其进程没有任何能力。但是有father进程来启动运行child可执行程序，其进程则有相应的能力。

上例中father和child的能力都设置的cap_dac_override和cap_sys_time两个能力。其实两个可执行程序设置的能力可以不同，各位读者可以自己修改其能力，套用公式进行计算。
## 4、以root用户身份执行程序

1、以root用户身份执行程序，则该进程所有能力的的P和I都置为1

2、以root用户身份执行程序，则该进程的E使能

```/*由于执行child的终端进程没有I能力，所有child进程的inheritable也为0， 其他能力为全1*/
xlzh@cmos:~/code/capability$ sudo ./child
child Cap data permitted: 0xffffffff, effective: 0xffffffff, inheritable 0x0
```
## 5、进程用户ID的变化对能力的影响

1、当一个进程的有效用户ID从0变化到非0， 那么所有的E能力清零

2、当一个进程的有效用户ID从非0变化到0，那么现有的P集合拷贝到E集合

3、如果一个进行原来的真实用户ID，有效用户ID，保存设置用户ID是0，由于某些操作这些ID都变成了非0，那么所有的的P和E能力全部清理

4、如果一个文件系统的用户ID从0变成非0，那么以下的能力在E集合中清除：CAP_CHOWN, CAP_DAC_OVERRIDE,  CAP_DAC_READ_SEARCH,  CAP_FOWNER,  CAP_FSETID,  CAP_LINUX_IMMUTABLE  (since  Linux  2.2.30),  CAP_MAC_OVERRIDE,  CAP_MKNOD，如果一个文件系统的用户ID从0变成非0，那么在P集合中使能的能力将设置到E集合中。
