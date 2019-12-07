---
layout:     post
title:      Deepin系统add-apt-repository失败的问题
subtitle:   专治各种小毛病。
date:       2019-10-28
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Deepin
    - Linux
---

> 不只是安装software-properties-common那么简单……

------

#### 问题

目前Deepin系统版本为15.11，貌似在15.10时切到了Debain的软件仓库，而不再是Ubuntu了，虽说比较稳定，但还不够新，比如git，版本就没有官方的新。
于是我尝试 [git官方的办法](https://git-scm.com/download/linux)：

```bash
sudo add-apt-repository ppa:git-core/ppa
```

报错，提示未找到命令。

#### 解决

这个问题在网上搜搜办法，很多回答都是让安装一个包：

```bash
sudo apt-get install software-properties-common
```

是的，确实解决了 **add-apt-repository** 命令找不到的问题，但实际add仓库源时，还是会出错，你可能会看到如下内容：

> Traceback (most recent call last):
>   File "/usr/bin/add-apt-repository", line 95, in <module>
>     sp = SoftwareProperties(options=options)
>   File "/usr/lib/python3/dist-packages/softwareproperties/SoftwareProperties.py", line 109, in __init__
>     self.reload_sourceslist()
>   File "/usr/lib/python3/dist-packages/softwareproperties/SoftwareProperties.py", line 599, in reload_sourceslist
>     self.distro.get_sources(self.sourceslist)    
>   File "/usr/lib/python3/dist-packages/aptsources/distro.py", line 93, in get_sources
>     (self.id, self.codename))
> aptsources.distro.NoDistroTemplateException: Error: could not find a distribution template for Deepin/stable

很尴尬啊兄弟！看这个报错就大概明白，还是因为Deepin/stable不适用这种三方ppa导致的。

我可不想就此放弃，经过一番仔细搜寻，找到一个[大佬的解决办法](https://bbs.deepin.org/forum.php?mod=viewthread&tid=177967)：

1. 编辑lsb-release文件：

   ```bash
   sudo deepin-editor /etc/lsb-release
   ```

2. 把已有内容的每行头加#注释掉，添加Ubuntu相关的内容：

   ```bash
   # DISTRIB_ID=Deepin
   # DISTRIB_RELEASE="15.11"
   # DISTRIB_DESCRIPTION="Deepin 15.11 "
   # DISTRIB_CODENAME=stable
   
   DISTRIB_ID=Ubuntu
   DISTRIB_RELEASE=16.04
   DISTRIB_DESCRIPTION="Ubuntu 16.04.6 LTS"
   DISTRIB_CODENAME=trusty
   ```

3. 此时再试试 **sudo add-apt-repository ppa:git-core/ppa** 看看，却报了另一个错：

   > The most current stable version of Git for Ubuntu.
   >
   > For release candidates, go to https://launchpad.net/~git-core/+archive/candidate .
   > More info: https://launchpad.net/~git-core/+archive/ubuntu/ppa
   > Press [ENTER] to continue or ctrl-c to cancel adding it
   >
   > gpg: keybox '/tmp/tmprpx7ycut/pubring.gpg' created
   > gpg: failed to start the dirmngr '/usr/bin/dirmngr': 没有那个文件或目录
   > gpg: connecting dirmngr at '/tmp/tmprpx7ycut/S.dirmngr' failed: 没有那个文件或目录
   > gpg: keyserver receive failed: No dirmngr

4. 不要慌，他说没有dirmngr，我们就安一个：

   ```bash
   sudo apt-get install dirmngr
   ```

最后再add仓库ppa就不会报错了。
搞定之后，就能顺利安装最新的git：

```bash
sudo apt-get update
sudo apt-get install git
```
