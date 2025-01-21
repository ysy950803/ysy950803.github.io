---
layout:     post
title:      快速实现Sublime Text的Kotlin高亮
subtitle:   又闲下来了。
date:       2021-06-18
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Kotlin
    - 鼓捣折腾
---

### 问题

Sublime Text是一款非常实用的编辑器软件，偶尔不想开大型IDE的时候，用它来看看代码还是不错的。

不过发现在用它来查看Kotlin代码时，默认是一片白，没有语言对应的高亮，点击右下角选择语言时也没有Kotlin这个选项（下图是解决问题之后的）。

![](https://blog.ysy950803.top/img/posts/011bfe859b0040145dc70d8350884280.webp)

### 解决

没有高亮看着多不舒服啊，如何快速搞定呢？已经有开源项目帮我们解决了。

#### 新方法（2023更新）

![](https://blog.ysy950803.top/img/posts/1f39488fd9c012c2ee97e514dc521a33.webp)

找到**Preferences**中的**Package Control**（如果没有这个选项，说明没有安装这个组件，在**Tools**中找到最后一个**Install Package Control**即可，稍等几分钟就会弹窗提示成功）。

![](https://blog.ysy950803.top/img/posts/04c66844ac7e1122fbb968433de7a126.webp)

打开Package Control后在弹窗中找到Install Package这项，双击或者回车均可，等待几秒就会弹出新的窗口，直接搜索Kotlin关键词，选中第一个，安装即可。

![](https://blog.ysy950803.top/img/posts/12b0814df2fcaf3d3e310ca6546a64c8.webp)

#### 旧方法（package下载链接已失效）

[GitHub - vkostyukov/kotlin-sublime-package: Sublime Text 2 Package for Kotlin Programming Language](https://github.com/vkostyukov/kotlin-sublime-package) ，我们查看项目的README，直接下载第一个 `Kotlin.sublime-package` 。

![](https://blog.ysy950803.top/img/posts/501b914b05ad5e69c4d8952bdce2dabf.webp)

下载下来发现文件名字是Kotlin.zip，没关系，直接改成Kotlin.sublime-package就行了。然后我们将此文件放到应用安装目录下面的**Packages**文件夹中，Windows可能是Data/Packages之类的目录，层级不多，稍微找一下就行，这个目录下面都是统一后缀的文件。

![](https://blog.ysy950803.top/img/posts/731fa9ac494266c11dc7e026b5daa98f.webp)

macOS在应用程序中找到Sublime Text然后右键**显示包内容**就能找到了。最后再重启Sublime Text，右下角就可以选择Kotlin语言高亮了。
