---
layout:     post
title:      MyBatis中插入（insert）后返回主键（key）的注解方式
subtitle:   那是真的牛逼。
date:       2019-03-17
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - 后端
    - Java
---

一般我们插入数据后需要知道其自增的主键key是多少，有两种方式：
#### 用@Options注解：
比如这里有个订单（Order）相关的DAO：

```java
public interface OrderDao {
    @Insert("INSERT INTO ...")
    @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
    void insertOrder(Order order);
}
```
这里的keyProperty表示对象中的成员变量，keyColumn表示数据库中的列名，因此我们这里数据库主键名称就是**id** ，其实此处不写keyColumn也是可以的，因为我们只是读不是写。此外，如果你Order实体中主键名称就叫"id"的话，keyProperty也不用写了。
实体类则如下：

```java
public class Order {
    private long id;
    ...
}
```
最后通过 **order.getId()** 方法就能拿到生成的主键，注意这里不是通过insertOrder方法的返回值来拿到的，所以我特意写成了void方法，如果写成int类型，返回的值只是表示本次插入是否成功而已（1 or 0）。

#### 用@SelectKey注解
还是和上面一样：

```java
@Insert("INSERT INTO ...")
@SelectKey(statement = "select last_insert_id()", keyProperty = "id", before = false, resultType = long.class)
void insertOrder(Order order);
```
这里的before指的是select语句是否在insert之前执行，显然我们这里需要先写后读，所以是false。
此外，select last_insert_id()也可以换成 **@@identity** ， 效果一样。

**参考：**
https://www.cnblogs.com/weiyinfu/p/6835301.html#6
https://stackoverflow.com/questions/4283159/how-to-return-ids-on-inserts-with-mybatis-in-mysql-with-annotations
