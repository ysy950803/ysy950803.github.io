---
layout:     post
title:      源码茶舍之android:externalService是什么属性？实现原理？
subtitle:   品味AOSP。
date:       2019-04-10
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - AOSP
---

## 发现

在AndroidManifest中声明Service时，偶然发现一个布尔类型的属性：`android:externalService`
示例如下：

```xml
<service
    android:externalService="true"
    ... />
```

如果minSDK小于24，会显示警告，很显然这是一个24以后的新东西。
先顾名思义一下，external的service，外置（外挂）的服务？它和 `android:exported` 以及 `android:isolatedProcess` 属性是什么关系？

## 初探

先谷歌一下，再百度一下，无果。竟然没有一个人解释这是什么东西，这更加激发了我的好奇心。
马上去Android官网搜，搜到service标签的文档，心中窃喜：
https://developer.android.com/guide/topics/manifest/service-element
翻完整个文档，发现居然也没有 `android:externalService` 的说明，难道是太新了忘了补充文档吗？
不过我们可以从中先复习一下 `android:exported` 以及 `android:isolatedProcess` 属性：

##### exported

> Whether or not components of other applications can invoke the service or interact with it — "true" if they can, and "false" if not. When the value is "false", only components of the same application or applications with the same user ID can start the service or bind to it.
> The default value depends on whether the service contains intent filters. The absence of any filters means that it can be invoked only by specifying its exact class name. This implies that the service is intended only for application-internal use (since others would not know the class name). So in this case, the default value is "false". On the other hand, the presence of at least one filter implies that the service is intended for external use, so the default value is "true".
> This attribute is not the only way to limit the exposure of a service to other applications. You can also use a permission to limit the external entities that can interact with the service (see the [permission](https://developer.android.com/guide/topics/manifest/service-element.html#prmsn) attribute).

其实这个属性大家也是耳熟能详了，从官网解释来看，主要就是限制该Service能否被其他应用调用。
同时，还特意解释了默认值的决定情况，即你不需要总是显式地声明此属性。

- 如果service标签下没有添加任何intent-filter，那么就默认为false，即不对外暴露，因为这种情况下其他应用不知道你的Service类名，当然就只能内部调用了（我觉得谷歌这个解释有点牵强，因为即便你知道包名和类名，但exported为false的话，你也调不了）
- 如果添加了至少一个能让外部调用的filter标签（比如action什么的），那么默认值就是true了。

最后，还提醒了此属性不是唯一一种限制外部调用的途径，permission也可以。

##### isolatedProcess

> If set to true, this service will run under a special process that is isolated from the rest of the system and has no permissions of its own. The only communication with it is through the Service API (binding and starting).

这个比较简单，顾名思义也知道是让Service独立运行到一个特定进程中。

## 挖掘

既然搜也搜不到，官网也藏着掖着，那就只能我们自己挖掘了。这怎么少得了Read the fucking code呢？
这就要从系统启动App并解析Manifest文件开始说起了，我们只简要地分析一下：
1、SystemServer进程启动**PackageManagerService**（PMS）服务；
2、PMS扫描文件目录的过程，会调用到**scanPackageLI**方法，此方法中会实例化**PackageParser**，这是关键；
3、PackageParser会解析Manifest中的各种标签，其中**parseService**便是解析service的。

来看看parseService方法的代码，长得一匹，得精简一下：

```java
private Service parseService(Package owner, Resources res,
        XmlResourceParser parser, int flags, String[] outError,
        CachedComponentArgs cachedArgs)
        throws XmlPullParserException, IOException {
    TypedArray sa = res.obtainAttributes(parser,
            com.android.internal.R.styleable.AndroidManifestService);

    ...

    Service s = new Service(cachedArgs.mServiceArgs, new ServiceInfo());
    ...
    // exported的解析
    boolean setExported = sa.hasValue(
            com.android.internal.R.styleable.AndroidManifestService_exported);
    if (setExported) {
        s.info.exported = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestService_exported, false);
    }

    ...

    s.info.flags = 0;
    ...
    // isolatedProcess的解析
    if (sa.getBoolean(
            com.android.internal.R.styleable.AndroidManifestService_isolatedProcess,
            false)) {
        s.info.flags |= ServiceInfo.FLAG_ISOLATED_PROCESS;
    }
    // externalService的解析看这里！
    if (sa.getBoolean(
            com.android.internal.R.styleable.AndroidManifestService_externalService,
            false)) {
        // 等我们去看看这个FLAG的注释就知道是干啥的了
        s.info.flags |= ServiceInfo.FLAG_EXTERNAL_SERVICE;
    }
    ...

    sa.recycle();

    ...

    int outerDepth = parser.getDepth();
    int type;
    while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
           && (type != XmlPullParser.END_TAG
                   || parser.getDepth() > outerDepth)) {
        if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
            continue;
        }

        if (parser.getName().equals("intent-filter")) {
            ServiceIntentInfo intent = new ServiceIntentInfo(s);
            ...
            s.intents.add(intent); // 注意这里的intents集合
        } else if (parser.getName().equals("meta-data")) {
            ...
        } else {
            ...
        }
    }

    if (!setExported) {
        // 这儿有个小发现，正好验证了我们上述的官方文档复习，intent-filter标签数量大于0时，exported自动就赋值为true了
        s.info.exported = s.intents.size() > 0;
    }

    return s;
}
```

接着我打开了ServiceInfo文件，总算是找到了官方解释：

```java
/**
 * Bit in {@link #flags}: If set, the service can be bound and run in the
 * calling application's package, rather than the package in which it is
 * declared.  Set from {@link android.R.attr#externalService} attribute.
 */
public static final int FLAG_EXTERNAL_SERVICE = 0x0004;
```

顺便，我发现Context源码中也新增了一个bindService的flag：

```java
/**
 * Flag for {@link #bindService}: The service being bound is an
 * {@link android.R.attr#isolatedProcess isolated},
 * {@link android.R.attr#externalService external} service.  This binds the service into the
 * calling application's package, rather than the package in which the service is declared.
 * <p>
 * When using this flag, the code for the service being bound will execute under the calling
 * application's package name and user ID.  Because the service must be an isolated process,
 * it will not have direct access to the application's data, though.
 *
 * The purpose of this flag is to allow applications to provide services that are attributed
 * to the app using the service, rather than the application providing the service.
 * </p>
 */
public static final int BIND_EXTERNAL_SERVICE = 0x80000000;
```

结合这俩来看，就非常明白了。稍作总结一下：
1、声明externalService为true就是让该Service可以绑定并运行在调用方的App中，而不是在声明这个Service的App中，这和我们最开始猜测的外置服务之意相符；
2、注释还进一步说明，此Service还同时须要设置isolatedProcess为true；
3、此Service的业务代码会在调用方App的包名环境下执行，因为它已经是独立进程（isolated process）了，从声明它的App那儿离家出走，改名换姓；
4、当然，此Service不能直接访问调用方App的数据。
5、目的是想从概念上分离Service提供方和使用方这二者（这是我个人理解）。

## 再挖

经过上述初步挖掘，我们算是搞清楚了externalService这个属性的含义和作用，但依然有一些疑惑。比如：
具体使用一定要带BIND_EXTERNAL_SERVICE这个flag吗？
只把isolatedProcess和externalService设为true就能用了吗？

要解决这些疑惑，有两个办法，一是自己去尝试，二是再次Read the fucking code！想不到吧？
先给出源码地址：https://android.googlesource.com/platform/cts/+/master/tests/tests/externalservice
其实这是CTS测试用到的单元测试代码，我们可以从单元测试项来看看有哪些重要的注意点。
这里具体只需要关注 [service/AndroidManifest.xml](https://android.googlesource.com/platform/cts/+/master/tests/tests/externalservice/service/AndroidManifest.xml) 和 [src/ExternalServiceTest.java](https://android.googlesource.com/platform/cts/+/master/tests/tests/externalservice/src/android/externalservice/cts/ExternalServiceTest.java) 这两个文件即可。

从单元测试的Manifest中我们可以了解到几种主要的失败情况：
1、没有把exported设成true：

```xml
<service android:name=".ExternalNonExportedService"
         android:isolatedProcess="true"
         android:externalService="true"
         android:exported="false"/>
```

就对应这个错误用例：

```java
/** Tests that BIND_EXTERNAL_SERVICE requires that an externalService be exported. */
public void testFailBindExternalNonExported() {
    Intent intent = new Intent();
    intent.setComponent(
            new ComponentName(sServicePackage, sServicePackage+".ExternalNonExportedService"));
    try {
        getContext().bindService(intent, mConnection,
                Context.BIND_AUTO_CREATE | Context.BIND_EXTERNAL_SERVICE);
        fail("Should not be able to BIND_EXTERNAL_SERVICE to non-exported service");
    } catch (SecurityException e) {
    }
}
```

**所以，exported也要同时设成true才行。**
2、没有把isolatedProcess设成true：

```xml
<service android:name=".ExternalNonIsolatedService"
         android:isolatedProcess="false"
         android:externalService="true"
         android:exported="true"/>
```

就对应此错误：

```java
/** Tests that BIND_EXTERNAL_SERVICE requires the service be an isolatedProcess. */
public void testFailBindExternalNonIsolated() {
    ...
        fail("Should not be able to BIND_EXTERNAL_SERVICE to non-isolated service");
    ...
}
```

3、**没有用BIND_EXTERNAL_SERVICE进行绑定也是不行的**：

```java
/** Tests that an externalService can only be bound with BIND_EXTERNAL_SERVICE. */
public void testFailBindWithoutBindExternal() {
    Intent intent = new Intent();
    intent.setComponent(new ComponentName(sServicePackage, sServicePackage+".ExternalService"));
    try {
        getContext().bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
        fail("Should not be able to bind to an external service without BIND_EXTERNAL_SERVICE");
    } catch (SecurityException e) {
    }
}
```

因此，最后正确的用法必须是这样声明：

```xml
<service
    android:name=".XXXService"
    android:exported="true"
    android:externalService="true"
    android:isolatedProcess="true" />
```

同时通过：

```java
bindService(intent, mConnection, Context.BIND_AUTO_CREATE | Context.BIND_EXTERNAL_SERVICE);
```

进行绑定，BIND_EXTERNAL_SERVICE是必须，其余flag根据自己需要决定。

## 后话

我专门搜了下AOSP的代码提交记录，发现这个功能3年前就开发好了，链接如下：
https://android.googlesource.com/platform/frameworks/base/+/b9a8666eb5504f022343fef9087135b7d937ddf8%5E%21/
Commit信息：

```
Add external services, a way to run isolated processes as a different package.

This adds android:externalService boolean attribute to <service>. If that
attribute is true, then bindService() may be called with
BIND_EXTERNAL_SERVICE to create the new service process under the calling
package's name and uid. The service will execute the code from the package in
which it is declared, but will appear to run as the calling application.

External services may only be used if android:exported="false" and
android:isolatedProcess="true".

Bug: 22084679
Bug: 21643067
Change-Id: I3c3a5f0ef58738316c5efeab9044e43e09220d01
```

改动也不多，就这么几个文件：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190410235236156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
其中最重要的逻辑就在ActiveServices当中，很长的方法，此处只保留关键新增部分：

```java
private ServiceLookupResult retrieveServiceLocked(Intent service,
        String resolvedType, String callingPackage, int callingPid, int callingUid, int userId,
        boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal,
        boolean allowInstant) {
    ServiceRecord r = null;
    ...
            ComponentName name = new ComponentName(
                    sInfo.applicationInfo.packageName, sInfo.name);
            if ((sInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0) { // 先验证flag，即对应externalService属性是否为true
                if (isBindExternal) { // isBindExternal表示是否是通过BIND_EXTERNAL_SERVICE绑定服务
                    if (!sInfo.exported) {
                    	// 你看，exported必须也是true，否则直接丢你一脸异常
                        throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                                " is not exported");
                    }
                    if ((sInfo.flags & ServiceInfo.FLAG_ISOLATED_PROCESS) == 0) {
                    	// isolatedProcess也必须true
                        throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                                " is not an isolatedProcess");
                    }
                    // Run the service under the calling package's application.
                    // （这里是源码注释，即下面的代码就是如何让调用方App拥有这个外置Service）
                    ApplicationInfo aInfo = AppGlobals.getPackageManager().getApplicationInfo(
                            callingPackage, ActivityManagerService.STOCK_PM_FLAGS, userId);
                    if (aInfo == null) {
                        throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " +
                                "could not resolve client package " + callingPackage);
                    }
                    // 其实就是重新设置了一遍ServiceInfo，让此Service改名换姓
                    sInfo = new ServiceInfo(sInfo);
                    sInfo.applicationInfo = new ApplicationInfo(sInfo.applicationInfo);
                    sInfo.applicationInfo.packageName = aInfo.packageName;
                    sInfo.applicationInfo.uid = aInfo.uid;
                    name = new ComponentName(aInfo.packageName, name.getClassName());
                    service.setComponent(name);
                } else {
                    throw new SecurityException("BIND_EXTERNAL_SERVICE required for " +
                            name);
                }
            } else if (isBindExternal) {
                throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                        " is not an externalService");
            }
	...
}
```
