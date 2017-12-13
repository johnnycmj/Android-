### 同步GET请求过程

```
//HTTP GET
    public String get(String url) throws IOException {
        //新建OKHttpClient客户端
        OkHttpClient client = new OkHttpClient();
        //新建一个Request对象
        Request request = new Request.Builder()
                .url(url)
                .build();
        //Response为OKHttp中的响应
        Response response = client.newCall(request).execute();
        if (response.isSuccessful()) {
            return response.body().string();
        }else{
            throw new IOException("Unexpected code " + response);
        }
    }

```

#### 第一步 OkHttpClient
 OkHttpClient client = new OkHttpClient();

 直接上代码

```
public OkHttpClient() {
    this(new Builder());
  }
```

```
public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
    }
```

```
Dispatcher dispatcher;  //分发器
    @Nullable Proxy proxy;  //代理
    List<Protocol> protocols;  //协议
    List<ConnectionSpec> connectionSpecs;  //传输层版本和连接协议
    final List<Interceptor> interceptors = new ArrayList<>();  //拦截器
    final List<Interceptor> networkInterceptors = new ArrayList<>();  //网络拦截器
    EventListener.Factory eventListenerFactory;
    ProxySelector proxySelector; //代理选择
    CookieJar cookieJar;   //cookie
    @Nullable Cache cache;  //缓存
    @Nullable InternalCache internalCache; //内部缓存
    SocketFactory socketFactory; //socket 工厂
    @Nullable SSLSocketFactory sslSocketFactory; //安全套接层socket 工厂，用于HTTPS
    @Nullable CertificateChainCleaner certificateChainCleaner; // 验证确认响应证书 适用 HTTPS 请求连接的主机名。
    HostnameVerifier hostnameVerifier; //  主机名字确认
    CertificatePinner certificatePinner; //  证书链
    Authenticator proxyAuthenticator;  //代理身份验证
    Authenticator authenticator;   // 本地身份验证
    ConnectionPool connectionPool;  //连接池,复用连接
    Dns dns;  //域名
    boolean followSslRedirects;  //安全套接层重定向
    boolean followRedirects;  //本地重定向
    boolean retryOnConnectionFailure;  //重试连接失败
    int connectTimeout;   //连接超时
    int readTimeout;  //read 超时
    int writeTimeout;  //write 超时
    int pingInterval;
```

new OkHttpClient() 主要实例化Builder ，做的就是初始化一些参数。

#### 第二步：创建 Request 对象
**Request request = new Request.Builder()
                .url(url)
                .build();**

 目前流行的Build模式，基本上开源框架都能看到。


```
Builder(Request request) {
      this.url = request.url;
      this.method = request.method;
      this.body = request.body;
      this.tag = request.tag;
      this.headers = request.headers.newBuilder();
    }
```
主要还是初始化请求需要的参数，如headers、url等。

#### 第三步： Response 对象
**Response response = client.newCall(request).execute();**

真正的执行 client.newCall().


```
@Override 
public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```
真正执行的是 RealCall.newRealCall()


```
  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }

  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```
1. client: 我们当前的OkHttpClient。
2. originalRequest ：上面构造的Request 对象。
3. forWebSocket; 是否切换协议，当response回复101时，会用到。

接下来看下execute()  这是一个同步方法，还有一个异步的执行方法enqueue()


```
@Override
  public Response execute() throws IOException {

    synchronized (this) {

      //  1
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    //2
    captureCallStackTrace();

    //3
    eventListener.callStart(this);
    try {
      // 4
      client.dispatcher().executed(this);

      //5.
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
        // 6
      client.dispatcher().finished(this);
    }
  }
```
1. 是否执行过，一个Call只能执行一次.
2. 创建一个跟踪堆栈
3. 开始处理，事件标记
4. 分发器，文档里说是异步请求的一个执行策略，但是这里是同步，这里只是用来标识一下.开始执行.
5. 重点返回Response 对象
6. 事件标记结束.

重点看getResponseWithInterceptorChain()


```
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    //所有的拦截器.
    List<Interceptor> interceptors = new ArrayList<>();
    // 1
    interceptors.addAll(client.interceptors());
    //2
    interceptors.add(retryAndFollowUpInterceptor);
    //3
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    //4
    interceptors.add(new CacheInterceptor(client.internalCache()));
    //5
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      //6
      interceptors.addAll(client.networkInterceptors());
    }

    //7
    interceptors.add(new CallServerInterceptor(forWebSocket));
    // 8
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```
1. client 自带的一些自定义拦截器.
2. 负责失败重试以及重定向的拦截器
3. 负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应的拦截器
4. 请求从缓存中读取和通过 responses写入缓存的拦截器
5. 打开一个到目标服务器的连接,并继续下一个拦截器
6. 添加client 的网络拦截器
7. 最后一个拦截器，调用服务
8. 创建RealInterceptorChain 并执行proceed

