---
layout: post
title: Nacos 长轮询实现方式-配置中心源码
category: 技术杂谈
tags: 技术杂谈
date: 2022-09-03
---

<meta name="referrer" content="no-referrer" />


### Nacos 长轮询实现方式-配置中心源码

前面的文章中我们用代码实现了一个简单的长轮询，重点是在AsyncContext，这个Selevt3.0提供的一个异步处理特性，如果不明白的可以再看下上篇文章[实现一个简单的长轮询](https://juejin.cn/post/7137124336783065125)，建议两篇文章结合来看会更容易理解，建议先看上篇文章！

下面我们看下Nacos是怎么实现的，Nacos的配置中心实现相对来说还是挺容易理解的。我们废话少说，挑重点的说，我们还是用上篇文章的图片作为引入：

![image-20220903115122987](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220903115122987.png)

-   首先客户端发起长轮询请求，监听变更的`dataId+group`，服务端收到客户端的请求，这时会挂起客户端的请求，如果在服务端设计的29.5s之内都没有发生变更，服务端会响应回客户端数据没有变更，客户端会继续发送请求。
-   如果在29.5s之内服务数据发生了变更，服务端会''推送''变更的数据到客户端。

#### Nacos客户端代码实现监听

上面也说到了客户端维护了一个长轮询任务，去检查服务端的配置信息是否发生变更，如果发生了变更，那么客户端拿到变更的groupKey再根据groupKey去获取配置项的最新值。

**首先生成配置服务类ConfigService**

```
         String serverAddr = "localhost";
         String dataId = "dynamic-thread-pool-demo";
         String group = "DEFAULT_GROUP";
         Properties properties = new Properties();
         properties.put("serverAddr", serverAddr);
         ConfigService configService = NacosFactory.createConfigService(properties);
```

通过反射创建NacosConfigService实例

![image-20220827190923203](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4abc79fc4e0d43fabca3fb00286516bd~tplv-k3u1fbpfcp-zoom-1.image)

在NacosConfigServic构造方法中构造了ClientWorker

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ab47b7217844d27bf908140483991e4~tplv-k3u1fbpfcp-zoom-1.image)

看一下ClientWorker构造方法中做了什么事情：我们主要看红色框起来的。

![image-20220826165415883](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3faec31228734db387a134a10bc5fd13~tplv-k3u1fbpfcp-zoom-1.image)

每10毫米执行一次`checkConfigInfo()`方法。

![image-20220826165931737](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2df9a2cf3659436697b70654563f9e78~tplv-k3u1fbpfcp-zoom-1.image)

重点都在LongPollingRunnable的类中的run方法中：

![image-20220826170836024](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f843db3d40a04abda2eb5515716224a2~tplv-k3u1fbpfcp-zoom-1.image)

第一处代码先检查本地是否有配置文件，如果有配置优先会用本地的配置文件，第二步就是访问监听服务端（`/listener`接口）变更的配置`dataId+group（changedGroupKeys）`,如果监听到变化的`changedGroupKeys`执行第三步访问服务端(`getConfig`接口)获取最新配置，第四步客户端继续执行次run方法重复上面的步骤。

我们到`checkUpdateDataIds`方法里面看一下：

![image-20220826172238995](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/086245b880764247bc4cc91332da4ecd~tplv-k3u1fbpfcp-zoom-1.image)

客户端是通过一个 http 的 post 请求去获取服务端的结果的，并且设置了一个超时时间：**30s**。

#### 客户端监听接口: listener

首先我们从客户端发送的 http 请求中可以知道，请求的是服务端的 /v1/cs/configs/listener 这个接口。

我们找到该接口对应的方法，在 ConfigController 类中，如下图所示：

> com.alibaba.nacos.config.server.controller.ConfigController.java

![image-20220826172524132](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9bf258e28a8498f9be898cc9b9c1cf2~tplv-k3u1fbpfcp-zoom-1.image)

Nacos 的服务端是通过 spring 对外提供的 http 服务，对 HttpServletRequest 中的参数进行转换后，然后交给一个叫 inner 的对象去执行。

下面我们进入这个叫 inner 的对象中去，该 inner 对象是 ConfigServletInner 类的实例，具体的方法如下所示：

> com.alibaba.nacos.config.server.controller.ConfigServletInner.java

![image-20220826172656606](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bcc2f72e39b47ea85a8d8ece92b8d20~tplv-k3u1fbpfcp-zoom-1.image)

这就到了长轮询的关键地方。

![image-20220827172954065](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40b815e9f25444ebb79bfb5ae34f001c~tplv-k3u1fbpfcp-zoom-1.image)

服务端拿到客户端提交的超时时间后，又减去了 500ms 也就是说服务端在这里使用了一个比客户端提交的时间少 500ms 的超时时间，也就是 29.5s，这正是为了避免客户端超时，**服务端一定要在客户端http超时之前返回！**

