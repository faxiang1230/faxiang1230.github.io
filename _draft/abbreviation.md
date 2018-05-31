# crypto

# memory
# interrupt
# process
#
# 文件系统
有几个传统抽象概念:文件，目录项，索引节点和安装点  
super_block:是硬盘上物理分区的超级块信息在操作系统中的反映(不是基于磁盘的文件系统会在使用现场创建超级块并保存到内存中),代表了一个具体的已安装文件系统  
inode:是硬盘上ext家族inode在OS中的反映,代表了一个具体的文件  
dentry:为了方便查找，引入的dentry概念，在物理设备上是没有直接对应信息的，每个dentry代表路径中的一个特定部分。
`/bin/vi`中`/`,`bin`,`vi`都是目录项对象，前两个是目录，最后一个是普通文件。
在路径中，每一个部分都是目录项对象。解析一个路径并遍历其分量时，执行的是常规的字符串比较过程，耗时而且繁琐，目录项对象的引入使其更加简单.
代表了路径的一个组成部分  
file:是面向进程的，代表由进程打开的文件(fd是面向用户的，和file一一对应，也是代表打开的文件)  

bdi_dev:backing device info,主要处理物理设备的读写缓冲，每一个物理磁盘抽象出来一个。基于RAM的或其他不是基于物理设备的，会创建一个noop_backing_dev_info  

dcache:VFS层遍历路径名中所有的元素并将他们逐个地解析成目录项对象，还要达到最深层目录，是一件非常费力的工作，会消耗大量的时间。内核将目录项对象缓存起来到dcache中  
file_system_type:每种文件系统一个定义，不管有没有实例安装  
vfsmount:代表安装点  