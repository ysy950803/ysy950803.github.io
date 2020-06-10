---
layout:     post
title:      授权后连接MySQL依然被拒绝访问（Access denied for user 'root'@'localhost'）
subtitle:   那是真的牛逼。
date:       2019-03-09
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - 后端
    - Java
    - 问题不大
---

我们在Spring Boot的应用配置中一般都会如下：

```
jdbc:mysql://123.123.123.123:3306/db_name?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true
```
123.123.123.123是MySQL数据库所在主机的IP地址，如果你想要远程访问数据库，就必须授权，一般这样操作：

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'IDENTIFIED BY '12345' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```
12345是你设置的访问密码。
但有时候我们轻量应用服务都是单机的，根本不用远程访问，也就是说上面的配置我会这样写：

```
jdbc:mysql://localhost:3306/byd?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true
```
然后查看Tomcat日志发现Access denied for user 'root'@'localhost'，于是乎我们自然想到上面授权的%并不通配localhost，所以单独给localhost授权：

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost'IDENTIFIED BY '12345' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```
一般到这应该没啥问题了，但我发现我的Web应用还是被拒绝访问，我有点慌了。马上去看mysql的user表：

```sql
SELECT user,host,password FROM mysql.user;
```
出现惊人一幕：

| user | host | password |
| ------ | ------ | ------ |
| root | % | *xxxxxx |
| root | 127.0.0.1 | *yyyyyy |
| root | ::1 | *yyyyyy |
| root | localhost | *xxxxxx |

这里 *xxxxxx 和 *yyyyyy 只是做示范，总之情况就是我刚才给localhost设置的密码和127.0.0.1的密码不一样，这说明以前这个数据库给127.0.0.1单独授权过，所以导致localhost解析到本机地址后密码不对应，最终被拒绝访问。

这里的解决办法就是把它们都设成一样的密码，就OK了：

```sql
UPDATE mysql.user SET password=PASSWORD('123456') WHERE user='root';
```
注意用PASSWORD()函数对密码进行加密，否则会变成明文。

