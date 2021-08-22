---
layout: post
title: 你真的了解线程中断吗？
category: 多线程与并发编程
tags: 多线程与并发编程
date: 2021-06-10
---

<meta name="referrer" content="no-referrer" />

#### 你真的了解线程中断吗？

何为中断？

计算机的世界里处处都有中断，任何工作都离不开中断，可以说整个计算机系统就是由中断来驱动的。那么什么是中断？简单来说就是CPU停下当前的工作任务，去处理其他事情，处理完后回来继续执行刚才的任务，这一过程便是中断！

#### 响应中断与非响应中断的区别

每个线程都有一个boolean类型的中断状态，当中断线程时，这个线程的中断状态将被设置为true,在Thread中包含了中断线程以及查询线程中断的方法，如下：

```java
public class Thread{
  public void interrupt(){.......}
  public boolean isInterrupted(){......}
  public static boolean interrupted(){......}
}
```

`interrupt()`方法能中断目标线程，而`isInterrupted()`方法能返回目标线程的中断状态。静态的`interrupted()`方法将清除当前的中断状态并返回它之前的值，这也是清除中断状态的唯一方法。

调用`interrupt()`方法并不意味着立即停止目标线程正在执行的工作，而只是传递了请求中断的消息。

下面我们来看一个例子来理解下这3个方法：

```java
public class InterruptTest {

    public static void main(String[] args) throws InterruptedException {

        Thread threadOne = new Thread(new Runnable() {
            @Override
            public void run() {
                boolean isInterrupted = Thread.currentThread().isInterrupted();
                System.out.println(isInterrupted);
                if (isInterrupted) {

                    System.out.println("hao");
                }

                System.out.println(Thread.currentThread().getName() + " threadOne isInterrupt1 " + Thread.currentThread().isInterrupted());
                while (!Thread.interrupted()){
                    System.out.println("--------");
                }
                System.out.println(Thread.currentThread().getName() + " threadOne isInterrupt2 " + Thread.currentThread().isInterrupted());
            }
        }, "t1");
        //启动线程
        threadOne.start();
        //设置中断标志
        threadOne.interrupt();
        System.out.println("interrupt 执行完毕。。。。。");
        threadOne.join();
        System.out.println("main over... ");
        System.out.println(Thread.currentThread().getName() + " isInterrupted3：" + threadOne.isInterrupted());
    }
}
```

输出结果：

```
interrupt 执行完毕。。。。。
true
hao
t1 threadOne isInterrupt1 true
t1 threadOne isInterrupt2 false
main over... 
main isInterrupted3：false
```

线程执行的顺序不一样打印的结果也不一样！

```
false
t1 threadOne isInterrupt1 false
--------
--------
--------
--------
--------
--------
--------
interrupt 执行完毕。。。。。
t1 threadOne isInterrupt2 false
main over... 
main isInterrupted3：false
```

```
false
t1 threadOne isInterrupt1 true
t1 threadOne isInterrupt2 false
interrupt 执行完毕。。。。。
main over... 
main isInterrupted3：false
```

main线程打印的 isInterrupted3一直都是false,因为没有线程去中断main线程，这里先忽略，只看t1线程；好好揣摩一下上面3种打印的结果就能明白这3个方法的作用；

当线程在调用`wait()`,`sleep()`,和`join()`方法的时候，这个时候如果收到中断请求(调用了`interrupt()`方法)或者在开始执行时发现某个已被设置好的中断状态时，将抛出一个异常！

```java
public class InterruptTest {

    @SneakyThrows
    public static void main(String[] args) {

        Thread threadOne = new Thread(new Runnable() {
            @Override
            @SneakyThrows
            public void run() {
                while (true) {
                    System.out.println("+++++++++++");
                    Thread.sleep(3000);
                    System.out.println("----------");
                }
            }
        }, "t1");

        InterruptTest t = new InterruptTest();
        threadOne.start();
        Thread.sleep(200);
        threadOne.interrupt();
    }
}
```

打印结果：

```
+++++++++++
Exception in thread "t1" java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at cn.haoxy.use.lock.reenlock.InterruptTest$1.run(InterruptTest.java:25)
	at java.lang.Thread.run(Thread.java:748)
```

