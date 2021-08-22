---
layout: post
title: 为啥有了AtomicLong还要在JDK1.8推出了LongAdder?
category: 多线程与并发编程
tags: 多线程与并发编程
date: 2021-07-05
---

<meta name="referrer" content="no-referrer" />


#### AtomicLong的问题

一个技术的出现肯定是为了解决一些问题。JDK1.8更新出LongAdder肯定是来解决一些已经出现的问题。在1.5就出现了AtomicLong，AtomicLong的作用是和LongAdder的作用是一样的；那肯定是AtomicLong有一些问题需要弥补；翻看AtomicLong的源码：

```JAVA
//以getAndAddLong方法为例
 public final long getAndAddLong(Object var1, long var2, long var4) {
        long var6;
        do {
            var6 = this.getLongVolatile(var1, var2);
        } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));

        return var6;
 }
```

翻阅源码可以看出很多方法几乎是所有有关这类的操作都是使用CAS操作。那CAS有什么问题呢？在并发数量不是很多的时候看不出什么问题，当并发量很大的时候就可以想象的出：例如有个N个线程进行加1操作，然后进行CAS操作，这时候只会有一个线程CAS成功，N-1个线程都是失败的，然后进行自旋；然后又有一个线程执行成功，剩下N-2个线程再次进行自旋；循环往复直到所有线程执行成功！这么一描述是不是觉得CAS在并发高的情况在效率会下降，这正式AtomicLong 的问题；LongAdder的出现就是为了解决这个问题的，那么下面我们就看看LongAdder怎么解决这个问题的；

#### LongAdder原理介绍

在介绍源码之前先对LongAdder原理进行一个大致的介绍：

可以说LongAdder是以空间换时间的方式来弥补AtomicLong的瓶颈问题。

LongAdder的基本思路就是分散热点，在AtomicLong中无论多少个线程都是对一个value进行累加，而在LongAdder中除了维护了一个value(`volatile long base`)值，还维护了一个数组

```java
		transient volatile Cell[] cells;
    @sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) {
          value = x; 
        }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }
    } 
```

虽然这个数组是间接维护的但是这不是重点；我们重点要知道这个数组中也维护了一个value值，目的很简单就是为了累加用的；不同的线程会命中到数组的不同槽中，各个线程只对自己槽内的那个value进行CAS操作，这样就达到了热点分散的目的；当并发不高的时候通过CAS直接操作base值，当并发高的时候CAS`base`有可能会失败，失败之后则会对Cell[]数组中的Cell[i]中的value进行CAS操作进行加1；原理很简单，下面我们就看看源码；

#### LongAdder源码分析

```java
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                //getProbe() & m 获取当前线程对应的槽位 Cell[getProbe() & m]
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
```

首先`add()`方法会先检查cells这个数组是否为空，如果等于空就会进行后面的casBase()对base进行累加操作，如果`casBase(b = base, b + x)`执行成功返回`true`然后取反就是`false`。那整体就为false,就不会进到if()方法里面整体结束；此篇文章也就结束了。。。。哈哈！

那现在我们假设cells不为空，就会进入到下面的判断逻辑：`as==null`以及`(m = as.length - 1) < 0`也为false。`getProbe()`方法是获取当前线程的hase值，然后与上m(数组的长度)就得到了指定位置上的`Cell[getProbe() & m]`这个对象然后赋值给a;我们假设这里不为空，然后进行`a.cas(v = a.value, v + x)`操作，这个操作就是拿到指定位置的Cell对象，对这个对象中的value值进行累加；如果并发不高很大可能是累加成功的，这里我们只能假设累加失败，有别的线程也在对这个value值进行累加那cas累加就会失败返回false然后取反就会进入`longAccumulate(x, null, uncontended);`方法；

