---
layout:     post
title:      "kernel OOPS"
date:       2017-07-07 23:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - kernel
  - oops
  - stability
---
# kernel OOPS  
## oops
什么是oops:  
oops原意是`哎呀`的意思，就是惊奇的预期.在Linux内核中是發生不正確的行為並產生一份錯誤報告的机制.多種類型的oops導致眾所周知的内核错误，但部分oops也允許繼續操作，但可靠度會打折扣。這個術語僅僅代表了一個錯誤。  

发生oops之后：  
1.通常会打印很多信息到dmesg中或者直接打印到控制台，从这里的信息可能就可以直接判断出是什么错误  
2.之后可能只是杀死导致oops的进程(包括kernel thread)，然后系统继续可以运行，但是可靠性将会降低    
3.严重的oops将会导致系统panic
## panic
什么是panic:  
严重的oops，系统认为已经不能继续正常运行时就会panic   
什么时候发生panic:  
Oops in interrupt, with panic_on_oops enabled, manual panic() calls  
HW failure, critical memory allocation fail, init/idle task killed, interrupt handler killed  

- BUG_ON产生的断言  
- 内存转换的相关错误：

```
“BUG: unable to handle kernel paging
request”
“... handle NULL pointer dereference” (when
bad_addr < PAGE_SIZE)
```
- Corrupted page table  
Kernel trying to execute NX-protected page  
Kernel trying to execute userspace page (Intel SMEP)  
Failed bounds check in kernel mode (Intel MPX feature)  
General protection fault, unhandled double fault  
- Soft lockup  
CPU spent 20s in kernel without reaching a schedule point
A warning, unless config/bootparam softlockup_panic enabled
Soft lockup can be harmless, so not good idea in production
- Hard lockup  
CPU spent 10s with disabled interrupts
- Detection of both combines several generic mechanisms  
High priority kernel watchdog thread updates soft lockup timestamp
hrtimer set to deliver periodic interrupts, increments hard lockup
counter and wakes up the watchdog thread
NMI perf event checks if hrtimers interrupts were processed and if
watchdog thread was scheduled
- Hung task check  
“INFO: task ... blocked for more than 120 seconds”
khungtaskd periodically processes tasks in uninterruptible
sleep and checks if their switch count changed
- RCU stall detector  
  - Detects when RCU grace period is too long (21s)
    CPU looping in RCU critical section or disabled interrupts, preemption or
    bottom halves, no scheduling points in non-preempt kernels
  - RT task preempting non-RT task in RCU critical section
- Several other debugging config options (later)

