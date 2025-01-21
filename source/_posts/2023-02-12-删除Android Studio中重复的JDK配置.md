---
layout:     post
title:      删除Android Studio中重复的JDK配置
subtitle:   又闲下来了。
date:       2023-02-12
author:     YSY
header-img: img/home-bg.jpg
catalog: true
cover: https://blog.ysy950803.top/img/posts/0792ce0e85c8aadf8a136b1e0eb19253.webp
tags:
    - Android
    - Java
    - Gradle
---

### 问题

可能因为一些不经意的操作，导致如下这种情况：出现多余重复的JDK路径配置，其实指向的是同一个路径。

![](https://blog.ysy950803.top/img/posts/0792ce0e85c8aadf8a136b1e0eb19253.webp)

强迫症犯了之后，就会想怎么干掉这个（2）。

### 解决

第一步：先打开你最近打开的项目，找到 `.idea/misc.xml` 看看里面有没有那个多余的JDK路径，如果有就直接把这个misc.xml文件删掉。然后完全退出整个Android Studio。

![](https://blog.ysy950803.top/img/posts/c44af2030f86e95f619e56c7f7df6254.webp)

第二步：在Android Studio缓存配置目录里找到 `options/jdk.table.xml` ，针对不同的系统路径不太一样：

> Windows
> C:\Users\username\AppData\Roaming\Google\AndroidStudioX.Y
>
> Linux
> /home/username/.config/Google/AndroidStudioX.Y
> 和
> /home/username/.local/share/Google/AndroidStudioX.Y
>
> macOS
> ~/Library/Application Support/Google/AndroidStudioX.Y

以我为例，可见：

![](https://blog.ysy950803.top/img/posts/f5f986644cff58097cb066f2c8c78895.webp)

第三步：编辑这个文件，把里面所有带（2）所在的标签块都删掉，主要是两个标签块：additional和jdk。

![](https://blog.ysy950803.top/img/posts/56aa81c36c9efc013e196c68c39d3b93.webp)
![](https://blog.ysy950803.top/img/posts/ac788720a7a78e40d730b4064804869d.webp)

删完了之后保存文件，再打开Android Studio，查看Gradle JDK配置，强迫症治好了。

![](https://blog.ysy950803.top/img/posts/376d38b36c37f7de9e5f049be9a9b87a.webp)
