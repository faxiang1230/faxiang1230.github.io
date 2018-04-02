---
layout:     post
title:      "Net Conntrack"
date:       2018-04-02 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - network
---
# Conntrack
在这里有必要再来看一眼netfilter框架，提供了数据包在网络栈传输路径上各个关键节点注册回调函数，从而对数据包进行各种操作:修改地址，丢弃数据包，记录日志等。所有接收、发送的数据都需要经过netfilter框架，下面图展示了路径中关键节点:  
![](../images/Netfilter-packet-flow.svg)  
绿色的表示IP层hook点，而蓝色表示mac层hook点;  
这张图展示了所有的关键节点，netconntrack,nat,filter等都是在节点上注册回调，回调函数通过优先级来决定调用次序，通过优先级的安排让不同的module可以相互协作.  
Netfilter有５个挂接点:PRE_ROUTING,LOCAL_IN,FORWARD,POST_ROUTING,LOCAL_OUT.  
conntrack利用其５个挂节点实现`连接跟踪`功能，每个连接可以提取出来`元信息`来标示每个连接,
例如TCP/UDP可以根据`protocol,src addr,dest addr, src port, dest port`来表示不同的连接，不过也不是所有的协议都可以提取出来`元信息`的，例如`ESP`根本没有`port`,ICMP也没有`port`,此时默认的将`protocol,src addr,dest addr`相同的当成同一个连接。
conntrack的主要目标是为`nat`打下基础，说白了主要用户就是`nat`功能。  
## 数据关系
![](../images/netconntrack1.jpeg)  
初始化的时候
## 初始化工作
和netfilter对应的用户空间工具,是基于netfilter框架的工具，常使用的有iptables,能够添加和删除netfilter规则，添加表等。在使用iptables的时候就可能会动态加载nf_conntrack
```
iptables -t nat -A INPUT -p tcp -j LOG
```
短短的一条规则需要加载这么多module:
```
nf_log_ipv4            16384  1
nf_log_common          16384  1 nf_log_ipv4
xt_LOG                 16384  1
iptable_nat            16384  1
nf_conntrack_ipv4      16384  1
nf_defrag_ipv4         16384  1 nf_conntrack_ipv4
nf_nat_ipv4            16384  1 iptable_nat
nf_nat                 28672  1 nf_nat_ipv4
nf_conntrack          131072  3 nf_conntrack_ipv4,nf_nat_ipv4,nf_nat
libcrc32c              16384  2 nf_conntrack,nf_nat
ip_tables              24576  1 iptable_nat
x_tables               36864  2 xt_LOG,ip_tables
```
nf_conntrack是这些里面的核心模块，被依赖最多。如果要撸代码的话，是需要从多个module的init开始看的，涉及的目录还比较多，容易看混了。  
nf_conntrack是核心通用的框架，里面就是初始化`init_net->netns_ct`中许多的表，分配空间，初始化一些限制。里面有两个数:`nf_conntrack_htable_size`和`nf_ct_expect_hsize`,分别是netns_ct中hash和expect_hash表的大小限制，如果没有人为干预的话，`nf_conntrack_htable_size`＝
```
if(totoal memory <=32MB)
  nf_conntrack_htable_size = 512
else if(totoal memory > 1G)
  nf_conntrack_htable_size = 16384
else
  nf_conntrack_htable_size = totoal memory / 16384
```
定义了同时维护的最大连接数，当连接限制过大时，浪费内存；当连接数限制过小时，需要不停地重建连接信息，效率较低。
除了空间上限制之外，不同的协议还定义了不同的超时机制，超时之后自动从hash上移除。  
而`nf_ct_expect_hsize`就是根据`nf_conntrack_htable_size`简单的计算得出:`nf_ct_expect_hsize = nf_conntrack_htable_size /256`  

实际上真正work的是`nf_conntrack_ipv4`,主要是注册netfilter的挂载点,填充nf_conntrack中协议相关部分
注册挂载点:
```
nf_conntrack_l3proto_ipv4_init
  ->nf_register_hooks(ipv4_conntrack_ops)
static struct nf_hook_ops ipv4_conntrack_ops[] __read_mostly = {
	{
		.hook		= ipv4_conntrack_in,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_conntrack_local,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_helper,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_HELPER,
	},
	{
		.hook		= ipv4_confirm,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
	{
		.hook		= ipv4_helper,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_HELPER,
	},
	{
		.hook		= ipv4_confirm,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
};
```
注册协议到`nf_ct_protos`中，在构建`元信息`时会调用具体的协议来构建信息。  

## 接收数据时工作流程

**nf_conntrack的所有工作流都是基于netfilter的框架的，在哪个位置做什么事情都是提前注册好的。**    
网络数据目的地址就是本机ip,采用udp协议。按照顺序是`ipv4_conntrack_in`,`ipv4_helper`,`ipv4_confirm`
```
nf_conntrack_in
   //根据数据信息解析出协议信息,然后校验一下.
   __nf_ct_l3proto_find
   l3proto->get_l4proto
   __nf_ct_l4proto_find
   l4proto->error
   //检索ct是否存在，不存在就新建并初始化，插入到unconfirmed表中。
   resolve_normal_ct
    hash_conntrack_raw  //计算出origin的tuple信息
    __nf_conntrack_find_get //根据tulple信息去hash表中去查找，没有找到就开始新建连接信息
    init_conntrack  //新建连接信息
    　　__nf_conntrack_alloc  //分配空间
    　　hlist_nulls_add_head_rcu(&ct->tuplehash[IP_CT_DIR_ORIGINAL].hnnode,
　　　　　　　&net->ct.unconfirmed);  //挂入unconfirmed表中
   nf_ct_timeout_lookup
   l4proto->packet
```
```
ipv4_helper  //是辅助方法的入口，不过这个辅助方法把nat都包含进去了，暂时不管这里的helper方法
```
```
ipv4_confirm
  __nf_conntrack_confirm //主要就是将ct从unconfirmed表中挪去，然后添加到hash表中
```
