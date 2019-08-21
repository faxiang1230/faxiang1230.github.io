# perf
## perf使用
perf被设计用来进行系统性能分析，而在他前面有ftrace框架了，为什么还造个新轮子呢?ftrace本身比较强大而且还有用户空间工具:kernerlshark,trace-cmd的支持，分别支持gui和cli模式，已经非常强大了，但是perf相比于ftrace，1.可以使用PMU来抓取系统活动，在不同架构上进行了封装，软件人员完全不需要了解PMU的细节就可以使用它采集数据.
利用ftrace的框架抓取数据后进行分析，它支持tracepoint以及kprobe事件，可以使用perf来完成ftrace事件的抓取，可以看作是trace-cmd的替代品
各种工具进行数据分析，它可以单独的record,之后进行report/script,内置的各种分析模式非常强大。
在进行系统分析时，通常都是从整体到局部的，逐渐聚焦问题,系统整体的性能情况可以通过atop/top/vmstat,brendangregg性能专家总结出的一副非常完备的地图,我们可以通过常用的系统工具查看cpu,内存，硬盘，网络来定位当前系统的瓶颈到底在哪:

![image](../images/linux_observability_tools.png)

![原图](www.brendangregg.com/Perf/linux_observability_tools.png)
上述工具大部分是通过系统提供的参数来查看的，相当于把系统当作一个黑盒，而深入的性能分析就需要深入到进程执行过程,通常是先从cpu分析开始。

### 剖析cpu
最常见的就是一个进程的cpu很高，可以通过perf的sample来
perf top -p xxx -v
有一些elf是没有debuginfo的，所以可能会只显示指令地址，没有对应的符号。
周期性对所有/特定的cpu进行采样，所以显示的百分比不是cpu利用率而是event占据所有的采样总量的百分比。可以使用top先定位出来具体的cpu繁忙程度，然后使用perf top -c xx -v -p yy来尽量缩减范围

还可以使用perf record/report
```
perf record -a -g -p xxxxx
perf report
```
### 剖析task
已经锁定到进程级别的时候，可以使用perf top和perf record/report,只是两者的采样的维度不一样。
### 调用栈


### 分析slab性能
你写了一个Module，使用kmalloc/kfree之类的接口或者使用专用的kmem_cache,你想要了解使用情况怎么样，不管在使用系统的Kmalloc或者私有的kemm_cache时都会根据cache line来作对齐，所以slab使用情况肯定会有一些浪费，可以通过perf kmem来了解slab的浪费情况。但是不能通过这个工具完成内存泄漏的分析。
perf kmem record
perf kmem stat --caller
还可以分析page的申请和释放情况:
perf kmem record --page
perf kmem stat --page
### perf sched
待完成
### perf lock
待完成
### perf probe
它基本上是封装了ftrace中的kprobe和uprobe接口，使用形式是非常相似的，但是相对于ftrace来说，更象是类似于trace-cmd的一种前端工具，而且提供了非常强大的分析和显示工具。

基于kprobes方式，使用方法和ftrace中基本是一样的，都是手动添加probe点，之后对其进行采集，定制格式化输出的方式基本和ftrace kprobe_event是一样的，参考内核文档`Documentation/trace/kprobetrace.txt`

## perf内部实现
### perf PMU
### perf tracepoint

## 为perf添加新的功能
## 为perf添加新的分析脚本
## 常见问题
- 问题1:perf找不到module的符号
当你自己编写module或者使用第三方非mainline中的Module时，在perf report是找不到他们的符号的，需要将module编译带有debuginfo并且放置到perf默认寻找的位置。

```
IN_TREE_DIR=/lib/modules/`uname -r`/kernel/modulename
mkdir -p $IN_TREE_DIR
cp modulename.ko $IN_TREE_DIR
depmod -a
```

## ref
1.www.brendangregg.com/perf.html
2.oliveryang.net/2016/07/linux-perf-tools-tips
