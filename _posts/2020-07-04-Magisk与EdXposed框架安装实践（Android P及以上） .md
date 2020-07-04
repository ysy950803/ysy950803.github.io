---
layout:     post
title:      Magisk与EdXposed框架安装实践（Android P及以上）
subtitle:   鼓捣鼓捣。
date:       2020-07-04
author:     YSY
header-img: img/tags-bg.jpg
catalog: true
tags:
    - 鼓捣折腾
    - Android
---

记得初中的时候还在用Android 2.3，高中开始刷机，每个月都去追论坛大佬的自定义精简ROM，不亦乐乎，这一晃都过了快十年了。从以前旧版Android（支持到8.x）的Xposed到现在的EdXposed、Magisk等玩机框架，大佬些的脚步从未停歇。模块开发生态也好了很多（得益于Github及各种文档）。不过由于现在ROM厂商越来越深度的定制和限制（当然也是考虑到普通用户的安全），ROOT起来也麻烦了不少。

其实我自从以前用Android 4.x时搞过机，后来也很久没接触这些了。最近把手边的测试机器（小米POCO F1，骁龙845，Android 10、MIUI 12）拿来搞了搞，发现操作过程也不是很麻烦，本文就仅作记录吧，方便以后查阅，毕竟各种链接每次都临时搜还是很费事。

## Xposed

