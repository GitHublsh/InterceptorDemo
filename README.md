# InterceptorDemo
OkHttp3 Interceptor Demo

#### 自定义拦截器示例From 官网

Interceptors are a powerful mechanism that can monitor, rewrite, and retry calls. Here's a simple interceptor that logs the outgoing request and the incoming response.

拦截器是一种强大的机制,可以监视、重写和重试调用.下面是一个简单例子,拦截发出的请求和传入的响应的日志.

	class LoggingInterceptor implements Interceptor {
	  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
	    Request request = chain.request();
	    
	    long t1 = System.nanoTime();
	    logger.info(String.format("Sending request %s on %s%n%s",
	        request.url(), chain.connection(), request.headers()));
	
	    Response response = chain.proceed(request);
	
	    long t2 = System.nanoTime();
	    logger.info(String.format("Received response for %s in %.1fms%n%s",
	        response.request().url(), (t2 - t1) / 1e6d, response.headers()));
	
	    return response;
	  }
	}
A call to chain.proceed(request) is a critical part of each interceptor’s implementation. This simple-looking method is where all the HTTP work happens, producing a response to satisfy the request.

调用 chain.proceed(request) 是每个拦截器的关键部分的实现.这个简单的方法存在所有HTTP工作发生的地方,生产满足请求的响应.

Interceptors can be chained. Suppose you have both a compressing interceptor and a checksumming interceptor: you'll need to decide whether data is compressed and then checksummed, or checksummed and then compressed. OkHttp uses lists to track interceptors, and interceptors are called in order.

拦截器可以多个链接.假设您有一个压缩拦截器和校验拦截器:你需要决定数据是先压缩然后校验,还是先校验后压缩.OkHttp使用列表追踪拦截器,拦截器按顺序被调用。


Application Interceptors 应用拦截器
Interceptors are registered as either application or network interceptors. We'll use the LoggingInterceptor defined above to show the difference.

拦截器可以被应用程序或网络注册,我们将使用上面定义的 LoggingInterceptor 显示两者之间的差异.

Register an application interceptor by calling addInterceptor() on:

注册一个应用拦截器通过 OkHttpClient.Builder调用 addInterceptor():

	OkHttpClient client = new OkHttpClient.Builder()
	    .addInterceptor(new LoggingInterceptor())
	    .build();
	
	Request request = new Request.Builder()
	    .url("http://www.publicobject.com/helloworld.txt")
	    .header("User-Agent", "OkHttp Example")
	    .build();
	
	Response response = client.newCall(request).execute();
	response.body().close();
The URL http://www.publicobject.com/helloworld.txt redirects to https://publicobject.com/helloworld.txt, and OkHttp follows this redirect automatically. Our application interceptor is called once and the response returned from  chain.proceed() has the redirected response:

URL http://www.publicobject.com/helloworld.txt 重定向到 https://publicobject.com/helloworld.txt, OkHttp 将会自动跟随这个重定向. 我们的应用拦截器被调用一次,响应通过 chain.proceed() 返回重定向的响应:

	INFO: Sending request http://www.publicobject.com/helloworld.txt on null
	User-Agent: OkHttp Example
	
	INFO: Received response for https://publicobject.com/helloworld.txt in 1179.7ms
	Server: nginx/1.4.6 (Ubuntu)
	Content-Type: text/plain
	Content-Length: 1759
	Connection: keep-alive
	We can see that we were redirected because response.request().url() is different from request.url(). The two log statements log two different URLs.

我们可以看到调用被重定向了,因为 response.request().url() 不同于 request.url(). 两个日志语句打印出两个不同的url.

Network Interceptors 网络拦截器
Registering a network interceptor is quite similar. Call addNetworkInterceptor() instead of addInterceptor():

注册一个网络拦截器和上面非常相似. 调用 addNetworkInterceptor() 来代替 addInterceptor():

	OkHttpClient client = new OkHttpClient.Builder()
	    .addNetworkInterceptor(new LoggingInterceptor())
	    .build();
	
	Request request = new Request.Builder()
	    .url("http://www.publicobject.com/helloworld.txt")
	    .header("User-Agent", "OkHttp Example")
	    .build();
	
	Response response = client.newCall(request).execute();
	response.body().close();
When we run this code, the interceptor runs twice. Once for the initial request to http://www.publicobject.com/helloworld.txt, and another for the redirect to https://publicobject.com/helloworld.txt.

当我们运行这段代码时,拦截器运行两次.第一次是初始化请求到 http://www.publicobject.com/helloworld.txt的时候调用,另一个用于重定向到 https://publicobject.com/helloworld.txt的时候.

	INFO: Sending request http://www.publicobject.com/helloworld.txt on Connection{www.publicobject.com:80, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=none protocol=http/1.1}
	User-Agent: OkHttp Example
	Host: www.publicobject.com
	Connection: Keep-Alive
	Accept-Encoding: gzip
	
	INFO: Received response for http://www.publicobject.com/helloworld.txt in 115.6ms
	Server: nginx/1.4.6 (Ubuntu)
	Content-Type: text/html
	Content-Length: 193
	Connection: keep-alive
	Location: https://publicobject.com/helloworld.txt
	
	INFO: Sending request https://publicobject.com/helloworld.txt on Connection{publicobject.com:443, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA protocol=http/1.1}
	User-Agent: OkHttp Example
	Host: publicobject.com
	Connection: Keep-Alive
	Accept-Encoding: gzip
	
	INFO: Received response for https://publicobject.com/helloworld.txt in 80.9ms
	Server: nginx/1.4.6 (Ubuntu)
	Content-Type: text/plain
	Content-Length: 1759
	Connection: keep-alive
	The network requests also contain more data, such as the Accept-Encoding: gzip header added by OkHttp to advertise support for response compression. The network interceptor's Chain has a non-null Connection that can be used to interrogate the IP address and TLS configuration that were used to connect to the webserver.

