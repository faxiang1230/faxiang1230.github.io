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
conntrack主要的作用是连接跟踪，对每一个来往的数据流都记录下来,它并不是一个单独的module,而是一种基础设施，主要是为NAT来服务的，还有期望连接,其他辅助方法。  
再往下它是构筑在netfilter框架上的一个模块,需要向netfilter注册hook方法。
它是如何区分每个数据流呢?根据协议的不同从中提取出来`元信息`，具有相同`元信息`的数据流它当做同一个连接;
例如TCP/UDP可以根据`protocol,src addr,dest addr, src port, dest port`来组成`元信息`。不过也不是所有的协议都可以提取出来`元信息`的，例如`ESP`根本没有`port`,同时也没有定义如何标示不同的连接函数时,此时默认的将`protocol,src addr,dest addr`相同的当成同一个连接。
conntrack模块一般只是提取`元信息`记录，并不会丢弃数据包，但是<font color=red>有时会因为错误或者异常而丢弃数据包</font>。

先再来看一下netfilter框架，它提供了数据包在网络栈传输路径上各个关键节点注册回调函数的能力，从而回调函数可以对数据包进行各种操作:修改地址，丢弃数据包，记录日志等。所有接收、发送的数据都需要经过netfilter框架，它有５个挂接点:`PRE_ROUTING,LOCAL_IN,FORWARD,POST_ROUTING,LOCAL_OUT`. 用户一般会使用netfilter的前端应用`iptables`来修改、查看这５张表.
下图展示了路径中关键节点:  
![](../images/netfilter-packet-flow.jpg)  
绿色的表示IP层hook点，而蓝色表示mac层hook点;  
这张图展示了所有表的关键节点和调用顺序，netconntrack,nat,filter等都是在节点注册回调，回调函数通过优先级来决定调用次序，通过优先级的安排让不同的module可以相互协作.  

## 数据关系
![](../images/netconntrack1.jpeg)  
假设现在只有一个net空间，里面指向唯一的一个`netns_ct`,其中维护了conntrack的信息
```
struct netns_ct {
	atomic_t		count;                   //现有的连接数
	unsigned int		htable_size;　　　　　//hash表大小
	struct hlist_nulls_head	*hash;　　　　　　//hash表头
	struct hlist_nulls_head	unconfirmed;　　//还没有确认双向元信息的连接都挂在这里
	struct hlist_nulls_head	dying;       //要移除的连接
	struct hlist_nulls_head tmpl;
	struct ip_conntrack_stat __percpu *stat;
	struct nf_ip_net	nf_ct_proto;
#ifdef CONFIG_NF_NAT_NEEDED
	struct hlist_head	*nat_bysource;
	unsigned int		nat_htable_size;
#endif
};
```
以下是经过删减的每个连接的元信息定义，根据协议不同有不同的表示方法:
```
struct nf_conntrack_tuple {
  源地址、源端口
	struct nf_conntrack_man src;

	目的地址，目的端口，协议类型
	struct {
		union nf_inet_addr u3;
		union {
			struct {
				__be16 port;
			} tcp;
			struct {
				__be16 port;
			} udp;
    ｝u;
		u_int8_t protonum;

		/* The direction (for tuplehash) */
		u_int8_t dir;
	} dst;
};
```
每个连接的抽象,里面我们暂时只关心连接的元信息和状态:
```
struct nf_conn {
  //ORIGIN和REPLY的元信息表示
	struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];

	连接的状态，刚开始跟踪连接时为CT_NEW,连接建立后变成CT_ESTABLISHED
	unsigned long status;
};
```
连接跟踪就是提取数据流中特征值,这个是和协议相关的，能够区分出不同的连接，然后记录到hash表中,它单独是没有什么用处的。    
主要是为`NAT`服务的，例如在`PREROUTING`处做了`DNAT`转换,连接信息的转换:

|DNAT前|DNAT后|
|--|--|
|src:10.0.0.2|src:10.0.0.2|
|dst:10.0.1.2|dst:10.0.1.3|

连接跟踪信息就是包含上面两组元信息,主要的点如下:

|ORIGINAL|src:10.0.0.2|dst:10.0.1.2|
|--|--|--|
|REPLY|src:10.0.1.3|dst:10.0.0.2|

当有数据`src:10.0.1.3 dst:10.0.0.2`的数据包到来的时候会自动转换为`src:10.0.1.2 dst:10.0.0.2`,这也是为什么设置NAT时可以只设置一个DNAT不必再设置一个逆向的SNAT,conntrack会帮你处理好的。  
nf_conntrack的代码比较零散,核心框架都是在`net/netfilter/nf_conntrack*`,和协议相关的(这里我们暂时只关心ipv4)位于`net/ipv4/netfilter/nf_conntrack*`,看代码的时候是有多个模块初始化的，所以比较乱。
## 初始化工作
我们并不直接使用conntrack和netfilter模块，而是使用基于netfilter框架的工具，常使用的有iptables、ip6tables,能够添加和删除netfilter规则，添加表等。
conntrack是以module的形式存在的，当我们使用iptables时就可能会动态加载nf_conntrack。当你第一次执行下面这条命令的时候:
```
iptables -t nat -A INPUT -p tcp -j LOG
```
你会发现短短的一条规则需要加载这么多module:
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
nf_conntrack是这些里面的核心模块，被依赖最多。如果要撸代码的话，是需要从多个module的init开始看的，涉及的目录还比较多，容易看混了.  

nf_conntrack是核心通用的框架，里面就是初始化`init_net->netns_ct`中许多的表，分配空间，初始化一些限制。
里面有两个数:`nf_conntrack_htable_size`和`nf_ct_expect_hsize`,分别是netns_ct中hash
和expect_hash表的大小限制，如果没有人为干预的话，`nf_conntrack_htable_size`＝
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

nf_conntrack其实并没有做什么，就是初始化了一些内存资源，真正做事的还是具体协议。这里我们看ipv4初始化过程中`nf_conntrack_ipv4`,它注册netfilter的hook,填充nf_conntrack中协议相关部分.  
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

下面举个例子:网络数据目的地址就是本机ip,采用udp协议。按照顺序需要经过的hook是`ipv4_conntrack_in`,`ipv4_helper`,`ipv4_confirm`
```
nf_conntrack_in
   //根据数据信息解析出协议信息,然后校验包完整性.
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
上面主要是根据数据包的信息，做一些校验，提取协议信息等，然后根据协议提供的元信息去hash表中去检索是否已经存在相同元信息的连接。
不存在的话就创建nf_conn信息，将信息暂时挂入`net->ct.unconfirmed`,此时连接的状态为`CT_NEW`.
```
ipv4_helper  //是辅助方法的入口，不过这个辅助方法把nat都包含进去了，暂时不管这里的helper方法
```
```
ipv4_confirm
  __nf_conntrack_confirm //主要就是将ct从unconfirmed表中挪去，然后添加到hash表中
```
confirm中做的就更加简单了，主要是从unconfirmed表中挪到hash表中，更改连接状态。
