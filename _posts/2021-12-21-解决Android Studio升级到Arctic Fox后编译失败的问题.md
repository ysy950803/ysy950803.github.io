---
layout:     post
title:      解决Android Studio升级到Arctic Fox后编译失败的问题
subtitle:   我都不知道为什么我能解决。
date:       2021-12-21
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - 问题不大
    - Android
---

### 问题

从Android Studio 4.1.3升级到最新的Arctic Fox之后，整个组件化工程会编译不过。
![在这里插入图片描述](https://img-blog.csdnimg.cn/0f42f86e474e42b582277a3f314f1a1a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6ZKI5Y-2,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)
编译错误显示：

> **e: [kapt] 'com.sun.tools.javac.util.Context' class can't be found ('tools.jar' is absent in the plugin classpath). Kapt won't work.**

很多同学知难而退，被迫回滚到4.1.3，那怎么行呢？

### 分析

首先说，这个错误不是组件化插件的问题，不是Kotlin的问题，也不是工程本身的代码问题，而是JDK没配置好导致的。

但是有人会问了，为什么旧版Studio用得好好的，升级到新版就有JDK的问题了？因为新版Studio默认内置了JDK 11，Gradle会使用11来运行，放弃了JDK 8（细心的同学已经发现，启动界面的Runtime version已经变成了11，不再是8了）。

如果不做任何设置，默认用JDK 11来编译工程，就会失败，当然错误提示不会是上面那个，是另一个，这里不赘述，我们还是以JDK 8来编译。

### 解决

1、我们先打开Studio的偏好设置（Preferences），然后**修改Gradle JDK为1.8**。

一般来说它会检测你系统安装的JDK，如果没有，你就手动Add一下，Add到Home那一层，图中我的路径都是JDK的默认安装路径（我的JDK当初都是用brew直接install的openjdk）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/92366ae7745842daad7b841fc269f15c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6ZKI5Y-2,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)
2、设置好之后，还要记得**配置你的终端环境变量**，这个是为了保证整个系统环境（包括脚本命令执行）都使用JDK 8，否则版本不一致也会出问题。

如果是bash就修改.bashrc，我是用的zsh，所以编辑.zshrc：

```bash
vim ~/.zshrc
```

追加内容：

```bash
export CLASSPATH=/Library/Java/JavaVirtualMachines/openjdk-8.jdk/Contents/Home/lib
export JAVA_HOME=/Library/Java/JavaVirtualMachines/openjdk-8.jdk/Contents/Home
```

保存并退出。

3、**重启Studio，尝试Rebuild工程**，如果没有问题，那恭喜你，不用继续往下看了。

当然，绝大多数情况，就会出现本文最开始那个错误提示：找不到tools.jar。我们知道，这个jar包是JDK lib目录下面的：
![在这里插入图片描述](https://img-blog.csdnimg.cn/7ed58e8b016847d38c5a43f8c6dd5d01.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6ZKI5Y-2,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)
既然环境变量也配置了，怎么就提示错误呢？

4、后来在网上搜了一下，发现是默认JDK的目录命名符号有问题，其实上面看到的 **/Library/Java/JavaVirtualMachines/openjdk-8.jdk** 是个软链接，指向的真实路径是：

**/usr/local/Cellar/openjdk@8/1.8.0+282/libexec/openjdk.jdk**

注意到里面的+号没有，这个玩意儿就是祸首。接下来我们修正错误：

```bash
cd /usr/local/Cellar/openjdk@8/
# 把+改成_
mv 1.8.0+282 1.8.0_282
# 去修正软链接，如果提示已存在就先sudo rm删掉
sudo ln -s /usr/local/Cellar/openjdk@8/1.8.0_282/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-8.jdk
```

此时再重新做一次上述第1步，设置JDK为1.8，再次Rebuild编译（建议先File → Invalidate Caches一下），成功。

### 总结

其实主要原因还是JDK配置的问题，可能新版Studio的Gradle在处理路径或者符号链接上面做了什么奇葩改动。

理论上讲，如果你的JDK本身就配置得非常完美，那么只需要做好第1步，就不会有其他问题了。

升级后部分UI中文变方块，解决办法是修改一下字体即可（Preferences → Appearance → Use custom font）。
