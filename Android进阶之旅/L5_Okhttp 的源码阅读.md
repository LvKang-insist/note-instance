首先先看一下用法：

```kotlin
val client = OkHttpClient.Builder().build()
val request = Request.Builder()
	.url("https://www.baidu.com")
	.build()
val call = client.newCall(request)
call.enqueue(object : okhttp3.Callback {
    override fun onFailure(call: okhttp3.Call, e: IOException) {

    }

    override fun onResponse(call: okhttp3.Call, response: okhttp3.Response) {
        Log.e("TAG", "onResponse: ${response.body?.string()}")
    }
})
```

------

### enqueue 方法

首先通过 builder 模式创建了一个 client。然后通过 newCall 方法创建了一个 Call ,最后通过这个 call 进行网络请求。

newCall 方法中传了一个参数，是 Request，这个 Request 是自己拼出来的。这个方法就是通过 request 创建一个 待用的网络请求。

```kotlin
override fun newCall(request: Request): Call {
  return RealCall.newRealCall(this, request, forWebSocket = false)
}

// RealCall 
companion object {
    fun newRealCall(
      client: OkHttpClient,  originalRequest: Request,   forWebSocket: Boolean
    ): RealCall {
        //创建一个 RealCall 
        //client ，requet，webSocket一般情况是用不到						
      return RealCall(client, originalRequest, forWebSocket).apply {
          //Transmitter
        transmitter = Transmitter(client, this)
      }
    }
  }
```

在 newRealCall 方法中返回了一个 RealCall 对象，由此可知，RealCall 是 Call 的实现类。

接着看一下 enqueue 方法：

```kotlin
call.enqueue(object : okhttp3.Callback {
    override fun onFailure(call: okhttp3.Call, e: IOException) {
    }
    override fun onResponse(call: okhttp3.Call, response: okhttp3.Response) {
        Log.e("TAG", "onResponse: ${response.body?.string()}")
    }
})

//RealCall 
  override fun enqueue(responseCallback: Callback) {
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    transmitter.callStart()
    client.dispatcher.enqueue(AsyncCall(responseCallback))
  }
```

在 enqueue 中 将 callback 转交给了 dispather，并创建了一个异步的Call  。下面分别看一下 dispatcher，enqueue 和 AsyncCall 分别是什么东西

- **dispatcher**

  ```kotlin
  val dispatcher: Dispatcher = builder.dispatcher
  
  //异步请求何时执行的策略。每个调度程序使用一个[ExecutorService]在内部运行调用。
  //如果您提供自己的执行器，它应该能够同时运行[配置的最大][maxRequests]调用数。
  class Dispatcher constructor() {
      
    // 最大请求数，超过后就会等一等
    @get:Synchronized var maxRequests = 64
    //...... 
      
    //主机的最大请求数，防止给服务器太大压力
    @get:Synchronized var maxRequestsPerHost = 5
      
    @get:Synchronized
    @get:JvmName("executorService") val executorService: ExecutorService
      get() {
        if (executorServiceOrNull == null) {
        executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
              SynchronousQueue(), threadFactory("OkHttp Dispatcher", false))
      }
        return executorServiceOrNull!!
      }
  }
  ```
  
  Dispatcher 主要是用来管理线程的，每一个新的请求的过程是需要一个单独的线程，这样不同的请求之间不会被挡着。
  
  他的内部实现用的是 ExecutorService ，
  
- **dispatcher 的 enqueue**

  ```kotlin
  internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      readyAsyncCalls.add(call)
  	//.....
    }
    promoteAndExecute()
  }
  ```

  调用 readyAsyncCalls ，这个是一个待命的队列，随时准备执行。

  ```kotlin
  private fun promoteAndExecute(): Boolean {
    assert(!Thread.holdsLock(this))
  
    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
      val i = readyAsyncCalls.iterator()
       // 遍历待命队列
      while (i.hasNext()) {
        val asyncCall = i.next()
  	  //判断 最大连接数 和 host 数量，不满足直接 break	
        if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
        if (asyncCall.callsPerHost().get() >= this.maxRequestsPerHost) continue // Host max capacity.
  	 // 从待命中移除	
        i.remove()
        asyncCall.callsPerHost().incrementAndGet()
         //将 asynCall 添加到 list 和 运行队列中
        executableCalls.add(asyncCall)
        runningAsyncCalls.add(asyncCall)
      }
      isRunning = runningCallsCount() > 0
    }
   
    for (i in 0 until executableCalls.size) {
      val asyncCall = executableCalls[i]
      // 调用 AsyncCall 的 executeOn 开始执行
      asyncCall.executeOn(executorService)
    }
    return isRunning
  }
  ```

  dispatcher 的 enqueue 主要就是进行了最大连接数和host的判断，然后将 asyncCall 添加到 一个 list 和 运行队列中，最后遍历 list 调用了 AsyncCall 的 executeOn 方法，

  下面看一下 AsyncCall

