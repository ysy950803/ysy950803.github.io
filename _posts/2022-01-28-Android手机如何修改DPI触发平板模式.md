---
layout:     post
title:      Android手机如何修改DPI触发平板模式
subtitle:   从最根本动手。
date:       2022-01-28
author:     YSY
header-img: img/404-bg.jpg
catalog: true
tags:
    - Android
    - 鼓捣折腾
---

### 前言

目前，微信可以支持平板和手机同时登录，不过并不是所有人都有Android平板设备。实际上，我们可以修改系统文件来达到目的。

我看了下网上（主要是酷安社区）很多方法其实都已经过时了，包括一些修改工具App。所以你以为改了实际上对微信没用，会发现其他软件都受dpi影响了，但是打开微信还是正常的手机分辨率，也就触发不了平板登录。

### 分析

下面我简单说一下原因，主要是很多ROM随着版本升级，可能那些系统配置的参数字段名称已经变了，如果大家还是一味地改以前的参数自然就没有效果，开发那些工具App的朋友们也不能保证及时更新这些修改。

以我的测试机器Redmi Note 9 Pro为例，使用工具修改dpi为280之后（原dpi为440），会发现 `/system/build.prop` 文件内容末尾追加了 `ro.sf.lcd_density=280`，这很明显不是修改而是新增，说明搭载MIUI 13最新开发版的Note 9 Pro对应的配置参数不是这个。

### 解决

解决方法非常简单，先保证手机已经ROOT，然后拉出配置文件：

```bash
adb root
adb remount
adb disable-verity
adb pull /system/build.prop
```

注意，如果你是第一次执行 `adb disable-verity` 命令，最好重启一下手机。

这里拉出来prop文件是为了方便检索字段和修改内容，修改之前记得备份原文件。打开文件后直接搜索 `density` ，会发现：

```bash
……
persist.miui.density_v2=440
……
```

显而易见，最新的MIUI系统使用了新的字段来配置dpi，而不是以前那个 `ro.sf.lcd_density`，所以直接修改这个就行了，改成280，保存文件，然后push回手机：

```bash
adb push build.prop /system/
```

重启就成功了。看看效果：

![在这里插入图片描述](https://img-blog.csdnimg.cn/667b993406aa418996a27f2422f7df37.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6ZKI5Y-2,size_20,color_FFFFFF,t_70,g_se,x_16)

当然，上述所说的改dpi的方式有利有弊，好处是比较通用，不需要改机型（因为不通过三方工具的话，手动改机型需要具体的机型代号，如下所示，改错了或许开不了机），缺点呢当然就是dpi会影响所有应用（不适合主力机），如果不想其他应用受影响还是改机型比较方便。

```bash
ro.product.system.brand=Xiaomi
ro.product.system.device=elish
ro.product.system.manufacturer=Xiaomi
ro.product.system.model=M2105K81AC
ro.product.system.name=elish
ro.product.system.marketname=Xiaomi Pad 5 Pro
```

### 不是MIUI怎么办

大家必须要知道的是，这些系统配置并不是一成不变的，可能某个Android大版本升级或者厂商自定义就会导致不同机型都不一样。所以还是自己动手去查看文件，搜索相关的关键字，来修改就行了。

此外，我这里跳过了ROOT这一步，大多数手机并不像MIUI开发版ROOT这么简单，主流方法会通过刷三方Rec和Magisk来ROOT，如果你具备这些前提条件，可能本文对你也没什么用了，因为成熟的模块或工具更好用（如果一直有人维护的话）。