下面我们在看下第一个红色框中的代码：`compareMd5()`方法；这个方法主要对比新旧MD5内容有没有变化。根据`dataId+group`组成的`groupKey`重新查询一下配置内容，对比新的内容和旧的内容有没有变化。这里注意一下旧MD5值是从客户端传过来的，客户端是从缓存cacheMap中拿过来的。这里简单说明一下：

![image-20220827175409625](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d1b1e2e7ae3428da89a9ec273cdd75d~tplv-k3u1fbpfcp-zoom-1.image)

服务端MD5对比代码：

![image-20220827174543709](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c886e97b7e74437cb7d9f12d73b33e2c~tplv-k3u1fbpfcp-zoom-1.image)

如果发生了变化就放到`List<String>changeGroups`集合中返回给客户端。但是如果这时没有发生变化的话服务端就挂起此次请求，利用`AsyncContext asyncContext = req.startAsync();`Selvet3.0新特性开启异步线程，上篇文章中[实现一个简单的长轮询](https://juejin.cn/post/7137124336783065125)有介绍这里就不多说了。

看一下上图中第二个框中的代码，开启一个线程去执行ClientLongPolling类中的run方法并把timeout传了进去，这里是29.5s哦；

![image-20220827181408183](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb3ee2bd32014860a114191129fadbc6~tplv-k3u1fbpfcp-zoom-1.image)

下面可以讲方法分成4步：

0.  创建一个调度的任务，调度的延时时间为 29.5s
0.  将该 ClientLongPolling 自身的实例添加到一个 allSubs（队列，维护长轮询订阅关系） 中去
0.  延时时间到了之后，首先将该 ClientLongPolling 自身的实例从 allSubs 中移除
0.  延时时间到了之后，将null写入 response 返回给客户端

总结上面的功能就是，此方法就是将客户端的请求挂起29.5秒，如果在这29.5s期间配置没有任何的变更就会返回给客户端null，客户端收到null，继续发起请求。

我们知道`AsyncContext`要调用`asyncContext.complete()`方法才说明这次异步调用结束啦。那`sendResponse`方法中肯定调用了`asyncContext.complete()`；这里我就不贴了，给大家说明一下调用链：`sendResponse()`--->`generateResponse()`--->`asyncContext.complete()`；

#### 发布配置接口 publishConfig

到这里只是说明29.5s之后配置没有发生变更，那如果发生变更了怎么又是怎么处理的呢？其实很简单我们接下来看看`ConfigController.publishConfig()`方法；

![image-20220827184018169](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de954f3fc9274c7ca8977d04e828987a~tplv-k3u1fbpfcp-zoom-1.image)

在fireEvent方法中触发了一个 ConfigDataChangeEvent 的事件！

fireEvent 方法实际上是触发的 AbstractEventListener 的 onEvent 方法

![image-20220827184414622](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbfbb807e138469faa98638cb7e1ab69~tplv-k3u1fbpfcp-zoom-1.image)

而 AbstractEventListener 是一个抽象类，所以实际注册的应该是 AbstractEventListener 的子类，所以我们需要找到所有继承自 AbstractEventListener 的类

![image-20220827184637455](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb0387c598774e7991224fe021b59d3e~tplv-k3u1fbpfcp-zoom-1.image)

我们看到了一个比较眼熟的类：就是我们刚刚一直在研究的 LongPollingService

所以当更新了配置之后，实际会调用到 LongPollingService 的 onEvent 方法！

![image-20220827184944299](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edc880746b20447cabaeac81f3071910~tplv-k3u1fbpfcp-zoom-1.image)

在onEvent 方法中执行了一个叫 DataChangeTask 的任务；猜测应该是通过这个DataChangeTask任务告诉客户端配置发生了变更。

![image-20220827185531069](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/41c04bccfa324c50be48e19ffeea0ebd~tplv-k3u1fbpfcp-zoom-1.image)

首先遍历allSubs队列找到与当前发生变更的配置项的 groupKey 相等的 ClientLongPolling 任务，找到之后删除对应的订阅关系，然后响应给客户端变化的groupKey，客户端拿到groupKey去查配置内容；这里又调用了`sendResponse()`方法，和上面定时线程中一样的流程；`sendResponse()`--->`generateResponse()`--->`asyncContext.complete()`；

![image-20220827190357849](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5becf1f046b84c1bbadac3ff01d31312~tplv-k3u1fbpfcp-zoom-1.image)

响应给客户端之后，结束整个异步线程。就完成了一次数据变更的 “推送” 操作了！

Nacos的实现逻辑和上篇[实现一个简单的长轮询](https://juejin.cn/post/7137124336783065125)大同小异，建议两篇文章结合来看会更容易理解，建议先看下上篇文章！如果对你有所帮助还请多多支持。
