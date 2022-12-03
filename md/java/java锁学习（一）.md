CreateTime:2019-05-13 19:55:44.0

## 作用
能够保证同一时刻，最多只有一个线程执行该段代码，以达到并发安全的效果

主要用于同时刻对线程间对任务进行锁

## 地位
synchronized是JAVA的原生关键字，是JAVA中最基本的互斥手段，是并发编程中的元老角色

## 不使用并发的后果

不使用并发会导致多线程情况下，同一个数据被多个线程同时更改，造成结果和预期不一致导致虫子的出生（bug）。

```
class MissRequestNumAsync implements Runnable {
    static public int i = 0;
    @Override
    public void run() {
        for (int j = 0; j < 100000; j ++) {
            i++;
        }
    }
}

public class SynchronizedObjectAction{
    static MissRequestNumAsync m1 = new MissRequestNumAsync();

   public static void main(String[] args) {
        System.out.println("消失的请求数-不使用并发的后果");

        //消失的请求数
        Thread t1 = new Thread(m1);
        Thread t2 = new Thread(m1);
        t1.start();
        t2.start();
		 System.out.println(MissRequestNumAsync.i);
	}
}
```

运行结果(并且每次都不一样)
```
消失的请求数-不使用并发的后果
102171
```

## 锁的分类

1. 对象锁(方法锁、同步代码块锁)
2. 类锁（静态锁、class对象）

### 对象锁

#### 同步代码块锁

直接撸

```
//代码块锁
//劣势：太多锁会增加业务复杂度
//解决方式： 成熟的工具类
//调试：  thread.getStats();
class SyncObjArea implements Runnable{
    static public int i = 0;
    Object sleepLock1 = new Object();
    Object sleepLock2 = new Object();
    @Override
    public void run() {
        //这里有抢占机制
        synchronized (sleepLock1) {
            System.out.println("我是同步代码块锁1");
            try{
                Thread.sleep(3000);
            }catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println(Thread.currentThread().getName()+"运行结束");
        }
        //这里有抢占机制
        synchronized (sleepLock2) {
            System.out.println("我是同步代码块锁2");
            try{
                Thread.sleep(3000);
            }catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println(Thread.currentThread().getName()+"运行结束");
        }
    }
}
```

运行结果(多次运行，结果不一)
```
我是同步代码块锁1
Thread-2运行结束
我是同步代码块锁2
我是同步代码块锁1
Thread-3运行结束
Thread-2运行结束
我是同步代码块锁2
Thread-3运行结束
同步代码块线程执行完毕
```

#### 方法锁

方法锁默认是对象级别的(this)

```
//方法锁
class SyncObjMethod implements Runnable {
    static public int i = 0;
    @Override
    public void run() {
      method();
    }

    public synchronized void method(){
        System.out.println("我是方法锁");
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+"运行结束");
    }
}
```

运行结果
```
我是方法锁
Thread-4运行结束
我是方法锁
Thread-5运行结束
方法锁形式  线程执行完毕
```

## 源码位置

[线程学习](https://github.com/lwl1989/javaTest/tree/master/src/Thread)

明天继续学习类锁