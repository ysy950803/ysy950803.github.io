---
layout:     post
title:      Android Shortcut启动导致其他Activity销毁问题
subtitle:   小问题而已。
date:       2021-09-27
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
---

### 问题

我们都知道，从API 25开始，Android加入了类似3D Touch一样的功能，即长按桌面图标可以弹出快捷方式菜单（最多4个）。早期国产系统的桌面Launcher并没有积极适这一功能，所以各大应用也懒得做，后来才逐渐完善。目前包括微信、支付宝等都可以长按弹出快捷方式，支付宝更是支持动态配置。

![](https://blog.ysy950803.top/img/posts/e4f6b19b36cdc2f51ed35e8eca7544c9.webp)

开发文档直接看官方的就行：[https://developer.android.com/guide/topics/ui/shortcuts](https://developer.android.com/guide/topics/ui/shortcuts?hl=zh-cn) ，静态快捷方式适配很简单，加xml文件就完事，此处不赘述。

但在实际体验开发过程中发现，通过快捷方式打开应用的对应页面后，其他Activity会被销毁。这并不是我们想要的效果。

### 简单分析

这个现象很像是在启动Activity时设置了 **CLEAR_TASK** 的标识，导致任务栈被清空。但是，从下列的使用示例来看，静态快捷方式又无法设置Intent的flag，相关逻辑由系统SDK内部实现。

```xml
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
  <shortcut
    android:shortcutId="compose"
    android:enabled="true"
    android:icon="@drawable/compose_icon"
    android:shortcutShortLabel="@string/compose_shortcut_short_label1"
    android:shortcutLongLabel="@string/compose_shortcut_long_label1"
    android:shortcutDisabledMessage="@string/compose_disabled_message1">
    <!-- 例1 -->
    <intent
      android:action="android.intent.action.VIEW"
      android:targetPackage="com.example.myapplication"
      android:targetClass="com.example.myapplication.ComposeActivity" />
    <!-- 例2 -->
    <intent
      android:action="android.intent.action.VIEW"
      android:data="xxx://xxx/xxx" />
  </shortcut>
  <!-- Specify more shortcuts here. -->
</shortcuts>
```

后来，我在官方文档看到这么一段：

> 静态快捷方式不能有自定义 intent 标记。 静态快捷方式的第一个 intent 始终设置有 Intent.FLAG_ACTIVITY_NEW_TASK 和 Intent.FLAG_ACTIVITY_CLEAR_TASK。这意味着，如果应用已在运行，则在静态快捷方式启动时，应用中的所有现有 Activity 都会被销毁。如果不希望出现这种行为，您可以使用 Trampoline Activity ……

### 解决

这个Trampoline意思就是设置一个跳板Activity，来分发启动目标Activity，并且我们需要让这个跳板Activity和应用的其他Activity不在一个栈中，很简单，设置 `taskAffinity` 属性即可：

```xml
<!-- AndroidManifest.xml -->
<activity
  android:name=".TrampolineActivity"
  android:taskAffinity="" />
  
<!-- xml/shortcuts.xml -->
<intent
  android:action="android.intent.action.VIEW"
  android:targetPackage="com.example.myapplication"
  android:targetClass="com.example.myapplication.TrampolineActivity" />
```

不显示设置taskAffinity，其默认值为包名，所以只要给我们的跳板Activity设置非包名的字符串就行。如此，再通过桌面长按快捷方式打开应用时，就不会销毁其他页面了。
