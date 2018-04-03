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
其实说白了在以太网中就是熟知的arp,不过在ipv6中则是ND(邻居发现协议).  
什么时候能够获取到邻接主机的IP地址呢?
```
1.主动发送arp请求，对应主机回复
2.对方发送arp请求给自己，自己回复
```
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
  //传输的方法,根据nud_state会有不同的回调函数
	int			(*output)(struct neighbour *, struct sk_buff *);
	const struct neigh_ops	*ops; //邻居的操作方法
	struct rcu_head		rcu;
	struct net_device	*dev;  //哪个网卡设备所属的邻居
	u8			primary_key[0];  //邻居的IP地址
};
```
