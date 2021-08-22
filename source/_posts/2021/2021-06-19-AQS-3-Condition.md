---
layout: post
title: AQS之条件队列Condition
category: 多线程与并发编程
tags: 多线程与并发编程
date: 2021-06-19
---

<meta name="referrer" content="no-referrer" />

### AQS之条件队列Condition

在前面的[AQS源码](https://juejin.cn/post/6964571893340831780)中我们分别讲述了AQS同步队列独占模式，共享模式锁的获取以及释放，这些都是在同步队列中；在AQS中还包含了条件队列，如下图：

<img src="http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20210520162521601.png" alt="image-20210520162521601" style="zoom:90%;" />

今天我们就单独来讲讲条件队列Condition，在我们的工作中你可能很少使用，但是在一些并发容器的源码中经常会看到Condition的身影，例如：`LinkedBlockingQueue`,`ArrayBlockingQueue`等等，这也是去看一些并发容器源码之前的基础。所以我觉得还是很有必要的；

注意： 条件队列和同步队列不同条件队列是由` firstWaiter`和`lastWaiter`组成!

#### AQS中条件队列介绍

我们举一个贴近生活的例子吧，例如我们排队去上厕所，通过排队最终获得了锁进入了厕所，但是不巧的是发现忘记带纸，遇到这种事情很无奈，但是也得接受这个事实，这时只能乖乖的出去准备好手纸(也就是进入了条件队列中等待),当然再出去之前还要把锁释放掉，好让后面排队的人进来，在准备好了手纸(条件满足)条件满足之后进入同步队列中去排队；

下面我看下Condition都包含哪些方法：

```java
 //响应线程中断的条件等待
   void await() throws InterruptedException;
   
   //不响应线程中断的条件等待
   void awaitUninterruptibly();
   
   //设置相对时间的条件等待(不进行自旋)
   long awaitNanos(long nanosTimeout) throws InterruptedException;
   
   //设置相对时间的条件等待(进行自旋)
   boolean await(long time, TimeUnit unit) throws InterruptedException;
   
   //设置绝对时间的条件等待
   boolean awaitUntil(Date deadline) throws InterruptedException;
   
   //唤醒条件队列中的头结点
   void signal();
   
   //唤醒条件队列的所有结点
   void signalAll();
```

看上面方法挺多，主要就是await开头和signal开头的方法；分别就是将当前线程进入条件队列和唤醒条件队列中的线程；

我们再去看一下Condition的使用方式：

```java
public class UseCondition {

    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();


    public void method1() {
        try {
            lock.lock();
            System.out.println("当前线程: " + Thread.currentThread().getName() + "进入等待状态");
            TimeUnit.SECONDS.sleep(3);
            System.out.println("当前线程: " + Thread.currentThread().getName() + "释放锁");
            condition.await();// Object wait
            System.out.println("当前线程: " + Thread.currentThread().getName() + "继续执行");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void method2() {
        try {
            lock.lock();
            System.out.println("当前线程: " + Thread.currentThread().getName() + "进入....");
            TimeUnit.SECONDS.sleep(3);
            System.out.println("当前线程: " + Thread.currentThread().getName() + "发出唤醒..");
            condition.signal();//Object notify
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        UseCondition uc = new UseCondition();
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                uc.method1();
            }
        }, "t1");
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                uc.method2();
            }
        }, "t2");
        t1.start();
        t2.start();
    }
}
```

输出结果：

```
当前线程: t1进入等待状态
当前线程: t1释放锁
当前线程: t2进入....
当前线程: t2发出唤醒..
当前线程: t1继续执行
```

开启了两个线程waiter和signaler，waiter线程开始执行的时候由于条件不满足，执行condition.await方法使该线程进入等待状态同时释放锁，signaler线程获取到锁之后更改条件，并通知所有的等待线程后释放锁。这时，waiter线程获取到锁，并由于signaler线程更改了条件此时相对于waiter来说条件满足，继续执行。

#### 源码深入分析-await()方法

我们对Condition有了大致的认识，我们就对await()方法进行解析：

```java
  public final void await() throws InterruptedException {
    				//如果当前线程被中断直接抛出异常
            if (Thread.interrupted())
                throw new InterruptedException();
    				//将节点加入到条件队列
            Node node = addConditionWaiter();
    				//释放到之前获取的所有锁资源。有可能会有重入锁
            long savedState = fullyRelease(node);
            int interruptMode = 0;
    				//检查当前节点的状态如果是-2就会在while循环里进行条件等待(在addConditionWaiter()方法中已经将当前节点的状态设置为了-2，所以说这里肯定会进入到while循环中) ，也判断了当前节点在不在同步队列中（node.prev == null 说明当前节点不在同步队列中）同步队列是有头节点的，而条件队列没有
            while (!isOnSyncQueue(node)) {
              	//挂起当前线程
                LockSupport.park(this);
              	//醒来之后检查自己是否被中断，这里也有可能会调出循环
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //走到这里说明节点已经条件满足被加入到了同步队列中或者中断了
            //这个方法很熟悉吧？就跟独占锁调用同样的获取锁方法，从这里可以看出条件队列只能用于独占锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            //走到这里说明已经成功获取到了独占锁，接下来就做些收尾工作
            //删除条件队列中被取消的节点
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

```

接下来我们带着几个问题来看下await方法，1.怎么将节点加到队列中的？2.怎么释放当前获取的锁？3.await()方法怎么退出的？
我们先看下怎么将节点加到队列中的？

```java
        private Node addConditionWaiter() {
            Node t = lastWaiter;
           	//检查最后一个节点是否被取消，如果最后一个节点被取消就删除队列中取消的节点
            if (t != null && t.waitStatus != Node.CONDITION) {
              	//删除队列中取消的节点
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
          	//将当前节点封装成Node,并将Node节点设置为-2
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
              	//lastwaiter如果等于null,说明目前队列为空，所以将firstWaiter也指向node节点
                firstWaiter = node;
            else
              	//不为空的话 将lastWaiter的下一个节点指向node
                t.nextWaiter = node;
          	//最后一个节点就是node的了
            lastWaiter = node;
            return node;
        }
```

`addConditionWaiter()`方法执行结束就将node节点加入到了条件队列中了。`unlinkCancelledWaiters()`方法删除队列中取消的节点就是删除链表中的节点，由于篇幅太长这里就不贴出来了，也不是重点。至此第一个问题就解决了！下面我们看看第二个问题。

文章最开始我们用上厕所的例子去介绍条件队列的时候说过，在他忘记带手纸的时候就好去条件队列中等待，但是要把当前获取的锁释放掉，这样别人才有机会去上厕所。

`fullyRelease(node)`方法释放掉当前线程获取的锁

```java
final int fullyRelease(Node node) {
        boolean failed = true;
        try {
          	//获取同步状态，这里有可能是重入锁，所以都要释放掉！
            int savedState = getState();
          	//释放锁，之前的文章中有讲解，这里不再赘述
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
              	//失败抛出异常
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
              	//如果释放锁失败，就将节点取消，所以在addConditionWaiter()方法中只需要判断最后一个节点是否是取消状态就可以了
                node.waitStatus = Node.CANCELLED;
        }
    }
```

这个方法很简单没有啥说的，唯一就是要把当前节点获取的所有锁都释放掉，不然的话别人就获取不了锁了！

走到这一步节点月加入到条件队列中，资源也释放了，接下来就是挂起了。`isOnSyncQueue()`方法就不展开说了，上面的注释基本也说清楚了；

`LockSupport.park(this);`将当前线程挂起，这个没有什么说的，重点是在唤醒之后的代码，唤醒有可能是调用了signal()方法唤醒也有可能是中断唤醒，所以这里醒来的第一件事就是检查自己是怎么被唤醒的？

```java
while (!isOnSyncQueue(node)) {
      //挂起当前线程
     LockSupport.park(this);
     //醒来之后检查自己是否被中断，这里也有可能会调出循环
     if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
           break;
      }   


     private int checkInterruptWhileWaiting(Node node) {
       //因为中断而加入同步队列：THROW_IE
       //因调用signal()而加入同步队列：REINTERRUPT
       //期间没有收到任何中断请求：0
            return Thread.interrupted() ? 
              (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) : 0;
        }

		//修改节点并加入到同步队列
    final boolean transferAfterCancelledWait(Node node) {
      	//如果这里CAS成功说明是中断发在signal方法之前，因为signal()方法先执行了的话状态就不是-2了，就已经是0了
      	// 这里我把signal()方法贴出来更容易理解
      
        /*   private void doSignal(Node node) {
         * 
         *      transferForSignal(node);
         *  }
         *  
         *    boolean transferForSignal(Node node) {
         					//这里进行了CAS,CAS成功就已经将-2改为了0
         *  		 if (!compareAndSetWaitStatus(node, Node.CONDITION, 0)){
         *            return false;
         *       }
         * }
         */
      	//所以说返回true表示节点由中断加入同步队列，返回false表示由signal加入同步队列
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
          //加入同步队列； https://juejin.cn/post/6964571893340831780 这篇文章中讲述了，这里就不展开了！ 
            enq(node);
            return true;
        }
      	//如果上面设置失败，说明节点已经被signal()方法唤醒，由于signal()方法会将节点加入到同步队列，这里只需要自旋等待即可！
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
```

方法走到这里要么返回`THROW_IE`，要么返回`REINTERRUPT`。那么方法再次来到while()循环中；这里肯定不等于0，然后就break掉！

```java
           while (!isOnSyncQueue(node)) { 
              	//挂起当前线程
                LockSupport.park(this);
              	//醒来之后检查自己是否被中断，这里也有可能会调出循环
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //走到这里说明节点已经条件满足被加入到了同步队列中或者中断了
            //这个方法很熟悉吧？就跟独占锁调用同样的获取锁方法，从这里可以看出条件队列只能用于独占锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            //走到这里说明已经成功获取到了独占锁，接下来就做些收尾工作
            //删除条件队列中被取消的节点
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
              	//根据中断模式进行响应的中断处理
                reportInterruptAfterWait(interruptMode);
        }
		//根据中断时机选择抛出异常或者设置线程中断状态
     private void reportInterruptAfterWait(int interruptMode) throws InterruptedException {
       			//如果中断模式是THROW_IE就抛出异常
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
              //实现代码为：Thread.currentThread().interrupt();
                selfInterrupt();
     }

```

break之后方法就来到了`if (acquireQueued(node, savedState) && interruptMode != THROW_IE)`中；`acquireQueued(node, savedState)`我们在[AQS源码](https://juejin.cn/post/6964571893340831780)中已经进行了讲解；

也就是说，结点从条件队列出来后又是乖乖的走独占模式下获取锁的那一套，等这个结点再次获得锁之后，就会调用reportInterruptAfterWait方法来根据这期间的中断情况做出相应的响应。如果是中断引起，interruptMode就为THROW_IE，再次获得锁后就抛出异常；如果是signal方法引起，interruptMode就为REINTERRUPT，再次获得锁后就重新中断。

到这里await()方法就结束啦！下面我们看下signal()方法:

#### 源码深入分析-signal()方法

signal()方法比较简单，我们大致看下：

```java
 public final void signal() {
      			//检查当前线程是否获取独占锁lock,如果没有则抛出异常
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
      			//获取条件队列中第一个节点，之后的操作都是针对这个节点
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
				//进行唤醒操作
        private void doSignal(Node first) {
            do {
              		//将firstWaiter指针向后移动一位，指向first的下一个节点，如果等于null的话说明队列中只有一个节点first;
                if ( (firstWaiter = first.nextWaiter) == null)
                  	//因为条件队列中没有节点了所以lastWaiter也制为空
                    lastWaiter = null;
              	//将头结点从等待队列中移除
                first.nextWaiter = null;
              	//将节点加入到同步队列，如果transferForSignal操作失败就去唤醒下一个结点
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

final boolean transferForSignal(Node node) {
  			//CAS将当前节点的ws从-2设置为0，这里就是和上面的中断两联系，如果transferAfterCancelledWait方法先将状态改变了, 导致这步CAS操作失败
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
  			//将该结点添加到同步队列尾部,这里返回的p节点是当前节点的前置节点；
        Node p = enq(node);
        int ws = p.waitStatus;
        //如果前置节点被取消或者修改状态失败则直接唤醒当前节点
        //此时当前节点已经处于同步队列中，唤醒会进行获取锁操作（acquireQueued()）或者正确的挂起操作
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }

```

#### 总结

由于条件队列是阻塞队列实现的关键组件，开始也说了一些并发容器实现的基础；所以我们还是有必要去了解一下原理的；首先明白两点

* 条件队列是建立在某个具体的锁上面
* 条件队列跟同步队列是两个队列，条件队列依赖条件唤醒，同步队列依赖锁释放唤醒

下面我们用一张图来总结一下这篇文章：

![](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/AQS%E6%9D%A1%E4%BB%B6%E9%98%9F%E5%88%97.png)