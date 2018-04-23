---
layout:     post
title:      "Linux内核Ramdisk(initrd)机制"
date:       2018-04-22 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - initrd
  - 转载
---
> 转载：http://www.cnblogs.com/armlinux/archive/2011/03/30/2396827.html  

摘要:对于Linux用户来说，Ramdisk并不陌生，可是为什么需要它呢？本文对Ramdisk在内核启动过程中的作用，以及它的内部机制进行深入介绍。

# initrd 和 initramfs在内核中的处理
- 临时的根目录rootfs的挂载
- initrd的解压缩
- 老式的initrd的处理
- cpio格式的initrd的处理

## initrd实例分析
在早期的Linux系统中，一般就只有软盘或者硬盘被用来作为Linux的根文件系统，因此很容易把这些设备的驱动程序集成到内核中。但是现在根文件系统可能保存在各种存储设备上，包括SCSI, SATA, U盘等等。因此把这些设备驱动程序全部编译到内核中显得不太方便。在Linux内核模块自动加载机制的介绍中，我们看到利用udevd可以实现实现内核模块的自动加载，因此我们希望根文件系统的设备驱动程序也能够实现自动加载。但是这里有一个矛盾，udevd是一个可执行文件，在根文件系统被挂载前，是不可能执行udevd的，但是如果udevd没有启动，那就无法自动加载根根据系统设备的驱动程序，同时也无法在/dev目录下建立相应的设备节点。为了解决这个矛盾，于是出现了`initrd`(boot loader initialized RAM disk)。initrd是一个被压缩过的小型根目录，这个目录中包含了启动阶段中必须的驱动模块，可执行文件和启动脚本。包括上面提到的udevd，当系统启动的时候，bootloader会把initrd文件读到内存中，然后把initrd的起始地址告诉内核。内核在运行过程中会解压initrd，然后把 `initrd`挂载为根目录，然后执行根目录中的`/initrc`脚本，您可以在这个脚本中运行initrd中的udevd，让它来自动加载设备驱动程序以及在/dev目录下建立必要的设备节点。在udevd自动加载磁盘驱动程序之后，就可以mount真正的根目录，并切换到这个根目录中。

您可以通过下面的方法来制作一个initrd文件。
```
# dd if=/dev/zero of=initrd.img bs=4k count=1024
# mkfs.ext2 -F initrd.img
# mount -o loop initrd.img  /mnt
# cp -r  miniroot/* /mnt
# umount /mnt
# gzip -9 initrd.img
```
通过上面的命令，我们制作了一个4M的initrd，其中miniroot就是一个根目录。最后我们得到一个名为initrd.img.gz的压缩文件。
利用initrd内核在启动阶段可以顺利的加载设备驱动程序，然而initrd存在以下缺点：

initrd大小是固定的，例如上面的压缩之前的initrd大小是4M(4k*1024)，假设您的根目录(上例中的miniroot/)总大小仅仅是 1M，它仍然要占用4M的空间。如果您在dd阶段指定大小为1M，后来发现不够用的时候，必须按照上面的步骤重新来一次。

initrd是一个虚拟的块设备，在上面的例子中，您可是使用fdisk对这个虚拟块设备进行分区。在内核中，对块设备的读写还要经过缓冲区管理模块，也就是说，当内核读取initrd中的文件内容时，缓冲区管理层会认为下层的块设备速度比较慢，因此会启用预读和缓存功能。这样initrd本身就在内存中，同时块设备缓冲区管理层还会保存一部分内容。为了避免上述缺点，于是出现了initramfs，它的作用和initrd类似，您可以使用下面的方法来制作一个initramfs：
```
# find miniroot/ | cpio -c -o > initrd.img
# gzip initrd.img
```
这样得到的initrd.img大小是可变的，它取决于您的小型根目录`miniroot/`的总大小，由于首选使用cpio把根目录进行打包，因此这个initramfs又被称为cpio initrd. 在系统启动阶段，bootload除了从磁盘上机制内核镜像bzImage之外，还要加载initrd.img.gz，然后把initrd.img.gz 的起始地址传递给内核。能不能把这两个文件合二为一呢？答案是肯定的，在Linux 2.6的内核中，可以把initrd.img.gz链接到内核文件(ELF格式)的一个特殊的数据段中，这个段的名字为.init.ramfs。其中全局变量__initramfs_start和__initramfs_end分别指向这个数据段的起始地址和结束地址。内核启动时会对.init.ramfs段中的数据进行解压，然后使用它作为临时的根文件系统。别看这个过程复杂，您只需要在make menuconfig中配置以下选项就可以了：
```
General setup  --->  
[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support     (../miniroot/)    Initramfs source file(s)
```
其中../miniroot/就是我们的小型根目录。这样就只需要一个内核镜像文件就可以了。 内核在启动过程中，必须对以下几种情况进行处理：
如果.init.ramfs数据段大小不为0(initramfs_end - initramfs_start != 0)，就说明这是initrd集成在内核数据段中。并且是cpio的initrd.

