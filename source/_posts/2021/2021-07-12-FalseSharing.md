---
layout: post
title: 并发性能杀手伪共享了解一下
category: 多线程与并发编程
tags: 多线程与并发编程
date: 2021-07-12
---

<meta name="referrer" content="no-referrer" />


#### 什么是伪共享

并发编程中一个无声的性能杀手！

伪共享的知识需要一点点的操作系统知识，我在操作系统篇都讲解这些知识大家可以看下这一篇文章来帮助理解什么事伪共享[程序员应该知道的操作系统知识--基础篇（三）](https://juejin.cn/post/6911252291152510984),这里就不做过多的赘述了。。。。

我们都知道CPU的缓存是分层的如下图：

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03bb89d244df441894dc7f0c7743f08a~tplv-k3u1fbpfcp-zoom-1.image" style="zoom:67%;" />

距离CPU核心的访问速度越快；了解了这个之后我们在了解一下什么叫局部性原理？局部性原理分为2个：

* 时间局部性：如果某个数据被访问，那么在不久的将来有可能还会被再次访问。
* 空间局部性：如果某个数据被访问，那么它相邻的数据可能很快也会被访问。

了解完CPU的分层和局部性原理我们再来了解一下缓存行的概念。

缓存/内存中数据都是以缓存行(cache line)为单位存储的,最常见的缓存行大小是64个字节。一个java的long类型是8个字节，因此一个缓存行可以存8个long类型的变量。

如果存在这样的场景，有多个线程操作不同的成员变量，但是是相同的缓存行，这个时候会发生什么？对，这就是我们今天要说的伪共享！

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d96ca41761af4cde8aaaf5b24e191e70~tplv-k3u1fbpfcp-zoom-1.image" alt="image-20210705164132641" style="zoom:80%;" />

如图所示，当Core_1需要x这个变量值，然后就会去看L1 Cache存不存在，不存在就看L2 Cache存不存在，如果缓存中都不存在，就会在主存中拿到，在主存中x变量和y变量是处在同一个缓存行中所以Core_1虽然用不到y变量，但是由于处在同一个缓存行所以也在拿过来！然后依次放到`L3 Cache`-->`L2Cache`-->`L1Cache`。

同样的道理如果Core_2需要用到y变量，这个时候只需从L3 Cache缓存中取就可以了；

如果这个时候Core_1把x变量改了，那么Core_2对应的缓存行需要设为失效状态(I)来保证缓存行的一致性；那么Core_2想对y变量更改，但是不幸的是此时的缓存已经失效了，那么Core_2就要从主存中拿数据。我们从[程序员应该知道的操作系统知识--基础篇（三）](https://juejin.cn/post/6911252291152510984),这篇文章中可以知道从主存中取数据是非常耗时的！

同样的道理，Core_2从主存中拿到了数据之后对y变量进行了修改，同样Core_1中的缓存也会设为失效状态！虽然Core_1,Core_2只对自己用到的数据进行了更改但是由于x,y变量同时在一个缓存行中，所以为了保证缓存行的一致性，怕对方使用到了脏数据所以就好通知对方要刷新缓存。

#### 代码验证

口说无凭！好，那我们就会代码来验证一下！

```java
public class FalseSharing_Test {

    public static long COUNT = 10000000L;

    private static class T {
        public volatile long x = 0L;
    }
	//初始化一个T数组，长度为2
    public static T[] arr = new T[2];

    static {
        arr[0] = new T();
        arr[1] = new T();
    }

    public static void main(String[] args) throws Exception {
        CountDownLatch latch = new CountDownLatch(2);

        Thread t1 = new Thread(() -> {
          	//T1（arr[0]）对x修改10000000次
            for (long i = 0; i < COUNT; i++) {
                arr[0].x = i;
            }
            latch.countDown();
        });
					 //T2（arr[1]）对x修改10000000次
        Thread t2 = new Thread(() -> {
            for (long i = 0; i < COUNT; i++) {
                arr[1].x = i;
            }
            latch.countDown();
        });

        final long start = System.nanoTime();
        t1.start();
        t2.start();
        latch.await();
        System.out.println((System.nanoTime() - start) / 100_0000);
    }
}

```

经过多次运行，上面代码执行完成用时大概在 228到446区间内；

进行改造：

```java
public class FalseSharing2_Test {


    public static long COUNT = 10000000L;

    private static class T {
        public long p1, p2, p3, p4, p5, p6, p7;
        public volatile long x = 0L;
        public long p9, p10, p11, p12, p13, p14, p15;
    }

    public static T[] arr = new T[2];

    static {
        arr[0] = new T();
        arr[1] = new T();
    }

    public static void main(String[] args) throws Exception {
        CountDownLatch latch = new CountDownLatch(2);

        Thread t1 = new Thread(() -> {
            for (long i = 0; i < COUNT; i++) {
                arr[0].x = i;
            }
            latch.countDown();
        });

        Thread t2 = new Thread(() -> {
            for (long i = 0; i < COUNT; i++) {
                arr[1].x = i;
            }
            latch.countDown();
        });

        final long start = System.nanoTime();
        t1.start();
        t2.start();
        latch.await();
        System.out.println((System.nanoTime() - start) / 100_0000);
    }
}

```

经过多次运行，上面代码执行完成用时大概在 88到115区间内；

这里只在`public volatile long x = 0L;`变量的前后增加了“占行符”。

没有改造过的代码中，两个x变量是处于同一个缓存行中；改造过之后的代码采用了“占行符“的方式，在x变量前后增加了7个long类型的数据，加上x本身也是long类型，刚好是64位，占用一个缓存行。`arr[0]`的中x变量和`arr[1]`中的x变量互不打扰！

#### 总结

在业务开发中我们一定要通过缓存行填充去解决掉潜在的伪共享问题吗？其实并不一定，因为伪共享是很隐蔽的，还有就是不同类型的计算机架构是不一样的，例如有的是32位系统有的是64位，所以说并不是每个系统都适合花费大量的精力去解决潜在的伪共享问题；

有哪些框架采用了呢？如果你看过我的上一篇文章[为啥有了AtomicLong还要造LongAdder](https://juejin.cn/post/6981263032865259527),`LongAdder`中就使用了这种方式来解决伪共享；同样在`ConcurrentHashMap`中`addCount()`方法中`CounterCell`对象中维护的`long`类型的`value`值也用到了这样方式来解决伪共享问题！翻阅他们的源码你会发现`@sun.misc.Contended`这个注解，加上这个注解的类会自动补齐缓存行，大家有兴趣可以去看一下！
