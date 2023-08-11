---
title: 解决macOS执行fastboot找不到设备的问题
date: 2023-08-11 15:49:01
cover: image-20230811154542258.png 
tags:
  - Android
  - AOSP
  - 鼓捣折腾
  - 问题不大
---

### 背景

最近准备给我的备用机Redmi Note 11 5G刷个类原生的三方ROM，MIUI实在是用腻了。搜罗了一番，在XDA上找到了一个基于Pixel Experience开发的ROM：[PixelExperience Plus for Redmi Note 11T/11S 5G/11 5G/POCO M4 Pro 5G (everpal)](https://forum.xda-developers.com/t/rom-13-unofficial-oss-vendor-pixelexperience-plus-for-redmi-note-11t-11s-5g-11-5g-poco-m4-pro-5g-everpal.4562525/)，它的实际开源地址是：[github.com/Xiaomi-MT6833/releases/releases/](https://github.com/Xiaomi-MT6833/releases/releases/)，可以直接在里面下载到最新的ROM刷机包和boot.img文件。

下载好`PixelExperience_Plus_everpal-13.0-20230410-1707-UNOFFICIAL.zip`和`boot.img`之后，开始刷机，其实过程非常简单（参考：[小米手机刷PixelExperience系统操作指南](https://miuiver.com/install-pixelexperience-on-xiaomi/)），保证手机已经先解锁BL，并和电脑连接，然后终端执行命令，忽略指南中刷vendor那一步：

```bash
adb reboot-bootloader
fastboot flash boot boot.img
fastboot reboot-recovery
```

进入Recovery后，先清除内部储存和缓存，点击Factory reset，再点击Format data/factory reset，点击format data确认格式化。然后点击右上角箭头图标返回主界面，点击Apply update，再点击Apply from ADB，接着电脑输入命令开始推包刷机：

```bash
adb sideload PixelExperience_Plus_everpal-13.0-20230410-1707-UNOFFICIAL.zip
```

完事之后，点击Reboot system now即可进入新系统！

### 问题

但是，我用macOS，直接就卡在了fastboot这一步，提示`< waiting for any device >`，可我明明已经用USB线连接好手机和电脑了呀！为什么会找不到设备？我以前在Windows和其他macOS上面刷机都是好的。

大概查阅了一下资料，有说adb驱动问题的，有说没开USB调试的，这些对我来说都不是问题，因为在电脑开机状态下，我是可以使用adb命令操作手机的，说明驱动什么的都没问题。

### 解决

思来想去，我发现我用的Apple官方自带的充电数据线（双头Type-C）来连接手机和电脑，会不会是线的问题？马上换成小米官方自带的数据线（单头Type-C），一头插手机，因为现在Mac电脑全是Type-C口了所以另一头插拓展坞，拓展坞再连电脑。

没想到刚一接完，终端就输出刷入boot成功了：

```bash
➜ fastboot flash boot boot.img
< waiting for any device >
Sending 'boot_b' (131072 KB)                       OKAY [  4.519s]
Writing 'boot_b'                                   OKAY [  0.591s]
Finished. Total time: 5.130s
```

原来还真是线材的问题，好气好笑！

搞定此问题之后，接下来按上述步骤无脑执行就OK了。终于可以体验原生Android了，看到Google图标的那一刻还是很激动的。

![image-20230811154542258](image-20230811154542258.png)
