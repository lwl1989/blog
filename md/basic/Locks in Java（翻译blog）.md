CreateTime:2015-10-08 13:28:01.0


###翻译自 [Jakob Jenkov](http://tutorials.jenkov.com/java-concurrency/locks.html#simple-lock

> 
 from Java 5 the package java.util.concurrent.locks contains several lock implementations, so you may not have to implement your own locks. But you will still need to know how to use them, and it can still be useful to know the theory behind their implementation. For more details, see my tutorial on the java.util.concurrent.locks.Lock interface.
> 

//JAVA5以来，在java.util.concurrent.locks包中提供了若干种锁的实现，所以所以你可能不必实现你自己的锁。但是你需要知道如何使用它们，并且知道实现后面的原理。通过下面的代码块，看我关于锁的教程。



> A Simple Lock
> 
> Let's start out by looking at a synchronized block of Java code:

一个简单的锁
让我们先来看看一个同步的代码块
```
public class Counter{

  private int count = 0;

  public int inc(){
    synchronized(this){
      return ++count;
    }
  }
}
```
> Notice the synchronized(this) block in the inc() method. This block makes sure that only one thread can execute the return ++count at a time. The code in the synchronized block could have been more advanced, but the simple ++count suffices to get the point across.

这里是一个自增的同步代码块，每当执行一个线程时，返回++count。这个代码快可以写得更完善，但是用于++count计算已经完全足够了。



>The Counter class could have been written like this instead, using a Lock instead of a synchronized block:

这个计数器类可能锁下面这样写的，使用一个锁替代同步代码块；
```
public class Counter{

  private Lock lock = new Lock();
  private int count = 0;

  public int inc(){
    lock.lock();
    int newCount = ++count;
    lock.unlock();
    return newCount;
  }
}
```

>The lock() method locks the Lock instance so that all threads calling lock() are blocked until unlock() is executed.
这个锁的方法lock() 能让实例lock阻塞直到unlock()方法被调用

>Here is a simple Lock implementation:
下面是一个简单的锁的实现：

```
public class Lock{

  private boolean isLocked = false;

  public synchronized void lock()
  throws InterruptedException{
    while(isLocked){
      wait();
    }
    isLocked = true;
  }

  public synchronized void unlock(){
    isLocked = false;
    notify();
  }
}
```
>Notice the while(isLocked) loop, which is also called a "spin lock". Spin locks and the methods wait() and notify() are covered in more detail in the text Thread Signaling. While isLocked is true, the thread calling lock() is parked waiting in the wait() call. In case the thread should return unexpectedly from the wait() call without having received a notify() call (AKA a Spurious Wakeup) the thread re-checks the isLocked condition to see if it is safe to proceed or not, rather than just assume that being awakened means it is safe to proceed. If isLocked is false, the thread exits the while(isLocked) loop, and sets isLocked back to true, to lock the Lock instance for other threads calling lock().

在循环while(isLocked)中,也被称为“自旋锁”,自旋锁和方法wait()、方法notify() 中有线程实现的细节。当islocked为true时，线程访问lock()方法停在wait()处。如果线程一直没有收到来自notify（）的唤醒信号，线程重新检查锁条件是否安全的进行与否，而不是直接将其唤醒。如果islocked为false，线程推出循环，并且设置islocked为true，将其返回，然后在其他线程调用LOCK的实例。

>When the thread is done with the code in the critical section (the code between lock() and unlock()), the thread calls unlock(). Executing unlock() sets isLocked back to false, and notifies (awakens) one of the threads waiting in the wait() call in the lock() method, if any.

当线程在临界区的代码完成时（在lock()和unlock()之间），将访问unlock(),执行unlock() 设置 isLocked为false并返回，唤醒一个正在等待的线程执行lock()方法（如果有的话）.


###Lock Reentrance
###锁折返
>Synchronized blocks in Java are reentrant. This means, that if a Java thread enters a synchronized block of code, and thereby take the lock on the monitor object the block is synchronized on, the thread can enter other Java code blocks synchronized on the same monitor object. Here is an example:

同步代码块在java里时折返。意思锁： 如果java线程中进入了一个同步代码块,从而锁住了一个监控的对象，这个线程能在统一个监控对象中输入其他的java代码块。
下面是一个例子：

```
public class Reentrant{

  public synchronized outer(){
    inner();
  }

  public synchronized inner(){
    //do something
  }
}
```

