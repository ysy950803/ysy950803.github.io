---
layout:     post
title:      删除Android Studio中重复的JDK配置
subtitle:   又闲下来了。
date:       2023-02-12
author:     YSY
header-img: img/home-bg.jpg
catalog: true
cover: https://imgconvert.csdnimg.cn/33252e541c8d4a02a31c34ee3d2657c1.png
tags:
    - Android
    - Java
    - Gradle
---

### 问题

可能因为一些不经意的操作，导致如下这种情况：出现多余重复的JDK路径配置，其实指向的是同一个路径。

![请添加图片描述](https://imgconvert.csdnimg.cn/33252e541c8d4a02a31c34ee3d2657c1.png)

强迫症犯了之后，就会想怎么干掉这个（2）。

### 解决

第一步：先打开你最近打开的项目，找到 `.idea/misc.xml` 看看里面有没有那个多余的JDK路径，如果有就直接把这个misc.xml文件删掉。然后完全退出整个Android Studio。

![请添加图片描述](https://imgconvert.csdnimg.cn/da2be5b4875f4119add1a7cc9ac0fbfb.png)

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

![请添加图片描述](https://imgconvert.csdnimg.cn/78715499487248dd9e2fe361a4963a20.png)

第三步：编辑这个文件，把里面所有带（2）所在的标签块都删掉，主要是两个标签块：additional和jdk。

![请添加图片描述](https://imgconvert.csdnimg.cn/bb73fc9f497747ed9530d58fb081a3c1.png)
![请添加图片描述](https://imgconvert.csdnimg.cn/b454c2a5a2eb473083261e5bad8e789a.png)

删完了之后保存文件，再打开Android Studio，查看Gradle JDK配置，强迫症治好了。

![请添加图片描述](https://imgconvert.csdnimg.cn/edac2262d762489287d39d5eb3c68a2b.png)
