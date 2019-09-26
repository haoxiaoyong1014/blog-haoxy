---
layout: post
title: Redis专题(五)--EhcacheCache和Redis做mybatis二级缓存对比
category: Redis专题
tags: redis
date: 2019-09-16
---


源码：

[使用EhcacheCache做二级缓存](https://github.com/haoxiaoyong1014/springboot-examples/tree/master/springboot-mybatis-myehcache)

[使用redis做二级缓存](https://github.com/haoxiaoyong1014/springboot-redis-examples/tree/master/springboot-mybatis-redis-cache)

我们都知道无论是使用redis做二级缓存，还是使用EhchcheCache做二级缓存，都需要去实现Cache接口，并实现其中的方法；使用EhchcheChche做二级缓存mybatis帮我们实现了，我们只需要引入相应的maven 依赖(坐标)即可；而使用Redis做二级缓存我们需要自己去实现Cache接口；

**Cache接口中的方法：**

```java
package org.apache.ibatis.cache;

import java.util.concurrent.locks.ReadWriteLock;

public interface Cache {
    String getId();

    void putObject(Object var1, Object var2);

    Object getObject(Object var1);

    Object removeObject(Object var1);

    void clear();

    int getSize();

    ReadWriteLock getReadWriteLock();
}

```
**下图是EhcacheCache做二级缓存:**
![image.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTE4MTMyOS1iZDkwMzNlM2RiMWNlMGRhLnBuZw?x-oss-process=image/format,png)
**下图是Redis做二级缓存:**

![image.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTE4MTMyOS1lNzQ0MjVmNmVhNDdhODliLnBuZw?x-oss-process=image/format,png)

详细代码就不贴出来了；这里贴出测试代码：

```java
@RequestMapping("/findAll")
    public String findAll() {
        long begin = System.currentTimeMillis();
        List<Person> persons = personService.findAll();
        long ing = System.currentTimeMillis();
        System.out.println(("请求时间：" + (ing - begin) + "ms"));
        return JSON.toJSONString(persons);
    }
    @RequestMapping("/cacheFindAll")
    public String cacheByFindAll() {
        long begin = System.currentTimeMillis();
        List<Person> persons = personService.findAll();
        long ing = System.currentTimeMillis();
        personService.findAll();
        long end = System.currentTimeMillis();
        System.out.println(("第一次请求时间：" + (ing - begin) + "ms"));
        System.out.println(("第二次请求时间:" + (end - ing) + "ms"));
        return JSON.toJSONString(persons);
    }
```
首先我们测试redis做二级缓存；我们先执行`findAll`方法；测试结果如下：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTE4MTMyOS1hMWUyOGUyNzZlNDU4Y2MwLnBuZw?x-oss-process=image/format,png)
然后执行`cacheByFindAll`方法，测试结果如下

![image.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTE4MTMyOS01YWRjOTJhNjA4YjI5NzMyLnBuZw?x-oss-process=image/format,png)
然后我们测试EhcacheCache做和二级缓存，执行`findAll`方法；测试结果如下：
![image.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTE4MTMyOS02NjQ4OTJjMjc0ZTBmNjA4LnBuZw?x-oss-process=image/format,png)
后执行`cacheByFindAll`方法，测试结果如下
![image.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTE4MTMyOS0wZDhiZjYzZTAxOWM5MWJmLnBuZw?x-oss-process=image/format,png)
使用Redis做二级缓存，除了第一次时间较长之外，之后的几次都几乎到70-80ms之间；
使用EhcacheCache做二级缓存，除了第一次时间较长之外，之后的几次基本上不花费时间；
是不是差别很大，这就是为什么mybatis帮我们实现了EhcacheCache做二级缓存的方式，虽然说redis很牛X,但得看用到什么地方；用好了事半功倍,用到了不该用的地方得不偿失；