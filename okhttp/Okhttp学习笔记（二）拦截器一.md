# Okhttp学习笔记（二）拦截器一
___
上回我们说到Okhttp通过责任链返回respose，接下来我们从拦截器开始，分析Okhttp都是这么返回
数据的。  

## RetryAndFollowUpInterceptor  

重定向追踪拦截器  
> This interceptor recovers from failures and follows redirects as necessary. It
may throw an {@link IOException} if the call was canceled.  


由此可知这个拦截器主要作用是从失败中恢复和重定向，接下来我们看他的intercept方法  

```Java
  @Override public Response intercept(Chain chain) throws IOException {
     Request request = chain.request();
     RealInterceptorChain realChain = (RealInterceptorChain) chain;
     Transmitter transmitter = realChain.transmitter();

     int followUpCount = 0;
     Response priorResponse = null;
     while (true) {
       transmitter.prepareToConnect(request);

       if (transmitter.isCanceled()) {
         throw new IOException("Canceled");
       }

       Response response;
       boolean success = false;
       try {
         response = realChain.proceed(request, transmitter, null);
         success = true;
       } catch (RouteException e) {
         // The attempt to connect via a route failed. The request will not have been sent.
         if (!recover(e.getLastConnectException(), transmitter, false, request)) {
           throw e.getFirstConnectException();
         }
         continue;
       } catch (IOException e) {
         // An attempt to communicate with a server failed. The request may have been sent.
         boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
         if (!recover(e, transmitter, requestSendStarted, request)) throw e;
         continue;
       } finally {
         // The network call threw an exception. Release any resources.
         if (!success) {
           transmitter.exchangeDoneDueToException();
         }
       }

       // Attach the prior response if it exists. Such responses never have a body.
       if (priorResponse != null) {
         response = response.newBuilder()
             .priorResponse(priorResponse.newBuilder()
                     .body(null)
                     .build())
             .build();
       }

       Exchange exchange = Internal.instance.exchange(response);
       Route route = exchange != null ? exchange.connection().route() : null;
       Request followUp = followUpRequest(response, route);

       if (followUp == null) {
         if (exchange != null && exchange.isDuplex()) {
           transmitter.timeoutEarlyExit();
         }
         return response;
       }

       RequestBody followUpBody = followUp.body();
       if (followUpBody != null && followUpBody.isOneShot()) {
         return response;
       }

       closeQuietly(response.body());
       if (transmitter.hasExchange()) {
         exchange.detachWithViolence();
       }

       if (++followUpCount > MAX_FOLLOW_UPS) {
         throw new ProtocolException("Too many follow-up requests: " + followUpCount);
       }

       request = followUp;
       priorResponse = response;
     }
   }
```
response的工作流程大致是这样的，启动死循环，调用下级责任链，如果有重定向的请求，则给request
重新赋值，开始下一轮循环，如果有没有则返回response，循环超过二十次则报错

## BridgeInterceptor

桥拦截器  
>  Bridges from application code to network code. First it builds a network request
from a user request. Then it proceeds to call the network. Finally it builds a user
response from the network response.  


应用到网络的桥，它由一个用户请求生成一个网络请求，然后继续请求网络，最后由网络响应生成一个
用户响应  
```Java
  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }
```  
从代码中分析可以得知，在这个拦截器中，这个类由原来的URL生成了一个request，根据不同情况还
会做支持长连接，Gzip等，然后又调用后面的链，返回一个原始的响应，如果是Gzip压缩后的再进行
解压缩。  
这时我们已经看到Okhttp是如何做到__支持Gzip了__，至于GzipSource这就是另一个故事了。我们
可以大概了解一个他是Square公司（Okhttp就是这家公司维护的）的另一个开源库，用来做IO操作

## CacheInterceptor  
缓存拦截器

> Serves requests from the cache and writes responses to the cache.  

从缓存中提供请求并把响应写到缓存中

```Java
@Override public Response intercept(Chain chain) throws IOException {
  Response cacheCandidate = cache != null
      ? cache.get(chain.request())
      : null;

  long now = System.currentTimeMillis();

  CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
  Request networkRequest = strategy.networkRequest;
  Response cacheResponse = strategy.cacheResponse;

  if (cache != null) {
    cache.trackResponse(strategy);
  }

  if (cacheCandidate != null && cacheResponse == null) {
    closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
  }

  // If we're forbidden from using the network and the cache is insufficient, fail.
  if (networkRequest == null && cacheResponse == null) {
    return new Response.Builder()
        .request(chain.request())
        .protocol(Protocol.HTTP_1_1)
        .code(504)
        .message("Unsatisfiable Request (only-if-cached)")
        .body(Util.EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
  }

  // If we don't need the network, we're done.
  if (networkRequest == null) {
    return cacheResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .build();
  }

  Response networkResponse = null;
  try {
    networkResponse = chain.proceed(networkRequest);
  } finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
      closeQuietly(cacheCandidate.body());
    }
  }

  // If we have a cache response too, then we're doing a conditional get.
  if (cacheResponse != null) {
    if (networkResponse.code() == HTTP_NOT_MODIFIED) {
      Response response = cacheResponse.newBuilder()
          .headers(combine(cacheResponse.headers(), networkResponse.headers()))
          .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
          .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
          .cacheResponse(stripBody(cacheResponse))
          .networkResponse(stripBody(networkResponse))
          .build();
      networkResponse.body().close();

      // Update the cache after combining headers but before stripping the
      // Content-Encoding header (as performed by initContentStream()).
      cache.trackConditionalCacheHit();
      cache.update(cacheResponse, response);
      return response;
    } else {
      closeQuietly(cacheResponse.body());
    }
  }

  Response response = networkResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .networkResponse(stripBody(networkResponse))
      .build();

  if (cache != null) {
    if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
      // Offer this request to the cache.
      CacheRequest cacheRequest = cache.put(response);
      return cacheWritingResponse(cacheRequest, response);
    }

    if (HttpMethod.invalidatesCache(networkRequest.method())) {
      try {
        cache.remove(networkRequest);
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
    }
  }

  return response;
}
```

这个拦截器，根据请求，取出缓存的response，根据当前时间戳，response，request得到缓存策略
生成新的response，和request。  
新的request == null表示强制缓存，也就是说该必须使用缓存了，如果这时候缓存为空，就该报错
了，否则，如果有缓存的response，则在拿到新response，合并新旧response，否则使用新的response
并更新或添加新的response
