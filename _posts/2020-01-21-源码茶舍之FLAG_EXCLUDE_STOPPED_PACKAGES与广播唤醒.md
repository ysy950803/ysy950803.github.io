---
layout:     post
title:      源码茶舍之FLAG_EXCLUDE_STOPPED_PACKAGES与广播唤醒
subtitle:   品味AOSP。
date:       2020-01-21
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - AOSP
---

### 发现

我们先随便实现一个BroadcastReceiver，静态注册：
```Kotlin
class TestReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        Log.w("TEST-1", "onReceive ${intent?.action}")
    }
}
```

```xml
<receiver android:name=".TestReceiver">
    <intent-filter>
        <action android:name="com.xxx.yyy.action_test_receiver" />
    </intent-filter>
</receiver>
```

其他诸如Activity什么的就不写了哈，然后我们启动这个测试App之后，用adb命令发一条广播：

```bash
adb shell am broadcast -p com.xxx.yyy -a com.xxx.yyy.action_test_receiver
```

其中参数p表示广播接收所在进程包名，a表示action。命令执行后终端会输出：

```bash
Broadcasting: Intent { act=com.xxx.yyy.action_test_receiver flg=0x400000 pkg=com.xxx.yyy }
Broadcast completed: result=0
```

然后查看logcat，我们可以如愿以偿地看到onReceive中的log。接下来我们杀掉进程，任意方式均可，这里我还是用adb命令，方便：

```bash
adb shell am force-stop com.xxx.yyy
```

杀进程后，再重复上面的广播发送命令，就会发现收不到广播了。这是为什么呢？表面看来这个问题很弱智，进程都死了当然不能再搞事。但实际上背后的逻辑还是值得探索的，系统也不是想象中那么简单地直接判断进程死活然后决定广播发送。

### 探秘

很显然我们要搞明白广播发送的底层逻辑。这里主要分析Framework层面的源码（基于Android 10），涉及到的关键类：

```bash
frameworks/base/core/java/android/content/Intent.java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
frameworks/base/services/core/java/com/android/server/pm/ComponentResolver.java
frameworks/base/services/core/java/com/android/server/IntentResolver.java
```

关于广播发送的细节，可以参考此文：https://www.jianshu.com/p/c5323a22f3f3，虽然源码版本不是最新，但基本逻辑差异不大，时序图也画得非常清晰。下面我们只针对文题简单地分析关键路径即可。

当我们乐呵呵地调用了 `sendBroadcast` 方法之后，会调用到AMS（ActivityManagerService）的 `broadcastIntent` 方法，进而调用内部的 `broadcastIntentLocked` 方法（此处插入一个题外话：很多同学可能经常见到源码里 `xxxLocked` 这种方法，这个locked是什么意思呢？顾名思义就是加锁咯，即你要调用这些locked后缀的方法时，必须保证是线程安全的，所以一般就会看到synchronize关键字，这也算是AOSP的编码规范吧）。

由于 `broadcastIntentLocked` 方法非常长，我们只截取关键片段：

```java
    final int broadcastIntentLocked(Intent intent/*省略18个参数*/) {
        intent = new Intent(intent);
		// ...
        // By default broadcasts do not go to stopped apps.
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
        // ...
        // Figure out who all will receive this broadcast.
        List receivers = null;
        List<BroadcastFilter> registeredReceivers = null;
        // Need to resolve the intent to interested receivers...
        if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
                 == 0) {
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
        // ...
    }
```

可以看到，起手就是new Intent(intent)，为什么不直接使用参数中的intent来进行后续操作呢？此处一个小细节可以看出源码逻辑的谨慎，去查Intent的构造方法就知道，这是对入参的拷贝，避免被其他线程修改。
然后最关键的便是下面的 `FLAG_EXCLUDE_STOPPED_PACKAGES` ，注释也写得很清楚，即**广播发送会排除（exclude）已停止运行的进程**。

但我初次分析时看了半天没发现是怎么排除的，于是找这个flag引用的地方，在 **Intent** 源码中发现一个方法，判断该intent是否要排除已停止的进程：

```java
    public boolean isExcludingStopped() {
        return (mFlags&(FLAG_EXCLUDE_STOPPED_PACKAGES|FLAG_INCLUDE_STOPPED_PACKAGES))
                == FLAG_EXCLUDE_STOPPED_PACKAGES;
    }
```

非常好，我们直接查 `isExcludingStopped` 方法的引用，发现在 **IntentResolver** 中：

```java
    private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
            boolean debug, boolean defaultOnly, String resolvedType, String scheme,
            F[] src, List<R> dest, int userId) {
        // ...
        final boolean excludingStopped = intent.isExcludingStopped();
		// ...
        for (i=0; i<N && (filter=src[i]) != null; i++) {
            // ...
            if (excludingStopped && isFilterStopped(filter, userId)) {
                if (debug) {
                    Slog.v(TAG, "  Filter's target is stopped; skipping");
                }
                continue;
            }
        }
        // ...
    }
```

这是通过intent构造resolve列表的一个私有方法，代码也非常清晰，此判断逻辑 `excludingStopped && isFilterStopped` 过滤了最后要启动的组件。其中 `isFilterStopped` 是真正判断进程是否已停止的方法，而 **excludingStopped** 是我们刚才传入的flag对应的控制标识。所以要过滤掉已停止进程有两个必要条件，**一是这个进程真的死了，二是开发者要通过flag来声明确实需要过滤**（是不是超人性化哈哈哈）。

因此，我们也可以声明不需要过滤，即给intent设置 `FLAG_EXCLUDE_STOPPED_PACKAGES` 的兄弟flag：`FLAG_INCLUDE_STOPPED_PACKAGES` （注意是 **include**），这样在发送广播时，即便是已停止的进程，也能接收到了。这就是上古时期通过广播唤醒死亡进程的方法，现在基本上被各大ROM厂商给优化没了，一般都是三方应用被禁，系统应用依然可以。

