---
layout:     post
title:      macOS下载AOSP的小坑
subtitle:   坑也太多了吧。
date:       2020-06-07
author:     YSY
header-img: img/tags-bg.jpg
catalog: true
tags:
    - Android
    - macOS
    - AOSP
---

## 爬坑

以下载Android X源码为例，过程和下载AOSP是一样的，只是分支不同而已。源码在线地址：[frameworks/support](https://cs.android.com/androidx/platform/frameworks/support)

### 安装Homebrew

在Mac上搞开发必须的包管理工具，类似Ubuntu上的apt。安装非常简单，一行命令的事情，来自官网（[Homebrew](https://brew.sh/index_zh-cn)）：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

### 安装repo工具

可以直接下载谷歌官方的repo源码（[https://gerrit.googlesource.com/git-repo](https://gerrit.googlesource.com/git-repo)）然后自己配置命令，但这样很麻烦，直接用brew安装整套Android编译工具：

```bash
brew cask install android-platform-tools
```

装完就可以直接使用repo命令了。

### 下载AOSP源码

详细过程可参考：[Win10用WSL下载AOSP](https://blog.ysy950803.top/2020/05/17/Win10用WSL下载AOSP/)

```bash
# 创建一个源码目录，命名随意
mkdir AOSP
cd AOSP
# 这里以AndroidX分支示例
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b androidx-master-dev
repo sync -j4
```

然后报错了：

> fatal: error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1056)

之前在Win10和Ubuntu甚至WSL都没有遇到这个坑。

**解决办法：**

打开repo工具的脚本，如果你是自己下载的git-repo源码那肯定知道在哪里，一般就是 `~/bin/repo` 。当然由于我们刚才是通过brew安装的，所以不在用户目录，直接文本编辑它：

```bash
sudo open -e /usr/local/bin/repo
```

在import sys那一行下面加入一段内容后如下：

```python
import sys

# For macOS START
import ssl

ssl._create_default_https_context = ssl._create_unverified_context
# For macOS END
```

保存后再去repo init就没问题了。这个主要原因是Python 2.7.9后在打开HTTPS链接时加入了SSL验证导致的，然鹅我们并不需要这些，所以上面的代码就是关闭这个。

## 私货

### 让终端默认支持git自动补全

可参考（[https://blog.csdn.net/tiancaijyy/article/details/84888868](https://blog.csdn.net/tiancaijyy/article/details/84888868)）

1、安装bash-completion

```bash
brew install bash-completion
# 成功后查看一下信息
brew info bash-comletion
```

2、在info里可以看到一个 `Add the following line to your ~/.bash_profile:` 提示，让你把下面命令加到你的~/.bash_profile文件中：

```bash
[[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"
```

3、然后请先确定自己的git版本：

```bash
git --version
# 比如我这里输出
git version 2.26.2
```

4、下载git自动完成脚本到bash-completion命令目录下，注意URL中的版本号替换：

```bash
curl https://raw.githubusercontent.com/git/git/v2.26.2/contrib/completion/git-completion.bash > /usr/local/opt/bash-completion/etc/bash_completion.d
```

5、然后使其生效：

```bash
brew unlink bash-completion
brew link bash-completion
```

最后重启（要command + Q完全退出）一下终端，再打开就可以自动补全各种命令（当然也包括git）了。

### 推荐安装oh-my-zsh

目前macOS上的终端默认已经是zsh了，而不是bash。推荐安装扩展版本的oh-my-zsh，无论是修改主题还是Tab自动补全都非常方便。安装方法还是很简单，按官方文档（[https://github.com/ohmyzsh/ohmyzsh](https://github.com/ohmyzsh/ohmyzsh)）：

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# 或者用wget
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
