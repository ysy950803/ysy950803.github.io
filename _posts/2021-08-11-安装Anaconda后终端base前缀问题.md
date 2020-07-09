---
layout:     post
title:      安装Anaconda后终端base前缀问题
subtitle:   杂七杂八。
date:       2021-08-11
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - AI
    - 问题不大
---

### 问题

最近想训练个小模型来用用，于是乎我在macOS安装 [Anaconda | Individual Edition](https://www.anaconda.com/products/individual#Downloads) ，选择的是图形界面安装（64-Bit Graphical Installer），整个过程很顺利，一路下一步就行了。

然鹅，安装完之后，我发现我的终端命令行前缀出现了一个base，像这样：

```bash
(base) -> ~
```

这就很无语。

### 原因

猜测原因应该是Anaconda安装后在shell的配置文件中注入了脚本，因为我用的是zsh，所以打开 `.zshrc` 文件可见：

```bash
# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/Users/xxx/opt/anaconda3/bin/conda' 'shell.zsh' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/Users/xxx/opt/anaconda3/etc/profile.d/conda.sh" ]; then
        . "/Users/xxx/opt/anaconda3/etc/profile.d/conda.sh"
    else
        export PATH="/Users/xxx/opt/anaconda3/bin:$PATH"
    fi
fi
unset __conda_setup
# <<< conda initialize <<<
```

但为了保证Anaconda正常使用，肯定不能把这段脚本删了。

### 解决

在官方文档 [Using the .condarc conda configuration file](https://conda.io/projects/conda/en/latest/user-guide/configuration/use-condarc.html#change-command-prompt-changeps1) 中发现了这么个配置：

> Change command prompt (changeps1)
> When using conda activate, change the command prompt from $PS1 to include the activated environment. The default is True.
>
> EXAMPLE:
>
> changeps1: False

那么我们只需要修改 `~/.condarc` 文件即可，追加内容后如下：

```bash
channels:
  - defaults
changeps1: False
```

然后刷新一下：

```bash
source ~/.condarc
```

**注意：**

我第一次在修改时，发现没有 `.condarc` 这个文件，这就很尴尬啊！解决办法是，至少启动一次 **Anaconda-Navigator** 这个应用程序，就会生成rc配置文件了。
