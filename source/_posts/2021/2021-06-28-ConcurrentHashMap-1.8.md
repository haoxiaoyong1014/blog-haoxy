---
layout: post
title: 你真的认真看过ConcurrentHashMap1.8源码吗？
category: 多线程与并发编程
tags: 多线程与并发编程
date: 2021-06-28
---

<meta name="referrer" content="no-referrer" />

在工作中我们经常会使用到HashMap,但是我们知道在并发环境下HashMap是线程不安全的，我们可以使用HashTable也是同HashMap一样是java.util包下属于java亲生的；但是如果你翻阅过HashTable的源码你会发现每个方法上都加了`synchronized`关键字；锁是方法级别的锁的颗粒度太大，就会影响性能！所以就算的java亲生的也不建议使用！那我们在并发环境下使用什么呢？这个时候`Doug Lea`大佬又站了出来，为什么是又呢？如果你看过我的AQS源码讲解你就知道AQS也是`Doug Lea`作者写的；可以说`Doug Lea`作者对HashMap进行了优化创作出了ConcurrentHashMap,同样是在JUC包下！

下面我们就看看ConcurrentHashMap怎么解决了线程不安全的问题；



#### 回顾1.7

我觉得在将ConcurrentHashMap1.8之前我觉得很有必要对1.7进行简单介绍，对比1.7和1.8发生的变化，这也是面试官经常问到的问题；

**分段**

在1.7中提出来分段锁的概念来解决HashTable锁的颗粒度太大的问题；

分段思想就是不像HashTable那样锁整个hash表，而是只锁hash表的一部分；

如何实现的呢？ConcurrentHashMap1.7中维护了一个Segment数组，每一个Segment对象中又维护了`HashEntry<K,V>[]`;这样看来Segment对象有点类似HashMap;为什么这么说呢？因为HashMap中也维护了`HashEntry<K,V>[]`;

有一点值得我们注意的是Segment对象继承了ReentrantLock,继承了ReentrantLock也就意味着Segment可以使用ReentrantLock中的方法；像`lock()` ,`unlock()`等这些ReentrantLock中的这些方法Segment一样也可以使用。

下面我们简单看一下ConcurrentHashMap1.7的源码：

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {

    final Segment<K,V>[] segments;

    // Segment是ConcurrentHashMap的静态内部类
     
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
    
        transient volatile HashEntry<K,V>[] table;
       
    }

    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        // ... 省略 ...
    }
}
```

我们看到这里的源码结合上面的文字大胆的想象一下它的结构：

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a156374b57d24854a033b32b9f7596e1~tplv-k3u1fbpfcp-zoom-1.image" alt="image-20210624113000883" style="zoom:67%;" />

由图像可以很清晰的看出这就是在HashMap的基础上多了Segment数组；也就是有两层数组，每一个Segment对象类似一个锁对象，当进行put操作的时候Segment就会将自己锁住，也就是你自能操作你所指向的HashEntry数组，同样其他线程也不会操作你的Segment！这就是分段锁带来的好处；

我们简单总结一下ConcurrentHashMap1.7的整体流程：

1. 先根据Key算出对应的Segment数组下标，index;
2. 获取index位置上的锁，segment[index].lock();
3. 生成一个HashEntry对象，entry=new HashEntry(hash,key,value);
4. 在hash一次计算出放到哪个HashEntry[]数组中的下标，index2
5. 添加到Segment对象内部的HashEntry[]数组或链表上，和HashMap的put方法逻辑类似
6. 释放index位置上的锁，segment[index].unlocak();

这里可以看出计算了两次了下标，一次是确定在哪个Segment[index]对象中，第二是确定放到哪个HashEntry[index]下或者说是某个HashEntry[index]的链表下；

ConcurrentHashMap1.7大致的流程就是这样，感兴趣的可以翻阅一下它的源码，这里我就不贴出源码了；下面重点介绍一下1.8的源码；

#### 1.8内部结构

我们先看下ConcurrentHashMap1.8的内部结构：

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V> implements ConcurrentMap<K,V>, Serializable {
 
  	static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
    }
    //hash表,装载Node数组，数据容器，在第一次put时初始化,大小始终是2的幂
    transient volatile Node<K,V>[] table;

    //扩容时使用，平时为 null，只有在扩容的时候才为非 null
    private transient volatile Node<K,V>[] nextTable;

    /**
     *  计数器，baseCount+CounterCell[] 记录容器的大小；
     *  size()方法会使用到
     */
    private transient volatile long baseCount;

    /**
     * 见下面文字
     */
    private transient volatile int sizeCtl;

  	//调整大小时要拆分的下一个表索引（加一）
    private transient volatile int transferIndex;

    //创建CounterCells时使用，0：空间，1：表示在忙
    private transient volatile int cellsBusy;

  
  	//计数器单元格表。大小是2的幂，结合baseCount使用
    private transient volatile CounterCell[] counterCells;
  
    
}
```