分析到此，其实只是有头有尾，但没有中间过程，`broadcastIntentLocked` 最终怎么就调到了 `buildResolveList` 呢？
我们回到上面的 `broadcastIntentLocked` 方法，其中的 `receivers` 对象存储的是静态广播的集合，`registeredReceivers` 则是动态广播的集合。我们只看静态广播即可，它来自于 `collectReceiverComponents` 方法，此方法最后返回的receiver肯定是被过滤后的：

```java
    private List<ResolveInfo> collectReceiverComponents(Intent intent, String resolvedType,
            int callingUid, int[] users) {
        // ...
        List<ResolveInfo> receivers = null;
        try {
            // ...
            for (int user : users) {
                // ...
                List<ResolveInfo> newReceivers = AppGlobals.getPackageManager()
                        .queryIntentReceivers(intent, resolvedType, pmFlags, user).getList();
                // ...
                if (newReceivers != null && newReceivers.size() == 0) {
                    newReceivers = null;
                }
                if (receivers == null) {
                    receivers = newReceivers;
                // ...
            }
        } catch (RemoteException ex) {
            // pm is in same process, this will never happen.
        }
        return receivers;
    }
```

这个方法内部逻辑较为简单，我们可以看到最终receivers的来源便是那个 `queryIntentReceivers` 方法，此方法实现在PMS（PackageManagerService）里面：

```java
    @Override
    public @NonNull ParceledListSlice<ResolveInfo> queryIntentReceivers(Intent intent,
            String resolvedType, int flags, int userId) {
        return new ParceledListSlice<>(
                queryIntentReceiversInternal(intent, resolvedType, flags, userId,
                        false /*allowDynamicSplits*/));
    }

    private @NonNull List<ResolveInfo> queryIntentReceiversInternal(Intent intent,
            String resolvedType, int flags, int userId, boolean allowDynamicSplits) {
        // ...
        synchronized (mPackages) {
            String pkgName = intent.getPackage();
            if (pkgName == null) {
                final List<ResolveInfo> result =
                        mComponentResolver.queryReceivers(intent, resolvedType, flags, userId);
                return applyPostResolutionFilter(
                        result, instantAppPkgName, allowDynamicSplits, callingUid, false, userId,
                        intent);
            }
            final PackageParser.Package pkg = mPackages.get(pkgName);
            if (pkg != null) {
                final List<ResolveInfo> result = mComponentResolver.queryReceivers(
                        intent, resolvedType, flags, pkg.receivers, userId);
                return applyPostResolutionFilter(
                        result, instantAppPkgName, allowDynamicSplits, callingUid, false, userId,
                        intent);
            }
            return Collections.emptyList();
        }
    }
```

最终receiver集合的查询操作在私有方法中，由 `mComponentResolver.queryReceivers` 得来，似乎越来越接近真相了，马上查看ComponentResolver：

```java
    List<ResolveInfo> queryReceivers(Intent intent, String resolvedType, int flags, int userId) {
        synchronized (mLock) {
            return mReceivers.queryIntent(intent, resolvedType, flags, userId);
        }
    }

    List<ResolveInfo> queryReceivers(Intent intent, String resolvedType, int flags,
            List<PackageParser.Activity> receivers, int userId) {
        synchronized (mLock) {
            return mReceivers.queryIntentForPackage(intent, resolvedType, flags, receivers, userId);
        }
    }
```

这里正好对应上面的两个不同情况下的调用，它们最终都会调用到 `queryIntent` 方法，此方法在ComponentResolver的一个静态内部类ActivityIntentResolver中实现：

```java
    private static final class ActivityIntentResolver
            extends IntentResolver<PackageParser.ActivityIntentInfo, ResolveInfo> {
        @Override
        public List<ResolveInfo> queryIntent(Intent intent, String resolvedType,
                boolean defaultOnly, int userId) {
            // ...
            return super.queryIntent(intent, resolvedType, defaultOnly, userId);
        }

        List<ResolveInfo> queryIntent(Intent intent, String resolvedType, int flags,
                int userId) {
            // ...
            return super.queryIntent(intent, resolvedType,
                    (flags & PackageManager.MATCH_DEFAULT_ONLY) != 0,
                    userId);
        }
        // ...
    }
```

内部类的中的 `queryIntent` 方法只是做了一些参数处理，进一步调用的是父类的实现，这个父类也就是起初我们提到的 **IntentResolver**：

```java
    public List<R> queryIntent(Intent intent, String resolvedType, boolean defaultOnly,
            int userId) {
        // ...
        if (firstTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                    scheme, firstTypeCut, finalList, userId);
        }
        if (secondTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                    scheme, secondTypeCut, finalList, userId);
        }
        if (thirdTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                    scheme, thirdTypeCut, finalList, userId);
        }
        if (schemeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                    scheme, schemeCut, finalList, userId);
        }
        filterResults(finalList);
        sortResults(finalList);
		// ...
        return finalList;
    }
```

看上面的 `buildResolveList` 方法，照应了开头分析的结果。最终返回的 **finalList** 也就是过滤之后的receiver集合。

### 总结

- 在广播发送的前序步骤（位于AMS）里，通过 `intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);` 设置了标识，以声明广播不发给已停止的进程，层层调用后最终在 **IntentResolver** 的 `buildResolveList` 方法中实现过滤。
- 在原生Android的设计逻辑中，若要突破上述限制，在 `sendBroadcast` 之前，给intent添加 flag：`FLAG_INCLUDE_STOPPED_PACKAGES` 即可。但鉴于各ROM厂商的正负优化，这个操作已经不适用了。
