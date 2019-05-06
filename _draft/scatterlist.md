## scatterlist产生的背景
>>转发:http://www.wowotech.net/memory_management/scatterlist.html

我没有去考究scatterlist API是在哪个kernel版本中引入的（年代太久远了），凭猜测，我觉得应该和MMU有关。因为在引入MMU之后，linux系统中的软件将不得不面对一个困扰（下文将以图片1中所示的系统架构为例进行说明）：
```
假设在一个系统中（参考下面图片1）有三个模块可以访问memory：CPU、DMA控制器和某个外设。CPU通过MMU以虚拟地址（VA）的形式访问memory；DMA直接以物理地址（PA）的形式访问memory；Device通过自己的IOMMU以设备地址（DA）的形式访问memory。
然后，某个“软件实体”分配并使用了一片存储空间（参考下面图片2）。该存储空间在CPU视角上（虚拟空间）是连续的，起始地址是va1（实际上，它映射到了3块不连续的物理内存上，我们以pa1,pa2,pa3表示）。

那么，如果该软件单纯的以CPU视角访问这块空间（操作va1），则完全没有问题，因为MMU实现了连续VA到非连续PA的映射。

不过，如果软件经过一系列操作后，要把该存储空间交给DMA控制器，最终由DMA控制器将其中的数据搬移给某个外设的时候，由于DMA控制器只能访问物理地址，只能以“不连续的物理内存块”为单位递交（而不是我们所熟悉的虚拟地址）。

此时，scatterlist就诞生了：为了方便，我们需要使用一个数据结构来描述这一个个“不连续的物理内存块”（起始地址、长度等信息），这个数据结构就是scatterlist（具体可参考下面第3章的说明）。而多个scatterlist组合在一起形成一个表（可以是一个struct scatterlist类型的数组，也可以是kernel帮忙抽象出来的struct sg_table），就可以完整的描述这个虚拟地址了。

最后，从本质上说：scatterlist（数组）是各种不同地址映射空间（PA、VA、DA、等等）的媒介（因为物理地址是真实的、实在的存在，因而可以作为通用语言），借助它，这些映射空间才能相互转换（例如从VA转到DA）。
```
![image](/img/scatterlist1.gif)  
图片1 cpu_dma_device_memory

![image](/img/scatterlist2.gif)  
图片2 cpu_view_memory  

## scatterlist API介绍
3.1 struct scatterlist

struct scatterlist用于描述一个在物理地址上连续的内存块（以page为单位），它的定义位于“include/linux/scatterlist.h”中，如下：
```
struct scatterlist {
#ifdef CONFIG_DEBUG_SG
       unsigned long   sg_magic;
#endif
       unsigned long   page_link;
       unsigned int    offset;
       unsigned int    length;
       dma_addr_t      dma_address;
#ifdef CONFIG_NEED_SG_DMA_LENGTH
       unsigned int    dma_length;
#endif
};
```

   page_link，指示该内存块所在的页面。bit0和bit1有特殊用途（可参考后面的介绍），因此要求page最低4字节对齐。
   offset，指示该内存块在页面中的偏移（起始位置）。
   length，该内存块的长度。

   dma_address，该内存块实际的起始地址（PA，相比page更接近我们人类的语言）。
   dma_length，相应的长度信息。

3.2 struct sg_table

在实际的应用场景中，单个的scatterlist是没有多少意义的，我们需要多个scatterlist组成一个数组，以表示在物理上不连续的虚拟地址空间。通常情况下，使用scatterlist功能的模块，会自行维护这个数组（指针和长度），例如[2]中所提到的struct mmc_data：
```
struct mmc_data {
   …

       unsigned int            sg_len;         /* size of scatter list */     
       struct scatterlist      *sg;            /* I/O scatter list */         
       s32                     host_cookie;    /* host private data */        
};
```
不过呢，为了使用者可以偷懒，kernel抽象出来了一个简单的数据结构：struct sg_table，帮忙保存scatterlist的数组指针和长度：
```
struct sg_table {
       struct scatterlist *sgl;        /* the list */
       unsigned int nents;             /* number of mapped entries */
       unsigned int orig_nents;        /* original size of list */
};
```
其中sgl是内存块数组的首地址，orig_nents是内存块数组的size，nents是有效的内存块个数（可能会小于orig_nents）。

