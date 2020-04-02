---
layout: post
title: SpringBoot自定义Starter 并制作一个简单的图床
category: SpringBoot
tags: SpringBoot
date: 2020-04-2
---

<meta name="referrer" content="no-referrer" />



上篇博客中讲述了[从SpringBoot源码到自己封装一个Starter](https://juejin.im/post/5e74971b518825492c05279e)，而并没有写一个真正的业务场景，这篇博客将自定义starter 添加第三方组件(图片存储OSS)；并根据自定义的starter制作一个图床；

#### 项目结构：

```
oss-spring-boot-project
│   README.md
│   pom.xml   
└───oss-spring-boot-parent
│		 │  pom.xml
│───oss-spring-boot-autoconfigure
│        │  src
│        │  pom.xml  
└───oss-spring-boot-starter
          │ pom.xml
```
这里放上源码地址看的更明了些！https://github.com/haoxiaoyong1014/oss-spring-boot-project

下层依赖上层；其中主要逻辑都在`oss-spring-boot-autoconfigure`模块中；

#### 开发步骤

##### 创建整体结构

* 创建一个名字为`oss-spring-boot-project`的工程
  * 这个pom.xml中主要存放一些上传到maven中央仓库所需的一些build插件以及module的管理
* 首先创建一个名字为`oss-spring-boot-parent`的module
  * 这个工程主要是对项目依赖版本的管理，避免一些冲突；
* 然后创建一个名字为`oss-spring-boot-autoconfigure`的module
  * 这里是自动装配的主要代码逻辑

* 之后创建一个名字为`oss-spring-boot-starter`的module
  * 这是主要提供给使用者使用的maven dependency

##### 代码逻辑实现

在`oss-spring-boot-autoconfigure`模块下的pom.xml文件中添加依赖：

```xml
	<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>2.1.4.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.aliyun.oss</groupId>
            <artifactId>aliyun-sdk-oss</artifactId>
            <version>3.8.0</version>
        </dependency>
    </dependencies>
```

**创建自动配置类**

```java
@Configuration
@EnableConfigurationProperties(OssProperties.class)
@ConditionalOnClass(OssLocalBean.class)
@ConditionalOnWebApplication
public class OssStarterAutoConfiguration {


    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "oss.config", name = "enable", havingValue = "true")
    public OssLocalBean defaultOssLocalBean(OssProperties ossProperties) {
        OssLocalBean oss = new OssLocalBean();
        oss.setAccessKeyId(ossProperties.getAccessKeyId());
        oss.setAccessKeySecret(ossProperties.getAccessKeySecret());
        oss.setBucketName(ossProperties.getBucketName());
        oss.setEndpoint(ossProperties.getEndpoint());
        oss.setEnable(ossProperties.getEnable());
        return oss;
    }

    @Bean
    @ConditionalOnMissingBean
    public OssService ossService () {
        return new OssService();
    }
}

```

每个注解的具体作用已经在上一篇博客[从SpringBoot源码到自己封装一个Starter](https://juejin.im/post/5e74971b518825492c05279e)说明的很清楚了，这里就不过多的叙述了；

新建一个`OssProperties`,声明该starter的使用者可以配置哪些配置项。

```java
@ConfigurationProperties(prefix = "oss.config")
public class OssProperties {

   private String endpoint;

   private String accessKeyId;

   private String accessKeySecret;

   private String bucketName;

   private Boolean enable;
    //....set
    //....get
}
```

使用过oss的都同学应该都知道除了enable是我自定义的以外其他都是必填项了；

在`resources`目录下新建一个`META-INF`目录并且创建一个`spring.factories`文件

```json
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  cn.haoxiaoyong.oss.starter.config.OssStarterAutoConfiguration
```

下面就是需要我们自己去封装使用oss的功能了，例如上传文件，下载文件，删除文件等等。。。

新建一个`OssService`功能类：

```java
public class OssService {

    @Autowired
    private OssLocalBean ossLocalBean;

    /**
     *	文件形式上传
     * @param key
     * @param file
     * @return
     */
    public String upload(String key, File file) {

        if (ossLocalBean.getEnable()) {

            OSS ossClient = null;
            try {
                ossClient = getClient();
                ossClient.putObject(ossLocalBean.getBucketName(), key, file);

                Date expiration = new Date(System.currentTimeMillis() + 3600L * 1000 * 24 * 365 * 100);
                URL url = ossClient.generatePresignedUrl(ossLocalBean.getBucketName(), key, expiration);
                return ossLocalBean.getProtocol() + url.getHost() + url.getPath();
            } finally {
                ossClient.shutdown();
            }
        }
        return null;
    }

    /**
     * 流式上传
     * @param key
     * @param inputStream
     * @return
     */
    public String upload(String key, InputStream inputStream) {
        if (ossLocalBean.getEnable()) {
            OSS ossClient = null;
            try {
                ossClient = getClient();
                ossClient.putObject(ossLocalBean.getBucketName(), key, inputStream);
                Date expiration = new Date(System.currentTimeMillis() + 3600L * 1000 * 24 * 365 * 100);
                URL url = ossClient.generatePresignedUrl(ossLocalBean.getBucketName(), key, expiration);
                return ossLocalBean.getProtocol() + url.getHost() + url.getPath();

            } finally {
                ossClient.shutdown();

            }
        }
        return null;
    }

    /**
     * 下载文件
     * @param key
     * @return
     * @throws IOException
     */
    public BufferedReader download(String key) throws IOException {
        if (ossLocalBean.getEnable()) {
            OSS ossClient = null;
            BufferedReader reader = null;
            try {
                ossClient = getClient();
                OSSObject ossObject = ossClient.getObject(ossLocalBean.getBucketName(), key);
                reader = new BufferedReader(new InputStreamReader(ossObject.getObjectContent()));
                while (true) {
                    String line = reader.readLine();
                    if (line == null) {
                        break;
                    }
                }
                reader.close();
                return reader;
            } finally {
                ossClient.shutdown();

            }
        }
        return null;
    }

    /**
     * 文件是否存在
     * @param key
     */
   
    public boolean exist(String key) {
        if (ossLocalBean.getEnable()) {

            OSS ossClient = null;

            try {
                ossClient = getClient();
                return ossClient.doesObjectExist(ossLocalBean.getBucketName(), key);
            } finally {
                ossClient.shutdown();

            }
        }
        return false;
    }

    /**
     * 删除文件
     * @param key
     */
    public void delete(String key) {
        if (ossLocalBean.getEnable()) {
            OSS ossClient = null;

            try {
                ossClient = getClient();
                ossClient.deleteObject(ossLocalBean.getBucketName(), key);
            } finally {
                ossClient.shutdown();

            }
        }
    }

    /**
     * get oss Client
     *
     * @return
     */
    private OSS getClient() {
        return new OSSClientBuilder().build(ossLocalBean.getEndpoint(), ossLocalBean.getAccessKeyId(), ossLocalBean.getAccessKeySecret());
    }
```

到这里基本上和上一篇博客[从SpringBoot源码到自己封装一个Starter](https://juejin.im/post/5e74971b518825492c05279e)流程是一致的！

接下来就是将`oss-spring-boot-autoconfigure`模块的`groupId`,`artifactId`,`version`引入到`oss-spring-boot-starter`模块中！

至此整个starter工程就完成了，接下来将其上传到maven中央仓库；这样任何人想使用都直接将

```xml
<dependency>
   <groupId>cn.haoxiaoyong.oss</groupId>
   <artifactId>oss-spring-boot-starter</artifactId>
   <version>0.0.2-beta</version>
</dependency>
```

此maven依赖添加pom.xml文件中即可！下图该项目在maven中央仓库的展示：

<img src="https://user-gold-cdn.xitu.io/2020/4/1/171361e2f2e397c9?w=1189&h=809&f=png&s=67874" style="zoom:60%;" />

<img src="https://user-gold-cdn.xitu.io/2020/4/1/171361e2f55cf00c?w=1278&h=601&f=png&s=75089" style="zoom:67%;" />

该工程代码：[SpringBoot自定义starter 添加第三方组件](https://github.com/haoxiaoyong1014/oss-spring-boot-project);欢迎star!

#### 使用自定义Starter制作一个图床

先看一下效果：

![](https://user-gold-cdn.xitu.io/2020/4/1/171361e2f74eece8?w=1550&h=658&f=gif&s=1018529)

创建`oss-picture-manage`工程；依赖maven依赖

```xml
 <dependencies>
    <dependency>
        <groupId>cn.haoxiaoyong.oss</groupId>
        <artifactId>oss-spring-boot-starter</artifactId>
        <version>0.0.2-beta</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>2.1.8.RELEASE</version>
    </dependency>
 </dependencies>
```

application.yml中配置：

```xml
oss:
  config:
    enable: true
    endpoint: 
    access-key-id: 
    access-key-secret: 
    bucket-name: 
    dir: blog/
```

前端代码：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>pic manage</title>
    <link rel="stylesheet" href="https://unpkg.com/element-ui@2.0.5/lib/theme-chalk/index.css">
    <script src="https://unpkg.com/vue/dist/vue.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/vue"></script>
    <script src="http://cdn.bootcss.com/vue-resource/1.3.4/vue-resource.js"></script>
    <script src="https://unpkg.com/element-ui@2.0.5/lib/index.js"></script>
</head>
<body>
<div id="test">

    <el-upload
            class="upload-demo"
            drag
            action="http://work.haoxiaoyong.cn:8876/upload"
            multiple="true"
            align="center"
            :on-success="handleSuccess"
    >
        <i class="el-icon-upload"></i>
        <div class="el-upload__text" style="font-size: 20px">将文件拖到此处，或<em>点击上传</em></div>
        <!--<div class="el-upload__tip" slot="tip">，且不超过500kb</div>-->
    </el-upload>
</div>

<footer align="center">
    <p>&copy; OSS 图床</p>
</footer>
<style>
    .el-upload-dragger {
        width: 688px;
        height: 300px;
    }
</style>

<script>
    var vue = new Vue({
        el: "#test",
        data: {},
        methods: {
            handleSuccess: function (response) {
                if (response.toString() === "fail") {
                    this.$message.error('上传失败');
                    return;
                }
                var h = this.$createElement;
                this.$notify({
                    title: '上传成功',
                    message: h('p', {style: 'color: teal'}, [
                        h('a', {style: 'word-break: break-word'}, response)
                    ]),
                    type: 'success',
                    duration: 0
                });
            }

        }
    });

</script>
</body>
</html>
```



Controller以及Service:

```java
@RestController
public class UploadController {


    @Autowired
    private UploadService uploadService;

    @RequestMapping("upload")
    public String upload(MultipartFile file) {
        return uploadService.upload(file);
    }
}

@Service
public class UploadService {

    @Autowired
    private OssService ossService;

    @Value("${oss.config.dir}")
    private String fileDir;

    public String upload(MultipartFile file) {
        String url = "fail";
        String fileName = fileDir + file.getOriginalFilename();
        try {
            if (!ossService.exist(fileName)) {
                url = ossService.upload(fileName, file.getInputStream());
                if (!StringUtils.isEmpty(url)) {
                    return url;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return url;
    }
}
```

至此，制作并使用Springboot Starter就完成了！可以直接使用此项目作为博客图床,此项目代码地址：https://github.com/haoxiaoyong1014/oss-picture-manage  欢迎star!

