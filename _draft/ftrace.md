# ftrace
ftrace是function trace的简称，代表的是一个trace体系框架,其中包括:  
[image](../images/ftrace-frame.png)  
1.如何创建probe点  
2.probe之后处理函数  
3.probe中处理函数输出结果到ring buffer  
4.ring buffer如何向用户输出结果.  
其中1和2实现方式可能会不一样，但是ftrace都是通过debugfs向用户空间提供接口，其他的所有插件都利用了3和4的基础结构。  

主要的插件:  
event tracing,function trace,dynamic ftrace,kprobe,uprobe

主要用途:  
目前ftrace最经常被用来trace这也是它最初设计的功能，在新版的kernel中还被用用来热升级，也就是livepatch;还有一种方式可以被用来进行Hook,最后一种使用的比较少。

## tracepoint
trace point有很长时间的历史了，在3.10以前在内核中还有`samples/tracepoint`的使用示例，但是自从ftrace发展很成熟之后就不再推荐使用了，所以这个示例也被mainline删除了。

trace point就是开发人员手动地静态的在内核代码中插桩，之后可以向这个点注册处理函数。代码片段示例:

预备工作的宏:
```
#define DEFINE_TRACE_FN(name, reg, unreg)				\
	static const char __tpstrtab_##name[]				\
	__attribute__((section("__tracepoints_strings"))) = #name;	\
	struct tracepoint __tracepoint_##name				\
	__attribute__((section("__tracepoints"))) =			\
		{ __tpstrtab_##name, 0, reg, unreg, NULL };		\
	static struct tracepoint * const __tracepoint_ptr_##name __used	\
	__attribute__((section("__tracepoints_ptrs"))) =		\
		&__tracepoint_##name;

#define DEFINE_TRACE(name)						\
	DEFINE_TRACE_FN(name, NULL, NULL);

#define DECLARE_TRACE(name, proto, args)                               \
       extern struct tracepoint __tracepoint_##name;                   \
       static inline void trace_##name(proto)                          \
       {                                                               \
               if (unlikely(__tracepoint_##name.state))                \
                       __DO_TRACE(&__tracepoint_##name,                \
                               TP_PROTO(proto), TP_ARGS(args));        \
       }                                                               \
       static inline int register_trace_##name(void (*probe)(proto))   \
       {                                                               \
               return tracepoint_probe_register(#name, (void *)probe); \
       }                                                               \
       static inline int unregister_trace_##name(void (*probe)(proto)) \
       {                                                               \
               return tracepoint_probe_unregister(#name, (void *)probe);\
       }
```
使用上面定义的宏生成tracepoint相关的信息,DECLARE_TRACE宏展开之后就是:`register_trace_subsys_event`和`unregister_trace_subsys_event`和tracepoint的函数`trace_subsys_event`;而DEFINE_TRACE展开之后就是将tracepoint的信息：名字放入`__tracepoints_strings`section中，tracepoint放入`__tracpoints`中。如此在插桩的时候就是插入了以函数，注册时候就是向插桩的函数处注册callback.
```
DECLARE_TRACE(subsys_event,            //宏展开之后
	TP_PROTO(struct inode *inode, struct file *file),
	TP_ARGS(inode, file));

DEFINE_TRACE(subsys_event);
static int my_open(struct inode *inode, struct file *file)
{
       int i;

       trace_subsys_event(inode, file);           //DEFINE展开之后就是trace_subsys_event函数
       return -EPERM;
}
```
注册tracepoint的callback:
```
static void probe_subsys_event(void *ignore,
			       struct inode *inode, struct file *file)
{
	path_get(&file->f_path);
	dget(file->f_path.dentry);
	printk(KERN_INFO "Event is encountered with filename %s\n",
		file->f_path.dentry->d_name.name);
	dput(file->f_path.dentry);
	path_put(&file->f_path);
}
register_trace_subsys_event(probe_subsys_event, NULL);
```
## event trace
使用 tracepoint 需要自己实现probe函数，而probe函数通常就是打印一些信息，每次都自己实现比较麻烦，而且还需要一个单独的内核模块来进行注册，不是很方便，所以在tracepoint上封装出了一个接口，probe函数采用固定的范式，根据tracepoint中组装成不同的probe函数，而probe函数的功能就是将信息输出到ring buffer中。

