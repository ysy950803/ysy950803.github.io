---
layout:     post
title:      MyBatis直接使用LocalDateTime时间类型以及MySQL时区问题排错
subtitle:   那是真的牛逼。
date:       2019-03-15
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - 后端
    - Java
    - 问题不大
---

## 时间类型

Java 8提供了新的时间API，相关介绍大家可以自行搜索或者直接参考这篇[Java中的时间与时区](https://blog.csdn.net/u012107143/article/details/78790378)，因此大家在写实体类时，可以放弃用以前的Date或者Timestamp类型了，直接用LocalDateTime类就行了，MyBatis从3.4.5版本开始就完全支持这种类型了，根本不用自己再去写什么类型转换，目前网上搜到的大部分文章还是让我们自己去实现，其实不用的。

我们来看看其官方文档（ https://github.com/mybatis/mybatis-3/releases ）：

> mybatis-3.4.5
@harawata harawata released this on 20 Aug 2017 · 472 commits to master since this release
Enhancements:
Make default enum type handler customizable. #971
Make mapper method and its interface type accessible to SqlProvider. #1055
Allow using configuration properties in SqlProvider. #1061
**Merge type handlers for JSR-310 (Java Date and Time API) into the core. #974**

JSR-310相关规范在这个版本就已经支持了，所以大家只要不小于此版本的就放心用吧。举个例子：

```java
public class User {
    private LocalDateTime createTime;
    // setter ... getter ... 省略了哈
}

public interface UserDao {
    @Insert("INSERT INTO user(create_time) values(#{createTime})")
    int insertUser(User user);
}

// 在set时间时，一般直接用now方法就好
user.setCreateTime(LocalDateTime.now());
```
这样就OK了，如果你数据库表存的是datetime类型的话，MyBatis自动帮你解析转换，你不用做额外工作。

**注意：**
这里LocalDateTime默认是不包含时区信息的，会取当前机器时间的时区，其实一般情况下，是没有问题的，我用阿里云的服务器（在深圳），直接打印出来就是：

```java
System.out.println(LocalDateTime.now());
// 输出的是北京时间
2019-03-15T16:51:37.121
```
当然这样可能你不是很放心，那么就指明时区：

```java
System.out.println(LocalDateTime.now(ZoneId.of("+08:00")));
```

## MySQL时区有问题（相差13或14小时）
这个问题最开始让我非常头疼，明明我的Tomcat和MySQL在同一个服务器上，Java代码打印时间出来都是对的，结果一插入数据库时间就错了。
然后进入数据库查看时间和时区：

```shell
mysql> select curtime();
mysql> show variables like '%time_zone%';
```
发现时间也没问题，都是北京时间，那为什么通过JDBC一插就差那么十几个小时呢？
问题的原因在这里：[一次 JDBC 与 MySQL 因 “CST” 时区协商误解导致时间差了 14 或 13 小时的排错经历](https://juejin.im/post/5902e087da2f60005df05c3d)

**解决办法：**
手动修改MySQL的时区，明确指定：

```shell
mysql> set global time_zone='+08:00';
mysql> set time_zone='+08:00';
mysql> flush privileges;
```
或者修改my.cnf配置文件，一劳永逸，添加：

```
[mysqld]
default-time-zone = '+08:00'
```
即可，最后记得重启MySQL服务，最好还能重启一下Tomcat。
