# Okhttp学习笔记（一）初窥Okhttp
___
Okhttp现在在Android网络开发中已经是毋庸置疑的必选网络框架，那么为什么用Okhttp呢？官网给
出了这样的解释：  

> HTTP is the way modern applications network. HTTP is the way modern applications
network. It’s how we exchange data & media. Doing HTTP efficiently makes your
stuff load faster and saves bandwidth.    
>  
> OkHttp is an HTTP client that’s efficient by default:  
>  
> * HTTP/2 support allows all requests to the same host to share a socket.  
> * Connection pooling reduces request latency (if HTTP/2 isn’t available).  
> * Transparent GZIP shrinks download sizes.  
> * Response caching avoids the network completely for repeat requests.  
>  
> OkHttp perseveres when the network is troublesome: it will silently recover from
common connection problems. If your service has multiple IP addresses OkHttp will
attempt alternate addresses if the first connect fails. This is necessary for IPv4+IPv6
and for services hosted in redundant data centers. OkHttp supports modern TLS features
(TLS 1.3, ALPN, certificate pinning).It can be configured to fall back for broad connectivity.  

也就是说Okhttp,大概有这几个优势
* HTTP/2 支持允许所有访问同一主机的请求共享一个socket  
* 利用连接池减少请求延迟（如果HTTP/2不可用)  
* 支持GZIP压缩
* 响应缓存减少重复请求

接下我们将分析Okhttp是如何做到这些的，不过在此之前。我们先看一下Okhttp的大概执行过程：  
```Java
  Request request = new Request.Builder().url(url).build();
  OkHttpClient okHttpClient = new OkHttpClient.Builder().build();
  Call realCall = okHttpClient.newCall(request);
  realCall.enqueue(new Callback() {
      @Override
      public void onFailure(Call call, IOException e) {

      }

      @Override
      public void onResponse(Call call, Response response) throws IOException {

      }
  });
```  
如图就是Okhttp的异步使用方法了。由上面的代码可以请求是在realCall里面执行的，这里面的realCall
是Call的一个子类RealCall。Call这个借口提供了enqueue()异步请求,excute()同步请求等方法
接下来我们看看enqueue是如何实现异步请求的？
```Java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.callStart();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```  
transmitter是连接网络层和应用层的桥梁，这个我们先放下，我们先看dispatcher，dispatcher
持有一个线程池,以及若干线程列表。而AsyncCall是NamedRunnable的子类，NamedRunnable又继，
承自Runnable，这里我们先看run方法    
```Java
  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  @Override protected void execute() {
    boolean signalledCallback = false;
    transmitter.timeoutEnter();
    try {
      Response response = getResponseWithInterceptorChain();
      signalledCallback = true;
      responseCallback.onResponse(RealCall.this, response);
    } catch (IOException e) {
      if (signalledCallback) {
        // Do not signal the callback twice!
        Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
      } else {
        responseCallback.onFailure(RealCall.this, e);
      }
    } finally {
      client.dispatcher().finished(this);
    }
  }

```
由此可以看出respanse对象是由getResponseWithInterceptorChain()这个方法得出的，接下来重
点来了
```Java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
      Response response = chain.proceed(originalRequest);
      if (transmitter.isCanceled()) {
        closeQuietly(response);
        throw new IOException("Canceled");
      }
      return response;
    } catch (IOException e) {
      calledNoMoreExchanges = true;
      throw transmitter.noMoreExchanges(e);
    } finally {
      if (!calledNoMoreExchanges) {
        transmitter.noMoreExchanges(null);
      }
    }
  }
```  
终于看到response了，由上面的代码可以看到，response是由chain返回来的，对于这个chain，OkHttp
是这么解释的
> A concrete interceptor chain that carries the entire interceptor chain: all application
A concrete interceptor chain that carries the entire interceptor chain: all application  
> If the chain is for an application interceptor then {@link #connection} must be null.
Otherwise it is for a network interceptor and {@link #connection} must be non-null.  

我们接着看代码，chain做了什么：
```Java
  public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
      throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.exchange != null && !this.exchange.connection().supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.exchange != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
        index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (exchange != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```  
从上面的代码可以看出，proceed取出了当前的拦截器，从当前拦截器以后生成了一个新链，当前的
拦截器调用新链产生response对象。自此，我们就看到了Okhttp最精巧的设计之一，责任链机制。
至于为什么要用这模式，我们慢慢分解
