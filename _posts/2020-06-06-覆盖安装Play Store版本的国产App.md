---
layout:     post
title:      覆盖安装Play Store版本的国产App
subtitle:   探索发现。
date:       2020-06-06
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - 鼓捣折腾
    - Android
---

## 前言

对于Android平台，如果在国内应用商店安装诸如微信、支付宝等国产大型App，相比去Google Play Store安装，会多要不少权限，即便权限无差异，在隐私政策等规则方面，Play Store也会更严格一些，能上架的应用肯定不敢乱搞。之前也有不少用户反映在Play Store安装的微信要流畅不少，某些功能细节也会有差异。

不过由于严格的审核机制，Play Store上的国产App更新频率普遍落后于国内应用商店，如果我们已经安装了国内的最新版本，正常情况下就没办法覆盖安装低版本了。Play Store上也会显示已安装，没有重新安装这种选项。可我们想在**不卸载原有版本且保留数据**的情况下覆盖安装Play Store上的版本，该怎么办呢？

下文以微信为例。

## 方案

第一种方法很简单，比如我现在装了国内应用商店上的微信，版本为7.0.14，那我可以等着Play Store上架7.0.15版本的微信后直接升级，自然就替换成了Play Store版本的微信。

第二种方法，拒绝等待。我们可以将Play Store上的微信下载下来，手动安装。但是，Play Store是没有提供Apk官方下载途径的，所以我们要去 [https://apkpure.com](https://apkpure.com) （专门提取官方Apk的网站）搜索WeChat即可，注意要下载Apk文件，不要下XAPK格式：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060612375117.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)

由于Play Store上目前最新版本低于我手机上的微信版本，直接copy到手机上手动安装会失败的，低版本不能覆盖高版本。所以，接下来我们需要通过adb命令来安装Apk了（在这之前请保证手机的开发者选项是打开状态且开启了其中的USB调试和USB安装）：

```bash
adb install -r -d WeChat_v7.0.13_apkpure.com.apk
```

`-r` 参数表示覆盖安装且保留数据，这对我们非常重要，微信几个GB的数据不是说清就能清的，要命。`-d` 参数表示强制将低版本覆盖安装到现有版本上，无论现有高低。

## 后话

安装后我发现小程序打不开了，提示模块正在更新中，但是过了半天都没反应（结果是我忘了使用科学上网，如果网络正常，小程序模块就会在后台通过谷歌服务来安装）。这证明了，Play Store版本的微信，默认不会自带小程序等额外模块，而是通过谷歌商店来更新的（可能是App Bundle），因为谷歌是不允许自己搞热更新的，所以微信的热更新自然就去掉了。

另外，我发现Play Store上的微信会使用系统的WebView内核（也就是Chrome那一套），公众号和小程序的使用体验流畅了很多很多，非常爽。当然，你也可以通过给自己发送debugtbs.qq.com网址并使用微信内置浏览器打开，然后手动关闭X5内核。

通过此方法，我把QQ、支付宝等都替换掉了。其中QQ有点奇葩，覆盖成低版本后打不开了，点icon没反应，怀疑可能是兼容问题，清除全部数据才好，看来降级太多个版本的话也会有风险，所以自己斟酌好再搞。但目前看来微信和支付宝是没问题的。至于淘宝，可以下载Play Store上的淘宝Lite，功能简洁，没那么多杂七杂八的，对于非深度用户足够了。
