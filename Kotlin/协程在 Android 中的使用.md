<<<<<<< HEAD
- 协程泄漏

   意思是你已经不需要协程了，但是他依然存在。

   例如：
=======
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
>>>>>>> e31b0f7cec67c84f459caf333b0b028024af7701
