---
layout:     post
title:      为何点击推送通知打不开Activity？Calling startActivity() from outside……
subtitle:   知其所以然。
date:       2019-11-19
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - AOSP
---

小米推送Android SDK有这么一个耳熟能详的方法：

```java
/**
 * 接收服务器向客户端发送的通知消息，在用户手动点击通知后触发
 */
public void onNotificationMessageClicked(Context context, MiPushMessage message) {
	...
	context.startActivity(intent);
}
```

如果在其中手动增加启动Activity的逻辑，会发现，点了没反应。把 `startActivity` 方法try-catch后，发现这么一个异常：

> Calling startActivity() from outside of an Activity context requires the 
> FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?

解读一下就是，说我要是从外面启动本应用的Activity需要传 `FLAG_ACTIVITY_NEW_TASK` 标识，还特别嘲讽地反问我一句：这真是你想要的？
![img1](https://img-blog.csdnimg.cn/20190702011449586.png)
我不想要我调你方法干嘛。

## 解决

解决就不说了，报错提示摆明了，要我传NEW_TASK，是的，就这么简单。

```java
public void onNotificationMessageClicked(Context context, MiPushMessage message) {
	...
	intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	context.startActivity(intent);
}
```

## 为什么

这异常到底是什么意思？找找系统源码（基于Android P源码）：
`frameworks/base/core/java/android/app/ContextImpl.java` 在此类中，大概900多行的位置：

```java
...
    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
        // maintain this for backwards compatibility.
        final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P)
                && (options == null
                        || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                            + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
...
```

从这段长注释可以看出：
1、并不是一定要带NEW_TASK，如果指定了任务栈，也没问题，这一点从判断逻辑中的 `ActivityOptions.fromBundle(options).getLaunchTaskId() == -1` 即可看出；
2、竟然这个异常在Android N到O上不会抛出，且谷歌指明这是一个Bug，只是为了兼容，保留了这个判断：`targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P` ，因此对于targetSDK小于N或大于等于P的应用，就会正常地抛出此异常。

回到最开始的推送click回调中，其中传入的**context**本身不属于**Activity**，而是**ApplicationContext**，所以没法启动另一个Activity，系统不知道它应该属于哪个任务栈，所以需要你指定，不管是通过NEW_TASK的方式还是Activity的 `android:taskAffinity` 属性。
再看，小米推送发通知的操作是系统推送服务框架执行的，服务框架本身不具有Activity组件，也没有任务栈，所以启动另一个Activity不指定task的话那按理来说就是不能。
那为什么在targetSDK在N到O之间就可以呢？谷歌说的Bug现象是什么呢？可以推理，不指定任务栈还强制启动一个Activity，那么该游离Activity虽然可以启动，但在最近任务列表里看不见；或者说系统为其指定了默认的task，当再从桌面正常启动应用时，最近任务就会出现两个MainActivity这种类似现象。
