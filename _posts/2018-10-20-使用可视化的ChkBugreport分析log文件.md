---
layout:     post
title:      使用可视化的ChkBugreport分析log文件
subtitle:   开阔视野。
date:       2018-10-20
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
---

> ChkBugreport是一款专门分析Android Bugreport文件的可视化工具。

- 下载源码
  在GitHub上把代码down下来：https://github.com/sonyxperiadev/ChkBugReport

- 编译jar包
  在源码的core/目录下有一个createjar.sh脚本，执行它！
  此时其实已经可以使用了，直接用命令：

  ```
  java -jar chkbugreport.jar bugreport_xxx.log
  ```

  会在同级目录下生成一个bugreport_xxx.log_out文件夹，打开里面的html文件即可查看可视化界面。

- 更方便地使用
  每次都运行上面的命令太繁琐了，我们可以把此工具加入系统环境。
  1、先下载脚本文件：http://sonyxperiadev.github.com/ChkBugReport/download/chkbugreport
  2、将脚本文件放到~/bin目录下，即$HOME/bin，给予此脚本执行权限：

  ```
  chmod +x chkbugreport
  ```

  3、把刚才编译好的jar包改名为chkbugreport.jar，并且也放到~/bin目录下。
  4、最后就可以愉快地使用chkbugreport命令直接解析log文件了：

  ```
  chkbugreport bugreport_xxx.log
  ```

- 可视化效果
  ![在这里插入图片描述](https://img-blog.csdn.net/20181020162212786?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
  ![在这里插入图片描述](https://img-blog.csdn.net/20181020162235770?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
  虽然是比较老的工具了，但还是相当实用的，对分析FC、ANR和一些事件流程很有帮助。
