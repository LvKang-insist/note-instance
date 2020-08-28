

### 为什么要使用 Kotlin 协程

- 轻量，高效
- 简单，好用

协程的特性

- 结构化并发：

  ```kotlin
  val scope = CoroutineScope(Dispatchers.Main).launch {
      launch {
          launch {
  
          }
      }
  
      launch {
  
      }
  }
  scope.cacel()
  ```

  在协程中可以进行嵌套的，并且具有子协程和父协程的概念。

  如上，启动了一个协程后，在内部有启动了多个子协程，在子协程中还可以继续启动协程。反过来看线程的话，可以再线程中创建线程，但是他们之间是没有任何关联的。

  协程的这种特性又被称为结构化并发，这种方式可以让协程非常方便的管理，比方说关闭协程，只需要关闭最外面的协程后，内部的协程都会被关闭。对于子协程也可以获取他的返回值并调用 cacel 进行关闭。

- 协程的取消

  调用 cacel 方法一定可以取消协程吗？

  答案是 No

  **协程的取消是 协作式的。**

  例如 会在执行 withContext 的时候判断协程是否取消，如果取消 withContext 里面的代码和外面的代码都不会执行。

  但是如果已经执行到 withContext 里面了，在取消协程。这个时候 withContext 里面的代码一定会执行完，执行完成后会判断有没有取消，如果取消则不会往下执行，否则继续执行。

  因为如此，所以协程的取消是协作式的。

- 协程中的异常

  try catch 捕获异常

  CoroutineExceptionHandler 处理协程异常。这种异常处理只能放在顶层协程中。

### Retrofit 对协程的支持

- Retrofit 是啥

  一个用于网络请求的库

- 什么叫对协程的支持？

  将 api 中定义的接口方法改为 suspend 方法即可，并且返回值不能使用 Call，因为是没有回调的。

  ```java
  /**
   * 普通请求
   */
  @GET("https://wanandroid.com/wxarticle/chapters/json")
  suspend fun get(): ResponseData<Bean>
  ```

### JetPack 对协程的支持

​	如 Lifecyle ，ViewModel，LiveData，Room 对协程的支持

- 协程泄漏

  意思是你已经不需要协程了，但是他依然存在。

- 内存泄露

  不被用到的内存无法被回收，本质上是对象泄露。如 Actvity  中有 AsyncTask，Activity 被关闭后，会有内存泄露吗？ 这个要看情况 ，**如果异步任务没有结束**，就会导致内存泄露，因为在 GC 中，线程是属于一个重量级的东西，在线程执行的过程中，GC 是绝对不会回收的，所以这种异常泄露实际上是线程的泄露。**如果异步任务结束**，就不会导致内存泄露，因为线程都执行完了，这些都会被 GC 回收掉。

  解决方案：用静态类 + 弱引用

- 协程泄露

  意思就是你已经不需要这个协程了，但是他还在工作，造成了泄露。实际上就是在关闭界面的时候协程内部的线程还在运转，就会导致内存泄露。所以他的本质是线程泄露。

  所以在界面关闭的时候需要取消协程

  在 onDestroy 中取消协程。当然不要使用 GlobalScope 创建协程，这个GlobalScope是全局的。

- CoroutineScope

  coroutineScope 内部的所有协程默认都会在 CoroutineScope 所指定的线程环境里执行，coroutineScope 被关闭后内部的所有协程都会被关闭

- Lifecyle ，ViewModel，LiveData，Room 对协程的支持

  - lifecycleScope

    ```kotlin
    //在 start 的时候执行
    lifecycleScope.launchWhenStarted {}
    //在 resume 的时候执行
    lifecycleScope.launchWhenResumed {}
    ```

  - viewModelScope

    ```
    viewModelScope.launch {
        tryCache(error, finally, block)
    }
    ```

  其他的支持都差不多

  并且在页面销毁的时候协程会自动取消

  

