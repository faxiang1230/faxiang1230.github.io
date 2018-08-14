https://blog.csdn.net/dog250/article/details/6451989
杀死linux的僵尸进程

linux并不把进程的树形结构导出给普通用户，然而在内核中，它却使用树形结构来管理进程。linux内核使用“子进程退出，父进程收尸，父进程退出，子进程被过继” 这种方式来管理进程的死亡，然而却少了一种，那就是父进程不给子进程收尸的情况 ，这就是僵尸进程的原因。
     既然知道了僵尸进程为何产生，那么想干掉它们就简单了。记住：任何没有人为因素的纯技术问题都是可以解决的！如何操作呢？很简单，就三步：
1.将僵尸进程从树形进程组织中摘除；
2.将僵尸进程过继给一个特定的进程；
3.该特定进程调用wait来回收掉它。
这三步岂不是很麻烦，直接干掉它的父进程不就得了，这样内核会自己将僵尸进程过继给别的进程或者init进程，然而有时我们不能这么做，如果它的父进程是个很重要的进程咋办，我们不能因为父辈抛弃了过早去世的孩子而责怪父亲，如果那样，linux内核的法律岂不是比我们还严重...既然父亲不要孩子了，那么建立一个收容所是必要的，使用上述三个步骤完成子进程空壳的过继和回收！这个收容所可以在内核空间也可以在用户空间，这不是最重要的。本文给出了一个预研例子：
1.首先给出一个用户态进程代码：
[cpp] view plain copy

    #include <unistd.h>  
    int main()  
    {  
            int pid = 0;  
             pid = fork();  
             if (pid == 0) { //子进程将瞬间变成僵尸，因为：1.父进程不回收；2.父进程不忽略  
             } else {  
                    while (1) {  
                //I'm VIP,though I am always sleeping!  
                            sleep(1);  
                    }  
             }  
    }  


2.然后给出一个内核模块代码：
[cpp] view plain copy

    unsigned long pid; //参数保存结束的僵尸进程的进程号  
    module_param(pid, long, S_IRUSR);   
    MODULE_PARM_DESC(pid, "pid");   
    struct task_struct *(*find)(struct pid *pid, enum pid_type type);  
    struct pid *(*get)(pid_t nr);  
    long (*wait1)(pid_t pid, void *v, int options, void *ru);  
    int __init rm_init(void){   
        find = 0xc1041aed;   //根据pid结构得到task_t函数的地址  
            get=0xc1041b81;         //根据pid得到pid结构体函数的地址  
        wait1 = 0xc1032e02;  
            struct pid* spid = (*get)(pid);  
        struct task_struct *tsk = (*find)(spid, PIDTYPE_PID);  
        tsk->real_parent = current;  
        tsk->parent = current;  
        list_del(&tsk->sibling);  
        list_add_tail(&tsk->sibling, &tsk->real_parent->children);  
        (*wait1)(pid, NULL, 0, NULL);  
        return 0;   
    }   
    void __exit rm_exit(void){   
    }   
    module_init(rm_init);   
    module_exit(rm_exit);   
    MODULE_LICENSE("GPL");  


上述的模块实现了僵尸进程的回收，虽然还不是很完美，然而起码证实了可行性，我们一些函数的地址还是通过procfs得到的。具体在代码润色方面，我有四个建议，这四个方式无论哪一个都是可行的，而且花不了太多时间，这里代码就从略了，如果写一下的话，充其量也只能锻炼一下c语言编程能力：
1.实现一个内核线程，专门实现模块init函数的逻辑，需要干掉的僵尸进程号通过procfs传入内核，然后在write例程中唤醒回收僵尸进程的内核线程；
2.实现一个用户态进程U，挂载一个信号A的处理函数，内部实现waitpid，通过procfs传入或者通过netlink传入内核的僵尸进程号代表的进程过继给用户态进程U，然后向U发送信号A；
3./dev/mem的机器码编程或者直接释放僵尸进程的task_t。

4.在/proc/<pid>/目录中加入kill-if-jiangshi文件，写入1如果该进程是僵尸，那么就调用上述模块的逻辑杀死它

 

     是不是linux应该提供一个系统调用，用于过继任何进程呢？不！那样就会搞乱整个系统，linux并不想把树形结构导出给用户！因此在模块中必须判断，需要结束的进程的status是“僵尸”！

