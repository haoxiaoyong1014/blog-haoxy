---
layout: post
title: .yml .properties文件下的参数值替换
category: Springboot
tags: springboot
toc: true
date: 2019-09-11
---

在应用springboot开发时,经常将一些配置信息记录到.yml文件中,或者记录到.properties文件中,举个晓丽子,当我们在介入微信登录,或者微信支付等第三方接口时,我们会将
微信api接口的路径 url 配置到.yml 或者.properties文件中;我们就以下面这个 url 为例:

`https://api.weixin.qq.com/sns/oauth2/access_token?appid=PPID&secret=SECRET&code=CODE&grant_type=authorization_code`

这是获取微信的access_token的接口,这里很明显要传入四个参数,`appid`,`secret`,`code`,`grant_type`,由于参数值是可变的,所以我们通常不会像下面那样直接配置到配置文件中;

下面是错误示范:

`wechatOAuthTokenUrl:https://api.weixin.qq.com/sns/oauth2/access_token?appid=PPID&secret=SECRET&code=CODE&grant_type=authorization_code`

如果我们在程序中提取wechatOAuthTokenUrl变量的值，那么对参数的赋值操作将变成一件非常棘手的事情。通常的做法是我们只配置url地址的前一小部分，即相当长时间不会改变的部分，即:
wechatOAuthTokenUrl=https://api.weixin.qq.com/sns/oauth2/access_token
然后再程序中，当我们想要发送请求的时候，只需要将4个参数构造成一个map即可发起请求，示例如下:

```java
public static void main(String[] args) throws IOException {
    // 读取属性文件，一般不会这么做，集成Spring时可做配置
    Properties properties = new Properties();
    InputStream inputStream = new FileInputStream("src/main/resources/wechat.properties");
    properties.load(inputStream);
    
    String wechatOAuthTokenUrl = properties.getProperty("wechatOAuthTokenUrl");
    
    // 参数map
    Map<String, String> map = new HashMap<>();
    map.put("appid", "A");
    map.put("secret", "B");
    map.put("code", "C");
    map.put("grant_type", "D");
    
    // 请求参数，封装了url、参数、header等信息
    RequestParam param = new RequestParam();
    param.setUrl(wechatOAuthTokenUrl);
    param.setBody(map);
    // 发送请求
    HttpClientUtil.getInstance().asyncSend(param);
}
```

那么还有没有其他方式呢？额，有，

**1,采用java.text包下的MessageFormat类**


采用该类，我们只需要将属性配置成如下格式即可：
`wechatOAuthTokenUrl:https://api.weixin.qq.com/sns/oauth2/access_token?appid={0}&secret={1}&code={2}&grant_type={3}`

测试类如下：

```java
@RestController
public class TestJobController {


    @Value("${wechatOAuthTokenUrl}")
    private String wechatOAuthTokenUrl;


    @RequestMapping("/rep")
    public String replaceWxUrl() {
        // 使用MessageFormat对字符串进行参数替换
        String format = MessageFormat.format(wechatOAuthTokenUrl, new Object[]{"PPID", "SECRET", "CODE", "authorization_code"});
        System.out.println(format);
        return format;
    }
}

```

输出结果:

`https://api.weixin.qq.com/sns/oauth2/access_token?appid=PPID&secret=SECRET&code=CODE&grant_type=authorization_code`

**2、采用String.format()进行匹配**

采用此方法，我们只需要将属性配置成如下格式即可:

`wechatOAuthTokenUrl:https://api.weixin.qq.com/sns/oauth2/access_token?appid=%s&secret=%s&code=%s&grant_type=%s`

测试类如下：

```java
@RestController
public class TestJobController {

    @Value("${wechatOAuthTokenUrl}")
    private String wechatOAuthTokenUrl;

    @RequestMapping("/rep2")
    public String replaceWxUrl2() {
        String format = String.format(wechatOAuthTokenUrl, new Object[]{"A", "B", "C", "D"});
        System.out.println(format);
        return format;
    }
}
```
输出结果:

`https://api.weixin.qq.com/sns/oauth2/access_token?appid=A&secret=B&code=C&grant_type=D`