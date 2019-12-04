---
layout: post
title: Redis专题(七)--基于Sentinel（哨兵）搭建实现Redis高可用集群
category: Redis专题
tags: Redis
date: 2019-10-29
---
<meta name="referrer" content="no-referrer" />

在前面的文章中我们讲了redis的主从复制的搭建,这边文章我们基于Sentinel（哨兵）搭建实现Redis高可用集群;

Redis哨兵为Redis提供了高可用性。实际上这意味着你可以使用哨兵模式创建一个可以不用人为干预而应对各种故障的Redis部署;

哨兵模式还提供了其他的附加功能，如监控，通知，为客户端提供配置。

下面是在宏观层面上哨兵模式的功能列表：

    监控：哨兵不断的检查master和slave是否正常的运行。
    通知：当监控的某台Redis实例发生问题时，可以通过API通知系统管理员和其他的应用程序。
    自动故障转移：如果一个master不正常运行了，哨兵可以启动一个故障转移进程，将一个slave升级成为master，其他的slave被重新配置使用新的master，并且应用程序使用Redis服务端通知的新地址。
    配置提供者：哨兵作为Redis客户端发现的权威来源：客户端连接到哨兵请求当前可靠的master的地址。如果发生故障，哨兵将报告新地址。

Redis哨兵是一个分布式系统:

哨兵自身被设计成和多个哨兵进程一起合作运行。有多个哨兵进程合作的好处有：

当多个哨兵对一个master不再可用达成一致时执行故障检测。这会降低错误判断的概率。
即使在不是所有的哨兵都工作时哨兵也会工作，使系统健壮的抵抗故障。毕竟在故障系统里单点故障没有什么意义。
Redis的哨兵、Redis实例(master和slave)、和客户端是一个有特种功能的大型分布式系统。在这个文档里将逐步从为了理解哨兵基本性质需要的基础信息，到为了理解怎样正确的使用哨兵工作的更复杂的信息(这是可选的)进行介绍。

集群架构图如下:

<img src="https://upload-images.jianshu.io/upload_images/15181329-cc0580e7d0e83fe7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

主从配置我们已经配置完成了,接下来就是配置哨兵:

在3个节点下分别配置`sentinel.conf`:

Master 6379:

<img src="https://upload-images.jianshu.io/upload_images/15181329-ff3af12dc9f0ae27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="carbon (1).png" style="zoom:50%;" />

Salve1 6380和Salve2 6381同上;

配置完毕在各个节点上启动哨兵:

在各个节点下的src目录下执行

`./redis-sentinel ../sentinel.conf`

启动后效果图:

<img src="https://upload-images.jianshu.io/upload_images/15181329-a58377f7b0a8ae9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="carbon.png" style="zoom:50%;" />
其他节点的效果就不放了;

**场景测试:**

场景1：master宕机

master 1的leader，会选取一个master 1的slave作为新的master。slave的选取是根据一个判断DNS情况的优先级来得到，优先级相同通过runid的排序得到，但目前优先级设定还没实现，所以直接获取runid排序得到slave 1。然后发送命令slaveofno one来取消slave 1的slave状态来转换为master。当其他sentinel观察到该slave成为master后,就知道错误处理例程启动了。sentinel A然后发送给其他slave slaveof new-slave-ip-port 命令，当所有slave都配置完后，sentinelA从监测的masters列表中删除故障master，然后通知其他sentinels。

现在6379是master节点:

<img src="https://upload-images.jianshu.io/upload_images/15181329-e67a423c3d4e9ceb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="carbon (2).png" style="zoom:50%;" />

关闭master节点以后;

查看当前谁是master,发现6381成为master

<img src="https://upload-images.jianshu.io/upload_images/15181329-e63c998c2479f652.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="carbon (3).png" style="zoom:50%;" />

场景2：重启之前关闭的 master 6379

```
[root@izuf63ehyqn5xpw1nkixd9z master]# cd src/
[root@izuf63ehyqn5xpw1nkixd9z src]# ./redis-server ../redis.conf 
```
查看6381master节点信息:

<img src="https://upload-images.jianshu.io/upload_images/15181329-3b4c62f5ef97d42d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="carbon (4).png" style="zoom:50%;" />
发现6379成为了slave节点;

下面自行可以测试一下, salve宕机然后重启;

