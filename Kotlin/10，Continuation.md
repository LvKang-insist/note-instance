在了解 Continuation 之前，首先要知道 suspend 函数为什么只能被 suspend 函数或者协程调用？

```kotlin
fun main() {
    foo() //报错：Suspend function 'foo' should be called only from a coroutine or another suspend function
}

suspend fun foo() {
    bar(1)
}

suspend fun bar(a: Int): String {
    return "Hello"
}
```

 	挂起函数“ foo ”只能从协程或其他挂起函数中调用 ，为什么会这样呢？其实 suspend 函数中会自动增加一个参数，而非 suspend 函数没有这个参数，调用的时候自然会保存了。

​	通过反编译就可以很清楚的看到这一点，如下：

```kotlin
    //https://github.com/work-helper/asm-bytecode-intellij 使用这个插件即可反编译
	@Nullable
    public static final Object foo(@NotNull Continuation<? super Unit> $completion) {
        // 调用 bar 函数
        Object object = StudyOneKt.bar((int)1, $completion);
        if (object == IntrinsicsKt.getCOROUTINE_SUSPENDED()) {
            return object;
        }
        return Unit.INSTANCE;
    }

    @Nullable
    public static final Object bar(int a, @NotNull Continuation<? super String> $completion) {
        return "Hello";
    }
```

​	suspend 函数会在参数的末尾自动加一个 Continuation 参数。正是因为这个参数，所以普通的函数无法调用 suspend 函数。

​	**Continuation 的泛型参数又是谁决定的呢？**  其实就是返回值的类型，通过上面的对比也能看出来。仔细观察，反编译后的函数返回值变成了 Object，至于为啥是 Object ，接着往下看。

------

**为啥两个函数的返回值都是 Object 呢?**

​		仔细看上面两个函数，他们并没有被挂起，他直接返回的是结果，这是一种情况

​		还有一种情况就是这个函数被挂起了，那么就会返回一个 挂起标志：**COROUTINE_SUSPENDED** ，让协程体的外部流程知道这个函数被挂起了，需要等待你的回调才能继续往下执行。

​		那么什么是正真的挂起呢？

```kotlin
//将回调转写成挂起函数
suspend fun getData(url: String): String = suspendCoroutine { continuation ->
    val response = OkHttpClient().newCall(Request.Builder().url(url).get().build()).enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
            continuation.resumeWithException(e)
        }

        override fun onResponse(call: Call, response: Response) {
            continuation.resume(response.body()?.string()!!)
        }
    })
```

​		正真的挂起必须异步调用 resume ，包括：切换到其他线程 resume 、单线程事件循环异步执行。如果直接 return 或者 没有切线程 则不是真正的挂起。

​		如果要将回调转为挂起函数，就需要使用 suspendCoroutine 获取挂起函数的 Continuation。 虽然知道 Continuation 是 suspend 函数的最后一个参数，但是编译器把它藏了起来，我们拿不到，但是通过  suspendCoroutine 就可以拿到这个参数。 回调成功使用 resume 或者 resumeWith 。回调失败则使用 resumeWithException

------

​		知道了 suspend 是如何挂起的，那么怎样创建一个协程呢？

​		通过 suspend 函数我们可以知道，**创建协程的关键就是 suspend 函数的最后一位参数 Continuation** 。这个参数就是用来在 suspend 函数执行完后进行回调的。suspend 挂起完成后，通过这个参数来进行回调。如上面的挂起函数

​		创建协程通常需要一个 suspend 函数，也需要一个 **API** createCoroutine

​		看一下官方的 API 

```kotlin
public fun <R, T> (suspend R.() -> T).createCoroutine(
    receiver: R,
    completion:Continuation<T>
): Continuation<Unit> //......

public fun <T> (suspend () -> T).createCoroutine(
    completion: Continuation<T>
): Continuation<Unit> //.....
```

​		首先这两个函数都是扩展函数，一个是带 receiver（对象），一个是没有 receiver 的。他们都需要一个 Continuation 的参数，并且都返回了一个 Continuation 。

​		一个协程创建之后一定会有两个 Continuation，一个是我们传进去的，这个是用来回调的。还有一个则是返回值 Continuation ，这个 Continuation 就是协程的本体，里面的 resume 全部执行完了之后就会调用我们传入得 Continuation。

所以正确的创建方式就是

```
 suspend {

    }.createCoroutine(object : Continuation<Unit> {
        override val context: CoroutineContext
            get() = TODO("not implemented")

        override fun resumeWith(result: Result<Unit>) {

        }
    }).resume(Unit)
```

