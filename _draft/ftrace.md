# ftrace
## function_graph
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
