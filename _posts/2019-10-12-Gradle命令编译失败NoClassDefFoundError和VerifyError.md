---
layout:     post
title:      Gradle命令编译失败NoClassDefFoundError和VerifyError
subtitle:   多加思索。
date:       2019-10-12
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - 问题不大
    - Gradle
    - Android
    - Java
---

### 问题

不知道大家在编译gradle项目的时候习惯直接在Android Studio这种IDE里面还是命令行操作。
今天在Deepin系统里面用命令编译：

```bash
./gradlew assembleDebug
```

直接报错：

> Exception in thread "main" java.lang.NoClassDefFoundError: org.gradle.wrapper.BootstrapMainStarter
>    at java.lang.Class.initializeClass(libgcj.so.17)
>    at org.gradle.wrapper.GradleWrapperMain.main(GradleWrapperMain.java:61)
> Caused by: java.lang.VerifyError: verification failed at PC 96 in org.gradle.wrapper.BootstrapMainStarter:start(([Ljava.lang.String;Ljava.io.File;)V): incompatible type on stack
>    at java.lang.Class.initializeClass(libgcj.so.17)
>    ...1 more

然后用Studio的Build Apk功能又没问题。顿时感觉奇怪。
去谷歌了一下这个错误，没找到什么实用的信息。

### 解决

冷静分析 -> 稍加思索 -> $^&*#@，想到了最开始安装Deepin的时候查到 **java -version** 还是 **1.5** ，以我多年的踩雷经验，应该是Java版本太旧导致的。

顺手一个openjdk8给安排上：

```bash
sudo apt-get install openjdk-8-jdk
```

安装完成，解决了。再次使用gradle命令就不会出错了。