- **AsyncCall**

  ```kotlin
  internal inner class AsyncCall( private val responseCallback: Callback) : Runnable {
  	
      fun executeOn(executorService: ExecutorService) {
        assert(!Thread.holdsLock(client.dispatcher))
        var success = false
        try {
          //执行  
          executorService.execute(this)
          success = true
        } catch (e: RejectedExecutionException) {
          val ioException = InterruptedIOException("executor rejected")
          ioException.initCause(e)
          transmitter.noMoreExchanges(ioException)
          responseCallback.onFailure(this@RealCall, ioException)
        } finally {
          if (!success) {
            client.dispatcher.finished(this) // This call is no longer running!
          }
        }
      }
  
      override fun run() {
        threadName("OkHttp ${redactedUrl()}") {
          var signalledCallback = false
          transmitter.timeoutEnter()
          try {
            //  
            val response = getResponseWithInterceptorChain()
            signalledCallback = true
            //
            responseCallback.onResponse(this@RealCall, response)
          } catch (e: IOException) {
            if (signalledCallback) {
              // Do not signal the callback twice!
              Platform.get().log(INFO, "Callback failure for ${toLoggableString()}", e)
            } else {
              //
              responseCallback.onFailure(this@RealCall, e)
            }
          } finally {
            client.dispatcher.finished(this)
          }
        }
      }
    }
  }
  ```

  注意 executeOn 方法，这个方法就是 在 dispatcher 的 enqueue 中调用的。在这里执行了 executorService 后，对应的 run 方法也就会执行。

  run 方法中，通过 getResponseWithInterceptorChain() 进行请求，并拿到响应 response，最后调用 callback，这个 callback 就是我们在请求的时候传入的 callback

  **在 run 中有一些比较重要的方法，我们到后面说**。

  到现在使用 enqueue 进行请求的大致逻辑已经非常清楚了，其中最关键的代码就是：

  ```kotlin
  client.dispatcher.enqueue(AsyncCall(responseCallback))
  ```

  **大致流程如下：**

  1，创建一个 RealCall ，然后调用 enqueue

  2，在 enqueue 中调用 dispatcher 的 enqueue

  3，在  dispatcher 的 enqueue 方法中触发了 AsyncCall 的 run 方法，

  4，在 run 方法中进行请求并响应

------

### execute 方法

```kotlin
val response = call.execute()
```

```kotlin
override fun execute(): Response {
  synchronized(this) {
    check(!executed) { "Already Executed" }
    executed = true
  }
  transmitter.timeoutEnter()
  transmitter.callStart()
  try {
    client.dispatcher.executed(this)
     // 非常直接，直接调用了。
    return getResponseWithInterceptorChain()
  } finally {
    client.dispatcher.finished(this)
  }
}
```

execute 中不需要切线程，所以就直接调用了。

------

### OkhttpClient 中的配置项

- dispatcher

  用来调度线程的，对性能进行优化等，例如 maxRequest = 64 等

- proxy

  代理

- protocols

  所支持 http 协议的版本

  ```kotlin
  enum class Protocol(private val protocol: String) {
  
    HTTP_1_0("http/1.0"),
  
    HTTP_1_1("http/1.1"),
      
    HTTP_2("h2"),
    //......
   }
  ```