网络请求也包含更多的数据,如 Accept-Encoding: gzip 头信息, OkHttp添加该头信息来通知并支持响应的压缩。网络拦截器链有一个非空连接,可用于查询用于连接到网络服务器的IP地址和TLS配置。

Choosing between application and network interceptors 在应用和网络拦截器之间做选择
Each interceptor chain has relative merits.

每个拦截器链都有自己的优点.

Application interceptors
Don't need to worry about intermediate responses like redirects and retries.
Are always invoked once, even if the HTTP response is served from the cache.
Observe the application's original intent. Unconcerned with OkHttp-injected headers like If-None-Match.
Permitted to short-circuit and not call Chain.proceed().
Permitted to retry and make multiple calls to Chain.proceed().

应用拦截器
不需要担心中间过程的响应,如重定向和重试.
总是只调用一次,即使HTTP响应是从缓存中获取.
观察应用程序的初衷. 不关心OkHttp注入的头信息如: If-None-Match.
允许短路而不调用 Chain.proceed(),即中止调用.
允许重试,使 Chain.proceed()调用多次.

Network Interceptors
Able to operate on intermediate responses like redirects and retries.
Not invoked for cached responses that short-circuit the network.
Observe the data just as it will be transmitted over the network.
Access to the Connection that carries the request.

网络拦截器
能够操作中间过程的响应,如重定向和重试.
当网络短路而返回缓存响应时不被调用.
只观察在网络上传输的数据.
携带请求来访问连接.

Rewriting Requests 重写请求
Interceptors can add, remove, or replace request headers. They can also transform the body of those requests that have one. For example, you can use an application interceptor to add request body compression if you're connecting to a webserver known to support it.

拦截器可以添加、删除或替换请求头信息.他们还可以改变的请求携带的实体.例如, 如果你连接到一个支持压缩的网络服务器你可以使用一个应用拦截器来添加请求实体压缩.

/** This interceptor compresses the HTTP request body. Many webservers can't handle this! */
/** 这个拦截器压缩了请求实体. 很多网络服务器无法处理它 */
	
		final class GzipRequestInterceptor implements Interceptor {
		  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
		    Request originalRequest = chain.request();
		    if (originalRequest.body() == null || originalRequest.header("Content-Encoding") != null) {
		      return chain.proceed(originalRequest);
		    }
	
	    Request compressedRequest = originalRequest.newBuilder()
	        .header("Content-Encoding", "gzip")
	        .method(originalRequest.method(), gzip(originalRequest.body()))
	        .build();
	    return chain.proceed(compressedRequest);
	  }
	
	  private RequestBody gzip(final RequestBody body) {
	    return new RequestBody() {
	      @Override public MediaType contentType() {
	        return body.contentType();
	      }
	
	      @Override public long contentLength() {
	        return -1; // We don't know the compressed length in advance!
	      }
	
	      @Override public void writeTo(BufferedSink sink) throws IOException {
	        BufferedSink gzipSink = Okio.buffer(new GzipSink(sink));
	        body.writeTo(gzipSink);
	        gzipSink.close();
	      }
	    };
	  }
	}
Rewriting Responses 重写响应
Symmetrically, interceptors can rewrite response headers and transform the response body. This is generally more dangerous than rewriting request headers because it may violate the webserver's expectations!

与重写请求对称,拦截器可以重写响应头信息和改变响应实体.这通常比重写请求头信息更加危险,因为它可能违反网络服务器的期望!

If you're in a tricky situation and prepared to deal with the consequences, rewriting response headers is a powerful way to work around problems. For example, you can fix a server's misconfigured Cache-Control response header to enable better response caching:

如果你在一个棘手的情况下,准备处理结果,重写响应头信息是一种强大的解决问题的方式.例如,您可以修复一个服务器配置错误的 Cache-Control 响应头信息,来确保更好的响应缓存:

/** Dangerous interceptor that rewrites the server's cache-control header. */
/** 重写服务器 cache-control 头信息的拦截器是危险的. */
private static final Interceptor REWRITE_CACHE_CONTROL_INTERCEPTOR = new Interceptor() {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Response originalResponse = chain.proceed(chain.request());
    return originalResponse.newBuilder()
        .header("Cache-Control", "max-age=60")
        .build();
  }
};
Typically this approach works best when it complements a corresponding fix on the webserver!

通常这种方法最好实现在相应的网络服务器上!

Availability 可用性
OkHttp's interceptors require OkHttp 2.2 or better. Unfortunately, interceptors do not work with OkUrlFactory, or the libraries that build on it, including Retrofit ≤ 1.8 and Picasso ≤ 2.4.

OkHttp的拦截器需要OkHttp 2.2或以上.不幸的是,拦截器不能和 OkUrlFactory同时工作,或其他库的构建,包括 Retrofit ≤ 1.8和 Picasso ≤ 2.4.