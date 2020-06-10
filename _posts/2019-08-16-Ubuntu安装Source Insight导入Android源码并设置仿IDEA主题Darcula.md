---
layout:     post
title:      Ubuntu安装Source Insight导入Android源码并设置仿IDEA主题Darcula
subtitle:   同样适用于Deepin哦！
date:       2019-08-16
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - 鼓捣折腾
    - Android
    - AOSP
    - Linux
---

### 预览
先来张图给大家感受下效果，然后我再慢慢道来过程，保证你避免每一步的坑。
![preview](https://img-blog.csdnimg.cn/20190816214656427.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
### 我们要做什么
1、由于Source Insight（以下简称SI）是Windows应用，所以不能直接安装在Linux上，于是我们要借助wine，所以第一步会介绍wine的安装过程和坑；
2、介绍SI在Linux（以Ubuntu 16.04为例）上的安装，并介绍如何导入免费证书（个人建议大家有条件还是付费支持一下SI这个软件，真的是个好东西，比IDEA看源码实在快太多了，我已经买了最新版4.0，还是挺良心的，一次性付费，239美刀）；
3、用SI导入AOSP源码并Sync文件建立索引，开头那张截图就是正在Sync，建立完成后就可以快速导航各个方法类引用等等了；
4、默认主题太刺眼，怎么办？还是习惯IDEA的风格，没事，我配置都写好了，只用你一键导入并改改字体大小即可！
### 第一步：安装wine
直接命令走起：
```shell
sudo apt-get install wine
```
过程中终端会显示一个确定页面，按Tab键选中并确定即可，然后再选“是”。
一般来说由于依赖问题，上述命令都是不能一次性安装成功的，这时候直接：
```shell
sudo apt-get install -f
```
好，我已经看穿了一切，这一步估计大多数用户依然是要出错的，且错误提示：

> dpkg: 处理归档
> /var/cache/apt/archives/xxx.deb
> (--unpack)时出错：  尝试覆盖共享的……

巴拉巴拉这类鬼东西，如果你也是，那么很好，下面一套连招即可解决：
```shell
sudo mv /var/lib/dpkg/info /var/lib/dpkg/info_back
sudo mkdir /var/lib/dpkg/info
sudo apt-get update
sudo apt-get install -f
sudo mv /var/lib/dpkg/info /var/lib/dpkg/info_back
sudo mv /var/lib/dpkg/info_back /var/lib/dpkg/info
```
安装成功后执行一下wine命令，没提示错误即可。
### 第二步：安装SI
先去官网下载最新版的exe：[https://www.sourceinsight.com/download/](https://www.sourceinsight.com/download/)
我这里版本号是4.0.0098，建议和我一样，如果下载困难的，可直接在我文末的网盘中下载本文所有资源。
用wine命令安装SI：
```shell
wine sourceinsight4098-setup.exe
```
这个时候会弹出Windows程序的安装过程，全部下一步即可，没有特殊配置，安装路径也最好不要改。
安装完成后你在Ubuntu的应用程序里已经可以搜到SI了，桌面上也会自动创建快捷方式：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190816221426934.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
然后不要急着打开，把网盘里下载下来的另一个exe文件复制到SI安装目录中：
```shell
cd ~/.wine/drive_c/Program\ Files\ \(x86\)/Source\ Insight\ 4.0/
cp ~/你下载并解压后的目录文件夹/sourceinsight4.exe sourceinsight4.exe
```
copy覆盖后，再打开SI，此时会弹出授权弹窗，选第三个导入证书，这时候选网盘里下载下来的si4.pediy.lic文件即可。
### 第三步：导入Android源码
成功打开进入SI之后，首次使用会提示import symbol之类的，可以不用管，直接关掉。
然后会继续提示是否创建一个新Project，这个时候就选是了，随便取个名字，然后点 **Browser**，选择自己的源码所在的目录即可，再 **Add All** 并勾选 **子目录** 就可以导入了。
这一步比较简单我就不赘述了，具体图文步骤可参考：[https://www.jianshu.com/p/b7b19bcc0425](https://www.jianshu.com/p/b7b19bcc0425)
**讲几点注意事项：**
1、导入成功后打开了SI主界面，但怎么啥都看不见？不要慌，这时候点一下菜单栏里的 **File > Open** 就好了。
2、如果发现菜单栏里的按钮都点不动怎么办？这个可能是wine的Bug，不要慌，多点几下SI主界面中间的空白处或者最小化再最大化一下，再去点 File 菜单，看是不是可以获取焦点了？
3、到此我们总算可以查看源码了，但按住Ctrl点击各种方法和变量等等，怎么提示 **Symbol Not Found** 呢？这是没建立索引的原因，选择菜单栏里的 **Project > Synchronize Files** 即可， **这个过程非常久，可能要数小时** （如果你导入了全部AOSP源码的话），总之一定要耐心等待，中途可能会导致SI整个界面停止响应，不要动，过一会儿就好了，Sync这一次以后就再也不用了（除非你文件有较大变动）。
如果Sync实在太慢，最开始就不要Add太多文件，选择几个需要的，再Add Tree即可。
### 第四步：SI主题改为IDEA的Darcula暗黑风格
1、默认字体太小了，我们先改改字体大小：
按 **Alt + Y** 快捷键（或者菜单栏 **Options > File Type Options**），然后改你喜欢的字体和大小即可：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190816223801624.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
改后确定会弹出一个窗口让你输入 **yes** ，是否应用于所有文件类型，我一般都是yes。
2、如何设置和IDEA一样的暗黑风格，如果不想自己一点一点手工DIY的同学，可以直接按我的来做：
先把整体主题设为自带的Black：
点击 **Options > Visual Theme > Black**，
然后点击菜单栏 **Options > Style Properties** ，在弹出的窗口中选择右边的 **Load** 按钮，选择网盘中下载的 **darcula-as.xml** 文件即可，Done。
### 附件
链接: [https://pan.baidu.com/s/1wVI61SDojBvxffNHct6NHQ](https://pan.baidu.com/s/1wVI61SDojBvxffNHct6NHQ) 提取码: **ij4s** 复制这段内容后打开百度网盘手机App，操作更方便哦！
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190816230010151.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
