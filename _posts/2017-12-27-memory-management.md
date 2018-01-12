---                                                                                                                                                                                                                
layout:     post
title:      "Linux内存管理"
date:       2017-12-27 23:00:00
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - kernel
  - memory
---

# Linux内存管理
限定：主要是指32位上，单节点，开启MMU，有HIGHMEM区域中的内存管理
- 物理内存管理的目标有两个:进程资源隔离，效率和性能；
通过虚拟地址和物理地址的转化，虚拟地址向进程提供统一的地址分布，而页表的转换保证了每个进程操作的物理地址不同。
对物理地址进行逻辑上分页创建了一种相同粒度上的映射单位，然后可以以一种全关联的方式进行映射管理，即一个虚拟地址理论上可以映射到任何的物理地址，实际上为了效率内核所占的部分内存是不能映射的.
效率方面，我们先看物理结构:
物理架构上来看，内存一般是接在被桥上，和显卡类似一样是离CPU最近的设备;然后有类似于南桥的东西，硬盘,USB,UART等都是连接在南桥上然后再和内存连接的。更细一点，CPU内部一般都有３级缓存，L1中数据和指令分开的,L2两者公用，L3是核间公用充当所有CPU和内存的缓冲。然后还有一个重要的设备DMA控制器，能够不占用cpu的情况下复制数据，这个真的是减少了CPU I/O等待时间。
所以效率分为两方面，利用cache来缓冲CPU和内存的速度，技术有per_cpu_pages,slab,就是减少cache miss率，性能的表现就是cache miss率；
页缓存和块缓存来缓存硬盘上的东西，减少磁盘I/O带来的延迟。
另外我们需要有DMA zone来专门进行设备I/O.

另外一方面是节约内存:
1.只有真正操作物理内存的时候再分配，这样就产生了缺页中断这种分配物理地址的手段
2.swap,swap,swap,讨厌的事情说３遍，将不经常使用的页交换到慢速设备上来腾出可用物理内存
呵呵，上面两种，一种在内存富裕的时候利用内存缓存来提升性能，另外一方面又尽量减少内存分配来节约内存(还有就是效率，例如exec),实现完美的平衡

- 其他问题:
我们知道的内存碎片问题，常常分配不出来物理连续的内存空间，围绕这个问题，有buddy系统的阶的概念，有MOVEABLE概念，有针对slab的内存压缩,针对普通内存的内存规整，CMA，KSM等技术

几种分配方式:buddy系统，per_cpu_pages,slab

在分页的基础上，我们有了buddy系统，管理以页为单位的内存，是最直接的内存管理，其他的per_cpu_pages,slab，malloc等最后都要buddy系统中直接或者间接地获得物理页.物理内存还希望是在物理上连续的，所以维护了不同阶的链表。
每当我们释放某个页之后，只是标识它是空闲的,但是它的数据却还在cache中,这样再次使用这个页时，可以直接操作cache中的数据，省却了从物理内存中load到cache的过程，因此诞生了per_cpu_pages管理.
应用程序分配的内存通常很小，远小于4k,分配的粒度太大了，这样就造成了很多页内浪费。发明了slab系列，是一种内存到内存的缓冲。