>Notice how both outer() and inner() are declared synchronized, which in Java is equivalent to a synchronized(this) block. If a thread calls outer() there is no problem calling inner() from inside outer(), since both methods (or blocks) are synchronized on the same monitor object ("this"). If a thread already holds the lock on a monitor object, it has access to all blocks synchronized on the same monitor object. This is called reentrance. The thread can reenter any block of code for which it already holds the lock.

请注意outer()和inner()声明Java同步，这相当于一个同步块（本）。如果一个线程调用outer()有从内outer()调用inner()没有问题，因为这两种方法（或块）同步，在同一监测对象（“自己”）。如果一个线程已将锁放在监视器对象上，它可以在同一监视对象上同步访问所有的块。这就是所谓的折返。线程可以进入任何代码块，它已经持有的锁。

>The lock implementation shown earlier is not reentrant. If we rewrite the Reentrant class like below, the thread calling outer() will be blocked inside the lock.lock() in the inner() method.

这个锁的是不可折返的。如果我们改写Reentrant类像下面这样，线程调用OUT方法将被inner方法中的锁阻挡住；
```
public class Reentrant2{

  Lock lock = new Lock();

  public outer(){
    lock.lock();
    inner();
    lock.unlock();
  }

  public synchronized inner(){
    lock.lock();
    //do something
    lock.unlock();
  }
}
```

>A thread calling outer() will first lock the Lock instance. Then it will call inner(). Inside the inner() method the thread will again try to lock the Lock instance. This will fail (meaning the thread will be blocked), since the Lock instance was locked already in the outer() method.


一个线程访问outer()会锁住Lock实例，再访问inner()，在inner方法里将再次对Lock进行上锁，这会失败（即线程会被阻塞），由于锁的实例已经锁在outer()方法。

>The reason the thread will be blocked the second time it calls lock() without having called unlock() in between, is apparent when we look at the lock() implementation:

原因的线程将被阻塞，这是它第二次访问lock()没有叫unlock()之间，很明显当我们看lock()实施：

```
public class Lock{

  boolean isLocked = false;

  public synchronized void lock()
  throws InterruptedException{
    while(isLocked){
      wait();
    }
    isLocked = true;
  }

  ...
}
```

>It is the condition inside the while loop (spin lock) that determines if a thread is allowed to exit the lock() method or not. Currently the condition is that isLocked must be false for this to be allowed, regardless of what thread locked it.


它是在while循环的条件（自旋锁）决定如果一个线程是允许退出lock()法与否。公用的条件isLocked必须锁false才被允许，无论如何，线程都会被锁住。


>To make the Lock class reentrant we need to make a small change:

现在想让LOCK类进行折返我们做一些改变：

```
public class Lock{

  boolean isLocked = false;
  Thread  lockedBy = null;
  int     lockedCount = 0;

  public synchronized void lock()
  throws InterruptedException{
    Thread callingThread = Thread.currentThread();
    while(isLocked && lockedBy != callingThread){
      wait();
    }
    isLocked = true;
    lockedCount++;
    lockedBy = callingThread;
  }


  public synchronized void unlock(){
    if(Thread.curentThread() == this.lockedBy){
      lockedCount--;

      if(lockedCount == 0){
        isLocked = false;
        notify();
      }
    }
  }

  ...
}
```

>Notice how the while loop (spin lock) now also takes the thread that locked the Lock instance into consideration. If either the lock is unlocked (isLocked = false) or the calling thread is the thread that locked the Lock instance, the while loop will not execute, and the thread calling lock() will be allowed to exit the method.

通知自旋锁现在还需要读取Lock实例的条件（配置），如果isLocked为false 或者 调用线程的线程锁Lock实例,这个while不会执行，并且线程调用lock方法将被允许推出方法。

>Additionally, we need to count the number of times the lock has been locked by the same thread. Otherwise, a single call to unlock() will unlock the lock, even if the lock has been locked multiple times. We don't want the lock to be unlocked until the thread that locked it, has executed the same amount of unlock() calls as lock() calls.
The Lock class is now reentrant.

此外，我们需要计算锁的次数，该锁已被相同的线程锁。否则，一个信号来unlock()会打开锁，即使锁已经被多次。我们不想将锁打开，直到线程锁，执行unlock()调用相同lock()。
LOCK类现在可以折返了。

###英语渣渣...

```
serveral  //若干种
implementations //实现
theory //原理
tutorial //教程
details //细节
advanced //先进
suffices //足够
instead //替代
parked //停
proceed //进行
critical  //临界
like below  //类似下面
reason //原因
determines //使..下决定
regardless //无论如何
```