这个接口虽然抽象的比较好，但由于它本身的复杂性使用起来仍然是比较复杂，所以使用起来仍然比较复杂，[lwn上专门有三篇文章](https://lwn.net/Articles/379903/)来讲解怎么使用它的:


## tracer
### function
### nop
### function_graph
## dynamic trace
## kprobe
kprobe 是很早前就存在于内核中的一种动态 trace 工具。kprobe 本身利用了 int 3（在 x86 中）实现了 probe 点，这对应于 A 部分的功能。使用 kprobe 需要用户自己实现 kernel module 来注册 probe 函数。可以看出 kprobe 并没有统一的 B、C 和 D。使用起来用户需要自己实现很多东西。不是很灵活。而在 function trace 出现后，kprobe 借用了它的一部分设计模式，实现了统一的 probe 函数（对应于图中的 B），并利用了 function trace 的环形缓存和用户接口部分，也就是 C 和 D 部分功能，用户可以使用读写 debugfs 中相关文件就可以控制 kprobe 的注册注销以及读取结果，非常方便。
## uprobe
## ringbuffer
## trace-cmd
## 问题
### 使用function_graph tracer崩溃问题
```
echo "function_graph" > /sys/kernel/debug/tracing/current_tracer
```
在上面的echo语句导致崩溃之后重启重启,使用kdump抓不到任何的crash.在vmware中报

   清除孤立的inode <inode>

我试图重现这个问题，将函数的current_tracer值replace成C程序中的其他东西：
```
include <stdio.h>                                                                 
#include <fcntl.h>                                                                 
#include <unistd.h>                                                                
#include <string.h>                                                                
#include <stdlib.h>                                                                
int openCurrentTracer() {                                                          
    int fd = open("/sys/kernel/debug/tracing/current_tracer", O_WRONLY);           
    if(fd < 0) exit(1); return fd;                                                 
}                                                                                  
int writeTracer(int fd, char* tracer) {                                            
    if(write(fd, tracer, strlen(tracer)) != strlen(tracer)) {                      
        printf("Failure writing %s\n", tracer); return 0;                          
    } return 1;                                                                    
}                                                                                  
int main(int argc, char* argv[]) {                                                 
    int fd = openCurrentTracer();                                                  
    char* blockTracer = "blk";                                                     
    if(!writeTracer(fd, blockTracer)) return 1; close(fd);                         
    fd = openCurrentTracer();                                                      
    char* graphTracer = "function_graph";                                          
    if(!writeTracer(fd, graphTracer))                                              
        return 1;                                                                  
    close(fd);                                                                     
    printf("Preparing to fail!\n");                                                
    fd = openCurrentTracer();                                                      
    if(!writeTracer(fd, blockTracer)) return 1;                                    
    close(fd);                                                                     
    return 0;                                                                      
}
```
奇怪的是，C程序不会崩溃我的系统。

我最初在使用Ubuntu（Unity环境）16.04 LTS时遇到了这个问题，并确认它是4.4.0和4.5.5内核中的一个问题。 我也在运行Ubuntu（Mate环境）15.10，在4.2.0和4.5.5内核上的机器上testing过这个问题，但是无法重现这个问题。 这只会让我更加困惑。

任何人都可以给我洞察发生了什么？ 具体来说，为什么我能够`write()`但不能`echo /sys/kernel/debug/tracing/ current_tracer`？

**更新**

正如vielmetti指出的，其他人也有类似的问题（如这里所见）。

>ftrace_disable_ftrace_graph_caller()假定它在jmp（e9）附近有一个5字节，则在ftrace_graph_call修改jmp指令。 然而，这是一个由2个字节组成的简短jmp（eb）。 而ftrace_stub()位于ftrace_graph_caller正下方，所以上面的修改会破坏ftrace_stub()中的内核oops的指令，如下所示的无效操作码：

补丁（如下所示）解决了echo问题，但我仍然不明白为什么echo先前打破了write()不是。

>https://lkml.org/lkml/2016/5/16/493
>http://zgserver.com/ftraceechofunction_graphcurrent_tracer.html

## 参考
https://www.ibm.com/developerworks/cn/linux/1609_houp_ftrace/  
https://static.lwn.net/kerneldoc/trace/ftrace-uses.html  
https://lwn.net/Articles/379903/  
https://lwn.net/Articles/381064/  
https://lwn.net/Articles/383362/  
