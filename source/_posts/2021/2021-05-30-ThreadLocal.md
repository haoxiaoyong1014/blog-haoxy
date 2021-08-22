---
layout: post
title: ThreadLocal你真的用不上吗？
category: 多线程与并发编程
tags: 多线程与并发编程
date: 2021-05-30
---

<meta name="referrer" content="no-referrer" />

#### ThreadLocal的作用以及应用场景

ThreadLocal算是一种并发容器吧，因为他的内部是有ThreadLocalMap组成，ThreadLocal是为了解决多线程情况下变量不能被共享的问题，也就是多线程共享变量的问题；ThreadLocal和Lock以及Synchronized的区别是：ThreadLocal是给每个线程分配一个变量（对象），各个线程都存有变量的副本，这样每个线程都是使用自己（变量）对象实例，使线程与线程之间进行隔离；而Lock和Synchronized的方式是使线程有顺序的执行，举一个简单的例子：目前有100个学习等待签字，但是老师只有一个笔，那老师只能按顺序的分给每个学生，等待A学生签字完成然后将笔交给B学生，这就类似Lock,Synchronized的方式。而ThreadLocal是，老师直接拿出一百个笔给每个学生；再效率提高的同事也要付出一个内存消耗；也就是以空间换时间的概念

##### 使用场景

* Spring的事务隔离就是使用ThreadLocal和AOP来解决的；主要是TransactionSynchronizationManager这个类；

* 解决SimpleDateFormat线程不安全问题；

  当我们使用SimpleDateFormat的parse()方法的时候，parse()方法会先调用Calendar.clear()方法，然后调用Calendar.add()方法，如果一个线程先调用了add()方法，然后另一个线程调用了clear()方法；这时候parse()方法就会出现解析错误；如果不信我们可以来个例子：

  ```java
  public class SimpleDateFormatTest {
  
      private static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
  
      public static void main(String[] args) {
          for (int i = 0; i < 50; i++) {
              Thread thread = new Thread(new Runnable() {
                  @Override
                  public void run() {
                      dateFormat();
                  }
              });
              thread.start();
          }
      }
  
      /**
       * 字符串转成日期类型
       */
      public static void dateFormat() {
          try {
              simpleDateFormat.parse("2021-5-27");
          } catch (ParseException e) {
              e.printStackTrace();
          }
      }
  }
  ```

  这里我们只启动了50个线程问题就会出现，其实看巧不巧，有时候只有10个线程的情况就会出错;

  ```java
  Exception in thread "Thread-40" java.lang.NumberFormatException: For input string: ""
  	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
  	at java.lang.Long.parseLong(Long.java:601)
  	at java.lang.Long.parseLong(Long.java:631)
  	at java.text.DigitList.getLong(DigitList.java:195)
  	at java.text.DecimalFormat.parse(DecimalFormat.java:2084)
  	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
  	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
  	at java.text.DateFormat.parse(DateFormat.java:364)
  	at cn.haoxy.use.lock.sdf.SimpleDateFormatTest.dateFormat(SimpleDateFormatTest.java:36)
  	at cn.haoxy.use.lock.sdf.SimpleDateFormatTest$1.run(SimpleDateFormatTest.java:23)
  	at java.lang.Thread.run(Thread.java:748)
  Exception in thread "Thread-43" java.lang.NumberFormatException: multiple points
  	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
  	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
  	at java.lang.Double.parseDouble(Double.java:538)
  	at java.text.DigitList.getDouble(DigitList.java:169)
  	at java.text.DecimalFormat.parse(DecimalFormat.java:2089)
  	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
  	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
  	at java.text.DateFormat.parse(DateFormat.java:364)
  	at  .............
  ```

  其实解决这个问题很简单，让每个线程new一个自己的SimpleDateFormat，但是如果100个线程都要new100个SimpleDateFormat吗？当然我们不能这么做，我们可以借助线程池加上ThreadLocal来解决这个问题；

  ```java
  public class SimpleDateFormatTest {
  
      private static ThreadLocal<SimpleDateFormat> local = new ThreadLocal<SimpleDateFormat>() {
          @Override
        	//初始化线程本地变量
          protected SimpleDateFormat initialValue() {
              return new SimpleDateFormat("yyyy-MM-dd");
          }
      };
  
      public static void main(String[] args) {
          ExecutorService es = Executors.newCachedThreadPool();
          for (int i = 0; i < 500; i++) {
              es.execute(() -> {
                	//调用字符串转成日期方法
                  dateFormat();
              });
          }
          es.shutdown();
      }
      /**
       * 字符串转成日期类型
       */
      public static void dateFormat() {
          try {
            	//ThreadLocal中的get()方法
              local.get().parse("2021-5-27");
          } catch (ParseException e) {
              e.printStackTrace();
          }
      }
  }
  
  ```

  这样就优雅的解决了线程安全问题；

