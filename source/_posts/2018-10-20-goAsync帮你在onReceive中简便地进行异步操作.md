---
layout:     post
title:      goAsync帮你在onReceive中简便地进行异步操作
subtitle:   开阔视野。
date:       2018-10-20
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
---

广播回调onReceive是在主线程跑的，所以我们不能在里面搞耗时操作，不然秒秒钟ANR。

又因为onReceive中的代码在执行完后，BroadcastReceiver对象就无效了，生命周期结束。
所以我们不能直接在里面起子线程，若应用进程被回收掉，线程的任务就可能无法完成。徒增不可控因素。

**解决：**
普遍的处理方式是在onReceive中再起一个IntentService去执行异步操作。这样就有了Service组件的保障，进程不会被轻易杀掉，但同时这个操作也比较重，代码实现上还得再去写个处理特殊业务的IntentService。

比如说我在onReceive里面只做一些很简单的耗时操作，可能就一两行代码我也要去写个Service？显然谷歌爹已经想到了这一点。

在API 11以后，BroadcastReceiver新增了一个静态内部类PendingResult，我们可以通过调用goAsync()方法来获取这个PendingResult对象，然后就可以愉快地进行子线程操作了，最后通过调用它的finish方法来结束广播接收者的生命。一切就变得可控了（相当于强行给Receiver续命）。

```java
...
    @Override
    public void onReceive(final Context context, final Intent intent) {
        final PendingResult result = goAsync();
        AsyncHandler.post(new Runnable() {
            @Override
            public void run() {
                // 在这里搞事情
                ...
                // 成功续命，可以手动结束了
                result.finish();
            }
        });
    }
...
```

**要注意的是，尽管这里可以异步操作，但耗时超过10s依然会爆ANR。**
其中AsyncHandler是一个简单封装的可复用的异步操作类（来自Android源码）：

```java
/**
 * Helper class for managing the background thread used to perform io operations
 * and handle async broadcasts.
 */
public final class AsyncHandler {
 
    private static final HandlerThread sHandlerThread = new HandlerThread("AsyncHandler");
    private static final Handler sHandler;
 
    static {
        sHandlerThread.start();
        sHandler = new Handler(sHandlerThread.getLooper());
    }
 
    public static void post(Runnable r) {
        sHandler.post(r);
    }
 
    public static void postDelayed(Runnable r, long delayedMills) {
        sHandler.postDelayed(r, delayedMills);
    }
 
    public static Message obtain(Runnable r) {
        return Message.obtain(sHandler, r);
    }
 
    public static void sendMessageDelayed(Message message, long delayedMills) {
        sHandler.sendMessageDelayed(message, delayedMills);
    }
 
    public static void removeCallbacks(int what) {
        sHandler.removeMessages(what);
    }
 
    private AsyncHandler() {}
}
```

最后，我们可以回味一下BroadcastReceiver源码是怎么描述PendingResult和goAsync的：

```java
    /**
     * State for a result that is pending for a broadcast receiver.  Returned
     * by {@link BroadcastReceiver#goAsync() goAsync()}
     * while in {@link BroadcastReceiver#onReceive BroadcastReceiver.onReceive()}.
     * This allows you to return from onReceive() without having the broadcast
     * terminate; you must call {@link #finish()} once you are done with the
     * broadcast.  This allows you to process the broadcast off of the main
     * thread of your app.
     *
     * <p>Note on threading: the state inside of this class is not itself
     * thread-safe, however you can use it from any thread if you properly
     * sure that you do not have races.  Typically this means you will hand
     * the entire object to another thread, which will be solely responsible
     * for setting any results and finally calling {@link #finish()}.
     */
    public static class PendingResult {
...
    /**
     * This can be called by an application in {@link #onReceive} to allow
     * it to keep the broadcast active after returning from that function.
     * This does <em>not</em> change the expectation of being relatively
     * responsive to the broadcast (finishing it within 10s), but does allow
     * the implementation to move work related to it over to another thread
     * to avoid glitching the main UI thread due to disk IO.
     *
     * @return Returns a {@link PendingResult} representing the result of
     * the active broadcast.  The BroadcastRecord itself is no longer active;
     * all data and other interaction must go through {@link PendingResult}
     * APIs.  The {@link PendingResult#finish PendingResult.finish()} method
     * must be called once processing of the broadcast is done.
     */
    public final PendingResult goAsync() {
        PendingResult res = mPendingResult;
        mPendingResult = null;
        return res;
    }
```
