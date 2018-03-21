---
layout:     post
title:      "conntrack模块引起的丢包"
date:       2018-03-21 23:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - network
---
# conntrack模块引起的丢包
## 问题描述
![](/images/conntrack_drop_packet.jpeg)
环境设置:在B机器上设置一条iptable规则做DNAT转发  
`iptables -t nat -A PREROUTING -p esp -j DNAT --to-destination 192.168.10.2`  
打开B机器的forward功能:  
`echo 1 > /proc/sys/net/ipv4/ipv4_forward`

问题步骤:  
1.A发送一个esp包到B,ip信息如图所示  
2.B接收到之后在PREROUTING处做DNAT,然后在路由阶段就是做转发到C机器  
3.C发送返回信息,同样是esp协议封装，经过B成功返回A  
4.在B机器上直接使用raw socket封装esp协议，并发送给A,失败，返回
`Operation not permitted`  
## 过程分解
标注完数据流之后，看起来挺正常的，除了红色的那一块，却又说不上哪不对。  

过程说明:我们知道`nat`功能是作为一种扩展存在的，其依赖conntrack功能。在`PREROUTING`处做`DNAT`,最终会计算出5元组信息`dstip:srcip,dstport:srcport,protocol`会创建链接信息并且挂入到`net->netns_ct->hash`上，具体信息:
```
ORIGINAL:from 192.168.1.2 to 192.168.1.1 protocol:esp
REPLY:from 192.168.10.2 to 192.168.1.1 protocol:esp
```
因为没有esp协议的tuple计算方法，只有ip层计算出来的tuple信息，最终在hash表上的信息就是上面的信息了。  
从C回到A就根据tuple信息做`SNAT`转换,所以我们通常只需要写一条DNAT或SNAT规则，conntrack会帮助我们在回来的路上做转换的。  

当B直接发送信息给A时，就很尴尬了，计算出的tuple信息如下:
```
ORIGINAL:from 192.168.1.１ to 192.168.1.２ protocol:esp
REPLY:from 192.168.1.2 to 192.168.1.1 protocol:esp
```
这条hash信息的`REPLY`key和已经存在的`ORIGINAL`key其实是已经冲突了,肯定是不能接着挂入到hash表上了，所以就直接丢了。
## 结论
实现自己的ESP tuple计算方法