上面我们对Thread中的`interrupt(),isInterrupted(),interrupted()`有所了解了；下面我们看看什么是响应中断和非响应中断。

其实上面抛出异常就是响应中断的一种方式；我们以Lock为例：

```java
 public static void main(String[] args) {

        Lock lock = new ReentrantLock();
        //这里先加锁的目的是 让下面的lock.lockInterruptibly()方法能进入到doAcquireInterruptibly()方法中
        lock.lock();

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {

                try {
                    //注意这里使用的是lockInterruptibly()方法-响应中断
                    lock.lockInterruptibly();
                    System.out.println("lockInterruptibly....");
                } catch (InterruptedException e) {
                    System.out.println(Thread.currentThread().getName() + " interrupted.");
                    e.printStackTrace();
                }
            }
        }, "t1");
        t1.start();
        Thread.sleep(1000);
   			//调用interrupt()方法将其中断
        t1.interrupt();
        Thread.sleep(5000);
    }
```

打印结果

```
t1 interrupted.
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at cn.haoxy.use.lock.reenlock.InterruptTest$3.run(InterruptTest.java:92)
	at java.lang.Thread.run(Thread.java:748)
```

这里调用了`lock.lockInterruptibly()`尝试去获取锁，在尝试获取锁的过程中调用了`t1.interrupt()`方法将t1线程设置中断标志，跟踪源码当代码来到`doAcquireInterruptibly()`方法中：

```java
 private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; 
                    failed = false;
                    return;
                }
             		// 1  判断是否真的需要挂起，如果需要挂起就会调用parkAndCheckInterrupt()方法将其挂起
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

这里重点是1处的代码`shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()`，其他代码我们已经在[JUC AbstractQueuedSynchronizer(AQS)源码-加解锁分析](https://juejin.cn/post/6964571893340831780)中详细的讲解了，这里就不在过多的赘述！

因为在调用`lock.lockInterruptibly()`方法之前在main线程中调用了`lock.lock()`方法,所以这里是要将线程挂起等待的。所以执行了`parkAndCheckInterrupt()`方法中的`LockSupport.park(this)`方法将线程挂起！挂起不是我们的重点,我们的重点是在挂起之后醒来的时候，整个main方法执行结束，`lock.lock()`结束，主线程释放锁，这个时候就会轮到线程t1去获取锁；t1线程就会在它挂起的地方醒来：

```java
   private final boolean parkAndCheckInterrupt() {
     		//挂起的地方
        LockSupport.park(this);
     		//醒来继续向下执行
        return Thread.interrupted();
    }
```

当醒来的继续向下执行`Thread.interrupted()`,在执行`Thread.interrupted()`方法的时候`t1.interrupt()`方法已经执行了，也就是中断t1线程；`Thread.interrupted()`返回`true`;那接下来就是`throw new InterruptedException()`;t1线程就被中断获取锁了；这就是Lock响应中断获取锁的过程；

有响应中断，当然也会有非响应中断：

```java
   public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

		
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; 
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

```

如果你看过我前面的文章你对上面的代码肯定很眼熟！

这个和响应中断获取锁的不同的地方在：

```java
//不响应中断
if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()){
      interrupted = true;
}         
//响应中断
  if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()){
       throw new InterruptedException();
  }
     
```

不响应中断就是简单的设置一下标志位让` interrupted = true`;

但是这里有一个问题一直也在困扰我，将` interrupted = true`也就是`acquireQueued(final Node node, int arg)`方法返回了`true`,那接下来就会执行`selfInterrupt();`方法；

```java
   static void selfInterrupt() {
     		//再次将当前线程设置为中断状态
        Thread.currentThread().interrupt();
    }
```

`selfInterrupt();`方法中调用了`interrupt()`方法；目的是什么呢？我个人觉得因为在线程醒来的时候在`parkAndCheckInterrupt()`方法里面调用了`Thread.interrupted()`方法将线程的标志改变了，当调`Thread.interrupted()`虽然返回了`true`,但是将中断标志改为了`false`,所以要在`selfInterrupt()`方法中调用`Thread.currentThread().interrupt()`将当前线程设置为中断状态（复位）；也就是还原用户的标志，用户判断线程状态自行决定怎么处理；