由于现在Android新版本的普及，新上市的手机基本都是8.0以上了。所以以前的Xposed框架已经不适用。EdXposed团队成为了后续版本研发的继任者，从[Xpsoed的wikipedia](https://zh.wikipedia.org/zh-hans/Xposed_(框架))中可以查到交接历史：

> 2017年7月，abforce在GitHub上发布了支持Nougat的xposed，不过此发布需在编译ROM前集成在源码中而不是以前直接卡刷的形式。
> 2017年10月，Xposed框架开始支持。[[3\]](https://zh.wikipedia.org/zh-hans/Xposed_(框架)#cite_note-3)
> 2019年1月，ElderDrivers完成了[EdXposed](https://github.com/ElderDrivers/EdXposed)的开发。EdXp是一个Magisk模块，依赖于[riru](https://github.com/RikkaApps/Riru)框架，成功将Xposed移植到了Android Pie上，成为最接近原版Xposed的框架。
> 2019年9月，[EdXposed](https://github.com/ElderDrivers/EdXposed)正式支持Android Q。
> 2020年1月，[EdXposed](https://github.com/ElderDrivers/EdXposed)与Xposed原开发团队达成共识，成为Xposed停止更新后的官方接任者。

当然，我们依然可以下载官方的资源来进行刷机：[https://repo.xposed.info/module/de.robv.android.xposed.installer](https://repo.xposed.info/module/de.robv.android.xposed.installer)，不过这不是本文重点。

## TWRP Recovery、Magisk、EdXposed

由于我的测试机是Android 10，所以要刷EdXposed。大致思路很简单，也是最容易成功的步骤：先刷三方Rec（这里是TWRP），然后装Magisk，最后通过Magisk装EdXposed。

在开始介绍步骤之前，我想说其实 [https://magisk.me/](https://magisk.me/) 是有热门刷机机型ROOT教程的，比起那些乱七八糟的二手资料要好很多，不过里面博客中的各种下载链接都打不开，后来我找到了替代的网站 [https://www.androidjungles.com/](https://www.androidjungles.com/)。本文所有下载链接都尽可能地提供官方原地址，保证权威有效（文末也会提供所有相关文件的网盘备份）。

**下面开始：**

0、这里默认电脑已经安装好 `adb`、`fastboot` 命令工具。

1、各机型对应的Recovery很可能是不同的，先找自己设备对应的下载：[https://twrp.me/Devices/](https://twrp.me/Devices/) ，比如我这里的机型打开后是这样：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200703235449500.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)

我们点Download Links中的链接，2选1即可，反正不管美洲还是欧洲速度都很慢。进去后选最新版本的img文件，再点进去进行下载。我这里最后下载下来是：[twrp-3.3.0-0-beryllium.img](https://dl.twrp.me/beryllium/twrp-3.3.0-0-beryllium.img) ，下载完后我们把它重命名成 `recovery.img` ，后面有用。

2、下载 [Disable-Force-Encryption-Treble.zip](https://drive.google.com/file/d/1As6z5v7NEIfOk67jXHBX34cbnFZdEjy9/view) ，后面在Rec中要用到，**不过这个和机型有关，部分机型可能不需要**，具体可查看 [https://magisk.me/](https://magisk.me/) 中的教程。

3、到Magisk的官方下载页面下载最新的Zip和Apk：[https://www.download-magisk.com](https://www.download-magisk.com) ，最终资源文件其实都在Github仓库中。我这里对应要下载的就是：

- [Magisk-v20.4.zip](https://github.com/topjohnwu/Magisk/releases/download/v20.4/Magisk-v20.4.zip)
- [MagiskManager-v7.5.1.apk](https://github.com/topjohnwu/Magisk/releases/download/manager-v7.5.1/MagiskManager-v7.5.1.apk)

4、上面下载好的两个zip都拷贝到手机里，然后打开终端，开始搞事情：

```bash
# 手机电脑连起来，先重启手机进入fastboot模式，命令和手动均可
adb reboot bootloader
# 然后把刚才下载并重命名好的img刷入
fastboot flash recovery recovery.img
# 执行完此就会自动进入Recovery模式
fastboot boot recovery.img
```

5、（无需安装这个zip的机型可忽略这一步）进入Rec后，可能第一次会有提示，点选 **Keep Read Only** ，然后在主界面点 **Install** ，选择刚才下载的 **DisableForceEncryption_Treble.zip** ，右滑Swipe安装。安装完后别急着点Reboot System，为了方便，按**返回键**到主界面中点击 **Reboot** ，选 **Recovery** 重启后自动进入Rec模式。

6、再次进入Rec，还是点Install，然后选刚才下载的Magisk-xxx.zip，进行安装（操作图文可见官方：[https://www.download-magisk.com](https://www.download-magisk.com)）。完事直接点Reboot System重启进入系统即可。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200704004046689.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)

7、不出意外，进入系统后，桌面上就会出现 **Magisk Manager** 的图标（这里请忽略EdXposed，那是后续手动装的），说明安装成功（所以刚才下载的Apk并没有太大用哈哈，当然可以覆盖安装一下保证版本最新）。打开Manager应该可以看到两项都安装成功了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200704004304427.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)

8、后面就简单了，参考EdXposed官方文档（[https://github.com/ElderDrivers/EdXposed](https://github.com/ElderDrivers/EdXposed)）的Install步骤即可：

- 在Magisk Manager的“下载”中安装 Riru（Riru - Core）和 Riru - EdXposed 后重启手机。
- 去 [https://github.com/ElderDrivers/EdXposedManager/releases](https://github.com/ElderDrivers/EdXposedManager/releases) 下载EdXposed Manager的Apk来安装。大功告成！

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200704012136271.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)

## 其他

- img文件重命名为recovery目的是为了覆盖系统原有的recovery，这样在用音量键上+电源键时可以手动进入到TWRP Rec。
- 建议全程保证网络能访问谷歌（你懂的），包括手机和电脑。否则，Manager中可能会出现一直在检查更新和模块列表加载不出来也下载安装不了的情况。
- 在上述第6步进行Magisk安装之前，某些教程会提到在Rec中 **Wipe** > **Format data** 来清除手机数据后再安装，注意如果你选择了清除数据，之前拷贝到手机中的zip文件会消失，你需要重启进系统再拷贝一次才行，比较麻烦，所以我没做清除的操作，貌似也没什么问题。
- 文件备份链接：https://pan.baidu.com/s/1csL1XgWn9O4HN4E8zLXq2A  密码：13av