##### **sizeCtl比较重要**

`sizeCtl`为0，代表数组未初始化， 且数组的初始容量为16

`sizeCtl`为正数，如果数组未初始化，那么其记录的是数组的初始容量，如果数组已经初始化，那么其记录的是数组的扩容阈值

`sizeCtl`为-1，表示数组正在进行初始化

`sizeCtl`小于0，并且不是-1，表示数组正在扩容， -(1+n)，表示此时有n个线程正在共同完成数组的扩容操作

#### 常用的构造方法

下面我们看下ConcurrentHashMap的常用的构造方法：

```java
//无参构造方法，默认大小为16
public ConcurrentHashMap() {}
```

```java
//给定一个初始容量，ConcurrentHashMap会基于这个值计算一个比这个值大2的幂次方作为初始容量  
public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
  //sizeCtl 的大小应该就代表了 ConcurrentHashMap 的大小，即 table 数组长度
        this.sizeCtl = cap; 
}
```

#### 添加元素put()/putVal()

```java
 public V put(K key, V value) {
        return putVal(key, value, false);
    }
//因为put方法太长了，这里先大致的过一变，我们一部分一部分的讲解
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
  			//获取key的hash值
        int hash = spread(key.hashCode());
  		  //记录某个桶上元素的个数，如果超过8个转成红黑树
        int binCount = 0;
  			//将table赋值给tab 进行循环
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
          	//第一个put操作tab肯定是等于null的，然后进行initTable()
            if (tab == null || (n = tab.length) == 0)
              	// 我们进入这个方法看看怎么初始化的 转 #code_1 处代码
                tab = initTable();
          	//(n - 1) & hash 计算key索引下标
          	//tabAT()该方法用来获取 table 数组中索引为 i 的 Node 元素,判断是否为空，然后赋值给f
          	//tabAT()方法内部=U.getObjectVolatile(tab,i);
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
              	//如果f==null,说明这个槽位没有元素，那直接cas将tab数组i这个位置设置为新Node，然后直接就break了
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                 
            }
          	//不等空的话继续判断这个元素f的hash是不是等于MOVED=-1表示在扩容；
            else if ((fh = f.hash) == MOVED)
              	//帮助扩容，这里我们后面说到扩容的时候会有讲解，这里先不说明
                tab = helpTransfer(tab, f);
            else {
              	//如果到这里说明f为头元素
                V oldVal = null;
              	//将头f锁住，也就是其他节点没有办法操作f以及f下面的链表
                synchronized (f) {
                  	//这里再次判断一下，是怕有其它线程将f删除掉，再次确保一下是否等于f
                    if (tabAt(tab, i) == f) {
                      	//当前为链表，在链表中查询新的元素是否存在，存在即覆盖，不存在即插入
                        if (fh >= 0) {
                            binCount = 1;
                          	//循环遍历 将f赋给e
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                              	// 新的元素和e元素hash是否相同，相同的话再比较equals,如果都相等直接覆盖
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                              	//走到这里说明和e说明hash不冲突并且eq不等
                                Node<K,V> pred = e;
                              	//判断e的下一个元素是不是为空，并将e的下一个元素赋给e,如果等于空就将新的元素插入到e.next位置（尾插），如果不等null继续循环重复以上操作
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                      	//判断f是不是为树结构，如果是树结构就想树中加入元素
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
              	//判断binCount是否等于0，不等于0的说明有新的元素添加，在上面的循环中对binCount++操作
                if (binCount != 0) {
                  	//判断链表长度是否大于8
                    if (binCount >= TREEIFY_THRESHOLD)
                      	//大于8的话就会变成红黑树，其实不是大于8就进行红黑树的，在treeifyBin()方法中又进行了数组长度的判断，看数组的长度是否大于64，如果数组的长度没有达到64就对数组进行扩容；
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
  			//到这里put的操作就结束了，addCount()方法的目的主要是累加元素的数量和扩容操作
  			//转 code_2处代码   记住这两个参数，后面会用到
        addCount(1L, binCount);
        return null;
    }
```