RealInterceptorChain中的proceed():

```
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    //1
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);

    //2
    Interceptor interceptor = interceptors.get(index);
    //3
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
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
1. 在chain 中调用下一个拦截器.
2. 获取下一个拦截器,在除了Clien之外的第一个拦截器就是RetryAndFollowUpInterceptor
3. 执行拦截器

#####  RetryAndFollowUpInterceptor  

```
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request(); // 获取Request
    RealInterceptorChain realChain = (RealInterceptorChain) chain;  //获取Chain
    Call call = realChain.call();  // 获取Call对象
    EventListener eventListener = realChain.eventListener();  // 获取事件集合

    streamAllocation = new StreamAllocation(client.connectionPool(), createAddress(request.url()),
        call, eventListener, callStackTrace);

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      //当连接被取消时释放连接
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {
        //1
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // Attach the prior response if it exists. Such responses never have a body.
      // 2
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

     //3
      Request followUp = followUpRequest(response);

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }
```
这个方法有点长，但是重点是在**response = realChain.proceed(request, streamAllocation, null, null);**
1. 执行 proceed,这里便是执行下一个拦截器,即BridgeInterceptor
2. 把之前的response附加上去，并把body置为空
3. 后续的状态码处理

下一个执行的是BridgeInterceptor。

##### BridgeInterceptor

```
@Override
  public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body(); // 获取请求体
    if (body != null) {
      MediaType contentType = body.contentType();
      //请求头：Content-Type
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      //请求头：长度
      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    //请求头:主域名
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

    //cookie
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    // okhttp 版本
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    //1
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
这个拦截器基本上是处理Header的信息。
1. 执行下一个拦截器即CacheInterceptor

##### CacheInterceptor

```
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
    //1
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
    // 2
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      // 3
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    // 4
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
这个拦截器主要是缓存相关的。
1. 如果禁止使用网络，并且缓存没有则失败.直接抛出504网关超时
2. 如果没有网络并且有缓存,直接从cache构造一个Response,并且请求结束.
3. 执行下一个拦截器，ConnectInterceptor。
4. 如果我们的缓存有一个response，那么将根据条件来判断获取： 如果是304直接从缓存构造

##### ConnectInterceptor

```
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    //继续下一个拦截器
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

很简单的一个拦截器打开一个到目标服务器的连接,并继续下一个拦截器

##### CallServerInterceptor

```
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    realChain.eventListener().requestHeadersStart(realChain.call());
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    realChain.eventListener()
        .responseHeadersEnd(realChain.call(), response);

    int code = response.code();
    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```

okhttp的最后一个拦截器。通过网络调用服务。

至此 okhttp的同步调用  execute()流程走完。
#### 总结

前面说了拦截器用了责任链设计模式,它将请求一层一层向下传，知道有一层能够得到Resposne就停止向下传递，然后将response向上面的拦截器传递，然后各个拦截器会对respone进行一些处理，最后会传到RealCall类中通过execute来得到esponse。

### 异步GET请求过程

```
private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url(url)
        .build();

    client.newCall(request).enqueue(new Callback() {
      @Override 
      public void onFailure(Call call, IOException e) {
        e.printStackTrace();
      }

      @Override 
      public void onResponse(Call call, Response response) throws IOException {
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

        Headers responseHeaders = response.headers();
        for (int i = 0, size = responseHeaders.size(); i < size; i++) {
          System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
        }

        System.out.println(response.body().string());
      }
    });
  }

```

```
@Override 
public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    //1
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
这里通过dispatcher的enqueue 调用异步。


```
synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```

解释： 当当前运行中的异步请求数量小于最大数，并且占用的Host数量小于最大数，则将这个Call加入runningAsyncCalls，并在线程池中运行。否则加入readyAsyncCalls中。

runningAsyncCalls和readyAsyncCalls ?

```
/** Ready async calls in the order they'll be run. */
  //正在准备中的异步请求队列
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  //运行中的异步请求
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  //同步请求
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

AsyncCall 是在RealCall中的一个内部类》

```
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```
 通过NamedRunnable 中的run执行 execute()

 这里的Response还是执行getResponseWithInterceptorChain()和同步里的一样。
 最后通过 responseCallback  回调回去。
