# Okhttp学习笔记（三）拦截器二
___
前面讲了RetryAndFollowUpInterceptor，BridgeInterceptor，CacheInterceptor接下来我们讲
一讲ConnectInterceptor
## ConnectInterceptor
连接拦截器  
> Opens a connection to the target server and proceeds to the next interceptor.

连接到服务器并且驱动下一个拦截器。  
这里我们假定你已经有一定计算机网络知识，知道网络分层，以及TCP握手。很明显这是网络层建立
tcp连接用的
```Java
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    Transmitter transmitter = realChain.transmitter();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

    return realChain.proceed(request, transmitter, exchange);
  }
```
核心代码只有一句`Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks)`
接下我们看具体他做了那些事
```Java
/** Returns a new exchange to carry a new request and response. */
   Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
     synchronized (connectionPool) {
       if (noMoreExchanges) {
         throw new IllegalStateException("released");
       }
       if (exchange != null) {
         throw new IllegalStateException("cannot make a new request because the
          previous response "
             + "is still open: please call response.close()");
       }
     }

     ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);
     Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);

     synchronized (connectionPool) {
       this.exchange = result;
       this.exchangeRequestDone = false;
       this.exchangeResponseDone = false;
       return result;
     }
   }
```
注释里面已经写明分会是一个Exchange对象，而这个Exchange对象是在这里new 出来的，那么这个
Exchange是干什么的呢
> Transmits a single HTTP request and a response pair. This layers connection management
 and events on {@link ExchangeCodec}, which handles the actual I/O.  

 我们可以知道ExchageCodec是Exchange的关键类，我们再来看看ExchangeCodec
 >  Encodes HTTP requests and decodes HTTP responses.

具体做了什么还是看不懂，我们再来看看ExchangeCodec出来的
```Java
  public ExchangeCodec find(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      return resultConnection.newCodec(client, chain);
    } catch (RouteException e) {
      trackFailure();
      throw e;
    } catch (IOException e) {
      trackFailure();
      throw new RouteException(e);
    }
  }
```
我们在来看看resultConnection的newCodec方法
```Java
  ExchangeCodec newCodec(OkHttpClient client, Interceptor.Chain chain) throws SocketException {
    if (http2Connection != null) {
      return new Http2ExchangeCodec(client, this, chain, http2Connection);
    } else {
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
      return new Http1ExchangeCodec(client, this, source, sink);
    }
  }
```
根据http1还是http2生成不同的ExchangeCodec看Http2ExchangeCodec和Http1ExchangeCodec的
作用分别是什么:
* http2
> Encode requests and responses using HTTP/2 frames.  

