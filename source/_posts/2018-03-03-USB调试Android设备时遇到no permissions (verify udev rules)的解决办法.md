---
layout:     post
title:      USB调试Android设备时遇到no permissions (verify udev rules)的解决办法
subtitle:   开阔视野。
date:       2018-03-03
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - 问题不大
    - Android
---

最近在Ubuntu上调试一些Android O系统的手机，出现adb root失败的情况，明明手机已经root了。
具体就是在执行adb devices查看连接的设备时，设备后显示 **no permissions (verify udev rules)** 字样。
根据官网的提示，去查文档：https://developer.android.com/studio/run/device.html#setting-up
可以看到下面的解决办法：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMzAzMTMzNjAxNTk4?x-oss-process=image/format,png)

------

也就是说，我们需要在 **/etc/udev/rules.d/** 下面创建一个 **51-android.rules** 文件，
我比较习惯用gedit，所以直接 **sudo gedit /etc/udev/rules.d/51-android.rules** 
如果是高通芯片的手机，直接在文件中输入：

```
SUBSYSTEM=="usb", ATTR{idVendor}=="05c6", MODE="0666", GROUP="plugdev"
```

并保存即可，再次adb root时，如果还失败，记得在手机上选择连接方式为传输文件，而不是只充电。

------

**提示：** 如果是别的机型，可能那个idVendor参数不一样，高通对应是05c6，具体可以查看谷歌的文档：
[https://developer.android.com/studio/run/device.html#VendorIds](https://developer.android.com/studio/run/device.html#VendorIds)
另外，要查看自己USB所连接的机型信息，可以用lsusb命令看。若出现Qualcomm，肯定就是高通了。