- connectionSpecs

  ConnectionSpecs 是用来指定 http 传输时的 socket 连接；对于 https 连接，ConnectionSpecs 是在构建 TLS 连接时向服务端说明客户端支持的 TLS 版本，密码套件的一类，

  在通过 https 连接时，需要向服务器附加上 客户端所支持的 TLS 版本，Cipher suite(可以接受的 对称，非对称和 hash 算法) 等数据。这些都在 ConnectionSpecs 这个类中。

   

  ```kotlin
  // 支持密码套件的列表
  private val RESTRICTED_CIPHER_SUITES = arrayOf(
      // TLSv1.3.
      CipherSuite.TLS_AES_128_GCM_SHA256,
      CipherSuite.TLS_AES_256_GCM_SHA384,
      CipherSuite.TLS_CHACHA20_POLY1305_SHA256,
  
      // TLSv1.0, TLSv1.1, TLSv1.2.
      CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
      CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
      CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
      CipherSuite.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
      CipherSuite.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256,
      CipherSuite.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256)
  
  //支持密码套件的列表
  private val APPROVED_CIPHER_SUITES = arrayOf(
      // TLSv1.3.
      CipherSuite.TLS_AES_128_GCM_SHA256,
      CipherSuite.TLS_AES_256_GCM_SHA384,
      CipherSuite.TLS_CHACHA20_POLY1305_SHA256,
  
      // TLSv1.0, TLSv1.1, TLSv1.2.
      CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
      CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
      CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
      CipherSuite.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
      CipherSuite.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256,
      CipherSuite.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,
  
      // Note that the following cipher suites are all on HTTP/2's bad cipher suites list. We'll
      // continue to include them until better suites are commonly available.
      CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,
      CipherSuite.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,
      CipherSuite.TLS_RSA_WITH_AES_128_GCM_SHA256,
      CipherSuite.TLS_RSA_WITH_AES_256_GCM_SHA384,
      CipherSuite.TLS_RSA_WITH_AES_128_CBC_SHA,
      CipherSuite.TLS_RSA_WITH_AES_256_CBC_SHA,
      CipherSuite.TLS_RSA_WITH_3DES_EDE_CBC_SHA)
      
       /** 安全的TLS连接需要最近的客户端和最近的服务器 */
      @JvmField
      val RESTRICTED_TLS = Builder(true)
          .cipherSuites(*RESTRICTED_CIPHER_SUITES)
          .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
          .supportsTlsExtensions(true)
          .build()
  
      /**
       *现代的TLS的配置，适用于大多数客户端和可以连接到的服务器，是OkHttp的默认配置
       */
      @JvmField
      val MODERN_TLS = Builder(true)
          .cipherSuites(*APPROVED_CIPHER_SUITES)
          .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
          .supportsTlsExtensions(true)
          .build()
  
      /**
       *向后兼容的配置，相比于MODERN_TLS，支持的TLS的版本变少了，只支持TLS_1_0版本
       */
      @JvmField
      val COMPATIBLE_TLS = Builder(true)
          .cipherSuites(*APPROVED_CIPHER_SUITES)
          .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2, TlsVersion.TLS_1_1, TlsVersion.TLS_1_0)
          .supportsTlsExtensions(true)
          .build()
  
  
      /** 未加密未认证的连接，就是HTTP连接*/
      @JvmField
      val CLEARTEXT = Builder(false).build()
  ```

  上面的 xxxx_TLS 配置的就是你要使用 http 还是使用 https 。

  如果你要使用 https ，那么你是用的 tls 版本是多少

- interceptors，networkInterceptors

  拦截器

- eventListenerFactory

  用来做统计的。

- cookieJar

  cookie 的存储器。okhttp 默认没有实现 cookie 的存储，需要自己去存储。

- cache

  缓存

- socketFactory

  用来创建 TCP 端口的 ，http 本身是没有端口的。

- certificateChainCleaner

  从服务器拿下来的证书，有时候会拿到很多个证书，certificateChainCleaner 是用来整理的，整理完后就是一个链或者说是一个序列，最后一个证书就是本地的根证书。

- hostnameVerifier

  给 https 做主机名验证的，用来验证对方的 host 是不是你需要访问的host

- certificatePinner

  证书固定器，用来验证自签名证书的！

- connectionPool

  连接池

- followRedirects

  遇到 301 的时候是否需要重定向

- followSslRedirects

  当你访问的是 http ，但是重定向到 https，或者是 访问的是 https ，但是重定向到 http 的时候 是否要重定向。

- retryOnConnectionFailure

  请求失败的时候是否需要重连

- connectTimeout

  tcp 连接的时间，超过这个事件则报错

- readTimeout

  下载响应的时候等待时间

- writeTimeout

  写入请求的等待时间

- pingInterval

  针对 webSocket 的，可以双向交互的通道，这就需要长连接了。pingInterval 表示多少时间 ping 一次。

