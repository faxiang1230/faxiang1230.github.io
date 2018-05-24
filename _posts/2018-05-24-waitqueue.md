---
layout:     post
title:      "waitqueue的简单工作过程"
date:       2018-05-23 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - kernel
---
# waitqueue的简单工作过程  
和waitqueue长的比较像的是workqueue,一个是等待队列，一个是工作队列。
workqueue是中断下半部的一种机制，将上半部不方便做的事情交给下半部，和软中断、tasklet不同的是workqueue属于进程上下文，可以睡眠。
如果我做，我就把函数指针挂到一个链表中，然后通知线程来这里去取;线程遍历链表然后挨个执行函数指针.  
而waitqueue是任务等待作用，本身就在进程上下文，暂时不需要占用CPU了，然后从runqueue中移除。还得需要再次唤醒啊。简单做法就是维护一个restqueue(休息队列)，唤醒的时候挨个唤醒。不过每次唤醒都得遍历restqueue,同时大部分任务平时都是在等待状态即位于restqueue中，所以遍历的开销有点大啊。
所以内核创建了多个链表，任务可以挂在单独的链表上。特定资源ready之后可以单独唤醒某个链表上的任务，唤醒的过程就是从链表中删除自己，更改自身状态，尝试回归到runqueue中。
## INTERRUPTIBLE和UNINTERRUPTIBLE和超时
INTERRUPTIBLE和UNINTERRUPTIBLE的区别大家都知道，一个是可以被信号唤醒，一个只能在特定资源唤醒，说白了就是在被唤醒的时候可中断睡眠的任务会同时检查是否有待处理信号和等待资源是否就位了，而不可中断睡眠的任务只会检查等待资源的状态。
什么时候才不会响应信号呢?
正在干一件耗时的事，例如IO操作，本来已经完成了99%,被信号打断了，我很恼火，干脆就不理会信号了  
从内核的使用场景上来看wait_event在IO和文件系统中使用的最为频繁，当我在进行IO的时候，需要处理信号，
而信号处理函数还需要从磁盘中load进来，在进行这个IO的时候又被打断了，场景过于复杂了，干脆就不响应信号了，专心等IO
## 处于不可中断睡眠的进程怎么处理
处于不可中断睡眠的进程不响应一切的信号，包括`SIGKILL`和`SIGSTOP`,怎么破?  
一种是预防,就是进入不可中断睡眠的进程使用等待超时
另外有人提出写一个内核模块根据传入的pid修改对应的task的状态为TASK_INTERRUPTIBLE。
不过只修改状态应该是不够的，信号的处理必须在进程重新被调度，从系统调用返回用户空间的时候才会处理，而进程在进入睡眠之前如果使用的类似于wait_event这样的方式，
它只检查资源是否就位,如果不是的话接着又睡了，根本不会被调度。所以这种方式理论上是不满足的。
## waitqueue代码赏析
可中断睡眠的进入和唤醒
```
#define wait_event_interruptible(wq_head, condition)				\
({										\
	int __ret = 0;								\
	might_sleep();								\            //在禁止休眠的上下文中进入睡眠会报一些警告信息
	if (!(condition))							\            //在真正调用schedule让出CPU前总是不甘心让出来，有机会就看看能不能重登宝座
		__ret = __wait_event_interruptible(wq_head, condition);		\
	__ret;									\
})

#define __wait_event_interruptible(wq_head, condition)				\
	___wait_event(wq_head, condition, TASK_INTERRUPTIBLE, 0, 0,		\
		      schedule())

#define ___wait_event(wq_head, condition, state, exclusive, ret, cmd)		\
({										\
	__label__ __out;							\
	struct wait_queue_entry __wq_entry;					\
	long __ret = ret;	/* explicit shadow */				\
										\
	init_wait_entry(&__wq_entry, exclusive ? WQ_FLAG_EXCLUSIVE : 0);	\
	for (;;) {								\
		long __int = prepare_to_wait_event(&wq_head, &__wq_entry, state);\　//再次检查信号状态，结合自己的状态来决定是否响应信号;之后决定是否将自己挂入到waitqueue链表上
										\
		if (condition)							\　　　　　　　　//检查资源就位标志,如果第一次睡眠之前没有
			break;							\
										\
		if (___wait_is_interruptible(state) && __int) {			\         //如果有待决信号并且自身也是可中断睡眠，然后就直接跳出去了;因为子prepare阶段发现有待决信号，就没加入到waitqueue链表上
			__ret = __int;						\
			goto __out;						\
		}								\
										\
		cmd;								\　　　　　　　　　　　　//调用schedule让出CPU，轮回
	}									\
	finish_wait(&wq_head, &__wq_entry);					\ //资源就位时就需要手动设置进程的状态，从waitqueue链表上删除自身
__out:	__ret;									\
})

void init_wait_entry(struct wait_queue_entry *wq_entry, int flags)
{
	wq_entry->flags = flags;
	wq_entry->private = current;
	wq_entry->func = autoremove_wake_function;   //总共做两件事，一个是唤醒这个任务，一个是从waitqueue中删除自己
	INIT_LIST_HEAD(&wq_entry->entry);
}
```
如果真的睡眠了，唤醒的时候waitqueue直接就维护了任务的状态和将自己从waitqueue链表上删除，还需要加入到runqueue上。
所以唤醒的时候通常要做两件事，一件是设置资源状态，另外一个是加入到runqueue中，自此形成一个闭环。
```
#define wake_up_interruptible(x)	__wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)
* changing the task state if and only if any tasks are woken up.
*/
void __wake_up(struct wait_queue_head *wq_head, unsigned int mode,
     int nr_exclusive, void *key)
{
 unsigned long flags;

 spin_lock_irqsave(&wq_head->lock, flags);
 __wake_up_common(wq_head, mode, nr_exclusive, 0, key);
 spin_unlock_irqrestore(&wq_head->lock, flags);
}
static void __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode,
			int nr_exclusive, int wake_flags, void *key)
{
	wait_queue_entry_t *curr, *next;

	list_for_each_entry_safe(curr, next, &wq_head->head, entry) {
		unsigned flags = curr->flags;
		int ret = curr->func(curr, mode, wake_flags, key);  //调用autoremove_wake_function，唤醒任务之后将自己从waitqueue中删除
	}
}
```
不可中断的睡眠，和可中断睡眠都是一样的，只不过里面不用理会信号
```
#define ___wait_event(wq_head, condition, state, exclusive, ret, cmd)		\
({										\
	__label__ __out;							\
	struct wait_queue_entry __wq_entry;					\
	long __ret = ret;	/* explicit shadow */				\
										\
	init_wait_entry(&__wq_entry, exclusive ? WQ_FLAG_EXCLUSIVE : 0);	\
	for (;;) {								\
		long __int = prepare_to_wait_event(&wq_head, &__wq_entry, state);\  //不理会信号，直接先加到waitqueue上
										\
		if (condition)							\　　　　　　　　　　　　　　　　　　　　　　　　//满足条件就
			break;							\
		cmd;								\　　　　　　　　　　　　　　　//调用schedule来让出CPU
	}									\
	finish_wait(&wq_head, &__wq_entry);					\        //资源就位时就需要手动设置进程的状态，从waitqueue链表上删除自身
__out:	__ret;									\
})
```
唤醒的操作和可中断睡眠没啥区别,同样都是唤醒任务加入到runqueue中