####  initTable（初始化table）

**#code_1**

```java
 private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
          	//sizeCtl<0 要么正在初始化，要么正在扩容，所以这里要让出CPU
            if ((sc = sizeCtl) < 0)
              	//让出cpu,进行自旋
                Thread.yield(); 
          	//把-1赋值给SIZECTL； 上面也介绍到SizeCtl等于-1表示数组正在进行初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                  	//这里再次判断table是否为空，为什么？下面会有说明
                    if ((tab = table) == null || tab.length == 0) {
                      	//这里sc就是sizeCtl,sizeCtl在构造方法中已经被赋值为table数组的初始化大小
                      	//可以看出如果是通过无参构造的话，默认大小就会是DEFAULT_CAPACITY=16
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                      	//初始化Node数组
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                      	//sc存放的是阈值，见下面的解释！
                        sc = n - (n >>> 2);
                    }
                } finally {
                  	//阈值赋值给sizeCtl，也验证了上面对sizeCtl的介绍
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

这里再次判断table是不是空，主要是解决并发问题，例如线程T1和T2同时进入到上面的While()循环中，判断table是否为空，这个时候都判断为true,`sc = sizeCtl) < 0` 都为false 这里只会有一个线程CAS成功，假如这里T1线程CAS成功，然后进行了初始化，那么如果没有再次判断的话T2过来又会初始化一次；这是不被允许的；

`sc = n - (n >>> 2);`  可以看做是 `n-(1/4)n=(3/4)n`，`3/4=0.75`   n表示Node数组大小，加入这里n=16,16向右移动2位，也就是除4；那sc的值就是12；（>>运算比除法效率高);

看到这里我们继续回到上面的put方法中！

####  addCount（累加计数，检测是否需要扩容）

**CounterCell的构造方法 -- 帮助理解addCount()**

```java
   @sun.misc.Contended static final class CounterCell {
     		//只是存了long 类型的value值 ；主要就是为了累加计数
        volatile long value;
        CounterCell(long x) { value = x; }
    }