* 解决过度传参问题；例如一个方法中要调用好多个方法，每个方法都需要传递参数；例如下面示例：

  ```java
  void work(User user) {
      getInfo(user);
      checkInfo(user);
      setSomeThing(user);
      log(user);
  }
  ```

  用了ThreadLocal之后：

  ```java
  public class ThreadLocalStu {
  
      private static ThreadLocal<User> userThreadLocal = new ThreadLocal<>();
  
      void work(User user) {
          try {
              userThreadLocal.set(user);
              getInfo();
              checkInfo();
              someThing();
          } finally {
              userThreadLocal.remove();
          }
      }
  
      void setInfo() {
          User u = userThreadLocal.get();
          //.....
      }
  
      void checkInfo() {
          User u = userThreadLocal.get();
          //....
      }
  
      void someThing() {
          User u = userThreadLocal.get();
          //....
      }
  }
  
  ```

每个线程内需要保存全局变量（比如在登录成功后将用户信息存到`ThreadLocal`里，然后当前线程操作的业务逻辑直接get取就完事了，有效的避免的参数来回传递的麻烦之处），一定层级上减少代码耦合度。

* 比如存储 交易id等信息。每个线程私有。
* 比如aop里记录日志需要before记录请求id，end拿出请求id，这也可以。
* 比如jdbc连接池（很典型的一个`ThreadLocal`用法）
* ....等等....

#### 原理分析

上面我们基本上知道了ThreadLocal的使用方式以及应用场景，当然应用场景不止这些这只是工作中常用到的场景；下面我们对它的原理进行分析;

我们先看一下它的set()方法;

```java
   public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

是不是特别简单，首先获取当前线程，用当前线程作为key,去获取ThreadLocalMap,然后判断map是否为空，不为空就将当前线程作为key,传入的value作为map的value值；如果为空就创建一个ThreadLocalMap,然后将key和value方进去；从这里可以看出value值是存放到ThreadLocalMap中；

然后我们看看ThreadLocalMap是怎么来的？先看下getMap()方法：

```java
//在Thread类中维护了threadLocals变量，注意是Thread类
ThreadLocal.ThreadLocalMap threadLocals = null; 

//在ThreadLocal类中的getMap()方法
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

这就能解释每个线程中都有一个ThreadLocalMap，因为ThreadLocalMap的引用在Thread中维护；这就确保了线程间的隔离；

我们继续回到set()方法，看到当map等于空的时候`createMap(t, value);`

```java
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

这里就是new了一个ThreadLocalMap然后赋值给threadLocals成员变量；ThreadLocalMap构造方法：

```java
       ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
         //初始化一个Entry   
         table = new Entry[INITIAL_CAPACITY];
         	//计算key应该存放的位置
          int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
          //将Entry放到指定位置
          table[i] = new Entry(firstKey, firstValue);
          size = 1;
          //设置数组的大小 16*2/3=10,类似HashMap中的0.75*16=12
          setThreshold(INITIAL_CAPACITY);
        }
```

这里写有个大概的印象，后面对ThreadLocalMap内部结构还会进行详细的讲解；

下面我们再去看一下get()方法：

```java
 public T get() {
        Thread t = Thread.currentThread();
      	//用当前线程作为key去获取ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) {
          	//map不为空，然后获取map中的Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
              	//如果Entry不为空就获取对应的value值
                T result = (T)e.value;
                return result;
            }
        }
      	//如果map为空或者entry为空的话通过该方法初始化，并返回该方法的value
        return setInitialValue();
    }
```

get()方法和set()都比较容易理解，如果map等于空的时候或者entry等于空的时候我们看看setInitialValue()方法做了什么事：

```java
  private T setInitialValue() {
      //初始化变量值 由子类去实现并初始化变量
        T value = initialValue();
        Thread t = Thread.currentThread();
      	//这里再次getMap();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
						//和set()方法中的
            createMap(t, value);
        return value;
    }

```



下面我们再去看一下ThreadLocal中的initialValue()方法；

```java
   protected T initialValue() {
        return null;
    }
```

设置初始值，由子类去实现；就例如我们上面的例子,重写ThreadLocal类中的initialValue()方法：

```java
    private static ThreadLocal<SimpleDateFormat> local = new ThreadLocal<SimpleDateFormat>() {
        @Override
      	//初始化线程本地变量
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };
```

createMap()方法和上面set()方法中createMap()方法同一个，就不过多的叙述了；剩下还有一个removve()方法

```java
   public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
           //2. 从map中删除以当前threadLocal实例为key的键值对
             m.remove(this);
     }