initrd是由bootloader加载到内存中的，这时bootloader会把起始地址和结束地址传递给内核，内核中的全局 initrd_start和initrd_end分别指向initrd的起始地址和结束地址。现在内核还需要判断这个initrd是新式的cpio格式的 initrd还是旧的initrd.

## initrd 和 initramfs在内核中的处理
临时的根目录rootfs的挂载
首选在内核启动过程，会初始化rootfs文件系统，rootfs和tmpfs都是内存中的文件系统，其类型为ramfs. 然后会把这个rootf挂载到根目录。 其代码如下：
```
[start_kernel() -> vfs_caches_init() -> mnt_init()]
void __init mnt_init(void)
{        
  ......      
  init_rootfs();       
  init_mount_tree();
}init_rootfs()注册rootfs文件系统，代码如下：

static struct file_system_type rootfs_fs_type = {   
     .name           = "rootfs",      
  .get_sb         = rootfs_get_sb,   
     .kill_sb        = kill_litter_super,};
int __init init_rootfs(void){   
     err = register_filesystem(&rootfs_fs_type);    
     return err;}
```
init_mount_tree会把rootfs挂载到/目录，代码如下：
```
static void __init init_mount_tree(void)
{     
    struct vfsmount *mnt;  
    struct mnt_namespace *ns;    
    mnt = do_kern_mount("rootfs", 0, "rootfs", NULL);        
    ......   
    set_fs_pwd(current->fs, ns->root, ns->root->mnt_root);      

    set_fs_root(current->fs, ns->root, ns->root->mnt_root);}
```
do_kern_mount()会调用前面注册的rootfs文件系统对象的rootfs_get_sb()函数，
```
[rootfs_get_sb() -> ramfs_fill_super() -> d_alloc_root()]
struct dentry * d_alloc_root(struct inode * root_inode)
{  
      struct dentry *res = NULL;  
      if (root_inode)
     {               

       static const struct qstr name = { .name = "/", .len = 1 };      
          res = d_alloc(NULL, &name);    
            if (res) {      
                  res->d_sb = root_inode->i_sb;  
                  res->d_parent = res;     
                   d_instantiate(res, root_inode);
               }     
   }        return res;}
```
从上面的代码中的可以看出，这个rootfs的dentry对象的名字为"/"，也就是根目录了。
## initrd的解压缩
在start_kernel()的最后，调用rest_init()，rest_init()会建立一个新的内核进程，并在这个内核进程中执行 kernel_init()函数，kernel_init()会调用populate_rootfs()来探测和解压initrd文件。这个函数需要处理上面的几种initrd的情况。
```
[kernel_init() -> populate_rootfs()]
static int __init populate_rootfs(void){        
  /* 如果__initramfs_end - __initramfs_start不为0，就说明这是和内核文件集成在一起的cpio的intrd。*/        
  char *err = unpack_to_rootfs(__initramfs_start,                         __initramfs_end - __initramfs_start, 0);        
  if (err)                
      panic(err);
      #ifdef CONFIG_BLK_DEV_INITRD        
      /* 如果initrd_start不为0，说明这是由bootloader加载的initrd，          
      * 那么需要进一步判断是cpio格式的initrd，还是老式块设备的initrd。         */             
  if (initrd_start) {
        #ifdef CONFIG_BLK_DEV_RAM                
        int fd;                
        /* 首先判断是不是cpio格式的initrd，也就是这里说的initramfs。*/                printk(KERN_INFO "checking if image is initramfs...");                
        /* 这里unpack_to_rootfs()的最后一个参数为1，表示check only，不会执行解压缩。*/                
        err = unpack_to_rootfs((char *)initrd_start,                        initrd_end - initrd_start, 1);                
        if (!err) {                        
          /* 如果是cpio格式的initrd，把它解压到前面挂载的根文件系统上，然后释放initrd占用的内存。*/                      
          printk(" it is/n");                        
          unpack_to_rootfs((char *)initrd_start,initrd_end - initrd_start, 0);                        
          free_initrd();                        
          return 0;                
        }                                
          /* 如果执行到这里，说明这是旧的块设备格式的initrd。                 
          * 那么首先在前面挂载的根目录上创建一个initrd.image文件，                 
          * 再把initrd_start到initrd_end的内容写入到/initrd.image中，                 
          * 最后释放initrd占用的内存空间(它的副本已经保存到/initrd.image中了。)。                 */                
          printk("it isn't (%s);
          looks like an initrd/n", err);                
          fd = sys_open("/initrd.image", O_WRONLY|O_CREAT, 0700);                
        if (fd >= 0) {                        
            sys_write(fd, (char*)initrd_start,initrd_end-initrd_start);                        sys_close(fd);                        
            free_initrd();                
        }        
        .....        
        return 0;
    }
rootfs_initcall(populate_rootfs);
```
经过populate_rootfs()函数的处理之后，如果是cpio格式的initrd，那么unpack_to_rootfs()函数已经把目录解压缩到之前mount的根目录上面了。但是如果是旧的块设备的initrd，unpack_to_rootfs()函数解压缩后得到的是一个块虚拟的设备镜像文件/initrd.image，对于这种情况，还需要进一步处理才能使用。接下来，kernel_init()就要处理这种情况。
```
static int __init kernel_init(void * unused){       
   ......        
   do_basic_setup();        
   /* 内核启动时，可以通过启动参数 rdinit=xxx 来指定启动的最后阶段，需要运行initrd中的哪一个可执行文件，         
   * 如果指定了这个参数，那么ramdisk_execute_command就会指向xxx这字符串，新cpio格式的initrd默认执行/init。         
   * 因此，如果如果ramdisk_execute_command为NULL， 就把它设置为/init。         
   */        
   if (!ramdisk_execute_command)                
   ramdisk_execute_command = "/init";        
   /* 现在，尝试访问ramdisk_execute_command，默认为/init，如果访问失败，说明根目录上不存在这个文件。          
   * 于是调用prepare_namespace()，进一步检查是不是旧的块设备的initrd         
   * (在这种情况下，还是一个块设备镜像文件/initrd.image，所以访问/init文件失败。)。         */        
   if (sys_access((const char __user *) ramdisk_execute_command, 0) != 0) {                
     ramdisk_execute_command = NULL;                
     prepare_namespace();        
   }        
   init_post();        
   return 0;
}
```
老式的initrd的处理
prepare_namespace()用于处理老式的initrd。
```
[kernel_init() -> prepare_namespace() -> initrd_load()]
int __init initrd_load(void){        
  if (mount_initrd) {                
    create_dev("/dev/ram", Root_RAM0);                
    if (rd_load_image("/initrd.image") && ROOT_DEV != Root_RAM0) {                        
      sys_unlink("/initrd.image");                        handle_initrd();                        
      return 1;                
      }       
    }        
    sys_unlink("/initrd.image");        
    return 0;
}
```
initrd_load()执行以下步骤：  
调用create_dev()建立设备文件节点/dev/ram，其实这也是一个ramfs文件系统。