```

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8aa5a8b916e34bdaba00ecd54bfacfa1~tplv-k3u1fbpfcp-zoom-1.image" alt="image-20210625145259089" style="zoom:80%;" />

**#code_2**

```java
private final void addCount(long x, int check) {
  			//在上面的内部结构进行介绍时就看到过CounterCell这个对象，上面说这个主要是结合baseCount使用来记录元素的个数
        CounterCell[] as; long b, s;
  			//判断counterCells是否为空，第一此肯定是空的
        if ((as = counterCells) != null ||
            //CAS操作，检查b是否等于BASECOUNT，如果等于就将baseCount+1赋给s 然后更新BASECOUNT，成功返回true,否则为false;  详细见下面的 #code_compareAndSwapLong说明
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
								// 只有as不能空的情况下才会对CounterCell进行累加,如果有多个线程同时去累加就会有失败的情况就和上面的CAS操作道理是一样的；如果失败然后取反就为true 就会进到fullAddCount()方法
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
              	//上面只要有一种情况为true都会进到这个方法 见 code_3处代码
              	//这个方法主要是初始化CounterCell数组和累计元素的个数
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
          	//拿到元素的个数，拿到这元素的个数的目的就是检查要不要进行扩容
            s = sumCount();
        }
    	//check就是上面传参中binCount,只要对向put数据了就binCount就会大于0
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            //当元素个数达到扩容阈值(sizeCtl)
        	//并且数组不为空
        	//并且数组长度小于限定的最大值
        	//满足以上所有条件，执行扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                //rs 是一个很大的值
                int rs = resizeStamp(n);
                 //sc小于0，说明有线程正在扩容，那么会协助扩容
                if (sc < 0) {
                    //扩容结束或者扩容线程数达到最大值或者扩容后的数组为null或者没有更多的桶位需要转移，结束操作
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    //扩容线程加1，成功后，进行协助扩容操作
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                         //协助扩容，newTable不为null，最上面我我们介绍到nextTable只会在扩容的时候才会用到
                        //见 code_4处代码
                        transfer(tab, nt);
                }
                //没有其他线性在扩容，给sizeCtl赋了一个很大的负数
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    //进行扩容，这里传入的是null
                    //见 code_4处代码
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

**#code_compareAndSwapLong说明**

`!U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)`；这段CAS操作的意思就是先比较b与主存中BASECOUNT的值是否相等，如果相等就将BASECOUNT的值更新为s;这里主要是为了防止并发情况的产生，假设现在有线程T1和T2都拿到baseCount都是为0；这时T1线程执行了这段CAS操作将0改为了1；那么主存中BASECOUNT值就为1，那这个时候T2再去执行这段CAS操作的时候，拿到baseCount还是0然后去和主存中的BASECOUNT去对比发现不相等，就会返回false; 返回false再取反就为true,接着就会进行到下面的`if()`判断条件，下面的多个条件只要有一个成立就会进入到`fullAddCount(x, uncontended)`方法中,这里我们假设第一次过来`as==null`，后面的条件就不会走了直接就进入到了`fullAddCount(x, uncontended)`方法！

#### fullAddCount

**#code_3**

```java
private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
  			//获取当前线程的hash值
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();     
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true;
        }
        boolean collide = false;  
  			//for循环
        for (;;) {
            CounterCell[] as; CounterCell a; int n; long v;
          	//判断counterCells是否为空，并将as数组的长度赋值给n
            if ((as = counterCells) != null && (n = as.length) > 0) {
              	//如果不为空。(n - 1) & h  = 长度-1 & 当前线程的hase值定位到某个CounterCell对象	
                if ((a = as[(n - 1) & h]) == null) {
                    //检查是否为忙碌状态，0:空闲 1：忙碌
                    if (cellsBusy == 0) {
                        //创建CounterCell对象 x是参数传过来的1
                        CounterCell r = new CounterCell(x); 
                        //再次检查cellsBusy的状态，如果等于0,将其设置为1
                        if (cellsBusy == 0 &&
                            U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {              
                                CounterCell[] rs; int m, j;
                                //主要是检查数组的【j = (m - 1) & h】这个位置有没有占用，如果等于空，就将上面创建的CounterCell对象赋值给数组rs的【j = (m - 1) & h】位置
                                if ((rs = counterCells) != null &&
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
                else if (!wasUncontended)      
                    wasUncontended = true;
              	//如果CounterCell对象不为空 首先对CounterCell对象中的value值进行累加
                else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                    break;
                else if (counterCells != as || n >= NCPU)
                    collide = false;            
                else if (!collide)
                    collide = true;
              	//如果上面的CAS累加失败了 将CELLSBUSY设置为忙碌状态
                else if (cellsBusy == 0 &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    try {
                        if (counterCells == as) {
                          	//对CounterCell[]数组进行扩容为原来的2倍，然后将旧的拷贝到新扩容
                            CounterCell[] rs = new CounterCell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            counterCells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                  
                }
                h = ThreadLocalRandom.advanceProbe(h);
            }
          	//第一次进来as==null的时候会进行这个else if 判断
          	//cellsBusy: 0表示空闲 1：表示在忙  
            else if (cellsBusy == 0 && counterCells == as &&
                    //将 CELLSBUSY由空闲设置为在忙状态
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {    
                  	//再次判断，依然是防止并发产生的问题
                    if (counterCells == as) {
                      	//初始化CounterCell数组 默认大小为2
                        CounterCell[] rs = new CounterCell[2];
                      	//h & 1: 要么等于0 要么等于1，一般可以用来判断h为单数还是复数例如：2的二进制位 0010 & 0001=0000；3的2进制为 0011 & 0001=0001
                        rs[h & 1] = new CounterCell(x);
                        counterCells = rs;
                        init = true;
                    }
                } finally {
                  	//完成初始化将cellsBusy设置为空闲
                    cellsBusy = 0;
                }
              	//然后进行break
                if (init)
                    break;
            }
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          
        }
    }
```

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/30c511724cc549d78ab71c1a977865aa~tplv-k3u1fbpfcp-zoom-1.image" alt="image-20210626162625012" style="zoom:67%;" />

计数的原理同LongAdder的实现原理是一样的，就是降低对value更新的并发数，也就是将对单一value的变更压力分散到多个value值上，降低单个value的“热度”。如果并发数量过多就会导致CAS失败的概率就会增加，重试的次数就会增加，越多的线程重试，CAS失败的概率就越高，从而形成恶性循环！

#### 扩容 transfer

**code_4**

多线程并发扩容

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
    	//如果是多cpu，那么每个线程划分任务，最小任务量是16个桶位的迁移
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                //两倍扩容创建新数组, 并赋值给成员变量nextTable
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            //记录线程开始迁移的桶位，从后往前迁移
            transferIndex = n;
        }
        int nextn = nextTab.length;
    	//创建fwd节点，已经迁移的桶位，会用这个节点占位（这个节点的hash值为-1--MOVED）
    	// 例如另一个线程在put的时候发现这个节点标注为fwd,那么这个线程就会去协助扩容
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                //i记录当前正在迁移桶位的索引值
               //bound记录下一次任务迁移的开始桶位
              //--i >= bound 成立表示当前线程分配的迁移任务还没有完成
                if (--i >= bound || finishing)
                    advance = false;
                //没有元素需要迁移 
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                 //计算下一次任务迁移的开始桶位，并将这个值赋值给transferIndex
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
             //如果没有更多的需要迁移的桶位，就进入该if
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                 //扩容结束后，保存新数组，并重新计算扩容阈值，赋值给sizeCtl
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                 //扩容任务线程数减1
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                     //判断当前所有扩容任务线程是否都执行完成
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    //所有扩容线程都执行完，标识结束
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
             //当前迁移的桶位没有元素，直接在该位置添加一个fwd节点
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            //当前节点已经被迁移
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                //当前节点需要迁移，加锁迁移，保证多线程安全
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                    //处理当前节点为链表的头结点的情况，构造两个链表，一个是原链表  另一个是原链表的反序排列
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                           //在nextTable的i位置上插入一个链表
                       		setTabAt(nextTab, i, ln);
                      		//在nextTable的i+n的位置上插入另一个链表
                       		setTabAt(nextTab, i + n, hn);
                       		//在table的i位置上插入forwardNode节点  表示已经处理过该节点
                       		setTabAt(tab, i, fwd);
                      		 //设置advance为true 返回到上面的while循环中 就可以执行i--操作
                       		advance = true;
                        }
                        //处理当前节点是TreeBin时的情况，操作和上面的类似
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }

```

码逻辑请看注释,整个扩容操作分为**两个部分**：

**第一部分**是构建一个 nextTable,它的容量是原来的两倍，这个操作是单线程完成的。新建 table 数组的代码为:`Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1]`,在原容量大小的基础上右移一位。

**第二个部分**就是将原来 table 中的元素复制到 nextTable 中，主要是遍历复制的过程。 根据运算得到当前遍历的数组的位置 i，然后利用 tabAt 方法获得 i 位置的元素再进行判断：

1. 如果这个位置为空，就在原 table 中的 i 位置放入 forwardNode 节点，这个也是触发并发扩容的关键点；
2. 如果这个位置是 Node 节点（fh>=0），如果它是一个链表的头节点，就构造一个反序链表，把他们分别放在 nextTable 的 i 和 i+n 的位置上
3. 如果这个位置是 TreeBin 节点（fh<0），也做一个反序处理，并且判断是否需要 untreefi，把处理的结果分别放在 nextTable 的 i 和 i+n 的位置上
4. 遍历过所有的节点以后就完成了复制工作，这时让 nextTable 作为新的 table，并且更新 sizeCtl 为新容量的 0.75 倍 ，完成扩容。设置为新容量的 0.75 倍代码为 `sizeCtl = (n << 1) - (n >>> 1)`，仔细体会下是不是很巧妙，n<<1 相当于 n 左移一位表示 n 的两倍即 2n,n>>>1，n 右移相当于 n 除以 2 即 0.5n,然后两者相减为 2n-0.5n=1.5n,是不是刚好等于新容量的 0.75 倍即 2n*0.75=1.5n



#### 总结

1.8改变的主要方面包括：

* 不采用segment而是采用synchronized+cas的方式来解决加锁问题，从而减少了锁的颗粒度(只锁当前链表)
* 设计了MOVED状态当resize的过程中如果有线程进行put数据发现这个数据的状态是fwd(-1),那么这个线程就会去协助扩容
* sizeCtl 的不同值来代表不同含义，起到了控制的作用。
* 摒弃ReentrantLock采用原生的synchronized；