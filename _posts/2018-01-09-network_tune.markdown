---
layout:     post
title:      "网络tuning"
subtitle:   " performance tuning"
date:       2018-01-09 23:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - network
---
# 网络tuning
## 网络框图
[老外的原文](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/)  
很多事情是在初始化的时候搞的，所以很有必要来看初始化的时候都干了什么  
### 初始化  

![](../images/network-tuning-softirq.jpg)  

Devices have many ways of alerting the rest of the computer system that some work is ready for processing. In the case of network devices, it is common for the NIC to raise an IRQ to signal that a packet has arrived and is ready to be processed. When an IRQ handler is executed by the Linux kernel, it runs at a very, very high priority and often blocks additional IRQs from being generated. As such, IRQ handlers in device drivers must execute as quickly as possible and defer all long running work to execute outside of this context. This is why the “softIRQ” system exists.
在内核中，设备是通过中断的方式来告诉CPU某些事件需要处理，在网络中更是如此，网卡设备通常都是发中断告诉CPU有新的网络包到了。我们都知道,在Linux系统中中断的优先级是最高的，除了在禁止中断的时候，中断处理可以终止任何的程序上下文从而进入中断上下文，在中断处理中最少也是要禁止本中断线，有时候还会禁止全局中断，为了减少因为中断处理而遗漏掉的中断事件通常分成中断处理和中断下半部,通常就是我们说的软中断。    
The “softIRQ” system in the Linux kernel is a system that kernel uses to process work outside of the device driver IRQ context. In the case of network devices, the softIRQ system is responsible for processing incoming packets. The softIRQ system is initialized early during the boot process of the kernel.
对中断处理中可以推后处理的部分放在软中断中，总共有32个位置静态注册软中断，目前只有12个位置被使用，最最重要的部分就是为了网络收发数据。在内核启动过程中就回注册软中断且在每个cpu上开启了一个`ksoftirqd`线程在软中断事件过多的时候来集中延迟处理软中断。  
所以网卡驱动注册的时候干得一件非常重要的事就是注册中断处理，注册软中断处理程序。
The diagram above corresponds to the softIRQ section of our network blog post and shows the initializing of the softIRQ system and its per-CPU kernel threads.

The initialization of the softIRQ system is as follows:

    softIRQ kernel threads are created (one per CPU) in spawn_ksoftirqd in kernel/softirq.c with a call to smpboot_register_percpu_thread from kernel/smpboot.c. As seen in the code, the function run_ksoftirqd is listed as thread_fn, which is the function that will be executed in a loop.
    The ksoftirqd threads begin executing their processing loops in the run_ksoftirqd function.
    Next, the softnet_data structures are created, one per CPU. These structures hold references to important data structures for processing network data. One we’ll see again is the poll_list. The poll_list is where NAPI poll worker structures will be added by calls to napi_schedule or other NAPI APIs from device drivers.
    net_dev_init then registers the NET_RX_SOFTIRQ softirq with the softirq system by calling open_softirq, as shown here. The handler function that is registered is called net_rx_action. This is the function the softirq kernel threads will execute to process packets.

Steps 5 - 8 on the diagram relate to the arrival of data for processing and will be mentioned in the next section. Read on for more!
### 接收数据
![](../images/network-tuning-nic_dma.jpg)
Data arrives from the network!

When network data arrives at a NIC, the NIC will use DMA to write the packet data to RAM. In the case of the igb network driver, a ring buffer is setup in RAM that points to received packets. It is important to note that some NICs are “multiqueue” NICs, meaning that they can DMA incoming packets to one of many ring buffers in RAM. As we’ll see soon, such NICs are able to make use of multiple processors for processing incoming network data. Read more about multiqueue NICs. The diagram above shows just a single ring buffer for simplicity, but depending on the NIC you are using and your hardware settings you may have multiple queues on your system.

Read more detail about the process describe below in this section of the networking blog post.

Let’s walk through the process of receiving data:

    Data is received by the NIC from the network.
    The NIC uses DMA to write the network data to RAM.
    The NIC raises an IRQ.
    The device driver’s registered IRQ handler is executed.
    The IRQ is cleared on the NIC, so that it can generate IRQs for new packet arrivals.
    NAPI softIRQ poll loop is started with a call to napi_schedule.

The call to napi_schedule triggers the start of steps 5 - 8 in the previous diagram. As we’ll see, the NAPI softIRQ poll loop is started by simply flipping a bit in a bitfield and adding a structure to the poll_list for processing. No other work is done by napi_schedule and this is precisely how a driver defers processing to the softIRQ system.

Continuing on to the diagram in the previous section, using the numbers found there:
```
    The call to napi_schedule in the driver adds the driver's NAPI poll structure to the poll_list for the current CPU.
    The softirq pending bit is set so that the ksoftirqd process on this CPU knows that there are packets to process.
    run_ksoftirqd function (which is being run in a loop by the ksoftirq kernel thread) executes.
    __do_softirq is called which checks the pending bitfield, sees that a softIRQ is pending, and calls the handler registered for the pending softIRQ: net_rx_action which does all the heavy lifting for incoming network data processing.
```
It is important to note that the softIRQ kernel thread is executing net_rx_action, not the device driver IRQ handler.