调用rd_load_image()把/initrd.image加载到/dev/ram中。

调用handle_initrd()把把块设备文件/dev/ram挂载到/root。

其中handle_initrd()代码如下：
```
[kernel_init() -> prepare_namespace() -> initrd_load() -> handle_initrd()]
static void __init handle_initrd(void){        
  ......                
  real_root_dev = new_encode_dev(ROOT_DEV);        
  /* 建立/dev/root.old设备文件。*/        
  create_dev("/dev/root.old", Root_RAM0);        
  /* 把/dev/root.old mount到/root目录。*/        
  /* mount initrd on rootfs' /root */        
  mount_block_root("/dev/root.old", root_mountflags & ~MS_RDONLY);        sys_mkdir("/old", 0700);        
  root_fd = sys_open("/", 0, 0);        
  old_fd = sys_open("/old", 0, 0);        
  /* move initrd over / and chdir/chroot in initrd root */        sys_chdir("/root");        
  sys_mount(".", "/", NULL, MS_MOVE, NULL);        
  /* chroot到/root目录，好了，现在/root目录成为当前的根目录。*/        sys_chroot(".");        
  /*         * In case that a resume from disk is carried out by linuxrc or one of         * its children, we need to tell the freezer not to wait for us.         */        
  current->flags |= PF_FREEZER_SKIP;        
  /* 建立一个线程，执行/linuxrc，这是旧的initrd默认执行的文件。*/        
  pid = kernel_thread(do_linuxrc, "/linuxrc", SIGCHLD);                
  ......
}
```
cpio格式的initrd的处理
对于新的cpio格式的initrd不需要额外的处理，因此kernel_init()继续执行：
```
[kernel_init() -> init_post()]
static int noinline init_post(void){       
   ......        
   /* 打开console，注意如果cpio格式的根目录中不存在/dev/console文件，        
    * 在unpack_to_rootfs()函数也会建立这个设备文件。         
    */             
    if (sys_open((const char __user *) "/dev/console", O_RDWR, 0) < 0)                printk(KERN_WARNING "Warning: unable to open an initial console./n");        
    /* 现在，标准输入，标准输出和标准错误全部都是/dev/console。*/        
    (void) sys_dup(0);        
    (void) sys_dup(0);        
    /* 执行ramdisk_execute_command指定的命令，默认为/init.*/        
    if (ramdisk_execute_command) {                run_init_process(ramdisk_execute_command);                
      printk(KERN_WARNING "Failed to execute %s/n",                                ramdisk_execute_command);        
      }        
      ......        
      run_init_process("/sbin/init");        
      run_init_process("/etc/init");        
      run_init_process("/bin/init");        
      run_init_process("/bin/sh");        
      panic("No init found.  Try passing init= option to kernel.");
    }
```
在调用run_init_process()执行/init之后，这个函数就不会返回了，一般的发行版本的Linux中，initrd中的/init脚本会启动udevd，加载必要的设备驱动程序，然后挂载真正的根文件系统，最后在执行真正的根文件系统上的initrd，这样就这个启动过程就顺利的交接了。

块设备的initrd不仅使用不方便，而且在内核中的处理过程也更加复杂，因此cpio的initrd肯定会取代它，推荐使用cpio格式的initrd.

initrd实例分析
如果您使用的是ubuntu，您可以执行以下的命令来看看它的initrd中的内容。
```
# mkdir /tmp/initrd# cp /boot/initrd.img-xxx /tmp/initrd/initrd.img.gz
# cd /tmp/initrd
# gunzip initrd.img.gz
# cat initrd.img | cpio -ivmd
```
现在，可以来看看这个根目录的init脚本到底做了什么。
```
# cat init
#!/bin/sh
# ubuntu用户一定很熟悉这个消息。
echo "Loading, please wait..."
......
exec run-init ${rootmnt} ${init} "$@" <${rootmnt}/dev/console >${rootmnt}/dev/console 2>&1
```
这个init脚本最后执行initrd中的run-init切换到真正的根文件系统中。
您可以对这个脚本进行修改，加入相关的打印信息，然后使用本文开头介绍的方法，重新制作一个cpio的initrd，然后使用这个initrd启动内核，快看看试验效果吧。
