---
layout: post
title: JUC CountDownLatch 从使用到源码解析
category: 多线程与并发编程
tags: 多线程与并发编程
date: 2021-05-24
---

<meta name="referrer" content="no-referrer" />

建议：本篇文章结合[JUC AbstractQueuedSynchronizer(AQS)源码-加解锁分析](https://juejin.cn/post/6964571893340831780)效果更佳！

#### 应用场景

CountDownLatch是一个比较有用的线程并发工具类，他的应用场景有很多，一般都是用在监听某些初始化操作上，等待初始化操作完成后然后通知某些主线程继续执行；在生活中例如接力赛吧，一个队员必须要接到另一个队员的接力棒才能跑；再例如我玩游戏的时候必须要先加载一些基础数据，基础数据加载完成之后才能开始游戏。这样的例子有很多；CountDownLatch的应用场景也有很多；在之前我们做电子合同的时候有个场景是要把用户上传的合同文档(一般都是.doc或者.docx),我们需要把这个.doc或者.docx文档转换成pdf;原因就是电子签章必须要签署到pdf上；一个文档页数有很多，我们就开启几个线程分页的去转换，我们必须要到等到所有线程都转换完成才能继续下一步的操作，就是这个场景我们就用到了CountDownLatch这个并发工具类，

#### 原理分析

它的内部提供一个计数器，在构造方法中必须要指定计数器的初始值，且计数器的初始值必须要大于0

`CountDownLatch countDownLatch = new CountDownLatch(2);`

另外它提供了一个`countDown()`方法来操作计数器的值，每调用一次`countDown()`方法计数器的值就会减1，直到计数器的值减为0，代表初始化操作已经完成，所有调用await方法而阻塞的线程都会被唤醒；可以进行下一步的操作！

#### 实例

```java
public class UseCountDownLatch {
  
    public static void main(String[] args) {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        Thread t1 = new Thread(() -> {
            try {
                System.out.println("进入线程t1" + "等待其他线程处理完成...");
                countDownLatch.await();
                System.out.println("t1线程继续执行...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "t1");
        Thread t2 = new Thread(() -> {
            try {
                System.out.println("t2线程进行初始化操作...");
                TimeUnit.SECONDS.sleep(3);
                System.out.println("t2线程初始化完毕,通知t1线程继续");
                countDownLatch.countDown(); //类似通知
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"t2");
        Thread t3 = new Thread(() -> {
            try {
                System.out.println("t3线程进行初始化操作...");
                TimeUnit.SECONDS.sleep(4);
                System.out.println("t3线程初始化完毕,通知t1线程继续");
                countDownLatch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"t3");
        t1.start();
        t2.start();
        t3.start();
    }
}

```

在初始化`CountDownLatch`时构造方法中传入了数值2，t1会等待t2和t3都调用`countDownLatch.countDown();`之后继续执行；执行结果如下：

```
进入线程t1等待其他线程处理完成...
t2线程进行初始化操作...
t3线程进行初始化操作...
t2线程初始化完毕,通知t1线程继续
t3线程初始化完毕,通知t1线程继续
t1线程继续执行...
```

#### API介绍

* **await():** 调用该方法的线程必须等到构造方法传入的值减到0的时候才能继续往下执行；
* **await(long timeout,TimeUnit unit):** 与上面的await方法功能一致，只不过这里有了时间限制，调用了该方法的线程等到指定的timeout时间后，不管构造方法传入的值是否减为0,都会继续往下执行；
* **countDown():** 使CountDownLatch初始值（构造方法中传入的值）减1；
* **long getCount():** 获取当前CountDownLatch维护的值；

#### 源码分析

下面贴一下`CountDownLatch`的构造方法：

```java
   public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```

`CountDownLatch`只有一个带参的构造方法，从这里可以看出count必须要大于0，不然会报错！在构造方法中只是new 了一个`Sync`对象；并赋值给了成员变量`sync`; 了解`AQS`的同学可以知道，`CountDownLatch`的实现依赖于`AQS`,它是`AQS`共享模式下的一个应用；

<img src="https://upload-images.jianshu.io/upload_images/15181329-b7d293c85766367a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image-20210507172847358.png" style="zoom:40%;" />

关于`AQS`我们后面有篇幅单独去讲解这个；上图可以看到不光`CountDownLatch`依赖于`AQS`,像`ReentrantLock`,`ReentrantReadWriteLock`,`Semaphore`都是依赖于`AQS`;可以看到`AQS`的重要性；同时也是面试的重点；

回归原题：`CountDownLatch`实现了一个内部类`Sync`并用它去继承`AQS`,这样就能使用`AQS`提供的大部分模板方法了；

我们看一下Sync内部类的代码：

```java
public class CountDownLatch {
    //Sync继承了AQS
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
		//Sync构造方法
        Sync(int count) {
            setState(count);
        }
		//获取当前同步状态
        int getCount() {
            return getState();
        }
	   //获取锁方法
        protected int tryAcquireShared(int acquires) {
            //在构造方法中state这个值必须要传入大于0的值，所以这里一直都是获取锁成功的;
            //直到每调用一次countDown()方法将state减1;
            return (getState() == 0) ? 1 : -1;
        }
		//释放锁方法
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                //获取锁状态
                int c = getState();
                //如果等于0，就不能再释放锁了
                if (c == 0)
                    return false;
                //否则将同步器减1
                int nextc = c-1;
                //使用CAS更新锁状态
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

```

<img src="https://upload-images.jianshu.io/upload_images/15181329-5bff99dc761beeec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:40%;" />

在获取锁方法`tryAcquireShared()`方法中：返回的是一个int类型的数值，分别为`-1`,`0`,`1`;他们分别表示：

- 负值：表示获取锁失败
- 零值：表示当前节点获取成功，但是后继节点不能再获取了
- 正值：表示当前节点获取成功，并且后继节点同样可以获取

`CountDownLatch`获取锁和释放锁的过程比较简单，我们在使用`CountDownLatch`的时候会调用`await()`方法来加锁，`countDown()`方法来解锁；下面我们先来看一下调用`awit()`方法的流程：

```java
public void await() throws InterruptedException {
     //以响应线程中断方式获取   
     sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
        //1 判断线程是否中断
        if (Thread.interrupted())
            //2以抛出异常的方式响应线程中断
            throw new InterruptedException();
        //3 尝试获取锁；在上面对正数，负数，0，3种取值已经进行了说明
        if (tryAcquireShared(arg) < 0)
            //4 获取锁失败
            doAcquireSharedInterruptibly(arg);
    }
```

关于是否响应线程中断后面文章会有介绍，感兴趣的可以关注我后面会有更新；

这里我们先关注3，4处的代码，首先调用了`tryAcquireShared(arg)`方法进行获取锁；这个方法就是上面我们贴出来的`tryAcquireShared(int acquires)`方法，看`state`是否等于`0`.如果等于`0`就返回`1`；表示加锁成功，否则返回-1表示不能获取锁。如果此方法返回`1`线程不必等待继续向下执行，如果此方法返回`-1`则进行` doAcquireSharedInterruptibly(arg)`方法:

```java
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

在[JUC AbstractQueuedSynchronizer(AQS)源码-加解锁分析](https://juejin.cn/post/6964571893340831780)这篇文章中我们讲述`doAcquireShared()`这个方法:

<img src="http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20210524111021488.png" alt="image-20210524111021488" style="zoom:60%;" />

基本上大致是一样的，唯一的区别就是在是否响应中断上有些区别；由于代码基本是一样的，这里就不过多的诉述了，建议看一下上一篇文章；

下面大致的说一下`countDown()`方法的过程：首先我们在`CountDownLatch`的构造方法中传入了一个数值count，这个数值赋给你内部类`Sync`,在`Sync`的构造方法中将count设置给了`State`同步状态，当每次调用`countDown()`方法的时候就会调用内部类Sync的`sync.releaseShared(1);`方法然后调用`tryReleaseShared()`方法实现自己释放锁的逻辑将`State`的值减1，每调用一次`countDown()`方法`state`就会减1，直到`state`减为`0`；

```java
public void countDown() {
     sync.releaseShared(1);
}

 public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

```

关于调用`countDown()`方法用一个生活中的例子解释更恰当；记得小时候家里的门都是使用**门闩（shuan栓）**去锁门；如果不知道**门闩**是什么，我在bai度搜索了图片：

<img src="http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20210523140427228.png" alt="image-20210523140427228" style="zoom:53%;" />

在保证更安全更牢固的情况下，可能一个门上会有多个门闩；当每调用一次`countDown()`就相当于打开一个门闩；直到每个门闩都会打开，这个门才能打开；

我们大致对`countDown()`方法有了了解，我们再去看源码，在源码中调用了`tryReleaseShared(arg)`方法去释放锁，tryReleaseShared方法如果返回true表示释放成功，返回false表示释放失败，只有当将同步状态减1后该同步状态恰好为0时才会返回true，其他情况都是返回false。

```java
		//释放锁方法
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                //获取锁状态
                int c = getState();
                //如果等于0，就不能再释放锁了
                if (c == 0)
                    return false;
                //否则将同步器减1
                int nextc = c-1;
                //使用CAS更新锁状态
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```

这里假设在构造`CountDownLatch`时count传入的是2，执行到这里`getState();`等于2，然后执行`c-1`，最后进行CAS将`c-1`的结果赋值给`state`,这时return的肯定是false；然后整个`releaseShared()`方法返回false;也就是说必须执行2次`countDown()`方法才会返回true;当下一次执行`countDown()`完毕之后，结果返回true,随后继续执行`doReleaseShared();`方法去唤醒同步队列的所有线程！

`doReleaseShared();`调用的是AQS类中的方法，在[JUC AbstractQueuedSynchronizer(AQS)源码-加解锁分析](https://juejin.cn/post/6964571893340831780)这篇文章中我们已经讲述过，这个不在阐述了；

