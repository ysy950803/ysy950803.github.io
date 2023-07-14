---
layout:     post
title:      源码茶舍之PackageManager获取注册Service数量问题
subtitle:   多查查，也不难。
date:       2019-11-02
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - AOSP
---

### 问题

今天有朋友遇到个问题，说bindService失败了，查了几步发现是由于PackageManager获取不到对应的Service组件导致的。具体示例代码如下：

```kotlin
val serviceInfos = packageManager.getPackageInfo("com.xxx.xxx", PackageManager.GET_SERVICES).services
Log.d("TEST", Arrays.toString(serviceInfos))
```

这里我们通过PackageManager获取到对应包名的PackageInfo，最终的serviceInfos是一个数组，包含**该应用注册的所有Service组件**。
但不同时候打印出来的数组长度竟然不同，也就是说某些Service一会儿有一会儿没有，这是为什么呢？

### 溯源

要搞清楚上面的问题，我们就要追本溯源啦！**在追踪的过程中我们时刻记得留意一切可能使services数组发生变化的逻辑**。

> 提示：以下Android系统源码均基于Android P。

先看看PackageInfo的源码中对services成员的注释描述：

```java
/**
 * Array of all {@link android.R.styleable#AndroidManifestService
 * &lt;service&gt;} tags included under &lt;application&gt;,
 * or null if there were none.  This is only filled in if the flag
 * {@link PackageManager#GET_SERVICES} was set.
 */
public ServiceInfo[] services;
```

可以看出，这里只提到了该数组包含AndroidManifest.xml中注册的所有Service组件，并没有说明有何具体过滤限制。那我们就只能从services赋值的源头找寻了。

PackageManager只是一层API，我们需要看它对应的系统服务，那么就是PackageManagerService，getPackageInfo相关方法：

```java
@Override
public PackageInfo getPackageInfo(String packageName, int flags, int userId) {
    return getPackageInfoInternal(packageName, PackageManager.VERSION_CODE_HIGHEST,
            flags, Binder.getCallingUid(), userId);
}

// 实际的内部方法，这里做了代码精简，只保留关键部分
private PackageInfo getPackageInfoInternal(String packageName, long versionCode,
        int flags, int filterCallingUid, int userId) {
    // ...

    // reader
    synchronized (mPackages) {
        // Normalize package name to handle renamed packages and static libs
        packageName = resolveInternalPackageNameLPr(packageName, versionCode);

        final boolean matchFactoryOnly = (flags & MATCH_FACTORY_ONLY) != 0;
        if (matchFactoryOnly) {
            final PackageSetting ps = mSettings.getDisabledSystemPkgLPr(packageName);
            if (ps != null) {
                // ...
                return generatePackageInfo(ps, flags, userId); // 生成PackageInfo实例
            }
        }

        PackageParser.Package p = mPackages.get(packageName);
        // ...
        if (!matchFactoryOnly && (flags & MATCH_KNOWN_PACKAGES) != 0) {
            final PackageSetting ps = mSettings.mPackages.get(packageName);
            // ...
            return generatePackageInfo(ps, flags, userId); // 生成PackageInfo实例
        }
    }
}
```

从getPackageInfoInternal方法的源码来看还只是一些权限校验和匹配，没有涉及到具体组件信息生成的逻辑，所以我们继续看generatePackageInfo方法：

```java
private PackageInfo generatePackageInfo(PackageSetting ps, int flags, int userId) {
    // ...
    if (p != null) {
        // ...
        PackageInfo packageInfo = PackageParser.generatePackageInfo(p, gids, flags,
                ps.firstInstallTime, ps.lastUpdateTime, permissions, state, userId);
        // ...
        return packageInfo;
// ...
```

同样地，我们只保留关键代码，可以看到生成PackageInfo的过程实际上是由PackageParser来处理。而且，到这里flags都还没解析判断呢，系统怎么知道我需要获取的是什么组件呢是吧？没错，最终逻辑基本都在PackageParser的相关方法里了：

