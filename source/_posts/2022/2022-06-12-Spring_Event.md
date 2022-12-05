---
layout: post
title: Spring事件监听机制原理
category: Spring源码
tags: Spring
date: 2022-06-12
---

<meta name="referrer" content="no-referrer" />

### Spring事件监听机制原理

#### 使用事件发布进行解耦

**定义一个通知事件并继承ApplicationEvent**

```java
public class NotifyEvent extends ApplicationEvent {

    private String str;
    
    private static final long serialVersionUID = 7099057708183571934L;

    public NotifyEvent(Object source, String str) {
        super(source);
        this.str = str;
    }
  
    public String getStr() {
        return str;
    }
}
```

**定义一个监听接口类并实现ApplicationListener**

```java
@Component
public class NotifyListeners implements ApplicationListener<NotifyEvent> {

    @Override
    public void onApplicationEvent(NotifyEvent event) {
        System.out.println("onApplicationEvent: " + event.getStr());
    }
}
```

当NotifyEvent事件发布时，会执行`onApplicationEvent`方法。实现代码解耦！

**发布事件**

```java
@Component
public class PublishEvent {

    @Resource
    //private ApplicationEventMulticaster applicationEventMulticaster;
    private ApplicationEventPublisher eventPublisher;

    private void publishEvent() {
        NotifyEvent event = new NotifyEvent(this,"HelloWorld");
        eventPublisher.publishEvent(event);
    }
  public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext();
        ac.register(SpringConfigurationListeners.class);
        ac.refresh();
        PublishEvent bean = ac.getBean(PublishEvent.class);
        bean.publishEvent();
    }
}
```

#### 源码分析

发布事件

![image-20220905195718379](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220905195718379.png)

![image-20220905195930482](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220905195930482.png)

获取多播器然后发布事件

![image-20220905200038490](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220905200038490.png)

问题1：applicationEventMulticaster事件多播器是什么时候初始化的呢？后面会讲到！

获取事件监听器，然后广播事件。

![image-20220906091802940](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906091802940.png)

**getApplicationListeners获取事件监听器**

![image-20220906094154964](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906094154964.png)

在retrieveApplicationListeners方法中主要是在defaultRetriever对象中的applicationListeners集合中

![image-20220906094634664](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906094634664.png)

**问题2：那么问题又来了defaultRetriever什么时候有值的呢？**

![image-20220906100548746](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906100548746.png)

我们在这里打一个断点，还记得问题1吗？我们也在applicationEventMulticaster属性上打个断点；

![image-20220906100753385](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906100753385.png)

![image-20220906101217066](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906101217066.png)

![image-20220906101139254](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906101139254.png)

```
org.springframework.context.support.AbstractApplicationContext#refresh中的initApplicationEventMulticaster（）方法中进行了初始化；
```

问题1就解决了！那下面我们解决一下问题2：

![image-20220906101951110](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906101951110.png)

![image-20220906102022700](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906102022700.png)

Bean初始化后置处理器

![image-20220906102130914](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906102130914.png)

![image-20220906102211794](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20220906102211794.png)

在初始化NotifyListeners Bean的时候！

