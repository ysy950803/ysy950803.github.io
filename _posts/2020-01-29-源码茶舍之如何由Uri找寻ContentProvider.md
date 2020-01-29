---
layout:     post
title:      源码茶舍之如何由Uri找寻ContentProvider
subtitle:   品味AOSP。
date:       2020-01-29
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - AOSP
---

### 引子

我们都知道四大组件之一ContentProvider的用处，它给大家提供一种统一的数据访问格式。调用者无需关心数据源于何处（如DB、XML文件和网络等），只需获取到对应的ContentResolver来进行增删查改即可。
自己实现一个Provider的时候，也会在配置文件中声明如下：

```xml
<provider
    android:name=".provider.TestProvider"
    android:authorities="com.xxx.yyy.provider"
    android:exported="true"
    android:readPermission="com.xxx.yyy.permission.READ_PROVIDER" />
```

其中 `authorities` 是该Provider的唯一标识，所以一般都写成包名与其他字符串的组合形式，若需提供数据给其他应用，则 `exported` 要设为true，同时比较规范的做法还需要加上读写权限。
然后，我们再从常见的查询操作说起：

```java
ContentResolver r = getContentResolver();
Uri uri = Uri.parse("content://com.xxx.yyy.provider/test_path/1");
Cursor c = r.query(uri, null, null, null, null);
// ...
```

如同访问某个网站，我们访问ContentProvider也需要一个URI，其数据格式：

- scheme前缀是固定的： **content://**
- 授权host：此例中为 **com.xxx.yyy.provider**
- 路径与参数：此例中为 **test_path/1**

那么，系统是如何通过这样一个URI来锁定对应的ContentProvider呢？

### 找寻

主要涉及源码（基于Android 10）：

```bash
frameworks/base/core/java/android/content/ContentResolver.java
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/ActivityThread.java
```

大致思路，便是追踪上述 `query` 方法中的参数uri，看看它的流向。根据源码设计的套路，起初几层调用都是看不到要害之处的，所以我们无需细读。来来来，先看ContentResolver的 `query` 方法：

```java
    @Override
    public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
            @Nullable String[] projection, @Nullable Bundle queryArgs,
            @Nullable CancellationSignal cancellationSignal) {
        // ...
        // 获取“不稳定”的Provider
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }
        IContentProvider stableProvider = null;
        Cursor qCursor = null;
        try {
            // ...
            try {
                // 尝试查询操作
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        queryArgs, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                // The remote process has died...  but we only hold an unstable
                // reference though, so we might recover!!!  Let's try!!!!
                // This is exciting!!1!!1!!!!1
                // 这段注释我特意没删，感觉特别皮。大意：远程进程已死亡，但我们还持有unstableProvider的引用，快试试回收它的资源！这真是一颗赛艇！（虽然我不知道到底这哪儿exciting了）
                unstableProviderDied(unstableProvider);
                // “不稳定”的Provider操作失败，获取“稳定”的Provider
                stableProvider = acquireProvider(uri);
                if (stableProvider == null) {
                    return null;
                }
                // 再次尝试查询操作
                qCursor = stableProvider.query(
                        mPackageName, uri, projection, queryArgs, remoteCancellationSignal);
                }
            if (qCursor == null) {
                return null;
            }
            // ...
        } catch (RemoteException e) {
            // ...
            return null;
        } finally {
            // 释放资源
        }
    }
```

从上述源码可得知，有两处代码在根据uri获取ContentProvider，即ContentResolver的 `acquireUnstableProvider` 和 `acquireProvider` 方法。先看看前者（后者最终殊途同归，本文不额外分析）：

```java
    public final IContentProvider acquireUnstableProvider(Uri uri) {
        if (!SCHEME_CONTENT.equals(uri.getScheme())) {
            // 这里硬核匹配字符串，凡是scheme不是content://的直接再见，所以它是固定的
            return null;
        }
        String auth = uri.getAuthority(); // 按例，此处获取到的字符串便包含"com.xxx.yyy.provider"
        if (auth != null) {
            // 此为ContentResolver中的抽象方法，由子Resolver各自具体实现
            return acquireUnstableProvider(mContext, uri.getAuthority());
        }
        return null;
    }
```

于是我们追踪到ContextImpl的静态内部类ApplicationContentResolver：

```java
    private static final class ApplicationContentResolver extends ContentResolver {
        @UnsupportedAppUsage
        private final ActivityThread mMainThread;
        // ...
        @Override
        protected IContentProvider acquireUnstableProvider(Context c, String auth) {
            return mMainThread.acquireProvider(c,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), false);
        }
    }
```

实际调用到ActivityThread当中去了，注意此时传递的关键参数已经是 **auth** 而不是uri了：

```java
    @UnsupportedAppUsage
    public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        // 获取已存在的Provider    
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
        	return provider;
        }
        // ...
        // 没获取到再尝试安装，这里来个插眼，等会有大用
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
```

看源码一般来说最好先深后广，且优先搞清热点代码。接下来我们看 `acquireExistingProvider` 方法：