```java
    final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
      	//获取当前线程的hash值并赋值给h变量
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); 
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            // #1 判断cells是否为空 以及 数组的长度是否大于0
            if ((as = cells) != null && (n = as.length) > 0) {
              	// #2 再次判断当前线程的位置是否有元素，为啥再次判断呢？因为上面有可能对该线程重新hash,一开始有可能多个线程都对Cell[0]这个位置进行累加value,经过一次重新hash,很有可能就不是0这个位置了，所以这里有可能是为空的，如果为空就进行向下执行，如果不为空就到代码#3处 
                if ((a = as[(n - 1) & h]) == null) {
             // cellsBusy 0表示空闲也可以说没有线程持有`锁`；1 ：表示忙碌状态，已有其他线程持有`锁`
                    if (cellsBusy == 0) {  
                      	//创建一个Cell对象r
                        Cell r = new Cell(x);  
                      	//casCellsBusy() 将0设置为1，表示忙碌，其他线程先等我完成你们再操作
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {    
                              	//下面一段的操作就是将r这个对象赋值到数组rs[j]这个位置
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           
                        }
                    }
                    collide = false;
                }
              	// #3 重新hash的话这里不会进,继续执行#4
                else if (!wasUncontended)       
                    wasUncontended = true;
         // #4 因为#2处代码 a = as[(n - 1) & h])不等于null，所以再次尝试进行对a=Cell[(n - 1) & h]中的value进行累加，成功就break,否则说明还有线程对当前value在进行累加操作，这个时候就会到代码#5处 要扩容了
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                else if (n >= NCPU || cells != as)
                    collide = false;            
                else if (!collide)
                    collide = true;
              // #5 扩容操作   
             //cellsBusy 0表示空闲也可以说没有线程持有`锁`；1 ：表示忙碌状态，已有其他线程持有`锁`
             // casCellsBusy() 将0设置为1，表示忙碌，其他线程先等我完成你们再操作
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {  
                          	//扩容原来的2倍
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                      	//将cellsBusy置为0。空闲状态了，其他线程可以操作了
                        cellsBusy = 0;
                    }
                  	//扩容结束又会进行重新的循环进入到#1处的代码
                    collide = false;
                    continue;                   
                }
                h = advanceProbe(h);
            }
          // #6
          // 如果#1处的代码 cells等于空 那么就会进行初始化Cell[]数组
          //下面就是初始化 rs这个数组，默认大小为2
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                      //h & 1: 要么等于0 要么等于1，一般可以用来判断h为单数还是复数例如：2的二进制位 0010 & 0001=0000；3的2进制为 0011 & 0001=0001
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
          	// 看base这个值还有其他线程在累加操作没；
          	// 再次尝试一下对base进行CAS累加操作
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          
        }
    }
```

上面的代码大致可以分为下面几部分：

* 首先是#1处的代码判断cells数组是否为空，为空的就会到#6处的代码进行初始化数组Cell[]
* 当#1处的代码cells数组不为空，就会到#2处再次判断当前线程的位置是否有元素，这里又会分成2种情况，一种就是有元素，另一种就是没有元素；
  * 我们首先假设这里是第一种情况，那么就会来到#3，#4，#5处的代码，#3处的代码不是我们的重点，最后肯定会执行#4处的代码，因为上面不为空那我肯定要尝试去累加这个这个对象中的value值；成功的话就直接break,不成功说明并发量特别的高，还有线程在对当前对象的value值进行修改；这个时候就会执行#5处的代码进行扩容了；扩容成功就会重新continue执行#1处的代码。
  * 还有另一种情况就是这个位置没有元素，没有元素的情况就比较的简单在该位置上创建一个Cell对象；



#### 总结

经过分析`LongAdder`中最核心的思想就是利用空间来换时间，将热点`value`分散成一个Cell列表来承接并发的CAS，以此来提升性能。那AtomicLong就可以被抛弃了吗？其实我觉得并不是，如果在并发程度没有那么高的情况下还是可以使用AtomicLong来节约一些内存空间；或者说对内存要求比较苛刻的情况下我们还是会用到AtomicLong的；

`LongAdder`的原理及实现都很简单，但其设计的思想值得我们品味和学习。