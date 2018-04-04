---
layout:     post
title:      "Linux网络邻接"
date:       2018-04-03 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - network
---
# Linux网络邻接
在发送数据时，网络层决定了如何路由，就是下一跳的IP地址.除此之外还需要在为出站数据包创建MAC头，吓一跳IP的MAC地址,邻接子系统就是负责发现当前链路上的节点，并将L3地址转换为L2地址.  
邻接子系统在以太网中就是熟知的arp,不过在ipv6中则是ND(邻居发现协议),这个协议本身就很简单，就是保存局域网内ip和mac的映射.  
整体上概念简单，不过在这个系统设计中有几种抽象:neigh_table代表一种协议族，neighbour代表协议族中的一个条目，neigh_ops是neighbour的操作方法,neigh_parms是neighbour的一些设定参数.  
怎么能够获取到邻接主机的IP地址呢?
```
1.主动发送arp请求，对应主机回复
2.对方发送arp请求给自己，自己回复
```
简单抓取arp报文:
```
$ arp  //查看arp条目
Address                  HWtype  HWaddress           Flags Mask            Iface
10.0.0.161               ether   c8:9c:dc:23:c2:01   C                     eth1
$ sudo arp -d 10.0.0.161　//删除arp条目
开启tcpdump,并ping 10.0.0.161,然后就看到arp报文主要内容
# tcpdump -n -i eth1 arp
18:32:46.003824 ARP, Request who-has 10.0.0.161 tell 10.0.0.160, length 28
18:32:46.004528 ARP, Reply 10.0.0.161 is-at c8:9c:dc:23:c2:01, length 46
```
从上面看出来:arp报文的一般形式是:以广播发送ARP请求，对应主机使用单播ARP回复.  
另外是当目的地址是广播地址或组播地址时，也不需要arp,L2地址是固定的。
## 数据结构
```
struct neighbour {
	struct neighbour __rcu	*next;
	struct neigh_table	*tbl;  //所属的协议族的table,目前只有ipv4和ipv6
	struct neigh_parms	*parms;  //设定参数，这个一般都是直接拷贝协议族中的neigh_parms
	unsigned long		confirmed;  //确认时间戳
	unsigned long		updated;  //更新时间戳
	rwlock_t		lock;
	atomic_t		refcnt; //引用计数
	struct sk_buff_head	arp_queue;
	unsigned int		arp_queue_len_bytes;
	struct timer_list	timer;  //回调函数中修改nud_state
	unsigned long		used;
	atomic_t		probes;
	__u8			flags;
	__u8			nud_state;　NUD:neighbor unreachability detection
	__u8			type;   //多播，广播还是单播
	__u8			dead;　　//dead被置位的邻居将由垃圾收集器回收删除
	seqlock_t		ha_lock;
	unsigned char		ha[ALIGN(MAX_ADDR_LEN, sizeof(unsigned long))];
	struct hh_cache		hh;  //邻接对象的硬件地址，在以太网中称之为MAC
  //传输的方法,根据nud_state会有不同的回调函数:arp_broken_ops,arp_direct_ops,arp_hh_ops,arp_generic_ops
	int			(*output)(struct neighbour *, struct sk_buff *);
	const struct neigh_ops	*ops; //邻居的操作方法
	struct net_device	*dev;  //哪个网卡设备所属的邻居
	u8			primary_key[0];  //邻居的IP地址
};
```
为了避免在每次传输数据包时都发送请求，内核将L3地址和L2地址之间映射存储在被称为邻接表的数据结构中，在ipv4中就是`arp_tbl`.

`neigh_table_init_no_netlink`执行邻接表的所有初始化工作  
## 工作过程
主要是看一下初始化过程中做了那些事，还有典型的发包过程中发送arp请求和arp响应.  
### 初始化
`net/ipv4/arp.c`中`arp_init`
```
neigh_table_init(&arp_tbl);
　初始化arp_tbl,分配内存,初始化工作队列
  neigh_table_init_no_netlink
  插入全局的链表neigh_tables中

注册arp协议，然后就可以接收处理arp报文了
dev_add_pack(&arp_packet_type);

创建/proc/net/arp
arp_proc_init();

创建/proc/sys/net/ipv4/neigh/
#ifdef CONFIG_SYSCTL
neigh_sysctl_register(NULL, &arp_tbl.parms, "ipv4", NULL);
#endif
}
```
### 第一次发送给某个IP
```
ip_finish_output2
  //根据目的地址查找下一跳地址
  nexthop = (__force u32) rt_nexthop(rt, ip_hdr(skb)->daddr);
  //通过ip去arp_tbl.nht中查找neighbour,此时肯定是没有的
  neigh = __ipv4_neigh_lookup_noref(dev, nexthop);
  //创建neighbour,主要就是通过arp_tbl.constructor来
  if (unlikely(!neigh))
    neigh = __neigh_create(&arp_tbl, &nexthop, dev, false);
      if (!IS_ERR(neigh))
        dst_neigh_output(dst, neigh, skb);
          if ((n->nud_state & NUD_CONNECTED) && hh->hh_len)
            return neigh_hh_output(hh, skb);
          else
            return n->output(n, skb);　//第一次就是neigh_resolve_output
```
仔细看一下`__neigh_create`中都干了哪些事:
```
__neigh_create
  neigh_alloc //分配neighbour
  tbl->constructor //arp初始化neighbour，主要初始化neighbour的output和ops指针函数.目前不知道有什么区别
  output = neigh_resolve_output //
  ops = arp_generic_ops
  neigh_hash_grow  //如果neighbour数目多了，是需要扩展arp_tbl->nht,新分配一个更大的表，将旧表中数据copy过去
```
arp报文真正要干的事是在`__neigh_event_send`中开始的，将会调用`__skb_queue_tail`将`skb`加入到`neighbour->arp_queue`中.稍后邻接定时器处理`neigh_timer_handler`将调用`neigh_probe`,最终调用`solicit`方法来发送数据包
```
arp_solicit
  1.根据/proc/sys/net/ipv4/conf/*/arp_announce中的配置选择源地址
    0:可以使用在任何接口上配置的ip,默认是这样的
    1:首先尝试使用属于目标子网的地址，如果没有这样的地址，就使用主ip地址
    2:使用主ip地址
  2.inet_select_addr选择一个和目的地址同网段的源地址
  3.arp_send发送请求，arp_create->arp_xmit
```
### arp报文接收和响应
在初始化时,注册了`arp_packet_type`,所以当接收到arp报文时可以进行响应.  
```
arp_rcv
  arp_process
    1.解析arp的头，源ip，目的ip,源mac,目的mac
    2.丢弃loopback和组播地址请求
    3.判断是否发送arp响应报文
      /proc/sys/net/ipv4/conf/*/arp_ignore
      /proc/sys/net/ipv4/conf/*/arp_filter
    4.在应答前将发送方加入邻接表或对其更新
    5.接收到ARP应答时,通过neigh_update更新邻接表
```
## 用户空间和邻接子系统的交互
查看
`/proc/net/arp`
`/proc/net/stat/arp_cache`
`arp`
`ip neigh show`
修改
`ip neigh add/del`
`ip ntable show`
静态ARP条目不会被邻接子系统垃圾回收器回收，但会在重启后消失.
