---
layout:     post
title:      IDE Eval Reset忘了重置进不去怎么办
subtitle:   偶尔探索一下，就能解决全网问题。
date:       2021-10-10
author:     YSY
header-img: img/404-bg.jpg
catalog: true
tags:
    - 问题不大
---

### 问题

我相信很多白嫖怪都知道目前JB全家桶“极为先进”的使用方法——其实就是无限重置30天（[IDE Eval Reset](https://zhile.io/2020/11/18/jetbrains-eval-reset-da33a93d.html)）。具体使用方法就不赘述了。

这里会出现一个问题，尤其是针对电脑上安装了多款JB家的IDE用户来说（比如我就是，IDEA、PyCharm和CLion都在使bai用piao），如果**超过30天未打开**其中某个IDE进行试用重置，那么你就会发现打不开了，要求你补充License，而且菜单栏也没有地方让你再去打开Eval Reset插件了。怎么办！？

或许你唯一能想到的办法就是完全卸载，清空一切相关配置文件和卸载残留，重新安装，重新试用。但是这样损失惨重啊，尤其是很多设置和项目又要重新导入。

### 解决

这个办法也是我偶然发现的，你在其他地方肯定搜不到。不过当你第一次遇到这个问题时，有个**前提：需要至少有一款IDE是可以打开使用的，也就是刚刚使用过Eval Reset并且还在30天内的。**

举例，比如我现在PyCharm打不开了，但是IDEA平时经常用到，所以能打开，接下来：

1、这里以macOS版本为例（其他系统也是类似的，就是路径不太一样，看后文），复制IDEA内部eval文件夹下面的key到PyCharm对应文件夹下面：

```bash
# xxx是你的用户名
cp /Users/xxx/Library/Application\ Support/JetBrains/IntelliJIdea2021.2/eval/idea212.evaluation.key /Users/xxx/Library/Application\ Support/JetBrains/PyCharm2021.2/eval/PyCharm212.evaluation.key
```

![](https://img-blog.csdnimg.cn/3ea46506bfc742e7925e8489ebefb274.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6ZKI5Y-2,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

注意key文件是否存在，版本一定要对应当前安装的版本，比如我这里212就是2021.2版本。复制后注意改名，比如idea212前缀要记得改成PyCharm212，如上述命令一行搞定。

2、此时再重新打开PyCharm会发现奇迹般地复活了，重置时间也和IDEA的一模一样。

#### 关于路径

这个插件是个[开源项目](https://gitee.com/pengzhile/ide-eval-resetter)，从其脚本文件源码可看到路径配置：

```java
// Resetter.java
protected static File getEvalDir() {
    String configPath = PathManager.getConfigPath();
    return new File(configPath, "eval");
}
```

上面这个PathManager实际上是 `com.intellij.openapi.application.PathManager` ，我们直接去GitHub搜IDEA的开源代码即可：[PathManager.java](https://github.com/JetBrains/intellij-community/blob/master/platform/util/src/com/intellij/openapi/application/PathManager.java)

所以Windows的路径一般是：`C:\Users\xxx\AppData\Roaming\JetBrains\IntelliJIdea2021.2\eval` ，Linux自行探索。

### 思考

从上述解决方法我们可以看出，无限重置时间之所以能生效，关键就在这些key文件。我大致了解了一下此插件源代码之后，可以得知key文件里面存储的信息：

```java
// LicenseFileRecord.java
@Override
public void reset() throws Exception {
    if (!FileUtil.delete(file)) {
        throw new Exception("Remove " + type + " failed: " + file.getAbsolutePath());
    }
    // 此处写入了当前时间戳，貌似就是这么简单
    try (DataOutputStream dos = new DataOutputStream(new FileOutputStream(file))) {
        dos.writeLong(~System.currentTimeMillis());
    }
}
```

因此，如果我们的所有key文件都超过30天过期了，就可以copy一下插件的源代码，自己去运行这些逻辑手动写入信息或生成key文件，具体操作就不赘述了。

总之，此插件还是有它的局限性，大部分代码都是GUI相关的，~~如果作者可以搞一个可执行脚本就好了~~，这样可以在命令行中拯救那些过期后打不开的JB全家桶。

**最新：** 后来发现呢，实际上作者有提供这样一个[临时脚本](https://gitee.com/pengzhile/ide-eval-resetter/tree/master/scripts)，可以让打不开IDE的（Evaluate for free显示不可选）重新获取权限。