```java
public static PackageInfo generatePackageInfo(PackageParser.Package p,
        int gids[], int flags, long firstInstallTime, long lastUpdateTime,
        Set<String> grantedPermissions, PackageUserState state, int userId) {
    // ...
    PackageInfo pi = new PackageInfo();
    pi.packageName = p.packageName;
    // ...
    if ((flags & PackageManager.GET_SERVICES) != 0) {
        // 这里的N就等于Manifest文件中实际声明的Service的数量
        final int N = p.services.size();
        if (N > 0) {
            int num = 0;
            final ServiceInfo[] res = new ServiceInfo[N];
            for (int i = 0; i < N; i++) {
                final Service s = p.services.get(i);
                // 关键就在这个判断，决定了哪些Service组件会被过滤掉
                if (state.isMatch(s.info, flags)) {
                    res[num++] = generateServiceInfo(s, flags, state, userId);
                }
            }
            // 由于返回的数组长度并不一定等于N，所以还需要专门trim一下数组
            pi.services = ArrayUtils.trimToSize(res, num);
        }
    }
    // ...
    return pi;
}
```

总算是找到老巢了，可以看到，最终返回的是pi对象，和传进来的p是不一样的。相关逻辑也很简单，从我的源码注释里可得知ServiceInfo数组之所以会发生变化，就是因为那个 **isMatch** 方法，如果它返回了false，那么这个Service组件不会返回给外部了。
继续深入，找到这个PackageUserState的isMatch方法：

```java
/**
 * Test if the given component is considered installed, enabled and a match
 * for the given flags.
 *
 * <p>
 * Expects at least one of {@link PackageManager#MATCH_DIRECT_BOOT_AWARE} and
 * {@link PackageManager#MATCH_DIRECT_BOOT_UNAWARE} are specified in {@code flags}.
 * </p>
 */
public boolean isMatch(ComponentInfo componentInfo, int flags) {
    final boolean isSystemApp = componentInfo.applicationInfo.isSystemApp();
    final boolean matchUninstalled = (flags & PackageManager.MATCH_KNOWN_PACKAGES) != 0;
    if (!isAvailable(flags)
            && !(isSystemApp && matchUninstalled)) return false;
    if (!isEnabled(componentInfo, flags)) return false; // 重点关注

    if ((flags & MATCH_SYSTEM_ONLY) != 0) {
        if (!isSystemApp) {
            return false;
        }
    }

    final boolean matchesUnaware = ((flags & MATCH_DIRECT_BOOT_UNAWARE) != 0)
            && !componentInfo.directBootAware;
    final boolean matchesAware = ((flags & MATCH_DIRECT_BOOT_AWARE) != 0)
            && componentInfo.directBootAware;
    return matchesUnaware || matchesAware; // 重点关注
}
```

哟，瞧瞧，这限制真的不少啊。对于三方非系统应用来说，我们暂时只用关心两个return分支。

### 解决

从上述的isMatch源码来分析问题排查办法。

第一个即isEnabled的检查，这个我们可以对应Service组件中的 `android:enabled` 属性，也就是说当你的组件被禁用时，那么对应Service的ServiceInfo就不会返回给外部了，这个很好理解，组件不可用时，外部肯定不能获取其信息，所以你要去bindService之类的操作肯定是抛异常的。当然，此属性默认值是true，但我们不排除业务逻辑中有动态设置false的可能，这个具体参考PackageManager的**setComponentEnabledSetting**方法，此处不赘述。

第二个即组件**direct-boot**（直接启动模式）的相关设置，这是从7.1之后出现的特性，对应 `android:directBootAware` 属性，该属性默认是false，即不支持该模式，那么很可能你的应用在设备加密锁屏后获取不到所需要的Service组件。可将**directBootAware**属性设为**true**后再尝试是否能解决本文问题，若涉及到Context的，还需要额外操作，具体参考谷歌官方文档中对直接启动模式的详细介绍和适配方式：[https://developer.android.com/training/articles/direct-boot.html](https://developer.android.com/training/articles/direct-boot.html)。
