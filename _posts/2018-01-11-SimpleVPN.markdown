---
layout:     post
title:      "SimpleVPN"
date:       2018-03-11 23:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - network
---
# 简单VPN示例
## 需求背景
A需要所有数据在网络上传输时都必须是经过加密的，以防止其他人窃取隐私，所以需要一种方案能够加密所有的网络数据。
其实现有的方案已经有好多了,像openvpn,IPSec,sock5等理论上都是能够达到目的的。下面就是根据openvpn的原理写的简要demo,加解密和密钥相关的可以自己增加.  
在此之前需要澄清一个概念,我们需要保护的对象是什么?是数据，是两个IP之间的数据。如何保护呢?通过密码技术，必有加密和解密两个对等环节。
我们平时看到的https连接，在我们看不到的地方进行了加解密协议协商，密钥协商等动作，加解密所用的加解密算法和密钥等都是网络上约定好的。所以我们的加密也是保护两个IP之间的数据传输，不是和任意IP的传输。  
下面的内容需要对IP的路由功能,linux的协议栈,netfilter有些了解  
## 方向
特别感谢大牛[Bomb250](http://blog.csdn.net/dog250/article/details/6964047),虽未谋面但深表佩服。  
需要解决两个问题
- 1.怎么能够拦截住用户的所有数据呢  

这个是分场景的，不同的场景需要细微的修改。不过我下面的所有核心都是利用IP层的路由功能和tun网络设备，这里先回顾一下搜索路由表的策略和tun设备的原理:
```
1.搜索匹配的主机地址
2.搜索匹配的网络地址
3.搜索默认路由表
```
tun设备的位置:  
![](/images/tun-position.jpg)  
tun设备实现了两种功能,一种是网络设备,有收发包的能力，注册了net_device接口；另外它还是一种misc设备,可以和用户空间通过`read/write`交互;我们可以通过tun来接收/转发所有的用户流量，向tun中写数据包来模仿物理网络返回的数据包。  
回归正题,拦截所有的网络数据的选择,理论上可以通过匹配主机的方式来拦截数据,不过你不可能知道用户访问的所有地址，所以这个看起来有点不可能，同样的道理匹配网络地址也不可能；就剩下默认路由一种选择了。不过存在一种情况默认路由是拦截不了的:用户和服务器都位于同一个局域网内(无线局域网内，虽然是局域网，但是通过无线传输时是需要加密的),路由查找策略根本就不会走到默认路由表项。此时可以通过`iptables & netfilter`的`nat`模块来做一些地址上的转换使数据包转换到tun设备上。
- 2.怎么区分不同的应用数据

应用走网络的时候都是通过socket来通信的，这里我们考虑的少点，只考虑ipv4家族系列,里面有两种类型的TCP/UDP协议，在协议描述中这两种都是有源端口和目的端口，以此来区分不同的应用.  
**场景1:**  
当客户端A访问服务端B,且不在同一个网段时,加密算法没有限制。我们可以通过加密整个的IP包之后进行传输.  
![](/images/SimpleVPN2.jpeg)  
原始的连接图，客户端B`10.0.0.161`访问server`10.1.2.5`上的服务，来往的数据包非常简单明确:  

![](/images/SimpleIpsecFlow2.jpeg)  
原图戳[这里](/images/SimpleIpsecFlow2.jpeg)  
**场景２:**  

1.客户端A访问服务端B,在同一个网段.  
因为同网段主机的流量，我们要实现直接的数据拦截需要添加特定的主机路由，但是我们最终的数据还是需要发送给B,没有办法直接区分加密前流量和加密后流量，最终的结果就是都流到了tun设备。
这里使用了iptables在`OUTPUT`阶段将非ESP的包全部`DNAT`到一个虚假的不在同网段的IP上，目的是使用默认路由即将用户数据发送给tun设备。  
2.使用ESP协议加密.  
ESP有两种模式:隧道模式和传输模式.在传输模式中,因为要做NAT,肯定需要重新校验，内核中是没有esp的校验方法的，所以包会被损坏丢弃。所以只能采取隧道模式,加密整个的IP包之后进行传输.  

整个的数据流向如图所示:  
![](/images/SimpleIpsecFlow.jpeg)  
[点击这里查看原图](/images/SimpleIpsecFlow.jpeg)  
上图中基本是利用的协议栈的路由功能和NAT表，处理如下:
需要提前设定一些iptables规则来引导流量走不同的路径:
```
client端设置:
iptables -t nat -A OUTPUT ! -p esp -d 10.0.0.160 -j DNAT --to-destination 10.0.1.160
设置tun设备作为默认路由

server端设置:
iptables -t nat -A PREROUTING -p esp -d 10.0.0.160 -j DNAT --to-destination 192.168.1.2
设置tun设备作为默认路由
```
1.应用发送的数据，在`OUTPUT`阶段因为是非ESP数据，所以全部`DNAT`到一个虚假的ip上，目的是为了路由到tun设备上。    
这个虚假的ip是需要有规律的，因为我们最终仍然需要发送到正确的目的主机上，所以需要在`3-4`中间还原回来正确的目的地址。  
所以在iptables规则中设置的统一是一个网络地址偏移，在Daemon中再还原回来。  
2.在`3-4`中Daemon拿到了用户数据，然后解析到了正确的目的地址，那么只需要外面加个壳子(IP头和ESP信息)，将原始的ip地址分别复制到壳子中的ip地址中，然后发出去  
3.服务端在接收到esp信息后在`PREROUTING`处`DNAT`到一个虚假的地址，这个地址没有什么要求，只是为了路由到tun设备上，真实的ip信息是通过解封装esp之后才拿到的
4.服务端Daemon程序拿到客户端应用的请求信息，这时候是需要通过tun设备发送给网络栈的，但是数据怎么返回呢?这里是通过更改一个发送包的源地址为一个虚假的IP地址(`10`中源地址已经被更改),然后tun设备作为默认路由能够拿到服务端服务返回的信息,然后再将目的地址转换回来.这样做是可以处理多个client端的数据的
5.服务端Daemon加上壳子(IP头和ESP信息),发送给客户端  
这里是有些问题的，因为ESP的特殊性出现了某些问题,看[这里](https://faxiang1230.github.io/2018/03/21/conntrack_drop_packet/),需要将源地址设置为虚假的IP地址,看步骤`8`,`16`   
6.客户端拿到服务端返回的数据之后呢,可以通过raw socket来接收ESP数据，然后解封装,得到服务端回应的真实报文，再重新塞回协议栈之前,需要照顾`NAT`的`conntrack`部分，将源地址恢复成虚假的地址，看步骤`2`,`21`  
[Demo地址](https://github.com/faxiang1230/openvpnforipsec.git)  

**缺陷:**
这个方案目前只能对付同网段中单服务器的情况，多个服务器除了设置多条`DNAT`规则之外暂时没有好办法