* http1
> * A socket connection that can be used to send HTTP/1.1 messages. This class strictly
  enforces the
 * following lifecycle:
 *
 * <ol>
 *     <li>{@linkplain #writeRequest Send request headers}.
 *     <li>Open a sink to write the request body. Either {@linkplain #newKnownLengthSink known
 *         length} or {@link #newChunkedSink chunked}.
 *     <li>Write to and then close that sink.
 *     <li>{@linkplain #readResponseHeaders Read response headers}.
 *     <li>Open a source to read the response body. Either {@linkplain #newFixedLengthSource
 *         fixed-length}, {@linkplain #newChunkedSource chunked} or {@linkplain
 *         #newUnknownLengthSource unknown length}.
 *     <li>Read from and close that source.
 * </ol>
 *
 * <p>Exchanges that do not have a request body may skip creating and closing the request body.
 * Exchanges that do not have a response body can call {@link #newFixedLengthSource(long)
 * newFixedLengthSource(0)} and may skip reading and closing that source.

可以看到对http2的请求和响应进行编码，对http1则做得更多一些可能会压缩或解压缩请求或响应。
我们再来看看，看看如何找到RealConnection
```Java
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
        int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
        boolean doExtensiveHealthChecks) throws IOException {
      while (true) {
        RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
            pingIntervalMillis, connectionRetryEnabled);

        // If this is a brand new connection, we can skip the extensive health checks.
        synchronized (connectionPool) {
          if (candidate.successCount == 0) {
            return candidate;
          }
        }

        // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
        // isn't, take it out of the pool and start again.
        if (!candidate.isHealthy(doExtensiveHealthChecks)) {
          candidate.noNewExchanges();
          continue;
        }

        return candidate;
      }
    }
```
做一个遍历，找到后如果是新的RealConnection或是健康的，就返回（猜测新的是最后一个避免死循
环）接下来我们看看findConnection方法
```Java
  /**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    RealConnection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (transmitter.isCanceled()) throw new IOException("Canceled");
      hasStreamFailure = false; // This is a fresh attempt.

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new exchanges.
      releasedConnection = transmitter.connection;
      toClose = transmitter.connection != null && transmitter.connection.noNewExchanges
          ? transmitter.releaseConnectionNoEvents()
          : null;

      if (transmitter.connection != null) {
        // We had an already-allocated connection and it's good.
        result = transmitter.connection;
        releasedConnection = null;
      }

      if (result == null) {
        // Attempt to get a connection from the pool.
        if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, null, false)) {
          foundPooledConnection = true;
          result = transmitter.connection;
        } else if (nextRouteToTry != null) {
          selectedRoute = nextRouteToTry;
          nextRouteToTry = null;
        } else if (retryCurrentRoute()) {
          selectedRoute = transmitter.connection.route();
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    List<Route> routes = null;
    synchronized (connectionPool) {
      if (transmitter.isCanceled()) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        routes = routeSelection.getAll();
        if (connectionPool.transmitterAcquirePooledConnection(
            address, transmitter, routes, false)) {
          foundPooledConnection = true;
          result = transmitter.connection;
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        result = new RealConnection(connectionPool, selectedRoute);
        connectingConnection = result;
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    connectionPool.routeDatabase.connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      connectingConnection = null;
      // Last attempt at connection coalescing, which only occurs if we attempted multiple
      // concurrent connections to the same host.
      if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, routes, true)) {
        // We lost the race! Close the connection we created and return the pooled connection.
        result.noNewExchanges = true;
        socket = result.socket();
        result = transmitter.connection;
      } else {
        connectionPool.put(result);
        transmitter.acquireConnectionNoEvents(result);
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```
首先注释验证了我们的猜想。这段代码前半段是复用，后半段是在没有可复用的情况下创建新的连接,
这里需要额外注意一下`result.connect()`他上面的注释写明了TCP,和TLS校验发生在这里，而且是
一个阻塞操作。
```Java
  public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled, Call call,
      EventListener eventListener) {
    if (protocol != null) throw new IllegalStateException("already connected");

    RouteException routeException = null;
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);

    if (route.address().sslSocketFactory() == null) {
      if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication not enabled for client"));
      }
      String host = route.address().url().host();
      if (!Platform.get().isCleartextTrafficPermitted(host)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication to " + host + " not permitted by network security policy"));
      }
    } else {
      if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
        throw new RouteException(new UnknownServiceException(
            "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"));
      }
    }

    while (true) {
      try {
        if (route.requiresTunnel()) {
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break;
          }
        } else {
          connectSocket(connectTimeout, readTimeout, call, eventListener);
        }
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
        eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
        break;
      } catch (IOException e) {
        closeQuietly(socket);
        closeQuietly(rawSocket);
        socket = null;
        rawSocket = null;
        source = null;
        sink = null;
        handshake = null;
        protocol = null;
        http2Connection = null;

        eventListener.connectFailed(call, route.socketAddress(), route.proxy(), null, e);

        if (routeException == null) {
          routeException = new RouteException(e);
        } else {
          routeException.addConnectException(e);
        }

        if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
          throw routeException;
        }
      }
    }

    if (route.requiresTunnel() && rawSocket == null) {
      ProtocolException exception = new ProtocolException("Too many tunnel connections attempted: "
          + MAX_TUNNEL_ATTEMPTS);
      throw new RouteException(exception);
    }

    if (http2Connection != null) {
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();
      }
    }
  }
```
建立隧道或建立socket连接。  
这时候我们可以看看Okhttp的前两个有点
> * HTTP/2 支持允许所有访问同一主机的请求共享一个socket  
> * 利用连接池减少请求延迟（如果HTTP/2不可用)  

Okhttp在请求连接的时候会根据URL查找出其域名地址，这样如果上一次请求过这个域名地址的话则，
复用RealConnection，这里我们需要解释一下RealConnection有Socket和rawSocket两个对象，分
别是网络层和应用层的Socket对象，HTTP和HTTP/2分别会对RealConnection进行不同程度的复用

至于下一个CallServerInterceptor比较简单，作用就是与服务器进行通信，把请求的头什么的全写
进去
