---
layout:     post
title:      InputMethodManager内存泄漏已成为历史
subtitle:   大人，时代变了。
date:       2020-11-27
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - AOSP
---

## 历史问题

相信做过很多业务开发的同学都遇到过Android应用的内存泄漏问题，虽然大部分泄漏都是我们自己**菜**导致的，但实际上系统服务也有可能出现内存泄漏。毕竟，代码都是人写的，AOSP也不是完美无瑕的。

说到系统服务，在处理文本输入的时候，我们以前经常会看到这样的泄漏：

![在这里插入图片描述](https://imgconvert.csdnimg.cn/20201127125205854.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70#pic_center)

这里大家也可自行搜索了解，大致上就是因为**InputMethodManager（下简称IMM）实例内部会持有View**，而View又持有Activity的引用，最终在Activity退出后没有正确处理View导致了Memory Leak。我们明白，系统服务生命周期一般是长于Activity的。

这里可以查看旧版AOSP源码（分支：android-9.0.0-r8）来取证：

```java
public final class InputMethodManager {
    ...
    /**
     * This is the root view of the overall window that currently has input
     * method focus.
     */
    View mCurRootView;
    /**
     * This is the view that should currently be served by an input method,
     * regardless of the state of setting that up.
     */
    View mServedView;
    /**
     * This is then next view that will be served by the input method, when
     * we get around to updating things.
     */
    View mNextServedView;
    ...
    /**
     * When the focused window is dismissed, this method is called to finish the
     * input method started before.
     * @hide
     */
    public void windowDismissed(IBinder appWindowToken) {
        ...
        synchronized (mH) {
            if (mServedView != null &&
                    mServedView.getWindowToken() == appWindowToken) {
                finishInputLocked();
            }
        }
    }
    ...
    /**
     * Disconnect any existing input connection, clearing the served view.
     */
    void finishInputLocked() {
        mNextServedView = null;
        if (mServedView != null) {
            ...
            mServedView = null;
            ...
        }
    }
    ...
    /**
     * Call this when a view is being detached from a {@link android.view.Window}.
     * @hide
     */
    public void onViewDetachedFromWindow(View view) {
        synchronized (mH) {
            ...
            if (mServedView == view) {
                mNextServedView = null;
                ...
            }
        }
    }
    ...
}
```

我们可以搜索源码发现虽然mServedView和mNextServedView都有在合适的时机做置空操作，**但最关键的输入焦点View即mCurRootView没有置空的地方**，这也是导致泄漏的主要原因。尤其是在列表视图（ListView，RecyclerView等）中如果itemView中带有输入框，尤其容易产生泄漏的问题。

曾经的解决办法通常都是反射操作IMM实例然后把这几个View对象强制置空，此处不再赘述。

## 大人，时代变了

我查阅了近几年的AOSP大版本源码，意外地发现，在Android 10的IMM中，这个内存泄漏的问题竟然修复了！有点惊奇的是，这个修复还是MIUI的工程师贡献的patch。

这个修复在2018年下半年就提交了，最终在Android 10才合入，下面的代码基于分支android-10.0.0_r30：

![](https://imgconvert.csdnimg.cn/20201127125312174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70#pic_center)

也就是说，在Android 9及以前，IMM的内存泄漏问题都没有得到官方的及时修复，最后还是国内厂商的工程师实在忍不住给修了（之前我还在MIUI的时候也给系统组提过这个bug）。

出于好奇，我查看了一下[这个patch的提交信息](https://cs.android.com/android/_/android/platform/frameworks/base/+/dff365ef4dc61239fac70953b631e92972a9f41f)：

![在这里插入图片描述](https://imgconvert.csdnimg.cn/20201127125344306.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70#pic_center)

看看描述，没差了，就是为了修复数年未解的IMM内存泄漏问题。不知道全球开发者为了这个玩意头疼了多久（毕竟Memory Leak也是一个项目质量指标的对吧，说白了影响你绩效 /狗头）。

这个问题也有对应的官方bug issue，大家有兴趣可以看看：[InputMethodManager#sInstance#mCurRootView cause memory leak](https://issuetracker.google.com/issues?q=116078227) ，最后也是得到了AOSP官方团队验证的：

![在这里插入图片描述](https://imgconvert.csdnimg.cn/20201127125406506.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70#pic_center)

## 进一步优化

虽然MIUI的大佬已经对此进行了修复，但IMM依然存在一些代码结构上的问题，可能导致了一些其他bug，官方团队在Android 11中对IMM源码做了[进一步优化](https://cs.android.com/android/_/android/platform/frameworks/base/+/970d9d2e0c979cf9a0ff0a79ef49044ed1363d4f) ，这次的改动还不小：

![在这里插入图片描述](https://imgconvert.csdnimg.cn/20201127125443171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70#pic_center)

这里我简单做一下介绍，大家感兴趣可以查看最新的源码。我们可以发现，在最新的IMM中，后面2个View已经从中去除了：

```java
public final class InputMethodManager {
    /**
     * This is the root view of the overall window that currently has input
     * method focus.
     */
    @GuardedBy("mH")
    ViewRootImpl mCurRootView;
    ...
}
```

只留下了mCurRootView的ViewRootImpl对象。原本IMM内很多跟mCurRootView相关的操作封装到了一个新建的**ImeFocusController**类中：

```java
public final class ImeFocusController {
	...
    private final ViewRootImpl mViewRootImpl;
    private boolean mHasImeFocus = false;
    private View mServedView;
    private View mNextServedView;

    @UiThread
    ImeFocusController(@NonNull ViewRootImpl viewRootImpl) {
        mViewRootImpl = viewRootImpl;
    }

    private InputMethodManagerDelegate getImmDelegate() {
        return mViewRootImpl.mContext.getSystemService(InputMethodManager.class).getDelegate();
    }
    
    @UiThread
    void onViewDetachedFromWindow(View view) {
        ...
        if (mServedView == view) {
            mNextServedView = null;
            mViewRootImpl.dispatchCheckFocus();
        }
    }

    @UiThread
    void onWindowDismissed() {
        ...
        if (mServedView != null) {
            getImmDelegate().finishInput();
        }
        getImmDelegate().setCurrentRootView(null);
        mHasImeFocus = false;
    }
    ...
    public View getServedView() {
        return mServedView;
    }

    public View getNextServedView() {
        return mNextServedView;
    }

    public void setServedView(View view) {
        mServedView = view;
    }

    public void setNextServedView(View view) {
        mNextServedView = view;
    }
}
```

我们可以看到，曾经的置空操作基本都放到了这个Controller中。mServedView和mNextServedView不再是IMM的成员，而是ImeFocusController的成员，且ImeFocusController又是ViewRootImpl的成员（此Controller实例化在ViewRootImpl的构造方法中）。

这个patch的优化，一定程度上解除了View对IMM的依赖，代码有效解耦。输入焦点处理的相关逻辑都转移到了View本身来控制，进一步避免了内存泄漏。

## 后话

其实每次我在搜到一些问题的解决资料时，都会关注一下帖子的发布时间，我发现IMM内存泄漏这个问题基本都是2019年之前的，好奇就去看了下最新的源码发现果然有所修复。Android系统还是在朝着越来越稳定，性能越来越优秀的方向发展。

在AOSP的Code Review平台上也可以发现，其实国内外各大手机厂商都对AOSP有着巨大的贡献，大家也不是一味埋头搞自己的定制，有bug还是会反哺修复的。再次感谢开源！

感兴趣可以浏览，会看到很多change的owner都不是Google的：

- [https://android-review.googlesource.com/q/xiaomi](https://android-review.googlesource.com/q/xiaomi)
- [https://android-review.googlesource.com/q/huawei](https://android-review.googlesource.com/q/huawei)
- [https://android-review.googlesource.com/q/oppo](https://android-review.googlesource.com/q/oppo)
- [https://android-review.googlesource.com/q/samsung](https://android-review.googlesource.com/q/samsung)
- [https://android-review.googlesource.com/q/sony](https://android-review.googlesource.com/q/sony)
