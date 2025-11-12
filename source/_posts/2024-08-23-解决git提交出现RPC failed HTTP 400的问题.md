---
title: 解决git提交出现RPC failed HTTP 400的问题
date: 2024-08-23 11:36:56
tags:
    - 鼓捣折腾
    - 问题不大
---

最近升级了macOS系统后（从12.5升到12.7），发现我的Hexo博客部署不上去了，执行：

```shell
hexo clean & hexo deploy
```

会提示错误：

```
Counting objects: 19815, done.
Compressing objects: 100% (5264/5264), done.
Writing objects: 100% (19815/19815), 44.91 MiB | 134.87 MiB/s, done.
Total 19815 (delta 14641), reused 19405 (delta 14283)
error: RPC failed; HTTP 400 curl 22 The requested URL returned error: 400 Bad Request
fatal: The remote end hung up unexpectedly
fatal: The remote end hung up unexpectedly
Everything up-to-date
```

看到400错误，还以为是权限有问题，尝试配置了git的用户名和邮箱信息：

```shell
git config --local user.email "xxx@xxx.xx"
git config --local user.name "yyy"
```

没用。查了下资料，问了一下GPT，让我把git的缓冲区搞大点：

```shell
git config http.postBuffer 524288000
```

还是没用。想着干脆升级一下git，是不是升级系统后这个自带的git版本有bug？先检查下当前版本：

```shell
git --version
# 输出 git version 2.39.3 (Apple Git-146)
```

会发现输出版本号里面有Apple的字样。还是升级吧：

```shell
# 直接用brew升级
brew install git
# 升完之后重新链接
brew link git --overwrite
# 再清理旧版
brew cleanup git
```

再检查版本，输出就是：`git version 2.46.0` 了。这个版本就能成功提交代码，没有上面的各种RPC啊400啊这些鬼问题了。 当然，你可以先尝试前面几步，如果能解决问题，就不需要升级了。

git升级后，你会发现，各种提示都变成中文了，just like this：

```
枚举对象中: 1446, 完成.
对象计数中: 100% (1446/1446), 完成.
使用 12 个线程进行压缩
压缩对象中: 100% (998/998), 完成.
...入对象中:  21% (312/1446), 10.13 MiB | 111.00 KiB/s
```

有人已经习惯了英文的，怎么办？可以改回来：

```shell
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

设置一下全局环境变量即可，如果要永久生效，就追加到你的shell配置文件中：`~/.bashrc` 或者 `~/.zshrc`，因人而异。