## 内存初始化
系统如何直到物理内存的信息？如何管理这些内存的？  
在x86上我们知道是通过e820寄存器获得系统物理内存信息，不是很了解x86,暂且不表.  
在ARM上有两种方式获得物理内存的信息，通过uboot传过来、通过device tree信息获得,分别在`setup_machine_tags`和`setup_machine_fdt`解析获得;目前android上大部分都是通过第二种方式传递给kernel，有一些使用uboot的仍然采用atags的方式传送过来。具体可以参考`Documentation/arm/Booting`  
获得信息之后如何管理这些内存?  
不管是x86还是arm，都会将信息保存在一种通用的结构中:`memblock`
```
struct memblock {
	phys_addr_t current_limit;
	struct memblock_type memory;  //所有可用的内存
	struct memblock_type reserved; //被系统占用的内存，将不会传递给zone管理
};
```
通过device tree传递的物理内存信息转换：  
在`setup_machine_fdt`中通过early_init_dt_scan_memory来获取memory的节点,最后加入到memblock中，里面分成两部分，总的内存memblock->memory和已经被占用的内存memblock->reserved;
```
drivers/of/fdt.c
early_init_dt_scan_memory(unsigned long node, const char *uname,
				     int depth, void *data)
{
	char *type = of_get_flat_dt_prop(node, "device_type", NULL);

	reg = of_get_flat_dt_prop(node, "linux,usable-memory", &l);
	endp = reg + (l / sizeof(__be32));
	while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {
		u64 base, size;

		base = dt_mem_next_cell(dt_root_addr_cells, &reg);
		size = dt_mem_next_cell(dt_root_size_cells, &reg);
		early_init_dt_add_memory_arch(base, size);
	}
}
```
上面是加入到memblock->memory中，表示总的可用物理内存;  
下面就是需要将内核占用的物理内存在reserved中标示出来，通常都有的内核的`text`,`data`, `initrd`,`swapper_pg_dir`,`dtb`,`DMA`  

然后在物理页面之后就是页表的初始化,在`prepare_page_table`清除VMALLOC之下的所有页表,重头戏都在`map_lowmem->create_mapping`,遍历pgd,pmd,然后分配pte空间，然后填充pte,pmd.在现在的arm中只有二级页表，pgd=pmd，关于pmd的操作就是操作pgd。
这里有个点:arm的MMU中PGD是20位，剩余的是标志位，但是在linux需要一些额外的标志位,YOUNG(表示页最近访问过),DIRTY(页内容被改变了)，而arm的硬件没有这些标志位，所以需要在软件上实现这些标志，在相应的操作中，置软件标志位.实际上pgd entry仍然是4字节的，每个entry可以指向2^8个pte项，每个pte项占据4字节，所以pgd指向的pte连续空间为1024字节空间，每个页4k空间，4个pgd可以指向同一页的pte，但是这里取巧操作，每两个pgd指向一个页，公2048字节，两外的2048字节保存软件标志位；而pgd在软件上看是每个entry占据8字节，指向一页空间的pte，相邻的两个地址可以共用一个页基地址；

下面映射dma,remap,fixmap,vmalloc区域。

下面就是填充pglist_data和各个zone的数据

## 页表
在启动MMU的系统中如何通过虚拟地址找到物理页？MMU中的作用？  
1.MMU中看到的物理地址，所以pgd,pte中存储的都是页物理地址  
2.linux理论上支持4极页表，而32位上一般使用2级页表，64位上使用3级页表  
3.内核的页表在swapper_pg_dir  
4.mm_struct->pgd是虚拟地址，因为处于线性映射，在装入MMU之前会转换为物理地址访问  

### arm页表

## 物理页分配和释放
### buddy
### alloc_page
分配时提供的参数:order和gfp_t,order是指阶数，需要分配`2^order`;gfp_t(Get free page)中有两种标志：从那个zone中分配，称为`Zone modifiers`，另外一种描述是`Action modifiers`；可以参考`include/linux/gfp.h`
理想情况下页的分配：
1.通过gfp_t中找到优先从哪个zone中分配，顺带确定后备分配区域zonelist
pglist_data中有一个zonelist node_zonelists，在初始化时确定这个数组里的顺序，从低到高分别是:Highmem->normal->dma,如果确定优先从normal区域中分配，当normal中不足时尝试从dma中获取，却不会去highmem中查找。
2.查找normal中的buddy中的free_area中链表，从order阶开始查找是否有足够的页满足要求
3.如果`free_area[order]`上没有的话，就继续查找`free_area[order]`到`free_area[MAX_ORDER]`,找到之后需要拆分高阶连续的页并且挨个放入低阶链表中(expand)
4.如果没有足够的页，尝试到后备分配区域中查找，需要满足条件:例如优先从normal中分配，现在需要到dma中分配，需要dma区域中`free page > watermark[high]+lowmem_reserve[1]`,lowmem_reserve的计算参考说明`Documentation/sysctl/vm.txt`中关于`lowmem_reserve_ratio`和`setup_per_zone_lowmem_reserve`实现方式
5.如果都不满足的话，尝试进行页面回收之后再次尝试分配
引起睡眠的分配API:
### slab
1.目的:slab是为小块内存分配提供的一种管理方式  
2.主要管理数据
3.实现方式:
4.创建、销毁
5.分配、释放
slab:  
slab可以看作是一组页，通常是一个页，当页从buddy中移到slab中管理时，slab就代表这个页
```
struct slab {
		struct {
			struct list_head list;          //挂入partial,free,full中
			unsigned long colouroff;        //color是指缓存行，colouroff是color offset,指缓存行可以占据的空间
			void *s_mem;		                //这个代表coloroff之后的位置
			unsigned int inuse;	            //这个slab中有多少个对象在被使用
			kmem_bufctl_t free;             //这个其实是char free[kmem_cache->num],每个字节和一个obj关联
			unsigned short nodeid;
		};
};
```
![image](../images/slab.jpg)

