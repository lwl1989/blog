CreateTime:2019-05-15 23:42:38.0

## 线程

个人理解：线程是一种运行单元，是进程内的拆分（就像一个房子，里面可以有电视在播放、人在吃饭），不同的线程可以同时干不同的事情。

其他：

    进程：进程（Process）是计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。

    协程：协程是比线程的更小的一种程序的拆分，和线程相比：
        线程有自己的上下文，切换受系统控制（系统调度【cpu切换 用户态 系统态（涉及锁）】）
        协程也有自己的上下文，但是其切换是由自己控制，由当前协程切换到其他协程由当前协程来控制。

为什么要引入这么多种奇奇怪怪的东西呢？

鄙人愚见： 从操作系统的设计原理来讲，一个cpu的核心只能同一时刻运行一个运算。但是，我们现在的cpu已经不是远古时代的cpu了（很久前就不是了）,因此是可以同时运行多个程序的。
但是，总不能我遇到一个请求我就将其新建一个进程吧（定义里说了，进程是基本单位），这样会消耗大量的资源和时间（当然也有这么处理的，比如php原本的cgi处理方式）。
因此，为了能更高效的利用cpu，产生了线程和协程的概念（和异步不一样，异步是为了解决耗时而产生的概念）。
这里要谈论到一个关键字【并行】，并行是指多个cpu实例或者多台机器同时执行一段处理逻辑，是真正的同时（同一时刻）。
并行和并发的区别，百度很容易找到。

举个栗子：
    假若cpu是1u1核，你搞N个线程还不如你搞串行执行快。
    假如cpu是2u2核，搞4个线程执行，可能程序会提高  无限接近 4 倍的效率，为什么不等于3呢，因为调度会消耗额外的时间。
    为什么栗子是4个线程，如果所有的任务都是计算密集型的，则创建的多线程数 = 处理器核心数就可以了
    如果io操作比较耗时，则根据具体情况调整线程数，此时 多线程数 = n*处理器核心数

