---
layout:     post
title:      Windows安装repo的真正解决方案
subtitle:   专治各种小毛病。
date:       2020-02-05
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Windows
    - AOSP
    - Android
---

> 2020-02-22更新！我发现谷歌在最近几天发布了git-repo 2.4版本，并更新了 [Microsoft Windows Details](https://gerrit.googlesource.com/git-repo/+/HEAD/docs/windows.md) 文档，直接解决了此文问题，比本文以前的三方解决办法简单很多很多。

### 最新官方解决方案
#### 一、基础设施
- 安装最新的Git for Windows（参考下面的旧文即可），目前版本是2.25.1
- **安装Python 3** ，目前版本是3.8.1，**不要安装Python 2**，这是和旧办法的不同之处
- 配置各种环境变量（参照旧文即可），Python 3在安装的时候勾选Add path那项就能自动配置

#### 二、搞起最新的repo工具
和旧文的**安装Repo**步骤类似，只不过所有都替换成谷歌官方的：
```bash
mkdir ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+rx ~/bin/repo
```
然后**注意**，先下载最新的repo工具源码，再进行init操作：
```bash
# 先随便新建源码目录
mkdir -p ~/AOSP/.repo
cd ~/AOSP/.repo
# clone工具集
git clone https://gerrit.googlesource.com/git-repo
# 一定要改文件夹名
mv git-repo repo
# 回到AOSP源码目录
cd ..
# 保证你成功
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-10.0.0_r25 --worktree
```
这里的 `--worktree` 参数非常重要，不加的话会出现 **error.GitError: Cannot initialize work tree for manifests** 错误。这个功能也是谷歌在这个月底才更新的。
![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/2/22/1706d37e0cdbf737?w=1003&h=274&f=png&s=67833)
最终我也试了下repo sync，repo upload等命令均无问题。

---

> 以下是旧文，强烈推荐分隔线以上的最新官方解决方案。

### 背景
2020真是魔幻的一年，受疫情影响，大家年后一段时间都远程办公了。奈何很多同事在家没有Linux开发环境，想在Windows上通过repo工具下载Android源码简直比登天还难。

网上搜来受去没几个讲透彻的，今天给大家整活。按下面步骤来，保证OK，我们最终以 **repo init** 执行成功为目标。

### 走起
#### 一、安装Git for Windows
先到官网（[https://git-scm.com/download/win](https://git-scm.com/download/win)）下载 **64-bit Git for Windows Setup** 然后安装，基本上一路下一步，但需要注意几点：

 - 第一步第一项有个Add icons什么的，即添加桌面图标，默认没勾，最好勾上。
 - 最后一步有3个Enable xxx，默认第3个（和symbollink相关）没勾，请把它勾上。
 
#### 二、安装Python 2.7
 先到官网（[https://www.python.org/downloads/release/python-2717/](https://www.python.org/downloads/release/python-2717/)）下载 **Windows x86-64 MSI installer** ，这貌似是Python 2时代的最后一个版本了。安装一路下一步即可。

#### 三、配置系统环境变量
上述俩基础组件装完了，检查一下环境变量，Path路径该加的加上，Windows如何查看并添加系统环境变量请大家自行搜索。需要添加如下：
```bash
C:\Program Files\Git\cmd
C:\Program Files\Git\bin
C:\Program Files\Git\usr\bin
C:\Python27\
C:\Python27\Scripts\
# 这一项不要忘了，先提前配置好，为repo做准备
C:\Users\你的用户名\bin
```
路径和你安装时的选择相关，切勿直接照抄。还是给大家整个图吧。
![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/2/22/1706d37e0d8cb23a?w=1595&h=827&f=png&s=421328)
#### 四、安装repo
repo原本是谷歌搞的一个方便下载AOSP的工具，基于git，但由于种种原因，不能直接在Windows上使用。但**好心的基佬Hub网友开发了一套改良版的repo**，适用于Windows，解决各种Error问题。

首先要把repo命令脚本搞定。在任意处打开 **Git Bash** （点桌面的快捷方式也可以），然后：
```bash
mkdir ~/bin
curl https://raw.githubusercontent.com/esrlabs/git-repo/stable/repo > ~/bin/repo
curl https://raw.githubusercontent.com/esrlabs/git-repo/stable/repo.cmd > ~/bin/repo.cmd
chmod a+rx ~/bin/repo
```
接下来，基本上就和Linux上的操作差不多了。但在repo init时，需要增加或修改 **repo-url** 参数，具体如下：
```bash
# 先随便新建源码目录
mkdir ~/AOSP
cd ~/AOSP
# 初始化
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-10.0.0_r25 --repo-url=https://github.com/esrlabs/git-repo.git
```
**注意点：**

 - 上述命令关键就在 `--repo-url=https://github.com/esrlabs/git-repo.git` ，替换掉原生的repo工具链，就能成功初始化了。
 - 这里用的是清华镜像源AOSP作示例，一般做ROM开发的公司会有自己的仓库地址，请自行修改init链接。
 - 不要忘了生成ssh的public key，在Windows下也一样：在Git Bash中执行 `ssh-keygen` 然后复制 `~/.ssh/id_rsa.pub` 文件中的内容添加到Gerrit等源码平台上即可。
![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/2/22/1706d37e0d0189ff?w=525&h=278&f=png&s=53959)
 - 如果发现上面下载速度太慢，可以把 [https://github.com/esrlabs/git-repo](https://github.com/esrlabs/git-repo) 项目直接下载下来并解压，复制解压后文件夹中的所有文件到源码目录的的 `.repo/repo` 子目录下面，然后再重新执行repo init命令，当然这次就不要带 **repo-url** 参数了。
![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/2/22/1706d37e0e867059?w=649&h=828&f=png&s=146096)
大功告成！
![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/2/22/1706d37e0eee702f?w=606&h=307&f=png&s=65927)
### 参考
 - [git-repo](https://github.com/esrlabs/git-repo)
 - [Microsoft Windows Details](https://gerrit.googlesource.com/git-repo/+/HEAD/docs/windows.md)
 - [windows环境下repo下载Android源代码](https://ressrc.com/2018/09/22/windows环境下repo下载android源代码/)
 - [window7下配置下载android源码环境,安装Repo](https://blog.csdn.net/nicolelili1/article/details/52527475)
 