```

源码的讲解就到这里，也都比较好理解，下面我们看看ThreadLocalMap的底层结构

#### ThreadLocalMap的底层结构

上面我们已经了解了ThreadLocal的使用场景以及它比较重要的几个方法；下面我们再去它的内部结构；经过上的源码分析我们可以看到数据其实都是存放到了ThreadLocal中的内部类ThreadLocalMap中；而ThreadLocalMap中又维护了一个Entry对象，也就说数据最终是存放到Entry对象中的；

```java
static class ThreadLocalMap {

        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
 
        }
          ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
  // ....................
}
```

Entry的构造方法是以当前线程为key,变量值Object为value进行存储的；在上面的源码中ThreadLocalMap的构造方法中也涉及到了Entry；看到Entry是一个数组；初始化长度为`INITIAL_CAPACITY = 16`；因为 Entry 继承了 WeakReference，在 Entry 的构造方法中，调用了 super(k)方法就会将 threadLocal 实例包装成一个 WeakReferenece。这也是ThreadLocal会产生内存泄露的原因；

##### 内存泄露产生的原因

<img src="http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20210528134958070.png" alt="image-20210528134958070" style="zoom:67%;" />

如图所示存在一条引用链： Thread Ref->Thread->ThreadLocalMap->Entry->Key:Value,经过上面的讲解我们知道ThreadLocal作为Key,但是被设置成了弱引用，弱引用在JVM垃圾回收时是优先回收的，就是说无论内存是否足够弱引用对象都会被回收；弱引用的生命周期比较短；当发生一次GC的时候就会变成如下：

<img src="http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20210528135713216.png" alt="image-20210528135713216" style="zoom:67%;" />

TreadLocalMap中出现了Key为null的Entry，就没有办法访问这些key为null的Entry的value,如果线程迟迟不结束（也就是说这条引用链无意义的一直存在）就会造成value永远无法回收造成内存泄露；如果当前线程运行结束Thread，ThreadLocalMap,Entry之间没有了引用链，在垃圾回收的时候就会被回收；但是在开发中我们都是使用线程池的方式，线程池的复用不会主动结束；所以还是会存在内存泄露问题；

解决方法也很简单，就是在使用完之后主动调用remove()方法释放掉；

##### 解决Hash冲突

记得在大学学习数据结构的时候学习了很多种解决hash冲突的方法;例如：

* 线性探测法（开放地址法的一种）： 计算出的散列地址如果已被占用，则按顺序找下一个空位。如果找到末尾还没有找到空位置就从头重新开始找；

  <img src="http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20210528170223972.png" alt="image-20210528170223972" style="zoom:50%;" />

* 二次探测法（开放地址法的一种）

<img src="http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20210528171123876.png" alt="image-20210528171123876" style="zoom:45%;" />

* 链地址法：链地址是对每一个同义词都建一个单链表来解决冲突，HashMap采用的是这种方法；

<img src="http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20210528171414508.png" alt="image-20210528171414508" style="zoom:50%;" />

* 多重Hash法：在key冲突的情况下多重hash,直到不冲突为止，这种方式不易产生堆积但是计算量太大；

* 公共溢出区法，这种方式需要两个表，一个存基础数据，另一个存放冲突数据称为溢出表；

上面的图片都是在网上找到的一些资料，和大学时学习时的差不多我就直接拿来用了；也当自己复习了一遍；

介绍了那么多解决Hash冲突的方法，那ThreadLocalMap使用的哪一种方法呢？我们可以看一下源码：

```java
 private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
   					//根据HashCode & 数组长度 计算出数组该存放的位置
            int i = key.threadLocalHashCode & (len-1);
						//遍历Entry数组中的元素
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
							//如果这个Entry对象的key正好是即将设置的key，那么就刷新Entry中的value；
                if (k == key) {
                    e.value = value;
                    return;
                }
							 // entry!=null,key==null时，说明threadLcoal这key已经被GC了，这里就是上面说到会有内存泄露的地方，当然作者也知道这种情况的存在，所以这里做了一个判断进行解决脏的entry（数组中不想存有过时的entry），但是也不能解决泄露问题，因为旧value还存在没有消失
                if (k == null) {
                  //用当前插入的值代替掉这个key为null的“脏”entry
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
				//新建entry并插入table中i处
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

从这里我们可以看出使用的是线性探测的方式来解决hash冲突！

源码中通过`nextIndex(i, len)`方法解决 hash 冲突的问题，该方法为`((i + 1 < len) ? i + 1 : 0);`，也就是不断往后线性探测，直到找到一个空的位置，当到哈希表末尾的时候还没有找到空位置再从 0 开始找，成环形！

#### 使用ThreadLocal时对象存在哪里？

在java中，栈内存归属于单个线程，每个线程都会有一个栈内存，其存储的变量只能在其所属线程中可见，即栈内存可以理解成线程的私有变量，而堆内存中的变量对所有线程可见，可以被所有线程访问！那么ThreadLocal的实例以及它的值是不是存放在栈上呢？其实不是的，因为ThreadLocal的实例实际上也是被其创建的类持有，（更顶端应该是被线程持有），而ThreadLocal的值其实也是被线程实例持有，它们都是位于堆上，只是通过一些技巧将可见性修改成了线程可见。