以上心思都比较直接，不过有一点，我们要仔细理解：

scatterlist数组中到底有多少有效内存块呢？这不是一个很直观的事情，主要有如下2个规则决定：

   1）如果scatterlist数组中某个scatterlist的page_link的bit0为1，表示该scatterlist不是一个有效的内存块，而是一个chain（铰链），指向另一个scatterlist数组。通过这种机制，可以将不同的scatterlist数组链在一起，因为scatterlist也称作chain scatterlist。

   2）如果scatterlist数组中某个scatterlist的page_link的bit1为1，表示该scatterlist是scatterlist数组中最后一个有效内存块（后面的就忽略不计了）。

3.3 API介绍

理解了scatterlist的含义之后，再去看“include/linux/scatterlist.h”中的API，就容易多了，例如（简单介绍一下，不再详细分析）：
```
#define sg_dma_address(sg)      ((sg)->dma_address)

#ifdef CONFIG_NEED_SG_DMA_LENGTH
#define sg_dma_len(sg)          ((sg)->dma_length)
#else
#define sg_dma_len(sg)          ((sg)->length)
#endif
```
   sg_dma_address、sg_dma_len，获取某一个scatterlist的物理地址和长度。
```
#define sg_is_chain(sg)         ((sg)->page_link & 0x01)
#define sg_is_last(sg)          ((sg)->page_link & 0x02)
#define sg_chain_ptr(sg)        \
       ((struct scatterlist *) ((sg)->page_link & ~0x03))
```
   sg_is_chain可用来判断某个scatterlist是否为一个chain，sg_is_last可用来判断某个scatterlist是否是sg_table中最后一个scatterlist。

   sg_chain_ptr可获取chain scatterlist指向的那个scatterlist。
```
static inline void sg_assign_page(struct scatterlist *sg, struct page *page)
static inline void sg_set_page(struct scatterlist *sg, struct page *page,
                              unsigned int len, unsigned int offset)
static inline struct page *sg_page(struct scatterlist *sg)
static inline void sg_set_buf(struct scatterlist *sg, const void *buf,
                              unsigned int buflen)

#define for_each_sg(sglist, sg, nr, __i)        \
       for (__i = 0, sg = (sglist); __i < (nr); __i++, sg = sg_next(sg))

static inline void sg_chain(struct scatterlist *prv, unsigned int prv_nents,
                            struct scatterlist *sgl)

static inline void sg_mark_end(struct scatterlist *sg)
static inline void sg_unmark_end(struct scatterlist *sg)

static inline dma_addr_t sg_phys(struct scatterlist *sg)
static inline void *sg_virt(struct scatterlist *sg)
```
   sg_assign_page，将page赋给指定的scatterlist（设置page_link字段）。
   sg_set_page，将page中指定offset、指定长度的内存赋给指定的scatterlist（设置page_link、offset、len字段）。
   sg_page，获取scatterlist所对应的page指针。
   sg_set_buf，将指定长度的buffer赋给scatterlist（从虚拟地址中获得page指针、在page中的offset之后，再调用sg_set_page）。

   for_each_sg，遍历一个scatterlist数组（sglist）中所有的有效scatterlist（考虑sg_is_chain和sg_is_last的情况）。

   sg_chain，将两个scatterlist 数组捆绑在一起。

   sg_mark_end、sg_unmark_end，将某个scatterlist 标记（或者不标记）为the last one。

   sg_phys、sg_virt，获取某个scatterlist的物理或者虚拟地址。
