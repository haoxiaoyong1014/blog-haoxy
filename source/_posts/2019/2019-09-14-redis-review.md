---
layout: post
title: Redis专题(三)--分布式Redis复习
category: Redis专题
tags: redis
toc: true
date: 2019-09-14
---

#### 复习要点

* 为什么使用redis
* 使用redis有什么缺点
* 单线程的redis为什么这么快
* redis的数据类型以及每种数据类型的使用场景(附案例)
* redis的过期策略以及内存淘汰机制
* redis和数据库双写一致性问题
* 如何应对缓存穿透和缓存雪崩问题
* 如何解决redis的并发竞争问题

#### 正文

##### 1,为什么使用redis

分析： 在项目中使用redis,主要是从两个角度去考虑：性能和并发。当然还具备可以做分布式锁等其他功能，但是如果只是为了分布锁这些其他功能，完全还有其他中间件(zookpeer等)代替，并不是非要使用redis。因此，这个问题主要从性能和并发两个角度去答。

回答：分为两点

(一) 性能

如下图所示，我们碰到需要执行耗时特别久，且结果不频繁变动的SQL,就特别适合将运行结果放入缓存，这样，后面的请求就去缓存中读取，使得请求能够迅速响应。

![](https://mmbiz.qpic.cn/mmbiz_png/DmibiaFiaAI4B1qSviae9BGFFmz5P1Xdw6Pnj6PtibHwpfnDV0If0jZoF2QdicflEfXqT6jgykQcHuQTpvQoghvYhcWA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

看到这里是不是想到了mybatis的二级缓存，实现mybatis的二级缓存方式有：<a href="https://github.com/haoxiaoyong1014/springboot-examples/tree/master/springboot-mybatis-myehcache">使用EhcacheCache做二级缓存</a>,和<a href="https://github.com/haoxiaoyong1014/springboot-redis-examples/tree/master/springboot-mybatis-redis-cache">使用redis做二级缓存</a>；

(二) 并发

如下图所示：在大并发的情况下，所有的请求直接访问数据库，数据库会出现链接异常，这个时候就需要使用redis做一个缓冲操作，让请求先访问到redis,而不是直接访问数据库。

![](https://mmbiz.qpic.cn/mmbiz_png/DmibiaFiaAI4B1qSviae9BGFFmz5P1Xdw6Pn9qLibiahZO3ZLACyDLoAm7d69vyy9u6M8CQqH9JsNPpN4HymibicmKaYNg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在这里也可以说一说redis的持久化；

##### 2,使用redis有什么缺点

回答：主要是四个问题

(一) 缓存和数据库双写一致性问题

(二) 缓存雪崩问题

(三) 缓存击穿问题

(四) 缓存的并发竞争问题

这四个问题，后文给出解决方案

##### 3,单线程的redis为什么这么快

回答：主要是一下三点

(一) 纯内存操作

(二) 单线程操作，避免了频繁的上下文切换

(三) 采用了非阻塞I/O多路复用机制

I/O多路复用机制:分配单个线程，通过跟踪每个I/O流的状态来管理多个I/O流

![](https://mmbiz.qpic.cn/mmbiz_png/DmibiaFiaAI4B1qSviae9BGFFmz5P1Xdw6PnNziadaf5biaVghVV3FjRaJ5ZloQoiaDeT5yU6UVzlhoFZksWIRjkvs7Tg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

参照上图,简单来说，就是我们的redis-cli在操作的时候，会产生具有不同事件类型的socket。在服务器端，有一段I/O多路复用程序，将其置入队列之中。然后，文件事件分配器，依次去队列中取，转发到不同的事件处理器中；

##### 4,redis的数据类型，以及每种数据类型的使用场景

是不是觉得这个问题很基础，其实我也这么觉得。然而根据面试经验发现，至少百分八十的人答不上这个问题。建议，在项目中用到后，再类比记忆，体会更深，不要硬记。基本上，一个合格的程序员，五种类型都会用到。

回答：一共五种

(一) String

这个其实没啥好说的，最常规的set/get操作，value可以是String也可以是数字。

使用场景：

* 一般做一些复杂的计数功能的缓存。(<a href="https://github.com/haoxiaoyong1014/springboot-examples/tree/master/springboot-redis-docker">Docker 部署 SpringBoot 项目整合 Redis 镜像做访问计数Demo</a>)

* 在发送短信或者邮件的时候存放验证码，等等。。。

(二) hash

这个value 存放的是结构化的对象，比较方便的就是操作其中的某个字段，

使用场景：

* 做单点登录的时候，就是用这种数据结构存储用户信息，以cookieId作为key，设置30分钟为缓存过期时间，能很好的模拟出类似session的效果。

(三) list

使用场景：

* 使用List的数据结构，可以做简单的消息队列的功能
* 可以利用lrange命令，做基于redis的分页功能，性能极佳,<a href="https://github.com/haoxiaoyong1014/redis-manage">Redis的后台管理</a>,结合前端项目<a href="https://github.com/haoxiaoyong1014/redis-manage-view">redis-manage-view</a>

(四) set

使用场景：

* 因为set堆放的是不重复的集合，所以可以做全局去重的功能
* 就是利用交集、并集、差集等操作，可以计算共同喜好，全部的喜好，自己独有的喜好等功能。<a href="https://github.com/haoxiaoyong1014/springboot-redis-examples/tree/master/springboot-redis-friends">基于Redis实现查询共同好友</a>

(五) sorted set

sorted set多了一个权重参数score,集合中的元素能够按score进行排列。可以做排行榜应用，取TOP N操作。

使用场景：

* 排行榜。<a href="https://github.com/haoxiaoyong1014/springboot-redis-examples/tree/master/springboot-redis-ranking">基于Redis实现商品排行榜</a>
* 做范围查询

##### 5,redis的过期策略以及内存淘汰机制

分析:这个问题其实相当重要，到底redis有没用到家，这个问题就可以看出来。比如你redis只能存5G数据，可是你写了10G，那会删5G的数据。怎么删的，这个问题思考过么？还有，你的数据已经设置了过期时间，但是时间到了，内存占用率还是比较高，有思考过原因么?

回答：

redis采用的是定期删除+惰性删除策略

为什么不用定时删除策略？

定时删除，用一个定时器来复杂监视key,过期则自动删除，虽然内存及时释放，但是十分消耗CPU资源；为什么会消耗CPU的资源呢？因为定期删除，redis默认每隔100ms检查一次，是否有过期的key,有过期的key则删除。需要说明的是redis不是每隔100ms将所有的key检查一次，而是随机抽取进行检查(如果每隔100ms,全部key进行检查，redis岂不是要卡死)。因此，如果只采用定期删除策略，一方面会导致cpu消耗大，另一方面会导致很多key到时间没有删除。

于是，惰性删除派上了用场，也就是说在你获取某个key的时候，redis会检查一下这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除。

采用定期删除+惰性删除就没有其他问题了吗？

不是的，如果定期删除没有删除key,然后你也没有及时去请求key,也就是说惰性删除也没有生效。这样redis的内存会越来越高。那么就应该采用内存淘汰机制。

在redis.conf中有一行配置

`maxmemory-policy volatile-lru`

该配置就是配内存淘汰策略的

* noeviction：当内存不足以容纳新写入数据时，新写入操作会报错。应该没人用吧。
* allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。推荐使用，目前项目在用这种。

* allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。应该也没人用吧，你不删最少使用Key,去随机删。
* volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。这种情况一般是把redis既当缓存，又做持久化存储的时候才用。不推荐
* volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。依然不推荐
* volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。不推荐

##### 6,redis 和数据库双写一致性问题

一致性问题是分布式常见的问题，还可以在分为最终一致性和强一致性，数据库和缓存双写，就必然会存在不一致的问题，答这个问题先明白一个前提，就是如果对数据库有强一致性要求，不能放缓存。我们所做的一切，只能保证最终一致性。另外，我们所做的方案其实从根本上说，只能说降低了不一致发生的概率，无法完全避免。因此，有强一致性要求的不能放缓存；

参考：

[分布式之数据库和缓存双写一致性方案解析](https://mp.weixin.qq.com/s?__biz=MzA5ODM5MDU3MA==&mid=2650863864&idx=1&sn=b0cad32ec778b4e3c95aacb9b2eba932&chksm=8b661fbdbc1196ab63df520397b78ee750d6d354d8381f4564e4af182f8cd6f968902ecd698d&scene=21#wechat_redirect)

[并发环境下，先操作数据库还是先操作缓存?](https://juejin.im/post/5d4a3196f265da03ab423727)

##### 7,如何应对缓存穿透和缓存雪崩问题

回答：

**缓存穿透**，即黑客故意去请求缓存中不存在的数据，导致所有的请求都堆到了数据库上，从而数据库连接异常。

解决方案：

(一) 利用互斥锁，缓存失效的时候，先去获得锁，得到锁了，再去请求数据库。没有得到锁，则休眠一段时间重试；

(二) 提供一个能迅速判断请求是否有效的拦截机制，比如，利用布隆过滤器，内部维护一系列合法有效的key。迅速判断出，请求所携带的Key是否合法有效。如果不合法，则直接返回。

**缓存雪崩**，即缓存同一时间大面积的失效，这个时候又来了一波请求，结果请求都怼到数据库上，从而导致数据库连接异常。

解决方案:

(一)给缓存的失效时间，加上一个随机值，避免集体失效。

(二)使用互斥锁，但是该方案吞吐量明显下降了。

(三)双缓存。我们有两个缓存，缓存A和缓存B。缓存A的失效时间为20分钟，缓存B不设失效时间。自己做缓存预热操作。然后细分以下几个小点



- I 从缓存A读数据库，有则直接返回
- II A没有数据，直接从B读数据，直接返回，并且异步启动一个更新线程。
- III 更新线程同时更新缓存A和缓存B。

##### 8,如何解决redis的并发竞争key问题

分析： 这个问题大致是，同时有多个子系统去set一个key,这个时候要注意什么呢？很多博客写道使用redis事务机制，我个人不是很推荐这种，因为我们的生产环境，基本上都是redis集群环境，做了数据分片操作，一个事务中有涉及多个key操作的时候，这多个key不一定都存在同一个redis-server上，因此redis的事务机制，不推荐；

(1)如果对这个key操作，不要求顺序
这种情况下，准备一个分布式锁，大家去抢锁，抢到锁就做set操作即可，比较简单。

(2) 如果对这个key操作，要求顺序

假设有一个key1,系统A需要将key1设置为valueA,系统B需要将key1设置为valueB,系统C需要将key1设置为valueC.

期望按照key1的value值按照 valueA–>valueB–>valueC的顺序变化。这种时候我们在数据写入数据库的时候，需要保存一个时间戳。假设时间戳如下

```
系统A key 1 {valueA  3:00}

系统B key 1 {valueB  3:05}

系统C key 1 {valueC  3:10}
```

那么，假设这会系统B先抢到锁，将key1设置为{valueB 3:05}。接下来系统A抢到锁，发现自己的valueA的时间戳早于缓存中的时间戳，那就不做set操作了。以此类推。

其他方法，比如利用队列，将set方法变成串行访问也可以。总之，灵活变通。

