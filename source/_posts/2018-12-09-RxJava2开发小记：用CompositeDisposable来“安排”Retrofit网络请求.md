---
layout:     post
title:      RxJava2开发小记：用CompositeDisposable来“安排”Retrofit网络请求
subtitle:   知名库之间的故事。
date:       2018-12-09
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Java
    - Android
---

## 情景

前不久项目遇到了偶现的OOM问题，看调用栈发现有RxJava相关，我们项目以RxJava2+RxAndroid+Retrofit2+OkHttp为基础设施的。
上谷歌搜了一转，最终竟踏入了[RxAndroid](https://github.com/ReactiveX/RxAndroid)的GitHub issue区，发现有个老外和我情景类似，原帖链接在此：https://github.com/ReactiveX/RxAndroid/issues/387， 他说他只用Retrofit发起1500个请求没毛病，但是加上RxAndroid就炸了，他怀疑RxAndroid有Bug。
这个问题被项目Owner（即JakeWharton大神）回复了，他给大家解释了这种框架组合的正确用法。下面是他的原话：

> The problem is that **Schedulers.io()** uses a cached thread pool without a limit and thus is trying to create 1500 threads. You should consider using a **Scheduler** that has a fixed limit of threads, or using RxJava 2.x's **parallel()** operator to parallelize the operation to a fixed number of workers.
> If you're using raw Retrofit by default it uses OkHttp's dispatcher which limits the threads to something like 64 (with a max of 5 per host). That's why you aren't seeing it fail.
> If you use **createAsync()** when creating the **RxJava2CallAdapterFactory** it will create fully-async **Observable** instances that don't require a **subscribeOn** and which use OkHttp's **Dispatcher** just like Retrofit would otherwise. Then you only need **observeOn** to move back to the main thread, and you avoid all additional thread creation.

大概意思是说，Schedulers.io()会用一个没有限制的缓存线程池，所以你需要一个有线程数量限制的调度器，或者用parallel操作符来进行并发操作（否则容易OOM）。
Retrofit默认组合OkHttp所用的线程数量是有限制的（比如64个），因此这位提问的老外才没出问题。
Jake大神建议，配合RxAndroid时，使用RxJava2CallAdapterFactory的createAsync方法来构造Retrofit实例，这样就会创建一个完全异步的Observeable，不需要再把它单独丢到自己指定的线程调度器（即Schedulers.io()）当中去，只需要observeOn主线程即可。

## 改进

看完这段话后，我发现自己的项目还真躺枪了，原本我是这么写的：

```java
public interface RetrofitInterface {
    // 获取远程服务器的数据
    @GET("test/getData")
    Observable<DataClass> getRemoteData();
}
```

```java
...
    public static RetrofitInterface createRequest() {
        return getRetrofit().create(RetrofitInterface.class);
    }
    private synchronized static Retrofit getRetrofit() {
        if (sRetrofit == null) {
            sRetrofit = new Retrofit.Builder()
                    .baseUrl("http://test.com")
                    .client(getHttpClient())
                    .addConverterFactory(GsonConverterFactory.create())
                    .addCallAdapterFactory(RxJava2CallAdapterFactory.create()) // 【问题主要是这里】
                    .build();
        }
        return sRetrofit;
    }
...
```

按照上面的意思，这里就该用**createAsync**而不是**create**了。
然后我的网络请求是这么写的：

```java
// 创建网络请求的Observable
Observable<DataClass> remoteDataObservable = RetrofitFactory.createRequest()
        .getRemoteData()
        .subscribeOn(Schedulers.io()) // 【问题主要是这里】
        .map(dataResponse -> {
            DataClass remoteData = dataResponse;
            // 假装对返回数据做了一些处理
            // ...
            return remoteData;
        })
        .observeOn(AndroidSchedulers.mainThread()); // 在UI线程中暗中观察并及时消费
```

按照上面的意思，这里**createAsync**已经会为你指定异步线程了，你就无需在外部再去调用**subscribeOn(Schedulers.io())**，所以把这句去掉。
**注意：** 这两步改动是一起的，缺一不可。
如此一来，就能避免这方面带来的内存溢出问题了。所以这也给我们提了个醒，Schedulers.io()不要随便用，它只适合做一些轻量的异步工作，不要试图用它支撑高并发。

## 后话

针对这个问题，我特意去看了看createAsync方法的源码注释：

```java
  /**
   * Returns an instance which creates asynchronous observables. Applying
   * {@link Observable#subscribeOn} has no effect on stream types created by this factory.
   */
  public static RxJava2CallAdapterFactory createAsync() {
    return new RxJava2CallAdapterFactory(null, true);
  }
```

没错，它专门提到：subscribeOn方法对createAsync最终构造出来的Observeable是没有影响的，也就是说，只要你用了createAsync，即便后续再调用subscribeOn(Schedulers.io())，都没什么用处，并不会作用到网络请求。一直到你调用observeOn之前（包括map等操作），线程都不会切换。

**调料包：** 关于RxJava的线程切换及操作符作用域，看看这篇应该就够了：https://www.jianshu.com/p/59c3d6bb6a6b
