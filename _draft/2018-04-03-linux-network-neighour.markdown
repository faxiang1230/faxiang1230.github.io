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
邻接子系统负责发现当前链路上的节点，并将L3地址转换为L2地址，在为出站数据包创建L2报头时，需要L2地址。
在这个系统设计中，neigh_table代表一种协议族，neighbour代表协议族中的一个条目，neigh_ops是neighbour的操作方法，根据其状态不同会有不同的操作方法。neigh_parms是neighbour的一些设定参数。  
其实说白了在以太网中就是熟知的arp,不过在ipv6中则是ND(邻居发现协议).  
什么时候能够获取到邻接主机的IP地址呢?
```
1.主动发送arp请求，对应主机回复
2.对方发送arp请求给自己，自己回复
```
抓取arp报文:
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
arp报文的一般形式是:以广播发送ARP请求，对应主机使用单播ARP回复  
另外是当目的地址是广播地址或组播地址时，也不需要arp,L2地址是固定的。而且在Linux中arp是有老化期限的，一个arp条目存在时间过长时
## 数据结构
```
struct neighbour {
	struct neighbour __rcu	*next;
	struct neigh_table	*tbl;
	struct neigh_parms	*parms;  //
	unsigned long		confirmed;  //确认时间戳
	unsigned long		updated;
	rwlock_t		lock;
	atomic_t		refcnt; //引用计数
	struct sk_buff_head	arp_queue;
	unsigned int		arp_queue_len_bytes;
	struct timer_list	timer;  //回调函数中修改nud_state
	unsigned long		used;
	atomic_t		probes;
	__u8			flags;
	__u8			nud_state;　NUD:neighbor unreachability detection
	__u8			type;
	__u8			dead;　　//dead被置位的邻居将由垃圾收集器回收删除
	seqlock_t		ha_lock;
	unsigned char		ha[ALIGN(MAX_ADDR_LEN, sizeof(unsigned long))];
	struct hh_cache		hh;  //邻接对象的硬件地址，在以太网中称之为MAC
  //传输的方法,根据nud_state会有不同的回调函数:arp_broken_ops,arp_direct_ops,arp_hh_ops,arp_generic_ops
	int			(*output)(struct neighbour *, struct sk_buff *);
	const struct neigh_ops	*ops; //邻居的操作方法
	struct rcu_head		rcu;
	struct net_device	*dev;  //哪个网卡设备所属的邻居
	u8			primary_key[0];  //邻居的IP地址
};
```
为了避免在每次传输数据包时都发送请求，内核将L3地址和L2地址之间映射存储在被称为邻接表的数据结构中，在ipv4中就是`arp_tbl`.
`neigh_table_init_no_netlink`执行林劫镖的所有初始化工作
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
