---
layout:     post
title:      源码茶舍之由一次简单的ANR分析深入了解Context
subtitle:   品味AOSP。
date:       2019-02-23
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - ANR
    - AOSP
---

> ANR是Android的老大难了，关于这方面的基础知识和深入好文都非常多，大家不妨谷歌一下。
> 最近搭载骁龙855的小米9也发布了，移动平台的设备性能越来越强，许多App大多时候其实都吃不完那么多计算资源。
> 说得可能不好听一点，很多烂代码要是在很多年前的手机上，本该导致卡顿（甚至是ANR）的，但由于如今强大的计算性能，卡顿几率大大减小了。从某方面来说增大了程序的容错，同时也掩盖了程序本身的缺陷。

今天的题目关键词是“简单分析”和“深入了解”，哈哈，可能对于大佬们来说这些内容并不深入，所以我措辞为“了解”，望轻喷。

## 分析traces文件

前段时间，业务质量平台报上来很多ANR，我是一看就头疼呀！每次心里都犯嘀咕，我怎么就从来没遇到ANR呢？你们到底是怎么使用的。
吐槽归吐槽，问题还是要解决的，Android的系统日志打包上来一般都会有traces.txt文件（还有event log等等，这里给大家硬广一下我另一篇[使用可视化的ChkBugreport分析log文件](https://blog.csdn.net/ysy950803/article/details/83214432)），也是我们分析这类问题的入口，里面记录了各个应用进程和系统进程的函数堆栈信息。于是乎，抓一份来瞧瞧：

```
"main" prio=5 tid=1 Blocked
group="main" sCount=1 dsCount=0 obj=0x75afba88 self=0x7fb0e96a00
...
at android.app.ContextImpl.getPreferencesDir(ContextImpl.java:483)
- waiting to lock <0x0cfeaaf2> (a java.lang.Object) held by thread 24
at android.app.ContextImpl.getSharedPreferencesPath(ContextImpl.java:665)
at android.app.ContextImpl.getSharedPreferences(ContextImpl.java:364)
- locked <0x09b0b543> (a java.lang.Class<android.app.ContextImpl>)
at android.content.ContextWrapper.getSharedPreferences(ContextWrapper.java:174)
at android.content.ContextWrapper.getSharedPreferences(ContextWrapper.java:174)
...
at com.xxx.receiver.xxx.onReceive(xxx.java:36)
...
```

这里简单解释一下，ANR无非就是UI线程Block了，所以我们找到形如 "main" prio=5 tid=1 Blocked 这样的片段，main表示主线程，prio即priority，线程优先级（这里不是重点），tid就是thread的id，即线程id，最后标记了Blocked，表示线程阻塞了。
接着的信息就是告诉你线程被哪个鬼lock了，关注这行：
**waiting to lock <0x0cfeaaf2> (a java.lang.Object) held by thread 24**
说明主线程的getPreferencesDir方法等着要去锁一个id为**0x0cfeaaf2**的Object类型的对象，但是被该死的tid=24的线程抢占了！让我来看看是谁，于是我们可以直接在traces文件里全局搜索0x0cfeaaf2或者tid=24这些字符串，锁定到如下日志：

```
"PackageProcessor" daemon prio=5 tid=24 Native
group="main" sCount=1 dsCount=0 obj=0x32c06af0 self=0x7fb0f36400
...
native: #06 pc 0000000000862c18 /system/framework/arm64/boot-framework.oat (Java_android_os_BinderProxy_transactNative__ILandroid_os_Parcel_2Landroid_os_Parcel_2I+196)
at android.os.BinderProxy.transactNative(Native method)
at android.os.BinderProxy.transact(Binder.java:620)
at android.os.storage.IMountService$Stub$Proxy.mkdirs(IMountService.java:870)
at android.app.ContextImpl.ensureExternalDirsExistOrFilter(ContextImpl.java:2228)
at android.app.ContextImpl.getExternalFilesDirs(ContextImpl.java:586)
- locked <0x0cfeaaf2> (a java.lang.Object)
at android.app.ContextImpl.getExternalFilesDir(ContextImpl.java:569)
at android.content.ContextWrapper.getExternalFilesDir(ContextWrapper.java:243)
at com.xxx.push.log.xxx.writeLog2File(xxx.java:100)
...
```

这里很明显就看到了 **locked <0x0cfeaaf2> (a java.lang.Object)** ，某个和推送服务相关的writeLog2File方法调用了getExternalFilesDirs，然后此方法进一步锁住了 **0x0cfeaaf2** 对象，没错，**这个对象和刚才主线程等待要锁的对象是同一个。**
所以主线程被tid=24的线程阻塞了，因为两个线程要需要同一把对象锁，tid=24线程一直占着茅坑，导致死锁，ANR就这么爆出来了。

## 了解Context

Context是一个抽象类，ContextImpl是Context的实现类（具体一些继承关系可参考[Context都没弄明白，还怎么做Android开发？](https://www.jianshu.com/p/94e0f9ab3f1d)，某大佬写的，比较全面）。
那么，上面的ANR我们重点关注的对象0x0cfeaaf2到底是谁呢？根据这一行：
**at android.app.ContextImpl.getPreferencesDir(ContextImpl.java:483)**
我们直接Read the fucking code，看看ContextImpl中这个方法在干啥：

```java
    private File getPreferencesDir() {
        synchronized (mSync) {
            if (mPreferencesDir == null) {
                mPreferencesDir = new File(getDataDir(), "shared_prefs");
            }
            return ensurePrivateDirExists(mPreferencesDir);
        }
    }
```

可见，这里涉及到shared_prefs文件的IO操作，系统考虑到线程安全，搞了个同步锁，mSync对象被锁住。这个mSync就是我们刚才反复提到的id为0x0cfeaaf2的Object对象，去看看它的实例化就知晓了：

```java
    private final Object mSync = new Object();
```

private final，两个关键字合体了，说明这个成员是不可变的，而且是私有的，不准继承，即在Context的生命周期内全局只实例化一次，这样才能在加锁的时候保证唯一性。
接下来又看刚才tid=24给对象加锁的方法，源码自然也在ContextImpl中：

```java
    @Override
    public File[] getExternalFilesDirs(String type) {
        synchronized (mSync) {
            File[] dirs = Environment.buildExternalStorageAppFilesDirs(getPackageName());
            if (type != null) {
                dirs = Environment.buildPaths(dirs, type);
            }
            return ensureExternalDirsExistOrFilter(dirs);
        }
    }
```

OK，它也有给mSync加锁的操作， **所以tid=24线程的getExternalFilesDirs方法先加锁，造成主线程的getPreferencesDir方法抢不到这把锁，这真是喧宾夺主啊！** 你区区一个子线程和主线程作对，分析到此我们基本清楚了这次ANR是怎么来的了。
这里我们进一步看看上面return的ensureExternalDirsExistOrFilter方法：

```java
    /**
     * Ensure that given directories exist, trying to create them if missing. If
     * unable to create, they are filtered by replacing with {@code null}.
     */
    private File[] ensureExternalDirsExistOrFilter(File[] dirs) {
        final StorageManager sm = getSystemService(StorageManager.class);
        final File[] result = new File[dirs.length];
        for (int i = 0; i < dirs.length; i++) {
            File dir = dirs[i];
            if (!dir.exists()) {
                if (!dir.mkdirs()) {
                    // recheck existence in case of cross-process race
                    if (!dir.exists()) {
                        // Failing to mkdir() may be okay, since we might not have
                        // enough permissions; ask vold to create on our behalf.
                        try {
                            sm.mkdirs(dir);
                        } catch (Exception e) {
                            Log.w(TAG, "Failed to ensure " + dir + ": " + e);
                            dir = null;
                        }
                    }
                }
            }
            result[i] = dir;
        }
        return result;
    }
```

我的天鸭，你看看，这操作多重啊，又是循环又是创建文件的，还有getSystemService这些系统服务对端调用，加在一起就是灰常耗时的操作，尤其是在文件目录极其散乱繁杂而且磁盘读写性能还不好的时候，此方法将进一步延长阻塞时间。
我又一想，什么SP啊，DB啊，外部存储啊这些我们平时经常访问啊，也并不是那么容易就ANR的。也就是说虽然上面的系统方法操作很繁杂，但应该不是导致最终问题的核心因素。
经过我反复分析traces文件，发现除了main线程在wait to lock这把锁，还有几个其它的子线程也在等待锁（有一些是访问App本地数据库的，最终调用也在ContextImpl中，和上面分析的两个方法类似）。说明当前这短暂的时间内，需要通过某个Context进行的IO操作太多了，各个线程都排着队要锁mSync，所以耗时操作不可怕，可怕的是一窝蜂全上来。自然就增大了ANR的风险。如果你反复遇到这种ANR，就应该考虑优化了。
最终，追溯到方法调用的源头，是在Application初始化时，各种SDK加载，以及一些业务逻辑触发。很显然，它们都是通过getApplicationContext来拿到的同一个Context引用，请求锁的也是同一个mSync。

## 结论与建议

- 调用Context相关的IO操作，不是启个子线程就高枕无忧了，由上面分析，mSync就这么一把，该阻塞还是阻塞，和是不是主线程无关。
- 尽量不要在Application的初始化时刻进行太多的方法调用，尤其是针对ApplicationContext的IO操作。
- 在主Activity中延后初始化，用IntentService进行异步操作（因为实例化一个Service就是另一个Context对象了）等都是比较好的优化方案。
- 所以为什么有大佬说不要滥用SharedPreference，它的性能并不是很好，从本文分析也可知它直接可能阻塞UI线程，试图寻找其它替代品吧。
- 广播接收onReceive里面可以用goAsync异步处理，见：[goAsync帮你在onReceive中简便地进行异步操作](https://blog.csdn.net/ysy950803/article/details/83216891)。
- ...想到再说，也欢迎大家补充。
