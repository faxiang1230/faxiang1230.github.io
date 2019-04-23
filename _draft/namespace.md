# namespace
namespace的作用是隔离，和它容易弄混的是cgroup，cgroup的作用是资源限制，还有一些cgroup类型是用来单纯的分组管理，没有进行资源限制,他们两个构成了容器的基础.
## pid namespace
```
#include<stdio.h>
#define _GNU_SOURCE
#include <linux/sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <signal.h>
#include <stdio.h>
#define STACK_SIZE (1024 * 1024)
#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)
static char child_stack[STACK_SIZE];
int child_main(void *arg)
{
	printf("child\n");
	execlp("/bin/bash","bash",NULL,NULL);
	return 1;
}
int main()
{
	pid_t child_pid;
   	child_pid = clone(child_main,child_stack+STACK_SIZE,SIGCHLD|CLONE_NEWPID,NULL);
   	if (child_pid == -1)
	   	errExit("clone");
   	wait(NULL);
   	return 0;
}
```
实验结果:
```
$ sudo ./a.out
child
# echo $$
1       --当前进程的pid是1
# exit
exit
$ echo $$
13256
```
## mount
```
#include<stdio.h>
#define _GNU_SOURCE
#include <linux/sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <signal.h>
#include <stdio.h>
#define STACK_SIZE (1024 * 1024)
#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)
static char child_stack[STACK_SIZE];
int child_main(void *arg)
{
	printf("child\n");
	system("mount -t proc proc /proc");
	execlp("/bin/bash","bash",NULL,NULL);
	return 1;
}
int main()
{
	pid_t child_pid;
   	child_pid = clone(child_main,child_stack+STACK_SIZE,SIGCHLD|CLONE_NEWPID|CLONE_NEWNS,NULL);
   	if (child_pid == -1)
	   	errExit("clone");
   	wait(NULL);
   	return 0;
}
```
实验结果:
```
$ sudo ./a.out
child
# pstree
bash───pstree
# echo $$
1
# ps aux        --我们对proc进行了隔离，所以新的进程看到的所有进程只有自己命名空间的
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  29792  5472 pts/10   S    09:54   0:00 bash
root        14  0.0  0.0  44432  3528 pts/10   R+   09:54   0:00 ps aux
```
## UTS