```java
    public final IContentProvider acquireExistingProvider( Context c, String auth, int userId, boolean stable) {
        synchronized (mProviderMap) {
            final ProviderKey key = new ProviderKey(auth, userId);
            // 关注这个存储Provider记录的的map，其实这里就是本文重点
            final ProviderClientRecord pr = mProviderMap.get(key);
            if (pr == null) {
                return null;
            }

            IContentProvider provider = pr.mProvider; // 最终获取Provider实例
            IBinder jBinder = provider.asBinder();
            if (!jBinder.isBinderAlive()) {
                // Provider所在进程已死，直接返回null
                handleUnstableProviderDiedLocked(jBinder, true);
                return null;
            }
            // ...
            return provider;
        }
    }
```

分析到这里，就自然而然有几个问题了， **ProviderKey** 是什么，怎么构造的？ **mProviderMap** 又是什么时候填充的？
带着问题，先看前者：

```java
    private static final class ProviderKey {
        final String authority;
        final int userId;

        public ProviderKey(String authority, int userId) {
            this.authority = authority;
            this.userId = userId;
        }

        @Override
        public boolean equals(Object o) {
            // ...
        }

        @Override
        public int hashCode() {
            // ...
        }
    }
```

可见， **ProviderKey** 是ActivityThread当中的一个内部POJO，非常普通，没有对入参做任何特殊处理。那么ContentProvider也就是根据 **authority** 和 **userId** 来唯一确定的，对应了文章开头的介绍。
此外，由于Android目前是多用户操作系统（国产ROM淡化了此概念，但应用双开、系统分身等功能实现均与多用户有关），所以这里用户id是必要的。

接下来看后一个问题， **mProviderMap** 从哪儿来？什么时候添加的Provider记录？很简单了，还是在ActivityThread当中，实例化如下：

```java
    @UnsupportedAppUsage
    final ArrayMap<ProviderKey, ProviderClientRecord> mProviderMap
        = new ArrayMap<ProviderKey, ProviderClientRecord>();
```

且仅有一处在进行 `put` 操作：

```java
    private ProviderClientRecord installProviderAuthoritiesLocked(IContentProvider provider,
            ContentProvider localProvider, ContentProviderHolder holder) {
        final String auths[] = holder.info.authority.split(";");
        final int userId = UserHandle.getUserId(holder.info.applicationInfo.uid);

        if (provider != null) {
            // ...
        }

        final ProviderClientRecord pcr = new ProviderClientRecord(
                auths, provider, localProvider, holder);
        for (String auth : auths) {
            final ProviderKey key = new ProviderKey(auth, userId);
            final ProviderClientRecord existing = mProviderMap.get(key);
            if (existing != null) {
                // ...
            } else {
                mProviderMap.put(key, pcr); // 在此处添加的
            }
        }
        return pcr;
    }
```

可见，ProviderClientRecord实例的构造是在这个 `installProviderAuthoritiesLocked` 私有方法中完成并添加到map中的。
这里有个小插曲**特别注意**：方法的第一行代码，对 **authority** 字符串进行了分割（分隔符为;），最终ProviderClientRecord的数量也取决于分割出来的数组。所以在Manifest配置文件中声明 `android:authorities` 属性时，可以填入多个授权host（就好比多个域名可以同时指向一个网站），以分号分割，难怪属性名要用复数呢。

接下来看看 `installProviderAuthoritiesLocked` 方法的调用处：

```java
    @UnsupportedAppUsage
    private ContentProviderHolder installProvider(Context context,
            ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
        if (holder == null || holder.provider == null) {
            // ...
        } else {
            provider = holder.provider;
            // ...
        }

        ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
            // ...
            IBinder jBinder = provider.asBinder();
            if (localProvider != null) {
                ComponentName cname = new ComponentName(info.packageName, info.name);
                ProviderClientRecord pr = mLocalProvidersByName.get(cname);
                if (pr != null) {
                    // ...
                } else {
                    // ...
                    // 第一处调用
                    pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                    // ...
                }
                retHolder = pr.mHolder;
            } else {
                ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
                if (prc != null) {
                    // ...
                } else {
                    // 第二处调用
                    ProviderClientRecord client = installProviderAuthoritiesLocked(
                            provider, localProvider, holder);
                    // ...
                }
                retHolder = prc.holder;
            }
        }
        return retHolder;
    }
```

由上， `installProviderAuthoritiesLocked` 方法的调用均在 `installProvider` 方法中。还记得上文的“插眼”吗？呼应上了。

### 总结

- 在我们使用ContentResolver来进行查询操作时，`query` 方法层层调用到 **ActivityThread** 的 `acquireExistingProvider` 方法，根据URI字符串当中的授权host（即 **authority** ）和当前所在用户的 **userId** 来获取对应的Provider实例。

- 当 `acquireExistingProvider` 获取不到时，则通过 `installProvider` 方法来安装Provider并把其载体 **ProviderClientRecord** 添加到 **mProviderMap** 中。

- AndroidManifest中声明Provider时， `android:authorities` 属性可以填多个字符串，以分号分割：

  ```xml
  <provider
      android:name=".provider.TestProvider"
      android:authorities="com.xxx.yyy.provider;cn.xxx.yyy.provider;net.xxx.yyy.provider"
      ... />
  ```

  如此可以写成多种不同host的URI，映射的却还是同一个ContentProvider。具体的好处我能想到的有几点：

  - 与同IP多域名的网站一样，域名多样化，提前抢占一些host，避免三方假冒。
  - 提供不同的URI分别给内部和外部开发者使用，便于区分和数据统计。