这里使用的是CreateCoroutine ，所以后面要使用 resume。也可以直接使用 startCoroutine。后面就不用 .resume 了。看一哈源码就知道是怎么回事了。

------

### 协程上下文

```kotlin
suspend {

    }.startCoroutine(object : Continuation<Unit> {
        override val context: CoroutineContext
            get() = TODO("not implemented")

        override fun resumeWith(result: Result<Unit>) {

        }
    })
```

上面的 CoroutineContext 则就是协程的上下文，其实就是数据集合

- 协程执行过程中需要携带数据
- 索引是 CoroutineContext.Key
- 元素是 CoroutineContext.Elment

### 上下文中的拦截器

拦截器是 ContinuationInterceptor 是协程上下文中的元素，它会存在上下文中，只不过他做的事比较高级

- 他可以篡改Continuation，可以对协程上下文所在协程的 Continuation 进行拦截。

   

  ```kotlin
  public interface ContinuationInterceptor : CoroutineContext.Element {
      public fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
  }
  ```

  可以看到他是 拦截器是继承自 上下文的 Elment。这个拦截方法接收一个 Continuation，并且返回一个 Continuation，而且他们的泛型都是一样的，所以才可以进行篡改。类似于 OkHttp 的拦截器。



### suspend fun main





### 案例1：自定义挂起恢复。加深挂起恢复

```kotlin
interface Generator<T> {
    operator fun iterator(): Iterator<T>
}

class GeneratorImpl<T>(private val block: suspend GeneratorScope<T>.(T) -> Unit, private val paramenter: T) :
    Generator<T> {
    override fun iterator(): Iterator<T> {
        return GeneratorIterator(block, paramenter)
    }
}

sealed class State {
    class NotRead(val continuation: Continuation<Unit>) : State()
    class Ready<T>(val continuation: Continuation<Unit>, val nextValue: T) : State()
    object Done : State()
}

abstract class GeneratorScope<T> internal constructor() {

    protected abstract val parameter: T
    abstract suspend fun yield(value: T)
}


class GeneratorIterator<T>(
    private val block: suspend GeneratorScope<T>.(T) -> Unit, override val parameter: T
) : GeneratorScope<T>(), Iterator<T>, Continuation<Any?> {

    override val context: CoroutineContext = EmptyCoroutineContext

    private var state: State

    init {
        val coroutineBlock: suspend GeneratorScope<T>.() -> Unit = {
            block(parameter)
        }
        val start = coroutineBlock.createCoroutine(this, this)
        state = State.NotRead(start)
    }


    private fun resume() {
        when (val currentState = state) {
            is State.NotRead -> currentState.continuation.resume(Unit)
        }
    }

    override fun hasNext(): Boolean {
        resume()
        return state != State.Done
    }

    override fun next(): T {
        return when (val currentState = state) {
            is State.NotRead -> {
                resume()
                return next()
            }
            is State.Ready<*> -> {
                state = State.NotRead(currentState.continuation)
                (currentState as State.Ready<T>).nextValue
            }
            State.Done -> throw IndexOutOfBoundsException("No value left")
        }
    }

    override fun resumeWith(result: Result<Any?>) {
        state = State.Done
        result.getOrThrow()
    }
	//挂起，注意这里没有进行恢复
    override suspend fun yield(value: T) = suspendCoroutine<Unit> { continuation ->
        state = when (state) {
            is State.NotRead -> State.Ready(continuation, value)
            is State.Ready<*> -> throw IllegalArgumentException("Cannot yield a value while ready")
            State.Done -> throw IllegalArgumentException("Cannot yield a value while done")
        }
    }
}


fun <T> generator(block: suspend GeneratorScope<T>.(T) -> Unit): (T) -> Generator<T> {
    return { parament: T ->
        GeneratorImpl(block, parament)
    }
}

fun main() {

    val nums = generator { start: Int ->
        for (i in 0..5) {
            yield(start + i)
        }
    }
    val seq = nums(10)

    for (i in seq) {
        println(seq)
    }
}
```

​	这段代码理解了好长时间，总结一哈，挂起函数的 continuation 如果没有进行 resume。那么这个挂起函数将会一直挂起。直到 resume 后才会继续往下执行。

### 案例2：非对称挂起恢复，加深挂起恢复的理解

