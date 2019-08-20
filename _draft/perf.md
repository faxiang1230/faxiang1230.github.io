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
perf top -p xxx
还可以使用perf record/report
```
perf record -a -g -p xxxxx
perf report
```
### 剖析task
### 调用栈
- 问题1:perf找不到module的符号
```
IN_TREE_DIR=/lib/modules/`uname -r`/kernel/modulename
mkdir -p $IN_TREE_DIR
cp modulename.ko $IN_TREE_DIR
depmod -a
```
## perf内部实现
### perf PMU
### perf tracepoint

## 为perf添加新的功能

## ref
1.www.brendangregg.com/perf.html
2.oliveryang.net/2016/07/linux-perf-tools-tips