slab分配总体样子:  
![image](../images/slab-all.jpg)

分配时，先从`kmem_cache->array_cache`中寻找，如果找不到再从`kmem_cache->kmem_cache_node->shared`中分配，然后再从`kmem_cache->kmem_cache_node->slabs_free`分配，不足的时候再从`per_cpu_pages`或`buddy`中分配.
### per_cpu_pages
适合单个页的分配
## 虚拟地址空间
### 寻找vma
### 插入vma
## 逆向映射
### anon_vma，anon_vma_chain
## 页缓存
其中有两个方式:页缓存和块缓存
页缓存的存在意义:读取规模比较大的操作时，使用页缓存更加方便
块缓存:在页需要回写的时候可以通过更小粒度的块而不是整页为单位回写,一般是文件系统的元数据，量特别小
而真正和硬盘IO打交道的时候都是通过发送bio请求，这点应该是没有啥区别的。
其本身就是为了效率而设计的，类似于cache的作用，所以需要设计几个点:
1.读写操作时首先会查询是否在缓存中，否则发生IO miss,需要从磁盘中加载，所以查询的操作需要高效，设计出了基数树；
缓存虽然好，但是总会存在负面作用:可能在缓存中的数据已经被修改了，但是没有被回写到磁盘上就被关机，所以设计出了回写机制:定时回写，直接回写，还有用户主动通过sync来回写。
这里为了可以管理按页处理和缓存的各种不同对象，内核使用了address_mapping抽象，将内存中的页和特定的块设备关联起来。每个地址空间都有一个宿主作为其数据来源，大多数情况下是标示一个文件的inode，少数情况下是设备本身，就是我们平时说的读文件、读设备。
address_mapping作为内存中的缓存页和inode之间的抽象层，存在一种这样的关系:每个文件系统挂载时都会生成super_block实例，关联着所有该文件系统中打开的所有文件对象inode,然
后address_mapping,然后就能遍历所有的缓存页的状态，这也是drop_caches的背后关系之一.
块缓存是如何组织的呢?

页缓存的实现?
分配页缓存:分配页，然后插入页缓存中:add_to_page_cache
查找页缓存:find_get_page
删除页缓存:从页缓存中删除释放回per_cpu_pageset/buddy中
页缓存的读写:分成两部分，分配页缓存并加入到对应的节点，然后创建bio请求等待读写完成
页缓存预读:假设页是顺序读写的，所以当发现符合顺序的规律时，就做出更加大胆的判断，预读更多的页；否则重新开始预测。
预读是在这３个地方控制的:
(1)do_generic_mapping_read,通用的读取例程
(2)缺页异常处理程序filemap_fault,负责为内存映射读取缺页
(3)`__generic_file_splice_read`,支持splice系统调用(一般不用鸟它)
```
if(absent(page)) {
  alloc_pages(8)
  read_pages(8)
}

```
## 页回收
## 缺页异常
