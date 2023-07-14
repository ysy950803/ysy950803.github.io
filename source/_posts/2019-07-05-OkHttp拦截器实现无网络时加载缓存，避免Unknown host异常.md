---
layout:     post
title:      OkHttp拦截器实现无网络时加载缓存，避免Unknown host异常
subtitle:   拦截器玩出花。
date:       2019-07-05
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
---

我们在创建OkHttp客户端时，可以添加接口数据缓存，真的很香：

```java
File cacheDir = ... // 缓存目录，可以是内部存储也可以是外部存储的目录
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .cache(new Cache(cacheDir, 20 * 1024 * 1024)) // 这里给了20MB缓存目录容量，超过后会自动清理
        ....
        .build();
```

然后我们会发现，先正常请求网络数据，然后断开网络连接，重新请求，并没有返回缓存。
而是出现一些诸如“Unknown host…”解析不了域名这种异常，查看之前设置的缓存文件目录，也确实有文件，可怎么就不加载呢？
![在这里插入图片描述](https://imgconvert.csdnimg.cn/20190705234709502.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
哦，结果还要配置一下缓存策略，回到我们的主题：拦截器。
我们可以在拦截器中实现网络连接判断并强制开起缓存：

```java
private static class CacheInterceptor implements Interceptor {
    @Override
    public okhttp3.Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Request.Builder requestBuilder = request.newBuilder();
        if (/*手机没联网*/) {
            requestBuilder.cacheControl(CacheControl.FORCE_CACHE); // 直接使用缓存
        }
        return chain.proceed(requestBuilder.build());
    }
}
... // 然后记得给Client添加拦截器
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        ....
        .addInterceptor(new CacheInterceptor())
        .build();
```

如此一来，断开网络后，就会正确地加载缓存数据了。
