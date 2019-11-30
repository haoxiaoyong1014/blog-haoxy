---
layout: post
title: Spring源码(一)构建Spring5.x源码
category: Spring源码
tags: spring
date: 2019-11-30
---

<meta name="referrer" content="no-referrer" />

#### 构建Spring5.x源码

* 构建条件 jdk1.8以上
* 下载安装gradle
* clone Spring源码

**Mac 安装gradle**

如果mac上安装了homebrew。用homebrew来安装很方便

命令如下：

`brew install gradle`

安装完毕后，输入：gradle -version 查看是否安装成功!

<img src="https://upload-images.jianshu.io/upload_images/15181329-b5bdfd680cf402fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="carbon.png" style="zoom:50%;" />

**Windows安装gradle**

* 下载:  https://services.gradle.org/distributions/  可以下载gradle-5.x-bin.zip以上的;4.x构建Spring5.x源码的时候回出现点问题;

<img src="https://upload-images.jianshu.io/upload_images/15181329-4a2b685664bed105.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

* 配置环境变量

  新建环境变量名GRADLE_HOME,变量值为Gradle的路径

<img src="https://upload-images.jianshu.io/upload_images/15181329-6a6d3288f27395b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

​	然后将他添加到PATH变量中: %GRADLE_HOME%\bin

<img src="https://upload-images.jianshu.io/upload_images/15181329-442f37dccf39a496.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

* 测试运行

  打卡cmd,运行: gradle -v

**下载 Spring 5.X 源码**

地址：https://github.com/spring-projects/spring-framework

Clone 下来使用Idea打开

<img src="https://upload-images.jianshu.io/upload_images/15181329-45708f1193de1d16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

配置正确gradle的安装路径;如果你的是Mac你的gradle路径可能在/usr/local/Cellar/gradle/5.x/libexec

如果觉得怕内存不足就在 Gradle VM options: `-XX:MaxPermSize=2048m -Xmx2048m -XX:MaxHeapSize=2048m`

显示idea 正在构建

<img src="https://upload-images.jianshu.io/upload_images/15181329-36765b986f3a1dfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

正常情况下，还会报一个错误;

<img src="https://upload-images.jianshu.io/upload_images/15181329-0c3a72d23139ef8b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

不必深究,点击OpenFile,将下面两行注释掉即可;

<img src="https://upload-images.jianshu.io/upload_images/15181329-f2573a7a0fb6e4b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

在重新Build,然后等待,时间可能会长些;最终依赖都下载完成之后

<img src="https://upload-images.jianshu.io/upload_images/15181329-23b0f49f761ef31e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

下面新建一个model 测试是否成功;

右键项目->new->Model

<img src="https://upload-images.jianshu.io/upload_images/15181329-7e4d16f68a01049f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

点击Next

<img src="https://upload-images.jianshu.io/upload_images/15181329-6f2b0ffc679919ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="image.png" style="zoom:50%;" />

Next->Finsh

然后再项目根目录下的`settings.gradle`文件中会出现`include 'test-demo'`;

<img src="https://upload-images.jianshu.io/upload_images/15181329-4ed44c01ad7184a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="carbon (1).png" style="zoom:50%;" />

在此model中的`build.gradle`文件中引入:`compile(project(":spring-context"))`

<img src="https://upload-images.jianshu.io/upload_images/15181329-5fa9f00edfd13898.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="carbon.png" style="zoom:50%;" />

在java 目录中新建UserDao,和 TestBean

```java
public class UserDao {

	public void test(){
		System.out.println("构建成功.....");
	}
}
```

```java
public class TestBean {

	public static void main(String[] args) {
		/**
		 * 测试是否构建成功
		 */
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(UserDao.class);
		UserDao bean = applicationContext.getBean(UserDao.class);
		bean.test();
		applicationContext.close();
 }
}
```

输出`构建成功.....`即ok!