[如何配置线程数量](https://blog.csdn.net/coslay/article/details/42062639?utm_source=blogxgwz5)

### java线程原理

> Java 给多线程编程提供了内置的支持。 一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。多线程是多任务的一种特别的形式，但多线程使用了更小的资源开销。这里定义和线程相关的另一个术语 - 进程：一个进程包括由操作系统分配的内存空间，包含一个或多个线程。一个线程不能独立的存在，它必须是进程的一部分。一个进程一直运行，直到所有的非守护线程都结束运行后才能结束。多线程能满足程序员编写高效率的程序来达到充分利用 CPU 的目的。

如图：

![](http://www.runoob.com/wp-content/uploads/2014/01/java-thread.jpg)


### java线程状态

新建状态: 使用 new 关键字和 Thread 类或其子类建立一个线程对象后，该线程对象就处于新建状态。它保持这个状态直到程序 start() 这个线程。

就绪状态: 当线程对象调用了start()方法之后，该线程就进入就绪状态。就绪状态的线程处于就绪队列中，要等待JVM里线程调度器的调度。

运行状态: 如果就绪状态的线程获取 CPU 资源，就可以执行 run()，此时线程便处于运行状态。处于运行状态的线程最为复杂，它可以变为阻塞状态、就绪状态和死亡状态。

阻塞状态: 如果一个线程执行了sleep（睡眠）、suspend（挂起）等方法，失去所占用资源之后，该线程就从运行状态进入阻塞状态。在睡眠时间已到或获得设备资源后可以重新进入就绪状态。可以分为三种：

等待阻塞：运行状态中的线程执行 wait() 方法，使线程进入到等待阻塞状态。

同步阻塞：线程在获取 synchronized 同步锁失败(因为同步锁被其他线程占用)。

其他阻塞：通过调用线程的 sleep() 或 join() 发出了 I/O 请求时，线程就会进入到阻塞状态。当sleep() 状态超时，join() 等待线程终止或超时，或者 I/O 处理完毕，线程重新转入就绪状态。

死亡状态: 一个运行状态的线程完成任务或者其他终止条件发生时，该线程就切换到终止状态。

### java线程实现方式

1. 通过实现 Runnable 接口；
2. 通过继承 Thread 类本身；
3. 通过 Callable 和 Future 创建线程。


### 优先级和锁

#### 优先级

java可以对线程设置优先级，示例如下：

    threadObj.setPriority(int level)

1. 在任意时刻，当有多个线程处于可运行状态时，运行系统总是挑选一个优先级最高的线程执行，只有当线程停止、退出或者由于某些原因不执行的时候，低优先级的线程才可能被执行

2. 两个优先级相同的线程同时等待执行时，那么运行系统会以round-robin的方式选择一个线程执行（即轮询调度，以该算法所定的）（Java的优先级策略是抢占式调度！）

3. 被选中的线程可因为一下原因退出，而给其他线程执行的机会：

    一个更高优先级的线程处于可运行状态（Runnable）
    线程主动退出（yield），或它的run方法结束
    在支持分时方式的系统上，分配给该线程的时间片结束

4. Java运行系统的线程调度算法是抢占式（preemptive）的，当更高优先级的线程出现并处于Runnable状态时，运行系统将选择高优先级的线程执行

5. 例外地，当高优先级的线程处于阻塞状态且CPU处于空闲时，低优先级的线程也会被调度执行


#### yield()和join()方法

为什么这两方法单独拿出来学习呢，因为这两方法涉及的方面比较复杂。

##### yield 

理论上，yield意味着放手，放弃，投降。一个调用yield()方法的线程告诉虚拟机它乐意让其他线程占用自己的位置。
这表明该线程没有在做一些紧急的事情。注意，这仅是一个暗示，并不能保证不会产生任何影响。

源码中定义
```
/**
  * A hint to the scheduler that the current thread is willing to yield its current use of a processor. The scheduler is free to ignore
  * this hint. Yield is a heuristic attempt to improve relative progression between threads that would otherwise over-utilize a CPU.
  * Its use should be combined with detailed profiling and benchmarking to ensure that it actually has the desired effect.
  */
```

就是说，我用了yield也不一定会释放使用权，只是为了减轻线程间竞争而采取的优化措施。

yield的几个关键点:

- Yield是一个静态的原生(native)方法
- Yield告诉当前正在执行的线程把运行机会交给线程池中拥有相同优先级的线程。
- Yield不能保证使得当前正在运行的线程迅速转换到可运行的状态
- 它仅能使一个线程从运行状态转到可运行状态，而不是等待或阻塞状态

##### join

> join() method suspends the execution of the calling thread until the object called finishes its execution.

也就是说，t.join()方法阻塞调用此方法的线程(calling thread)，直到线程t完成，此线程再继续；通常用于在main()主线程内，等待其它线程完成再结束main()主线程。

```
    /**
     *  Waits at most <code>millis</code> milliseconds for this thread to  
     * die. A timeout of <code>0</code> means to wait forever.    
     */
    //此处A timeout of 0 means to wait forever 字面意思是永远等待，其实是等到t结束后。
    public final synchronized void join(long millis)    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

可以看出，Join方法实现是通过wait（小提示：Object 提供的方法）。
当main线程调用t.join时候，main线程会获得线程对象t的锁（wait 意味着拿到该对象的锁),调用该对象的wait(等待时间)，直到该对象唤醒main线程 ，比如退出后。
这就意味着main 线程调用t.join时，必须能够拿到线程t对象的锁。

#### 锁

锁的具体讲解在我这两边学习笔记里

[java锁学习（一）](https://my.oschina.net/lwl1989/blog/3049066)

[java锁学习（二）](https://my.oschina.net/lwl1989/blog/3049584)


### 所有测试代码

[测试代码](https://github.com/lwl1989/javaTest/tree/master/src/Thread)