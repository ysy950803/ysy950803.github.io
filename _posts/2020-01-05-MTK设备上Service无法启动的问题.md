---
layout:     post
title:      MTK设备上DuraSpeed导致Service无法启动的问题
subtitle:   知其所以然。
date:       2020-01-05
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - AOSP
---

> 没想到联发科还会改framework，有点小惊讶……

### 背景

问题的起因是我们的系统应用无法调起与其他部门联动的某个Service组件了。查日志发现有这么一行：

```shell
1276-2330/system_process D/ActivityManager: bringUpServiceLocked, suppress to start service!
```

提示内容大概是说Service的启动被禁止了，当时我比较纳闷，难道谷歌又搞了什么幺蛾子？哈哈，而且最奇怪的是只在那一台特定型号的设备上复现问题，其他机型一切正常。
这台设备最大的不同就是处理器是MTK的，其他正常机型是高通的。

### 探查

既然日志内容如此明显，问题也比较好查了，我们去看看这行log是在Android源码的哪一行出现的。正好最近谷歌推出了官方的源码检索平台：[Android Code Search](https://cs.android.com/)，可以在线搜索AOSP和AndroidX的代码，简直方便。
ActivityManager这个TAG的log有很多地方，我们直接查找 `bringUpServiceLocked` 方法所在的代码即可。
![在这里插入图片描述](https://imgconvert.csdnimg.cn/20200105180914692.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
对应的文件路径为：**frameworks/base/services/core/java/com/android/server/am/ActiveServices.java**，然而呢，我根本没有找到“suppress to start service”这个内容。

很容易想到，这是MTK夹带的私货，当我把源码切到该机型对应分支之后，果然找到了这段代码。由于保密原因这里不方便公开源码哈，但其实逻辑非常简单，就是MTK修改了代码，在 ``bringUpServiceLocked`` 方法内插入了自己的判断：**如果应用包名不在某个特定的白名单里，就会被禁止启动其他应用的Service组件**。其目的是为了防止不同应用之间的相互唤醒，初衷还是好的。但这样一刀切的方式，未免有点不妥。

这个限制其实对应了MTK的一个进程管理功能，叫 **DuraSpeed**，网上很容易搜到相关资料，是MTK为了缓解手机长时间使用后的性能下降问题而开发的，没想到这些逻辑已经植入了framework代码，我原本以为联发科作为一家硬件厂商，只会动framework以下的代码。

### 解决

那么这个问题如何解决呢？由上可知，从应用层肯定是没法子的，只能从底层来解决。直接给系统组的大佬提issue咯！

- **方法一：** 如果有条件修改ROM源码，找到这段代码，改之，这是最简单粗暴的。

- **方法二：** 上面我们也提到了，限制逻辑中有一个白名单，在MTK的独立实现的framework修饰代码中，把包名加进去即可。

- **方法三：** 其实这个 **DuraSpeed** 是一个可选功能，只不过在联发科的源码配置文件中默认开启了。一般来说这个配置文件在ROM源码的 **device/厂商名/机型代号** 目录下面，有一个 **ProjectConfig.mk** 文件，我们可以找到如下内容：

  ```makefile
  MTK_DURASPEED_DEFAULT_ON = yes
  MTK_DURASPEED_SUPPORT = yes
  ```

  将此两者改成no即可关闭该功能。最后我试了一下，重新打包编译ROM后，果然解决了起初的问题。
