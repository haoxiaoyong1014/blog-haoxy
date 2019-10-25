---
layout: post
title: Redis专题(六)--搭建Redis主从复制以及它的优缺点
category: Redis专题
tags: redis
date: 2019-10-25
---

在Redis的持久化中曾提到，Redis高可用的方案包括持久化、主从复制（及读写分离）、哨兵和集群。其中持久化侧重解决的是Redis数据的单机备份问题（从内存到硬盘的备份;
而主从复制则侧重解决数据的多机热备。

### 主从复制概述

主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(master)，后者称为从节点(slave)；数据的复制是单向的，只能由主节点到从节点。

默认情况下，每台Redis服务器都是主节点；且一个主节点可以有多个从节点(或没有从节点)，但一个从节点只能有一个主节点。

### 主从复制的作用

**主从复制的作用主要包括:**

    数据冗余：主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。

    故障恢复：当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复；实际上是一种服务的冗余。

    负载均衡：在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点），分担服务器负载；尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量。

    高可用基石：除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础。
    
### 实现主从复制

**1. 建立复制**

需要注意，主从复制的开启，完全是在从节点发起的；不需要我们在主节点做任何事情。

从节点开启主从复制，有3种方式：

1、配置文件

在从服务器的配置文件中加入：slaveof <masterip> <masterport>

2、启动命令

redis-server启动命令后加入 --slaveof <masterip> <masterport>

3、客户端命令

Redis服务器启动后，直接通过客户端执行命令：slaveof <masterip> <masterport>，则该Redis实例成为从节点。

上述3种方式是等效的，下面以配置文件的方式为例，看一下当执行了slaveof后，Redis主节点和从节点的变化。

**1,下载安装**

`wget http://download.redis.io/releases/redis-4.0.11.tar.gz`

**2.解压缩**

`tar -zxvf redis-4.0.11.tar.gz`

**3.安装编译,进入到解压缩目录下,执行以下两个命令即可**

```
make 
make install
```
**4,通过cp复制成如下几个master,slave1,slave2目录**

```
hxyMacmini:redis haoxiaoyong$ ls
master slave1 slave2
```
**5,修改从节点配置文件**

1).从节点一
 
```
protected-mode yes
slave-serve-stale-data yes
port 6380
slaveof 127.0.0.1 6379  #主节点的ip和端口
```
2).从节点二

```
protected-mode yes
slave-serve-stale-data yes
port 6381
slaveof 127.0.0.1 6379  主节点的ip和端口
```

### 启动测试

1.首先启动主节点,然后启动从节点,命令一样的,进入到src下通过以下命令进行启动

`./redis-server ../redis.conf`

2.测试

在主节点上通过info replication查看节点信息,连接主节点的客户端,通过以下命令

`redis-cli -p 6379`

然后输入info replication

```
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=42,lag=1
slave0:ip=127.0.0.1,port=6381,state=online,offset=42,lag=1
master_replid:16d69933dc4525a5a3bd707e7e097121416ffe11
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:42
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:42
127.0.0.1:6379> 

```

3.在主节点上添加一条数据,在从节点上进行查看是否数据进行同步了.

1).添加数据(主)

```
127.0.0.1:6379> set haoxy 'a good boy'
OK
127.0.0.1:6379> get haoxy
"a good boy"
127.0.0.1:6379> 
```

2).从节点获取数据

```
hxyMacmini:src haoxiaoyong$ ./redis-cli -p  6380
127.0.0.1:6380> get haoxy
"a good boy"
127.0.0.1:6380> 
```
以上就完成了redis的主从架构的搭建和数据的同步

### 主从复制优缺点：

**优点：**

* 支持主从复制，主机会自动将数据同步到从机，可以进行读写分离
* 为了分载Master的读操作压力，Slave服务器可以为客户端提供只读操作的服务，写服务仍然必须由Master来完成
* Slave同样可以接受其它Slaves的连接和同步请求，这样可以有效的分载Master的同步压力。
* Master Server是以非阻塞的方式为Slaves提供服务。所以在Master-Slave同步期间，客户端仍然可以提交查询或修改请求。
* Slave Server同样是以非阻塞的方式完成数据同步。在同步期间，如果有客户端提交查询请求，Redis则返回同步之前的数据

**缺点：**

* Redis不具备自动容错和恢复功能，主机从机的宕机都会导致前端部分读写请求失败，需要等待机器重启或者手动切换前端的IP才能恢复。
* 主机宕机，宕机前有部分数据未能及时同步到从机，切换IP后还会引入数据不一致的问题，降低了系统的可用性。
* Redis较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂。