```kotlin
sealed class Status {

    class Created(val continuation: Continuation<Unit>) : Status()
    class Yielded<P>(val continuation: Continuation<P>) : Status()
    class Resumed<R>(val continuation: Continuation<R>) : Status()
    object Dead : Status()

}

class Coroutine<P, R>(
    override val context: CoroutineContext = EmptyCoroutineContext,
    private val block: suspend Coroutine<P, R>.CoroutineBody.(P) -> R

) : Continuation<R> {

    //伴生对象，一个类中只能存在一个，类似于 java 的静态方法
    companion object {
        fun <P, R> create(
            context: CoroutineContext = EmptyCoroutineContext,
            block: suspend Coroutine<P, R>.CoroutineBody.(P) -> R
        ): Coroutine<P, R> {
            println(1)
            return Coroutine(context, block)
        }
    }


    inner class CoroutineBody {
        var parameter: P? = null

        suspend fun yield(value: R): P = suspendCoroutine { continuation ->
            val previousStatus = status.getAndUpdate {
                println(7)
                when (it) {
                    is Status.Created -> throw IllegalArgumentException("A了ready started")
                    is Status.Yielded<*> -> throw IllegalArgumentException("A了ready yielded")
                    is Status.Resumed<*> -> Status.Yielded(continuation)
                    Status.Dead -> throw IllegalArgumentException("A了ready dead")
                }
            }
            println(8)
            (previousStatus as? Status.Resumed<R>)?.continuation?.resume(value)
        }
    }

    private val body = CoroutineBody()

    private val status: AtomicReference<Status>

    val isActive: Boolean
        get() = status.get() != Status.Dead

    init {
        println(2)
        val coroutineBlock: suspend CoroutineBody.() -> R = {
            block(parameter!!)
        }
        val start = coroutineBlock.createCoroutine(body, this)
        status = AtomicReference(Status.Created(start))
    }

    override fun resumeWith(result: Result<R>) {
        println("哈哈哈哈哈")
        val previousStatus = status.getAndUpdate {
            when (it) {
                is Status.Created -> throw IllegalArgumentException("Never started")
                is Status.Yielded<*> -> throw IllegalArgumentException("A了ready yielded")
                is Status.Resumed<*> -> {
                    Status.Dead
                }
                Status.Dead -> throw IllegalArgumentException("Already dead")
            }
        }
        (previousStatus as Status.Resumed<R>).continuation.resumeWith(result)
    }

    suspend fun resume(value: P): R = suspendCoroutine { coninuation ->
        //getAndUpdate 传入的函数将会赋值给 status
        val preiousStatus = status.getAndUpdate {
            println(4)
            when (it) {
                is Status.Created -> {
                    body.parameter = value
                    Status.Resumed(coninuation)
                }
                is Status.Yielded<*> -> {
                    Status.Resumed(coninuation)
                }
                is Status.Resumed<*> -> throw IllegalArgumentException("A了ready yielded")
                Status.Dead -> throw IllegalArgumentException("A了ready yielded")
            }
        }
        println(5)
        when (preiousStatus) {
            is Status.Created -> {
                println("---------")
                preiousStatus.continuation.resume(Unit)
            }
            is Status.Yielded<*> -> {
                println("+++++++++++++++")
                (preiousStatus as Status.Yielded<P>).continuation.resume(value)
            }
        }
    }
}
//自定义拦截器
class Dispatcher : ContinuationInterceptor {
    override val key = ContinuationInterceptor

    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> {
        return DispcherContinuation(continuation)
    }

}

//对象 continuation 代替 DispcherContinuation 实现 Continuation 接口
class DispcherContinuation<T>(val continuation: Continuation<T>) : Continuation<T> by continuation {

    private val executor = Executors.newSingleThreadExecutor()

    override fun resumeWith(result: Result<T>) {
        executor.submit {
            continuation.resumeWith(result)
        }
    }

}

suspend fun main() {

    val prodducer = Coroutine.create<Unit, Int>(Dispatcher()) {
        println(6)
        for (i in 0..3) {
            println(Thread.currentThread().name + " send $i")
            yield(i)
        }
        200
    }

    val consumer = Coroutine.create<Int, Unit>(Dispatcher()) { param: Int ->
        for (i in 0..3) {
            val value = yield(Unit)
            println(Thread.currentThread().name + " receive $value")
        }
    }

    while (prodducer.isActive && consumer.isActive) {
        println(3)
        val result = prodducer.resume(Unit)
        println("/////////////////////////////////////////////")
        consumer.resume(result)
    }

}
```

理解了一哈，感觉就是不断地挂起和恢复。执行到 resume 后，将 resume 的 continuation 保存起来，将别的 suspend 恢复，然后又调用 yeild ，在yeild 中会挂起，并将 resume 函数恢复。while 循环里面继续往下执行。感觉就是不断地挂起和恢复。

