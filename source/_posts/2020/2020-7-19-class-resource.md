---
layout: post
title: S关于使用this.getClass().getResource()获取文件时遇到的坑
category: 技术杂谈
tags: 技术杂谈
date: 2020-07-19
---

<meta name="referrer" content="no-referrer" />



#### 关于使用this.getClass().getResource()获取文件时遇到的坑

最近在工作中遇到需要读取配置文件，然后第一想法就是将文件放到项目的`resources`目录下,

然后使用：

```java
String fileName = "config/zh.md"
String path = this.getClass().getResource("/").getPath()  + fileName;
System.out.println(path);// D:/example/exam01/target/classes/config/zh.md
```

在IDE工具中开发及Debug时一切都正常，但是打成Jar包发布到线上时就会出现java.io.FileNotFoundException

```
java.io.FileNotFoundException: file:/usr/local/exam01-1.0-SNAPSHOT.jar!/BOOT-INF/classes!/config/zh.md (No such file or directory)
```

错误信息也已经很明显了，就是因为文件不存在，但是在IDE中是可以正常运行了，那为什么打成jar包放到服务器中就不行了呢？

仔细检查报错路径发现在磁盘确实不存在这样一条路径，因为路径从 `.../exam01-1.0-SNAPSHOT.jar/...`开始，后面的文件路径都是打到Jar包中的，磁盘没有后面 `.../BOOT-INF/classes!/config/zh.md`这样的目录；

在Jar包中的文件在磁盘是没有实际路径的，所以这时候通过 `this.getClass()..getResource()` 无法获取文件。

**解决方式一**

直接将需要的文件上传到服务器指定的文件夹下，如果把文件路径写死，就太low了，也不符合编码规范。而且存在各种隐患例如：不同的环境发布到不同的服务器上，开发一个服务器，测试一个服务器，生产一个服务器，每个服务器中都要上传一份；如果误删或者迁移项目忘记迁移这个文件就麻烦了；

**解决方式二**

可以通过 `this.getClass()..getResourceAsStream("/config/zh.md")` 能够正常获取到文件流。

**xxx.class.getResource("")** 和 **xxx.class.getClassLoader().getResource("")**

上面问题已经解决了，我们看下**xxx.class.getResource("")** 和 **xxx.class.getClassLoader().getResource("")**的区别；



![image-20200719184851542](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20200719184851542.png)

1,其实，`class.getResource("/") == class.getClassLoader().getResource("")；`

Class.getResource和ClassLoader.getResource本质上是一样的，都是使用ClassLoader.getResource加载资源的。

**对于Class.getResource：**

先获取文件的路径path，不以'/'开头时，默认是从此类所在的包下取资源；path以'/'开头时，则是从项目的ClassPath根下获取资源。

**对于ClassLoader.getResource：**

同样先获取文件的路径，path不以'/'开头时，首先通过双亲委派机制，使用的逐级向上委托的形式加载的，最后发现双亲没有加载到文件，最后通过当前类加载classpath根下资源文件。

对于getResource("/")，'/'表示Boot ClassLoader中的加载范围，因为这个类加载器是C++实现的，所以加载范围为null。

2，以上两种方法返回的都是 java.net.URL对象，如果需要得到相应的String类型，可以用以下方法：

`xxx.class.getResource("").getPath();`

`xxx.class.getResource("").getFile();`

或者通过`InputStream input = getClass().getClassLoader().getResourceAsStream("config\\config.properties"); `获取IO流;

#### 3.类加载器ClassLoader

我们都知道 Java 文件被运行，第一步，需要通过 `javac` 编译器编译为 class 文件；第二步，JVM 运行 class 文件，实现跨平台。

而 JVM 虚拟机第一步肯定是 **加载 class 文件**，所以，**类加载器**实现的就是（来自《深入理解Java虚拟机》）：

通过一个类的全限定名来获取描述此类的二进制字节流

类加载器有几个重要的特性：

* 每个类加载器都有自己的预定义的**搜索范围**，用来加载 class 文件；

*  每个类和加载它的类加载器共同确定了这个类的唯一性，也就是说如果一个 class 文件被**不同的类加载器加载**到了 JVM 中， 那么这两个类就是不同的类，虽然他们都来自同一份 class 文件；

 **3.1 双亲委派模型**

*  所有的类加载器都是有层级结构的，每个类加载器都有一个父类类加载器（通过组合实现，而不是继承），除了**启动类加载器（Bootstrap ClassLoader）**

*  当一个类加载器接收到一个类加载请求时，首先将这个请求委派给它的父加载器去加载，所以每个类加载请求最终都会传递到顶层的启动类加载器，如果父加载器无法加载时，子类加载器才会去尝试自己去加载；

通过双亲委派模型就实现了类加载器的三个特性：

**委派（delegation）**：子类加载器委派给父类加载器加载；

**可见性（visibility）**：子类加载器可访问父类加载器加载的类，父类不能访问子类加载器加载的类；

**唯一性（uniqueness）**：可保证每个类只被加载一次，比如 `Object` 类是被 Bootstrap ClassLoader 加载的，因为有了双亲委派模型，所有的 Object 类加载请求都委派到了 Bootstrap ClassLoader，所以保证了只被加载一次。

**3.2 Java 中的类加载器**

从 JVM 虚拟机的角度来看，只存在两种不同的类加载器：

*  启动类加载器（Bootstrap ClassLoader），是虚拟机自身的一部分；

*  所有其他的类加载器，独立于虚拟机外部，都继承自抽象类 `java.lang.ClassLoader`

而绝大多数 Java 应用都会用到如下 3 中系统提供的类加载器：

*  **启动类加载器（Bootstrap/Primordial/NULL ClassLoader）**：顶层的类加载器，没有父类加载器。负责加载 /lib 目录下的，或则被 -Xbootclasspath 参数所指定路径中的，

并被 JVM 识别的（仅按文件名识别，如 rt.jar，名字不符合的类库即使放在 lib 目录也不会被加载）类库加载到虚拟机内存中。所有被 Bootstrap classloader 加载的类，

它的 Class.getClassLoader 方法返回的都是 null，所以也称作 NULL ClassLoader。

*  **扩展类加载器（Extension CLassLoader）**：由 `sun.misc.Launcher$ExtClassLoader` 实现，负责加载 `<JAVA_HOME>/lib/ext` 目录下，或被 `java.ext.dirs` 系统变量所指定的目录下的所有类库；

* **应用程序类加载器（Application/System ClassLoader）**：由 `sun.misc.Launcher$AppClassLoader` 实现。它是 ClassLoader.getSystemClassLoader() 方法的默认返回值，

所以也称为系统类加载器（System ClassLoader）。它负责加载 classpath 下所指定的类库，如果应用程序没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

如下，就是 Java 程序中的类加载器层级结构图：

![image-20200719214403877](http://cg-mall.oss-cn-shanghai.aliyuncs.com/blog/image-20200719214403877.png)