panic的过程:  
May trigger crash dump if configured, or reboot after delay
## BUG
## WARN
## 错误类型  
### watchdog:
### bad page map:
### kernel NULL pointer:
### kernel page fault:
### bad page state:
## kernel oops info  
```
[  266.491864] ------------[ cut here ]------------
[  266.491904] kernel BUG at mm/rmap.c:399!   需要配置CONFIG_DEBUG_BUGVERBOSE
[  266.491934] invalid opcode: 0000 [#1] SMP  invalid opcode：是发生了BUG();
0000是异常特定的错误码，如果错误类型是page fault时特别有用,
Bit 0 – Present
Bit 1 – Write
Bit 2 – User
Bit 3 – Reserved write
Bit 4 – Instruction fetch
[#1] oops计数器，后面跟了kernel的重要配置：SMP PREEMPT DEBUG_PAGEALLOC KASAN
[  266.491962] Modules linked in: amdkfd amd_iommu_v2 radeon cfbfillrect
cfbimgblt cfbcopyarea drm_kms_helper ttm fuse
该设备上拥有哪些设备，哪些被编译成了module,可以推断出部分kernel的配置
可能包含如下的tainted标志：
P – proprietary
O – out-of-tree
F – force-loaded
C – staging
E – unsigned
X – external
+/- – being loaded/unloaded
[  266.492043] CPU: 3 PID: 5155 Comm: java Not tainted 3.19.0-rc3-kfd+ #24
[  266.492087] Hardware name: AMD BALLINA/Ballina, BIOS
WBL3B20N_Weekly_13_11_2 11/20/2013
CPU还有另外的一些硬件信息
[  266.492141] task: ffff8800a3b3c840 ti: ffff8800916f8000 task.ti:
ffff8800916f8000
[  266.492191] RIP: 0010:[<ffffffff81126630>]  [<ffffffff81126630>]
unlink_anon_vmas+0x102/0x159
正在被知行的指令即出错的指令，转换为function+offset;如果是built-in，可以直接使用
addr2line -e vmlinux address来得出源文件行数;
如果是module，需要objdump -dSr x.ko来找function的偏移对应的源代码行数  
[  266.492249] RSP: 0018:ffff8800916fbb68  EFLAGS: 00010286
[  266.492285] RAX: ffff88008f6b3ba0 RBX: ffff88008f6b3b90 RCX: ffff8800a3b3cf30
[  266.492331] RDX: ffff8800914b3c98 RSI: 0000000000000001 RDI: ffff8800914b3c98
[  266.492376] RBP: ffff8800916fbba8 R08: 0000000000000002 R09: 0000000000000000
[  266.492421] R10: 0000000000000008 R11: 0000000000000001 R12: ffff88008f686068
[  266.492465] R13: ffff8800914b3c98 R14: ffff88008f6b3b90 R15: ffff88008f686000
[  266.492513] FS:  00007fb8966f6700(0000) GS:ffff88011ed80000(0000)
knlGS:0000000000000000
[  266.492566] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033发生错误的原因  
[  266.492601] CR2: 00007f50fa190770 CR3: 0000000001b31000 CR4: 00000000000407e0
寄存器的值，这个看的时候需要对着反汇编来看，从发生错误的哪行指令来反推
[  266.492652] Stack:
[  266.492665]  0000000000000000 ffff88008f686078 ffff8800916fbba8
ffff88008f686000
[  266.492714]  ffff8800916fbc08 0000000000000000 0000000000000000
ffff88008f686000
[  266.492764]  ffff8800916fbbf8 ffffffff8111ba5d 00007fb885918000
ffff88008edf3000
RSP指向的原始的栈顶信息
[  266.492815] Call Trace:
[  266.492834]  [<ffffffff8111ba5d>] free_pgtables+0x8e/0xcc
[  266.492873]  [<ffffffff8112253e>] exit_mmap+0x84/0x116
[  266.492907]  [<ffffffff8103f789>] mmput+0x52/0xe9
[  266.492940]  [<ffffffff81043918>] do_exit+0x3cd/0x9c9
[  266.492975]  [<ffffffff8170c1ec>] ? _raw_spin_unlock_irq+0x2d/0x32
[  266.493016]  [<ffffffff81044d7f>] do_group_exit+0x4c/0xc9
[  266.493051]  [<ffffffff8104eb87>] get_signal+0x58f/0x5bc
[  266.493090]  [<ffffffff810022c4>] do_signal+0x28/0x5b1
[  266.493123]  [<ffffffff8170ca0c>] ? sysret_signal+0x5/0x43
[  266.493162]  [<ffffffff81002882>] do_notify_resume+0x35/0x68
[  266.493200]  [<ffffffff8170cc7f>] int_signal+0x12/0x17
调用栈的过程，是通过回溯栈信息的方式得到的；？表示这个地址并不完全完全适应该帧，可能是上一次执行失败了
[  266.493235] Code: e8 03 b7 f4 ff 49 8b 47 78 4c 8b 20 48 8d 58 f0 49 83
ec 10 48 8d 43 10 48 39 45 c8 74 55 48 8b 7b 08 83 bf 8c 00 00 00 00 74 02
<0f> 0b e8 a4 fd ff ff 48 8b 43 18 48 8b 53 10 48 89 df 48 89 42
RIP周围的指令，RIP是被<>括住的，这里是of ob,代表了ud2；
可以使用scripts/decodecode来反汇编这里的指令：scripts/decodecode < oops.txt
[  266.493404] RIP  [<ffffffff81126630>] unlink_anon_vmas+0x102/0x159
[  266.493447]  RSP <ffff8800916fbb68>
[  266.508877] ---[ end trace 02d28fe9b3de2e1a ]---
[  266.508880] Fixing recursive fault but reboot is needed!
```

## 内核关于BUG的选项
```
CONFIG_BUG - emit BUG traps.  Nothing happens without this.
当BUG()发生时，会产生一个标准的无效操作码ud2：0F 0B,然后触发一个异常
CONFIG_GENERIC_BUG - enable this code.
CONFIG_GENERIC_BUG_RELATIVE_POINTERS - use 32-bit pointers relative to
the containing struct bug_entry for bug_addr and file.
CONFIG_DEBUG_BUGVERBOSE - emit full file+line information for each BUG
__bug_table__实现了一个symbols段，当异常发生时并检查出是ud2时会到__bug_table__中查询，
得出file+line信息  
CONFIG_BUG and CONFIG_DEBUG_BUGVERBOSE are potentially user-settable
(though they're generally always on).
CONFIG_GENERIC_BUG is set by each architecture using this code.

To use this, your architecture must:

1. Set up the config options:
   - Enable CONFIG_GENERIC_BUG if CONFIG_BUG

2. Implement BUG (and optionally BUG_ON, WARN, WARN_ON)
   - Define HAVE_ARCH_BUG
   - Implement BUG() to generate a faulting instruction
   - NOTE: struct bug_entry does not have "file" or "line" entries
     when CONFIG_DEBUG_BUGVERBOSE is not enabled, so you must generate
     the values accordingly.

3. Implement the trap
   - In the illegal instruction trap handler (typically), verify
     that the fault was in kernel mode, and call report_bug()
   - report_bug() will return whether it was a false alarm, a warning,
     or an actual bug.
   - You must implement the is_valid_bugaddr(bugaddr) callback which
     returns true if the eip is a real kernel address, and it points
     to the expected BUG trap instruction.
```

## 如何解栈帧
