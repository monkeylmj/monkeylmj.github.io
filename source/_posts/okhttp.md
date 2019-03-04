---
title: okhttp源码解析
date: 2019-03-03 22:18:27
tags:
 - 源码解析
 - 网络
---

简要总结一下OKHttp框架的整体流程，结合其中用到的设计模式进行说明。

<!-- more -->

## 1. 使用方式

```java
//1.创建Client
OkHttpClient client = new OkHttpClient();

//2.创建请求
Request request = new Request.Builder()
                .url("xxx")
                .build();
//3.执行(同步or异步)
okHttpClient.newCall(request).execute();
okHttpClient.newCall(request).enqueue(responseCallback);
```



## 2. 源码设计模式解析

### 2.1 OKHttpClient和Request对象的实例化

OKHttpClient和Request对象包含大量的属性，通过**Builder模式**按需配置，方便地创建对象。

@Request.Builder

```java
public static class Builder {
    HttpUrl url;
    String method;
    Headers.Builder headers;
    RequestBody body;
    Object tag;

    public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
    }

    Builder(Request request) {
      this.url = request.url;
      this.method = request.method;
      this.body = request.body;
      this.tag = request.tag;
      this.headers = request.headers.newBuilder();
    }

    public Builder url(HttpUrl url) {
      if (url == null) throw new NullPointerException("url == null");
      this.url = url;
      return this;
    }
    
    //省略...
}
```



@OKHttpClient 中包含大量的请求过程的重要属性，可通过Builder来配置

```java
class OKHttpClient {
  final Dispatcher dispatcher;
  final @Nullable Proxy proxy;
  final List<Protocol> protocols;
  final List<ConnectionSpec> connectionSpecs;
  final List<Interceptor> interceptors;
  final List<Interceptor> networkInterceptors;
  final EventListener.Factory eventListenerFactory;
  final ProxySelector proxySelector;
  final CookieJar cookieJar;
  final @Nullable Cache cache;
  final @Nullable InternalCache internalCache;
  final SocketFactory socketFactory;
  final @Nullable SSLSocketFactory sslSocketFactory;
  final @Nullable CertificateChainCleaner certificateChainCleaner;
  final HostnameVerifier hostnameVerifier;
  final CertificatePinner certificatePinner;
  final Authenticator proxyAuthenticator;
  final Authenticator authenticator;
  final ConnectionPool connectionPool;
  final Dns dns;
  final boolean followSslRedirects;
  final boolean followRedirects;
  final boolean retryOnConnectionFailure;
  final int connectTimeout;
  final int readTimeout;
  final int writeTimeout;
  final int pingInterval;
}
```



### 2.2 okHttpClient.newCall创建Call对象

```java
@Override
public Call newCall(Request request) {
   return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

这里看到OKHttpClient类实现了Call.Factory接口，方法中创建了Call的实现类RealCall。

@Call.Factory

```java
  interface Factory {
    Call newCall(Request request);
  }
```

这里用到了**工厂方法模式**，Factory接口提供newCall方法，不同的工厂实现类可以继承来生产Call类型的对象。目前只有OKHttpClient一个工厂实现类，满足开闭原则，以后可以直接添加其他工厂来生成不同的Call，方便扩展。



### 2.3 Call执行

上一步生成的RealCall开始执行，这里选择同步执行的方式来举例往下查看流程，异步流程类似（只是交给dispatcher放到线程池里执行）

@RealCall#execute

```java
@Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

逻辑非常简单，核心就一句话： `Response result = getResponseWithInterceptorChain()`，顾名思义这里就是通过一层层拦截器最终获取到Response，整个流程结束。



### 2.4 拦截器的实现

OKHttp网络请求流程的核心部分，使用了**责任链模式**来完成请求发送者和请求处理者的解耦（只需将请求发到链上即可，无须关系处理的细节）。

@RealCall#getResponseWithInterceptorChain

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

首先，构造了一个interceptors列表，其中加入了一些默认的拦截器，client.interceptors和client.networkInterceptors为自定义的拦截器（构造OKHttpClient时加入），所有拦截器都实现了Interceptor接口。

然后，构造了一个职责链 RealInterceptorChain持有了所有的拦截器，然后将请求发到职责链上。



@RealInterceptorChain#proceed

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,RealConnection connection) throws IOException {
     //...
     // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    //...
    return response;
  }
```

处理的逻辑很简单：取出index所在的拦截器，执行intercept方法进行拦截，传入的参数为新构造的职责链（index+1，指向了下一个拦截器）。

UML图如下：

![](https://i.loli.net/2019/02/23/5c70cba2935de.png)



最后，写一个简单的自定义拦截器，看下大概的流程：

```java
public class TestInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request(); //取出request
        
        //请求前拦截进行一些处理...
        
        Response response = chain.proceed(request); //传给下一个拦截器
        
        //响应后拦截进行一些处理...
        
        return response;
    }
}
```

到此，OKHttp的网络请求流程已经完整！



## 总结

![](https://i.loli.net/2019/02/23/5c70be03dc3e8.png)

OKHttp网络请求框架的实现很优雅，各个类的职责明确，整个流程跟把大象装在冰箱里一样简单：

1、组装Request

2、Client发送Request（经过各个拦截器的处理）

3、返回Response