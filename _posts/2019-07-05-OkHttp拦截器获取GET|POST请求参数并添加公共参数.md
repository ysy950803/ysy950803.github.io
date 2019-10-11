---
layout:     post
title:      OkHttp拦截器获取GET/POST请求参数并添加公共参数
subtitle:   拦截器玩出花。
date:       2019-07-05
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - OkHttp
---

我们在创建OkHttp客户端时，可以添加各种拦截器，真的很香：

```java
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .addInterceptor(new XXXInterceptor())
        .addInterceptor(new YYYInterceptor())
        ....
        .build();
```

服务端经常需要我们做一些参数校验的工作，需要在本地先把请求参数封装起来再加密之类的，那么我们可以用拦截器来实现获取所有get或post参数，还可以添加一些公共参数，这样就不用在每个接口定义那去加了：

```java
private static class ParamsInterceptor implements Interceptor {
    private static final String METHOD_GET = "GET";
    private static final String METHOD_POST = "POST";
    private static final String HEADER_KEY_USER_AGENT = "User-Agent";

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Request.Builder requestBuilder = request.newBuilder();
        HttpUrl.Builder urlBuilder = request.url().newBuilder();

        if (METHOD_GET.equals(request.method())) { // GET方法
            // 这里可以添加一些公共get参数
            urlBuilder.addEncodedQueryParameter("key_xxx", "value_xxx");
            HttpUrl httpUrl = urlBuilder.build();
            
            // 打印所有get参数
            Set<String> paramKeys = httpUrl.queryParameterNames();
            for (String key : paramKeys) {
                String value = httpUrl.queryParameter(key);
                Log.d("TEST", key + " " + value);
            }
            
            // 将最终的url填充到request中
            requestBuilder.url(httpUrl);
        } else if (METHOD_POST.equals(request.method())) { // POST方法
        	// FormBody和url不太一样，若需添加公共参数，需要新建一个构造器
            FormBody.Builder bodyBuilder = new FormBody.Builder();
            // 把已有的post参数添加到新的构造器
            if (request.body() instanceof FormBody) {
                FormBody formBody = (FormBody) request.body();
                for (int i = 0; i < formBody.size(); i++) {
                    bodyBuilder.addEncoded(formBody.encodedName(i), formBody.encodedValue(i));
                }
            }
            
            // 这里可以添加一些公共post参数
            bodyBuilder.addEncoded("key_xxx", "value_xxx");
            FormBody newBody = bodyBuilder.build();
            
            // 打印所有post参数
            for (int i = 0; i < newBody.size(); i++) {
            	Log.d("TEST", newBody.name(i) + " " + newBody.value(i));
            }

			 // 将最终的表单body填充到request中
            requestBuilder.post(newBody);
        }

        // 这里我们可以添加header
        requestBuilder.addHeader(HEADER_KEY_USER_AGENT, getUserAgent()); // 举例，调用自己业务的getUserAgent方法
        return chain.proceed(requestBuilder.build());
    }
}
```

为了方便大家参考，我再补充一下import的包，因为这里OkHttp和Retrofit的一些类（比如Response）可能有重名，一定要各自用各自的，千万不要导入错了：

```java
import okhttp3.FormBody;
import okhttp3.HttpUrl;
import okhttp3.Interceptor;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
```

我没有水贴，![在这里插入图片描述](https://img-blog.csdnimg.cn/20190705233527983.png)。