---
layout:     post
title:      "OkHttp自定义重试拦截器出现IllegalStateException: cannot make a new request……异常"
subtitle:   拦截器玩出花。
date:       2019-09-20
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
---

### 问题

OkHttp自定义重试拦截器常见实现方式：

```java
private static class RetryInterceptor implements Interceptor {
    @Override
    public okhttp3.Response intercept(Chain chain) throws IOException {
        int retryCount = 0;
        Request request = chain.request();
        okhttp3.Response response = chain.proceed(request);
        while (!response.isSuccessful() && retryCount < RETRY_MAX_COUNT) {
            retryCount++;
            response = chain.proceed(request);
        }
        return response;
    }
}
```

但是没有人告诉我，在3.14.x版本上会出现这个异常：

> AndroidRuntime: java.lang.IllegalStateException: cannot make a new request because the previous response is still open: please call response.close()

第一反应去OkHttp的issue查查，果然有和我同病相怜的：
[https://github.com/square/okhttp/issues/4986](https://github.com/square/okhttp/issues/4986)

### 解决

```java
while (!response.isSuccessful() && retryCount < RETRY_MAX_COUNT) {
    retryCount++;
    response.close(); // 很简单，加上这一句
    response = chain.proceed(request);
}
```

### 调查

虽说很快就解决了，但是要是异常提示不告诉你要close呢，我们就要自己去探索了。所以还是忍不住扒下来源码看了看，搜索这个异常提示位于：
**okhttp/src/main/java/okhttp3/internal/connection/Transmitter.java**
可以看到，是在newExchange方法中抛出的异常：

```java
  /** Returns a new exchange to carry a new request and response. */
  Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    synchronized (connectionPool) {
      if (noMoreExchanges) {
        throw new IllegalStateException("released");
      }
      if (exchange != null) {
        throw new IllegalStateException("cannot make a new request because the previous response "
            + "is still open: please call response.close()");
      }
    }
	……
  }
```

为了简单点，我们就直接搜索exchange在什么时候置的空，之所以response.close()能解决问题那就说明close方法最终会调用到exchange = null。查出来2个地方：

```java
  // 第一个调用处是来自RetryAndFollowUpInterceptor这一内部重试实现，此处可以忽略
  public void exchangeDoneDueToException() {
    synchronized (connectionPool) {
      if (noMoreExchanges) throw new IllegalStateException();
      exchange = null;
    }
  }

  // 这个方法应该才是最终调用
  @Nullable IOException exchangeMessageDone(
      Exchange exchange, boolean requestDone, boolean responseDone, @Nullable IOException e) {
    boolean exchangeDone = false;
    synchronized (connectionPool) {
      if (exchange != this.exchange) {
        return e; // This exchange was detached violently!
      }
      boolean changed = false;
      if (requestDone) {
        if (!exchangeRequestDone) changed = true;
        this.exchangeRequestDone = true;
      }
      if (responseDone) {
        if (!exchangeResponseDone) changed = true;
        this.exchangeResponseDone = true;
      }
      // 其实从这里的语义就能看出，只有responseDone了，exchange才会置空，正好对应response.close()
      if (exchangeRequestDone && exchangeResponseDone && changed) {
        exchangeDone = true;
        this.exchange.connection().successCount++;
        this.exchange = null;
      }
    }
    if (exchangeDone) {
      e = maybeReleaseConnection(e, false);
    }
    return e;
  }
```

**回溯一部分调用路径：**
ResponseBodySource#close -> complete --> Exchange#bodyComplete --> Transmitter#exchangeMessageDone
**然后从Response的close方法开始看**：
其实就是调用了ResponseBody的close：

```java
  @Override public void close() {
    Util.closeQuietly(source());
  }
  // source是一个抽象方法，返回的BufferedSource是一个Closeable
  public abstract BufferedSource source();
```

RealResponseBody进行了实现：

```java
public final class RealResponseBody extends ResponseBody {
  ……
  private final BufferedSource source;

  public RealResponseBody(
      @Nullable String contentTypeString, long contentLength, BufferedSource source) {
    this.contentTypeString = contentTypeString;
    this.contentLength = contentLength;
    this.source = source;
  }
  ……
  @Override public BufferedSource source() {
    return source;
  }
}
```

正好对应Exchange的openResponseBody方法：

```java
  public ResponseBody openResponseBody(Response response) throws IOException {
    try {
      ……
      ResponseBodySource source = new ResponseBodySource(rawSource, contentLength);
      // 将ResponseBodySource的实例作为RealResponseBody的source传入
      return new RealResponseBody(contentType, contentLength, Okio.buffer(source));
    } catch (IOException e) {
      eventListener.responseFailed(call, e);
      trackFailure(e);
      throw e;
    }
  }
```

调用就这样合龙了。
所以我们调用Response#close方法最终就会走到Transmitter#exchangeMessageDone，对exchange置空，这样就不会抛出那个异常了。
