---
layout: post
title: 利用Nacos作为配置中心动态修改线程池核心参数
category: 技术杂谈
tags: 技术杂谈
date: 2022-08-11
---

<meta name="referrer" content="no-referrer" />

#### 利用Nacos作为配置中心动态修改线程池

#### 背景

这篇文章的主要核心原理都来自于这个开源项目[dynamic-tp](https://dynamictp.cn/)，可以说是对这个开源项目的源码分析，也是对这个开源项目中涉及到的技术点进行学习总结。

#### 从这篇文章中能学到的技术点

从这篇文章中能学到的技术点，也就是从这个[dynamic-tp](https://dynamictp.cn/)开源项目中学习到的技术点（这里只列举了这个项目的冰山一角）：

* 利用Nacos作为配置中心，动态监听Nacos配置变更。
* 项目启动之初获取Nacos配置，将yml文件中配置的线程池注册到IOC容器中。这里在文章的最后再介绍，利用Spring提供的扩展点。
* .yml文件映射到具体的对象中，对比是否有变更。
* Spring提供的扩展点学习。

#### 那我们开始吧

以下都是对[dynamic-tp](https://dynamictp.cn/)这个开源项目进行了简化，首先看一下我的Nacos配置以及配置类：

```yml
spring:
  dynamic:
    tp:
      enabled: true
      executors:
        - threadPoolName: commonExecutor
          corePoolSize: 2
          maximumPoolSize: 8
          queueCapacity: 200
          keepAliveTime: 50
          allowCoreThreadTimeOut: false
        - threadPoolName: dynamicThreadPoolExecutor1
          corePoolSize: 4
          maximumPoolSize: 6
          queueCapacity: 400
          keepAliveTime: 50
          allowCoreThreadTimeOut: false  
```

```java
@Data
@ConfigurationProperties(prefix = "spring.dynamic.tp")
public class DynamicThreadPoolProperties {

    /**
     * 是否开始动态配置
     */
    private boolean enabled = true;

    /**
     * 线程池基础配置
     */
    private List<ThreadPoolProperties> executors;

}
```

```java
@Data
public class ThreadPoolProperties {

    /**
     * 线程池的名称
     */
    private String threadPoolName;

    /**
     * 核心线程数
     */
    private int corePoolSize = 1;

    /**
     * 最大线程数
     */
    private int maximumPoolSize = DynamicThreadPoolConst.AVAILABLE_PROCESSORS;

    /**
     * 线程队列数
     */
    private int queueCapacity = 1024;

    /**
     * 是否允许核心线程超时
     */
    private boolean allowCoreThreadTimeOut = false;

    /**
     * 超时时间
     */
    private long keepAliveTime = 30;

    /**
     * Timeout unit.
     */
    private TimeUnit timeUnit = TimeUnit.SECONDS;
}
```

上面这个技术点就不用多说了吧，yml文件配置的内容会映射到`DynamicThreadPoolProperties`类中,多个的话会映射成集合的形式。

介绍完了基础的配置，那我们开始介绍核心一点东西：**监听Nacos配置变更的事件**。

监听Nacos配置变更的方式有多种，可以使用`实现 ApplicationListener<NacosConfigReceivedEvent>`的方式也可以使用注解`@NacosConfigListener`。

下面我们就使用`实现 ApplicationListener<NacosConfigReceivedEvent>`的方式：

```java
@Component
public class NacosRefresher extends AbstractRefresher implements ApplicationListener<NacosConfigReceivedEvent> {

    @Override
    public void onApplicationEvent(NacosConfigReceivedEvent nacosConfigReceivedEvent) {
        refresher(nacosConfigReceivedEvent.getContent());
    }
}
```

当Nacos的配置发生了变更的时候会执行`onApplicationEvent`方法；

AbstractRefresher类提供了refresher方法，至于为啥做成一个抽象的是因为不光Nacos可以作为配置中心，zookeeper也可以。像[dynamic-tp](https://github.com/dromara/dynamic-tp)作者不光提供了Nacos,Zookeeper,还有Apollo。

上面提到了当Nacos的配置发生了变更的时候会执行`onApplicationEvent`方法，`onApplicationEvent`方法中调用了AbstractRefresher类提供的`refresher`方法:

```java
@Slf4j
public abstract class AbstractRefresher {

    @Resource
    private DynamicThreadPoolRegistry dynamicThreadPoolRegistry;

  	//项目启动之初就与.yml文件中的配置项进行映射了
    @Resource
    private DynamicThreadPoolProperties dtpProperties;

    public void refresher(String content) {
        if (StringUtils.isBlank(content)) {
            log.warn("DynamicTp refresh, empty content.");
            return;
        }
        //目前先支持yml解析
        YamlConfigParser yamlParser = new YamlConfigParser();
        Map<Object, Object> yamlMap = yamlParser.doParse(content);
        doRefresher(yamlMap);
    }

    protected void doRefresher(Map<Object, Object> properties) {
        if (MapUtils.isEmpty(properties)) {
            log.warn("DynamicTp refresh, empty properties.");
            return;
        }
        PropertiesBinder.bindDtpProperties(properties,dtpProperties);
        doRefresher(dtpProperties);
    }

    protected void doRefresher(DynamicThreadPoolProperties dtpProperties) {

        dynamicThreadPoolRegistry.refresher(dtpProperties);
    }
}

```

DynamicThreadPoolProperties类在项目启动之初就与.yml文件中的配置项进行映射了，目前这里是原始值。DynamicThreadPoolRegistry类就是核心类了，后面会有介绍，我们先看看`refresher`方法。

在`refresher`方法中有个yml文件解析类：YamlConfigParser类：

```java
public class YamlConfigParser {
    public Map<Object, Object> doParse(String content) {

        if (StringUtils.isEmpty(content)) {
            return Collections.emptyMap();
        }
        YamlPropertiesFactoryBean bean = new YamlPropertiesFactoryBean();
        bean.setResources(new ByteArrayResource(content.getBytes()));
        return bean.getObject();
    }
}
```

YamlPropertiesFactoryBean是Spring提供给我们使用yml文件解析类，将yml配置`content`内容传进去会解析成Map结构：

![image.png](https://upload-images.jianshu.io/upload_images/15181329-e59647d20a2dd49e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是不是也是个技术点！

紧接着下面还有一个技术点，我们看下`doRefresher`方法中：`PropertiesBinder.bindDtpProperties(properties,dtpProperties);`

`properties`：是从变更的yml文件中解析出来的Map，是变更之后的新内容。

`dtpProperties`:是项目启动之初从原始yml文件中解析出来的内容，是旧内容。

当这两个参数传入到`PropertiesBinder.bindDtpProperties(properties,dtpProperties)`方法中，`dtpProperties`对像中的值就变成了最新的。

![image.png](https://upload-images.jianshu.io/upload_images/15181329-8130b7c87ec13ddf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是springBoot给我们提供的，是不是又学到了。这也是学习开源项目的好处，会学到一些捷径。

业务开发中如果你需要对比两个对象的属性值差别，也有一个开源的工具：[equator](https://github.com/dadiyang/equator) 感兴趣可以去看下，这个工具也是在[dynamic-tp](https://github.com/dromara/dynamic-tp)开源项目中学到的。

下面就到了我们的核心类：DynamicThreadPoolRegistry

```java
@Slf4j
@Component
public class DynamicThreadPoolRegistry {
		
  	//这里就不用多说了，项目中所有配置的线程池都会注入到这个Map结构中，以线程池的名称为key
    @Resource
    private Map<String, ThreadPoolExecutor> threadPoolExecutorMap = new HashMap<>();

    public void refresher(DynamicThreadPoolProperties dtpProperties) {
        if (Objects.isNull(dtpProperties) || CollectionUtils.isEmpty(dtpProperties.getExecutors())) {
            log.warn("DynamicTp refresh, empty threadPoolProperties.");
            return;
        }
				//遍历Map,给线程池重新设置值。
        dtpProperties.getExecutors().forEach(x -> {
            if (StringUtils.isBlank(x.getThreadPoolName())) {
                log.warn("DynamicTp refresh, threadPoolName must not be empty.");
                return;
            }
            ThreadPoolExecutor executor = threadPoolExecutorMap.get(x.getThreadPoolName());
          	//具体的设置方法
            refresher(executor, x);
        });
    }

    private void refresher(ThreadPoolExecutor executor, ThreadPoolProperties properties) {
        if (properties.getCorePoolSize() < 0 ||
                properties.getMaximumPoolSize() <= 0 ||
                properties.getMaximumPoolSize() < properties.getCorePoolSize() ||
                properties.getKeepAliveTime() < 0) {

            log.error("DynamicTp refresh,  invalid configuration: {}", properties);
            return;
        }
      
        doRefresher(executor, properties);
    }

    private void doRefresher(ThreadPoolExecutor executor, ThreadPoolProperties properties) {

        executor.setCorePoolSize(properties.getCorePoolSize());
        executor.setMaximumPoolSize(properties.getMaximumPoolSize());
        executor.setKeepAliveTime(properties.getKeepAliveTime(), properties.getTimeUnit());
        executor.allowCoreThreadTimeOut(properties.isAllowCoreThreadTimeOut());
        val blockingQueue = executor.getQueue();
        if (blockingQueue instanceof LinkedBlockingQueue) {
            log.warn("DynamicTp refresh, {} :capacity cannot be changed.", blockingQueue);
        }
        if (blockingQueue instanceof VariableLinkedBlockingQueue) {
            ((VariableLinkedBlockingQueue<Runnable>) blockingQueue).setCapacity(properties.getQueueCapacity());
        }
    }
}
```

原理就是线程池ThreadPoolExecutor自己提供的方法：

![image.png](https://upload-images.jianshu.io/upload_images/15181329-46fab9c23bea361a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此动态修改线程池的参数就讲述完了。示例中涉及到代码没有完全贴出来，此示例代码在github上[dynamic-thread-pool](https://github.com/haoxiaoyong1014/dynamic-thread-pool)

在这里你会看到一个有点陌生的队列`VariableLinkedBlockingQueue`,这个队列是可以改变容量的，我们一般创建线程池时使用的队列是`LinkedBlockingQueue`，它的默认大小是`Integer.MAX_VALUE`容易内存溢出，当然你也可以在构造`LinkedBlockingQueue`时指定capacity容量大小，但是capacity是final修饰是没有办法改变的，`VariableLinkedBlockingQueue`队列就给它完善了这一点。使用`VariableLinkedBlockingQueue`队列我们就可以根据自身业务发展情况动态设置队列容量的大小。当然在[dynamic-tp](https://github.com/dromara/dynamic-tp)项目中还有设计比较巧妙的队列`MemorySafeLinkedBlockingQueue`，内存安全队列。这些队列都是可以拿到自己项目中直接使用的。这两个队列在很多开源项目中都使用到了，设计确实很巧妙。在这个项目中学习到了很多。

**到这里我们还有一个Spring提供的扩展技术点没有讲：**

>  项目启动之初获取Nacos配置，将yml文件中配置的线程池注册到IOC容器中

我的示例中没有涉及到这一点，但是dynamic-tp项目中是涉及到了的。下面我们就简单介绍一下这个扩展点，不光Mybatis使用到这个扩展点，很多需要融合Spring的开源框架都会使用到：Nacos，Dubbo，Apollo，RocketMQ太多了就不列举了。

`项目启动之初获取Nacos配置`这个上面也介绍了一种方式使用`@ConfigurationProperties(prefix = "spring.dynamic.tp")`将配置映射到指定的对象中，当然这个是有条件的，就是在使用此对象的类必须是Spring Bean(已经在IOC容器中)。

那如果不是呢？当然有别的方法，可以在Environment中获取。既然说到这里就把方法贴出来吧：

```java
@Slf4j
public class DtpBeanDefinitionRegistrar implements EnvironmentAware {

    private Environment environment;

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

       	DynamicThreadPoolProperties dtpProperties = new DynamicThreadPoolProperties();
        PropertiesBinder.bindDtpProperties(environment, dtpProperties);
        List<ThreadPoolProperties> executors = dtpProperties.getExecutors();
    }
}  
```

利用了SpringBoot给我们提供的捷径`PropertiesBinder.bindDtpProperties();`，这样`dtpProperties`对象中就有值了。和上面那个方法只是入参不一样，上面的入参是一个Map结构，这里是一个Environment对象。

下面我们继续讲我们的Sping扩展点：`将yml文件中配置的线程池注册到IOC容器中`。或者说将外部配置的对象，再或者说指定的类，注入到IOC容器中，在不适用@Bean，@Component，@ComponentScan时，我们使用 ImportBeanDefinitionRegistrar接口。我们先来一个简单的列子：

首先我定一个注解（模仿Mybatis的@Mapper）：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed
public @interface HXY {

    String value() default "";
}
```



```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
			  //Map<String, Object> annotationAttributes = annotationMetadata.getAnnotationAttributes(ComponentScan.class.getName());
        //String[] basePackages = (String[]) annotationAttributes.get("basePackages");

        MyClassPathBeanDefinitionScanner scanner = new MyClassPathBeanDefinitionScanner(beanDefinitionRegistry, false);
				//指定加上@HXY注解的类放入IOC容器成为Bean
        scanner.addIncludeFilter(new AnnotationTypeFilter(HXY.class));
        scanner.doScan("cn.haoxiaoyong.example");
        //scanner.doScan(basePackages);
    }
}
```

这里要借助`ClassPathBeanDefinitionScanner`

```java
public class MyClassPathBeanDefinitionScanner extends ClassPathBeanDefinitionScanner {
    
    public MyClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
        super(registry, useDefaultFilters);
    }
    @Override
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        return super.doScan(basePackages);
    }
    @Override
    public void addIncludeFilter(TypeFilter includeFilter) {
        super.addIncludeFilter(includeFilter);
    }
}
```

只要加上@HXY注解的类就会注册到IOC成为Spring Bean。这也是@Mapper的实现方式。

如果你不想使用注解可以更简单一点：

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
	
    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {

        beanDefinitionRegistry.registerBeanDefinition("testHxyBean",new RootBeanDefinition(TestHxyBean.class)); 
    }
}
```

**注意**：要在启动类，或者配置类上加`@Import(MyImportBeanDefinitionRegistrar.class)`

在[dynamic-tp](https://github.com/dromara/dynamic-tp)项目中使用了构造方法注入的，原理就是：拿到yml文件中配置的线程池构造参数，我们一般构造一个线程池都需要：

```java
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue)
```

在yml文件中我们也配置了这些构造参数。我们还配置线程池的名称，然后利用BeanDefinitionBuilder填充ConstructorArgumentValues对象和MutablePropertyValues；最后还是要调用BeanDefinitionRegistry类的`registerBeanDefinition`方法注册Bean。具体的可以看下这个开源项目。至此这个扩展点也介绍完了。欢迎mark！

鸣谢：

yanhom大佬

dynamic-tp项目地址：https://github.com/dromara/dynamic-tp

dynamic-tp官网：https://dynamictp.cn/

