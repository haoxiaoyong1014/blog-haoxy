---
layout: post
title: 别人那没有的工具类
category: utils
tags: utils
date: 2020-03-6
---

<meta name="referrer" content="no-referrer" />

[检查一个对象中的参数值是否为空](#检查一个对象中的参数值是否为空)
[Bean以及集合之间浅拷贝](#Bean以及集合之间浅拷贝)
[List集合根据单多属性条件去重](#List集合根据单或多属性条件去重)

#### 检查一个对象中的参数值是否为空

工作中常用的工具类整理

```java
public class ValidUtils {

    public static <T> JSONObject inspect(T t, String requireParams) {
        JSONObject jsonResult = new JSONObject();
        jsonResult.put("code", CommonResultEnum.CHECK_VALID.code());
        String jsonString = JSONObject.toJSONString(t);
        JSONObject jsonObject = JSONObject.parseObject(jsonString);
        String[] params = requireParams.split(",");
        for (String param : params) {
            if (jsonObject.getString(param) == null || jsonObject.getString(param).isEmpty()) {
                jsonResult.put("code", CommonResultEnum.CHECK_VALID_ERROR.code());
                jsonResult.put("msg", String.format(CommonResultEnum.CHECK_VALID_ERROR.message(), param));
                break;
            }
        }
        return jsonResult;
    }
}

```

测试：

```java
    @Test
    public void testValidUtils() {
        User user = new User();
        user.setUsername("张思睿");
        user.setPassword("123456");
        //user.setAge(23);
        JSONObject jsonResult = ValidUtils.inspect(user, "username,password,age");
        System.out.println(jsonResult.get("code") + ": " + jsonResult.get("msg"));

    }
```

打印：

`20201: age参数为空`

#### Bean以及集合之间浅拷贝

```java
public class ConverterUtils {

    public static <V, C> C convert(V v, Class<C> beanClass) {
        if (ObjectUtils.isEmpty(v)) {
            return null;
        } else {
            C instance = null;
            try {
                instance = beanClass.newInstance();
                BeanUtils.copyProperties(v, instance);
            } catch (Exception var4) {
                var4.printStackTrace();
            }

            return instance;
        }
    }

    public static <V, C> List<C> convertList(List<V> vList, Class<C> beanClass) {
        List<C> cList = new ArrayList();
        if (CollectionUtils.isEmpty(vList)) {
            return cList;
        } else {
            vList.stream().forEach((v) -> {
                cList.add(convert(v, beanClass));
            });
            return cList;
        }
    }
}

```

测试convert方法：

```java
	@Test
    public void testConverterByConverterUtils() {
        User user = User.builder()
                .username("张思睿").password("123456")
                .age(25).gender("男").hobby("女")
                .build();
        Persion persion = ConverterUtils.convert(user, Persion.class);
        System.out.println(persion);
    }
```

打印：

```json
Persion(username=张思睿, age=25, gender=男, hobby=女)
```

测试convertList方法：

```java
private static List<User> users = new ArrayList<>();

    static {
        User user = User.builder()
                .username("张兔").password("123456")
                .age(25).gender("女").hobby("男")
                .build();
        User user2 = User.builder()
                .username("张思睿").password("123456")
                .age(25).gender("男").hobby("女")
                .build();
        users.add(user);
        users.add(user2);
    }

    /**
     * 测试 converterList方法
     * 具体实现 {@link ConverterUtils}
     */
    @Test
    public void testConverterListByConverterUtils() {
        List<Persion> persions = ConverterUtils.convertList(users, Persion.class);
        persions.stream().forEach((persion) -> {
            System.out.println(persion);
        });
    }
```

打印：

```json
Persion(username=张兔, age=25, gender=女, hobby=男)
Persion(username=张思睿, age=25, gender=男, hobby=女)
```
#### List集合根据单或多属性条件去重
```java
public class UniqueUtils {

    public static <T> List<T> distinctByKeys(List<T> t, String... fields) {
        Stream<T> tStream = t.stream().filter(new Predicate<T>() {
            Map<Object, Boolean> seen = new ConcurrentHashMap<>(10);

            @Override
            public boolean test(T t) {
                boolean flag = false;
                try {
                    for (String field : fields) {
                        Field declaredField = t.getClass().getDeclaredField(field);
                        declaredField.setAccessible(true);
                        flag = seen.putIfAbsent(declaredField.get(t), Boolean.TRUE) == null;
                    }
                } catch (NoSuchFieldException | IllegalAccessException e) {
                    e.printStackTrace();
                }
                return flag;
            }
        });
        return tStream.collect(Collectors.toList());
    }
}
```
**测试根据单个属性去重：**
```java
public class UniqueUtilsTest {

    private static List<Persion> list = new ArrayList<>();

    static {
        list.add(new Persion("张思睿", 23));
        list.add(new Persion("里斯", 24));
        list.add(new Persion("王武", 25));
        list.add(new Persion("张思睿", 26));
    }

    /**
     * 测试根据一个属性去重
     */
    @Test
    public void distinctByKeyTest() {
        UniqueUtils.distinctByKeys(list, "username").forEach(
                l -> System.out.println(l.getUsername() + "," + l.getAge()));
    }
}
```
**打印结果：**
```json
张思睿,23
里斯,24
王武,25
```
**测试根据多个属性去重**
```java
public class UniqueUtilsTest {

    private static List<Persion> list2 = new ArrayList<>();

    static {
        list2.add(new Persion("张思睿", 23));
        list2.add(new Persion("里斯", 24));
        list2.add(new Persion("王武", 25));
        list2.add(new Persion("张思睿", 26));
    }
    /**
     * 测试根据多个属性去重
     */
    @Test
    public void distinctByKeysTest() {
        UniqueUtils.distinctByKeys(list2, "username","age").forEach(
                l -> System.out.println(l.getUsername() + "," + l.getAge()));
    }
}

```
**测试结果：**
```json
张思睿,23
里斯,24
王武,25
张思睿,26
```
注意集合中的对象属性值；在`distinctByKeys()`方法中传入了两个属性，当这两个属性的值都一样的时候才会去重；

持续更新中。。。。

具体代码地址：https://github.com/haoxiaoyong1014/common-utils
[备用地址](https://github.com/haoxiaoyong1014/common-utils)
欢迎star!

