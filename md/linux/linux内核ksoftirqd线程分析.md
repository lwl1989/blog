CreateTime:2021-10-13 16:47:19.0

# 系统现象
我们执行ps会发现一个叫ksoftirqd的线程
```
 ps -ef | grep ksoftirqd
root         6     2  0 8月19 ?       00:01:15 [ksoftirqd/0]
root        14     2  0 8月19 ?       00:01:13 [ksoftirqd/1]
root        20     2  0 8月19 ?       00:00:51 [ksoftirqd/2]
root        26     2  0 8月19 ?       00:00:56 [ksoftirqd/3]
root        31     2  0 8月19 ?       00:00:58 [ksoftirqd/4]
root        36     2  0 8月19 ?       00:00:56 [ksoftirqd/5]
root        41     2  0 8月19 ?       00:01:01 [ksoftirqd/6]
root        46     2  0 8月19 ?       00:01:00 [ksoftirqd/7]
root        51     2  0 8月19 ?       00:00:46 [ksoftirqd/8]
root        56     2  0 8月19 ?       00:00:31 [ksoftirqd/9]
root        61     2  0 8月19 ?       00:00:57 [ksoftirqd/10]
root        66     2  0 8月19 ?       00:01:20 [ksoftirqd/11]
root        71     2  0 8月19 ?       00:01:29 [ksoftirqd/12]
root        76     2  0 8月19 ?       00:00:58 [ksoftirqd/13]
root        81     2  0 8月19 ?       00:00:49 [ksoftirqd/14]
root        86     2  0 8月19 ?       00:00:48 [ksoftirqd/15]
```

# 软中断：

从上面的结果可以看出，每个处理器都有一个这样的线程。所有的线程的名字都叫做ksoftirq/n，区别在于n，他对应的是处理器的编号。

这个线程就是专门用于处理软中断的程序。

软中断：

UN1X系统提供软中断机制作为进程通信的一种手段。软中断是通过发送规定的信号到指定进程，对方进程定时地查询有无外来信号，若有则按约定进行处理，处理完毕，返回断点继续执行原来的指令。可见，软中断是对硬中断的一种模拟。软中断存在较大的时延，不象硬中断能获得及时响应。例如，对方进程若处在阻塞队列，那么只有等到该进程执行时才能查询软中断信号。显然，从软中断信号发出到对方响应，时间可能拖得很长。此外，**软中断处理程序运行在用户态，硬中断处理程序则运行在核心态**。

优先级：

	中断>软中断>用户进行

对于软中断，内核会在几个特殊的时机执行（注意执行和调度的区别，调度软中断只是对软中断打上待执行的标记，并没有真正执行），而在中断处理程序返回时处理是最常见的。

# 代码分析

在<Softirq.c(kernel)>中
```c
static int ksoftirqd(void * __bind_cpu)
{
    set_user_nice(current, 19);
    current->flags |= PF_NOFREEZE;

    set_current_state(TASK_INTERRUPTIBLE);  //处于等待队伍中，等待资源有效时唤醒（比方等待键盘输入、socket连接、信号等等）

    while (!kthread_should_stop()) {
        preempt_disable();
        if (!local_softirq_pending()) {
            preempt_enable_no_resched();
            schedule();
            preempt_disable();
        }

        set_current_state(TASK_RUNNING); //正在运行或处于就绪状态

        while (local_softirq_pending()) {  //当有软中断未解决
            // cpu正常可用
            if (cpu_is_offline((long)__bind_cpu))
                goto wait_to_die;
            do_softirq();  //执行软中断  这里一般执行的就是 注册的回调函数,比如epoll
            preempt_enable_no_resched();  //抢占cpu计数-1 但不立即抢占式调度
            cond_resched();  //在内核态主动让出cpu可以调用cond_resched,让出cpu，此时就是等待调度
            preempt_disable();   //cpu抢占计数+1  就是当cpu重新调度到此处时，提高计数
			/**
			当从内核态返回到用户态的时候，要检查是否进行调度，而调度要看两个条件：
			1. preempt_count是否为0
			2. rescheduled是否置位
			因此上方让出cpu之前先将preempt_count-1（否则cond_resched可能失败），调度到ksoftirq又加回来
         */
        }
        preempt_enable();
        set_current_state(TASK_INTERRUPTIBLE); //执行完一次   标记线程进入等待队列
    }
    set_current_state(TASK_RUNNING); //这里是为了执行之后的处理 保证本线程正常终止
    return 0;

wait_to_die:
    preempt_enable();
    /* Wait for kthread_stop */
    set_current_state(TASK_INTERRUPTIBLE);
	//kthread_should_stop()返回should_stop标志。它用于创建的线程检查结束标志,并决定是否退出
    while (!kthread_should_stop()) {  //只要不接受到结束信号，就继续进行调度
        schedule();
        set_current_state(TASK_INTERRUPTIBLE);
    }
    set_current_state(TASK_RUNNING); //这里是为了执行之后的处理 保证本线程正常终止
    return 0;
}
```

# 小知识

### cpu抢占

在中断或临界区代码中，线程需要关闭内核抢占，因此，互斥机制（如：自旋锁（spinlock）、RCU等）、中断代码、链表数据遍历等需要关闭内核抢占，临界代码运行完时，需要开启内核抢占。关闭/开启内核抢占需要使用内核抢占API函数preempt_disable和 preempt_enable。

关于抢占cpu的几个函数认知
```
内核抢占API函数说明如下（在include/linux/preempt.h中）：
preempt_enable() //内核抢占计数preempt_count减1
preempt_disable() //内核抢占计数preempt_count加1
preempt_enable_no_resched()　 //内核抢占计数preempt_count减1，但不立即抢占式调度
preempt_check_resched () //如果必要进行调度
preempt_count() //返回抢占计数
preempt_schedule() //核抢占时的调度程序的入口点
```

### 线程状态

看完这段代码，要了解一下linux线程的状态：

- TASK_RUNNING：正在运行或处于就绪状态：就绪状态是指进程申请到了CPU以外的其它全部资源。正所谓：万事俱备，仅仅欠东风.提醒：一般的操作系统教科书将正在CPU上运行的进程定义为RUNNING状态、而将可运行可是尚未被调度运行的进程定义为READY状态。这两种状态在Linux下统一为 TASK_RUNNING状态.
- TASK_INTERRUPTIBLE：处于等待队伍中，等待资源有效时唤醒（比方等待键盘输入、socket连接、信号等等），但能够被中断唤醒.普通情况下，进程列表中的绝大多数进程都处于TASK_INTERRUPTIBLE状态.毕竟皇帝仅仅有一个（单个CPU时），后宫佳丽几千；假设不是绝大多数进程都在睡眠，CPU又怎么响应得过来.
- TASK_UNINTERRUPTIBLE：处于等待队伍中，等待资源有效时唤醒（比方等待键盘输入、socket连接、信号等等），但不能够被中断唤醒.
- TASK_ZOMBIE:僵死状态。进程资源用户空间被释放，但内核中的进程PCB并没有释放。等待父进程回收.
- TASK_STOPPED:进程被外部程序暂停（如收到SIGSTOP信号，进程会进入到TASK_STOPPED状态），当再次同意时继续运行（进程收到SIGCONT信号，进入TASK_RUNNING状态）。


### ksoftirqd创建过程

inux/init/main.c（do_pre_smp_initcalls）-> linux/kernel/softirq.c（spawn_ksoftirqd）->  linux/kernel/softirq.c（cpu_callback）

[创建过程](https://blog.51cto.com/yorrick/1216060 "创建过程")