### 开始处理网络数据
![](../images/network-tuning-net_rx_action.jpg)
Now, data processing begins. The net_rx_action function (called from the ksoftirqd kernel thread) will start to process any NAPI poll structures that have been added to the poll_list for the current CPU. Poll structures are added in two general cases:

    From device drivers with calls to napi_schedule.
    With an Inter-processor Interrupt in the case of Receive Packet Steering. Read more about how Receive Packet Steering uses IPIs to process packets.

We’re going to start by walking through what happens when a driver’s NAPI structure is retreived from the poll_list. (The next section how NAPI structures registered with IPIs for RPS work).

The diagram above is explained in depth here), but can be summarized as follows:

    net_rx_action loop starts by checking the NAPI poll list for NAPI structures.
    The budget and elapsed time are checked to ensure that the softIRQ will not monopolize CPU time.
    The registered poll function is called. In this case, the function igb_poll was registered by the igb driver.
    The driver’s poll function harvests packets from the ring buffer in RAM.
    Packets are handed over to napi_gro_receive, which will deal with possible Generic Receive Offloading.
    Packets are either held for GRO and the call chain ends here or packets are passed on to net_receive_skb to proceed up toward the protocol stacks.

We’ll see next how net_receive_skb deals with Receive Packet steering to distribute the packet processing load amongst multiple CPUs.

![](../images/network-tuning-netif_receive_skb.jpg)

Network data processing continues from netif_receive_skb, but the path of the data depends on whether or not Receive Packet Steering (RPS) is enabled or not. An “out of the box” Linux kernel will not have RPS enabled by default and it will need to be explicitly enabled and configured if you want to use it.

In the case where RPS is disabled, using the numbers in the above diagram:
```
    netif_receive_skb passes the data on to __netif_receive_core.

    __netif_receive_core delivers data to any taps (like PCAP).
    __netif_receive_core delivers data to registered protocol layer handlers. In many cases, this would be the ip_rcv function that the IPv4 protocol stack has registered.
```
In the case where RPS is enabled:
```
    netif_receive_skb passes the data on to enqueue_to_backlog.
    Packets are placed on a per-CPU input queue for processing.
    The remote CPU’s NAPI structure is added to that CPU’s poll_list and an IPI is queued which will trigger the softIRQ kernel thread on the remote CPU to wake-up if it is not running already.
    When the ksoftirqd kernel thread on the remote CPU runs, it follows the same pattern describe in the previous section, but this time, the registered poll function is process_backlog which harvests packets from the current CPU’s input queue.
    Packets are passed on toward __net_receive_skb_core.
    __netif_receive_core delivers data to any taps (like PCAP).
    __netif_receive_core delivers data to registered protocol layer handlers. In many cases, this would be the ip_rcv function that the IPv4 protocol stack has registered.
```
### 协议栈处理和socket接口
Next up are the protocol stacks, netfilter, berkley packet filters, and finally the userland socket. This code path is long, but linear and relatively straightforward.

You can continue following the detailed path for network data. A very brief, high level summary of the path is:

    Packets are received by the IPv4 protocol layer with ip_rcv.
    Netfilter and a routing optimization are performed.
    Data destined for the current system is delivered to higher-level protocol layers, like UDP.
    Packets are received by the UDP protocol layer with udp_rcv and are queued to the receive buffer of a userland socket by udp_queue_rcv_skb and sock_queue_rcv. Prior to queuing to the receive buffer, berkeley packet filters are processed.

Note that netfilter is consulted multiple times throughout this process. The exact locations can be found in our detailed walk-through.
## 网络调节
### 网卡相关
NIC驱动都实现了ethtool相关的方法,当然实现的多少和具体的硬件特性，驱动版本相关的，所以针对网卡调节的工具是ethtool  
网卡驱动可能有多个接收和发送队列，当有多个队列的时候可以多个DMA操作并行搬移数据，这样处理数据的速度肯定会提升;每个队列的大小也可以调节，可以通过增加队列的长度来容纳更多的网络数据；

网卡还可以将中断打包，传统的是来一个数据包就发送一个中断通知，频发打断CPU，可以设置中断发起的频率，从时间和空间上来设置，接收数据之后n时间如果再没有数据接收到就发起中断，超出多少了个数据包才发出中断。这两种角度的考量在脏页回写的调节上也有体现。时间和数据包设置的比较小时，延迟更小，但是影响CPU的效率

上面是增加网络数据的接收能力和中断触发速度，然后下面就是软中断处理过程。
## 软中断调节
在软中断处理时，如果过多的软中断占用CPU则会让用户程序完全得不到执行，系统响应就慢。软中断处理中有两种方式来将过多的软中断交给`ksoftirqd`来处理：  
1.时间上是否过长  
2.数据数量是否过多  
## sock的buffer
## Cache利用率
## RPS
RPS:Receive Packet Steering  
RPS启用单一NIC rx队列被几个CPU的softirq负载，加速从NIC rx队列中流出的速度，防止单一NIC rx队列
的网络流量瓶颈  
`/sys/class/net/ethX/queues/rx-N/rps_cpus`
## RFS
## 工具
