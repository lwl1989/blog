CreateTime:2019-05-14 20:09:04.0

## 类锁
类锁
!!!! java类有很多对象 ，但是只有一个class对象 !!!!

所以，类锁，就是针对当前类的Class对象的锁

类锁同一时刻只能被一个对象获取
1. synchronized放在static方法上(静态锁)
2. synchronized放在class对象上

### 静态锁

```
class SyncClassStatic implements Runnable {
    @Override
    public void run() {
        method();
    }

    public static synchronized void method() {
        System.out.println("我是类锁中静态锁");
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+"运行结束");
    }
}
result:
我是类锁中静态锁
Thread-6运行结束
我是类锁中静态锁
Thread-7运行结束
类锁，静态锁运行完毕
```

### class对象锁
```
class SyncClassObj implements Runnable{
    @Override
    public void run() {
        method();
    }

    private void method(){
        synchronized (SyncClassObj.class) {
            System.out.println("我是类锁中class对象锁");
            try{
                Thread.sleep(3000);
            }catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println(Thread.currentThread().getName()+"运行结束");
        }
    }
}
result:
我是类锁中class对象锁
Thread-8运行结束
我是类锁中class对象锁
Thread-9运行结束
类锁，class对象运行完毕
```

## 常见场景和问题

1. 两个线程同时访问一个对象的同步方法会怎样？

    依次执行（锁住的是同一个对象）

2. 两个线程访问的是两个对象的同步方法又会怎样？

	随机执行（锁住的是不同的对象）

3. 两个线程访问的是synchronized的静态方法会怎样？

	依次执行（锁住的是同一个类)

4. 同时访问同步方法与非同步方法会怎样？

	只锁加了synchronized的方法，其他方法不会收到影响
```
class SyncObjLock implements Runnable {


    @Override
    public void run() {
        if(Thread.currentThread().getName().equals("Thread-0")) {
            syncMethod();
        }else{
            asyncMethod();
        }
    }

    private synchronized void syncMethod()
    {
        System.out.println("我被锁了");
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+"运行结束");
    }

    private void asyncMethod()
    {
        System.out.println("我没被锁");
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+"运行结束");
    }
}
同时开始，同时结束：
我被锁了
我没被锁
Thread-0运行结束
Thread-1运行结束
```
5. 访问同一个对象的不同的普通同步方法会怎样？
```
class SyncDiff implements Runnable{
    @Override
    public void run() {
        if(Thread.currentThread().getName().equals("Thread-0")) {
            syncMethod1();
        }else{
            syncMethod2();
        }
    }


    private synchronized void syncMethod1()
    {
        System.out.println("我被锁了，我是方法1");
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+"运行结束");
    }


    private synchronized void syncMethod2()
    {
        System.out.println("我被锁了,我是方法2");
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+"运行结束");
    }
}
同时开始，同时结束：
我被锁了
我没被锁
Thread-0运行结束
Thread-1运行结束
```

6. 同时访问静态sync和非静态sync方法

```
class SyncStaticOrNo implements Runnable {
    @Override
    public void run() {
        if(Thread.currentThread().getName().equals("Thread-0")) {
            syncMethod1();
        }else{
            syncMethod2();
        }
    }


    private synchronized void syncMethod1()
    {
        System.out.println("我被对象锁");
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+"运行结束");
    }


    private static synchronized void syncMethod2()
    {
        System.out.println("我是类锁");
        try{
            Thread.sleep(3000);
        }catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+"运行结束");
    }
}
同时开始，同时结束：
我被锁了
我没被锁
Thread-0运行结束
Thread-1运行结束
```


7. 方法抛出异常后，会释放锁吗？

会，jvm帮我们释放了
```
class SyncExc implements Runnable{


    @Override
    public void run() {
            method1();
    }

    private synchronized void method1()
    {
        System.out.println(Thread.currentThread().getName()+"开始");
        try{
            Thread.sleep(3000);
        }catch (Exception e) {
            e.printStackTrace();
        }
        throw new RuntimeException("");
//        System.out.println("运行结束");
    }

}
```

```
Thread-0开始
Exception in thread "Thread-0" java.lang.RuntimeException: 
	at Thread.SyncExc.method1(SynchronizedObject.java:187)
	at Thread.SyncExc.run(SynchronizedObject.java:176)
	at java.lang.Thread.run(Thread.java:748)
Thread-1开始
Exception in thread "Thread-1" java.lang.RuntimeException: 
	at Thread.SyncExc.method1(SynchronizedObject.java:187)
	at Thread.SyncExc.run(SynchronizedObject.java:176)
	at java.lang.Thread.run(Thread.java:748)

```

## 锁的使用场景及分析

整个系统全局同步用类锁，局部锁用对象锁

对比下，就像一样，锁的范围越大，性能自然越低(粒度：类锁>对象锁  性能：类锁<对象锁)