# TCP问题
## TIME_WAIT
## MSS,SNDBUF,
1、MSS(Max Segment Size) 是TCP数据包每次能够传输的最大数据分段，其中并不包括TCP首部。
而且MSS只出现在syn报文段中。一般来说，MSS的值在不分段的情况会越大越好，比如一个外出接口的MSS值
是MTU减去IP和TCP首部长度。

2、窗口大小(Window Scaling)是个动态的值，因为TCP是用的滑动窗口协议，传输数据的速率都是根据窗口大小来调整的。可以
把窗口理解为一个缓存，而且窗口大小跟MSS是没有任何关系的。

3、窗口是为了控制传输过程中的速度。而MSS只是控制TCP报文段大小。

4.SNDBUF和RECVBUF,这两个纯粹是linux实现

非阻塞有什么用呢?目前我们我看到的场景都是阻塞的，默认也是阻塞的，那么为什么保留非阻塞呢？那些场景需要使用非阻塞？怎么使用？