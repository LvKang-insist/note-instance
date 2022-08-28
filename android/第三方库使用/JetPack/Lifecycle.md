### 前言

一个可以感知 Activity / Fragment 生命周期的组件，在生命周期拥有者和观察者之间快速方便的建立一种联系，当生命周期的拥有者的生命周期发生变化时，观察者就会收到对应的通知，另外还可以判断生命周期所处的生命周期状态

Lifecycle 是多个 JetPack 组件的基础，例如我们非常熟悉的 LiveData 就是以 Lifecycle 为基础实现的生命周期感知型容器。



### 了解 Lifecycle

首先我们得知道他的作用是啥，使用它的好处是啥。Lifecycler 主要就是为了简化生命周期的感知复杂度。在没有 Lifecycle 之前，我们都是通过接口回掉的方式或者直接调用的方式去感知生命周期，这样既不美观而且复杂度也比较高。

Lifecycle 整体上采用的是观察者模式，核心的 API 是 `LifecycleObserver` 和 `LifecycleOwner`：

- LifecycleObserver：观察者 
- LifecycleOwner：被观察者
- Lifecycle：定义了生命周期的标准行为模式，是 Lifecycle 框架的核心类，框架也提供了一个默认实现 LifecycleRegistry

再使用的时候生命周期的宿主需要实现 LifecycleOwner，并且将生命周期状态分发给 Lifecycle，从而间接的分发给观察者。

```kotlin
//被观察者
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```

```kotlin
//观察者
public interface LifecycleObserver {}
```

默认的 ComponentActivity 和 Fragment 已经实现了 `LifecycleOwner` 接口，并且返回了一个默认的 Lifecycle 实现类 `LifecycleRegistry` 。

#### Lifecycle 的使用

```kotlin
// 过时方式（lifecycle-extensions 不再维护）
implementation "androidx.lifecycle:lifecycle-extensions:2.4.0"

// 目前的方式：
def lifecycle_version = "2.5.0"

// Lifecycle 核心类
implementation "androidx.lifecycle:lifecycle-runtime:$lifecycle_version"
// Lifecycle 注解处理器（用于处理 @OnLifecycleEvent 注解）
kapt "androidx.lifecycle:lifecycle-compiler:$lifecycle_version"
implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"
// 应用进程级别 Lifecycle
implementation "androidx.lifecycle:lifecycle-process:$lifecycle_version"
```

- 方式一：注解

    ```kotlin
    //创建观察者
    public class MyObserver implements LifecycleObserver {
        private static final String TAG = "MyObserver";
      
        //该注解表示方法需要监听指定的生命周期时间
        @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
        public void connectListener(){
            Log.e(TAG, "connectListener:  onResume" );
        }
    
        @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
        public void disconnectListener(){
            Log.e(TAG, "disconnectListener: onPause");
        }
    }
    ```

    ```kotlin
    public class MainActivity extends AppCompatActivity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            //将 Lifecycle 对象和LifecycleObserver 对象进行绑定
            getLifecycle().addObserver(new MyObserver());
        }
    }  
    ```

- 方式二：LifecycleEventObserver,非注解方式，推荐使用

    ```kotlin
    lifecycle.addObserver(object :LifecycleEventObserver{
        override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
            when(event){
                Lifecycle.Event.ON_CREATE -> TODO()
                Lifecycle.Event.ON_START -> TODO()
                Lifecycle.Event.ON_RESUME -> TODO()
                Lifecycle.Event.ON_PAUSE -> TODO()
                Lifecycle.Event.ON_STOP -> TODO()
                Lifecycle.Event.ON_DESTROY -> TODO()
                Lifecycle.Event.ON_ANY -> TODO()
            }
        }
    })
    ```

- 方式三：DefaultLifecycleObserver，非注解方式，推荐使用

    ```kotlin
    // DefaultLifecycleObserver 是 FullLifecycleObserver 接口的空实
    lifecycle.addObserver(object : DefaultLifecycleObserver {
        
        override fun onCreate(owner: LifecycleOwner) {}
    
        override fun onStart(owner: LifecycleOwner) {}
    
        override fun onResume(owner: LifecycleOwner) {}
    
        override fun onPause(owner: LifecycleOwner) {}
    
        override fun onStop(owner: LifecycleOwner) {}
    
        override fun onDestroy(owner: LifecycleOwner) {}
    })
    ```

> 注意：Lifecycle 内部会禁止一个观察者注册到多个宿主上，如果绑定了多个宿主，那么就不知道以哪个生命周期为主了。

#### Lifecycle 的宿主

Android 默认提供的宿主有三个：

- Activity，具体的实现在 `ComponentActivity` 中

- Fragment

- ProcessLifeccycleOwner

    前两个宿主的生命周期都已经很熟悉了，第三个宿主则提供整个应用进程级别 Activity 的生命周周期

    ```kotlin
    ProcessLifecycleOwner.get().lifecycle.addObserver(object :LifecycleEventObserver{
        override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
            when(event){
                //应用进程启动时分发，只有一次
                Lifecycle.Event.ON_CREATE -> TODO()
                //应用进入前台(START)时分发，可能分发多次
                Lifecycle.Event.ON_START -> TODO()
                //应用进入前台(RESUME)时分发，可能分发多次
                Lifecycle.Event.ON_RESUME -> TODO()
                //应用退出前台(PAUSE)时分发，可能分发多次
                Lifecycle.Event.ON_PAUSE -> TODO()
                //应用退出前台(STOP)时分发，可能分发多次
                Lifecycle.Event.ON_STOP -> TODO()
                //不会被分发
                Lifecycle.Event.ON_DESTROY -> TODO()
            }
        }
    
    })
    ```

自定义宿主：

其实自定义宿主也就是自定义一个 `LifeccyleOwner` 的实现类，然后通过 `LifecycleRegistry` 将生命周期手动的分发给观察者即可。

```kotlin
class MyActivity : Activity(), LifecycleOwner {

    private lateinit var lifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleRegistry = LifecycleRegistry(this)
        lifecycleRegistry.markState(Lifecycle.State.CREATED)
    }

    public override fun onStart() {
        super.onStart()
        lifecycleRegistry.markState(Lifecycle.State.STARTED)
    }

    override fun getLifecycle(): Lifecycle {
        return lifecycleRegistry
    }
}
```



### Lifecycle 实现原理分析

#### 注册观察者

`lifecycle.addObserver()` 最终会分发到调度器 `LifecycleRegistiy` 中，在内部会将观察者和观察持有者的状态包装为一个结点，并在注册时将观察者状态同步推进到与宿主相同的状态中。

`LifecycleRegistry.java`

```kotlin
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    //观察者的初始状态，要么是 DESTROYED 或者是 INITIALIZED
    //以确保观察者可以看到完整的事件流
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    //对观察者进行包装
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    // 以 observer 为键，保存到 map 中，如果之保存过，会直接返回之前的，没有保存返回 null
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

   //如果之前保存过，直接退出
    if (previous != null) {
         return;
    }

   //将观察者推进到最新状态
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        //.....
        statefulObserver.dispatchEvent(lifecycleOwner, event);
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }
}
@Override
public void removeObserver(@NonNull LifecycleObserver observer) {
   mObserverMap.remove(observer);
}

static class ObserverWithState {
  State mState;
  LifecycleEventObserver mLifecycleObserver;

  ObserverWithState(LifecycleObserver observer, State initialState) {
    //用适配器包装观察者，实现对不同观察者的统一分发
    mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
    mState = initialState;
  }
  
  //最终 LifecycleRegistry.java 里面会调用到这里完成通知观察者
  void dispatchEvent(LifecycleOwner owner, Event event) {
    // 通过事件获得下一个状态
    State newState = getStateAfter(event);
    mState = min(mState, newState);
    // 回调 onStateChanged() 方法
    mLifecycleObserver.onStateChanged(owner, event);
    mState = newState;
  }
}
```

`Lifecycling.java`

为了适配不同的观察者，例如注解形式，或者是继承形式的观察者，`LifecycleRegistry` 为观察者提供了对应的适配器对象，

```kotlin
static LifecycleEventObserver lifecycleEventObserver(Object object) {
    boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
    boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
    //1，同时实现了 LifecycleEventObserver 和 FullLifecycleObserver
    if (isLifecycleEventObserver && isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                (LifecycleEventObserver) object);
    }
    //2，只实现了FullLifecycleObserver 
    if (isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
    }
		//3，只实现了LifecycleEventObserver
    if (isLifecycleEventObserver) {
        return (LifecycleEventObserver) object;
    }
		//4，使用注解的形式
    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass);
    if (type == GENERATED_CALLBACK) {
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                    constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
   //5,反射调用
    return new ReflectiveGenericLifecycleObserver(object);
}
```

`FullLifecycleObserverAdapte.java`

```java
class FullLifecycleObserverAdapter implements LifecycleEventObserver {

    private final FullLifecycleObserver mFullLifecycleObserver;
    private final LifecycleEventObserver mLifecycleEventObserver;

    FullLifecycleObserverAdapter(FullLifecycleObserver fullLifecycleObserver,
            LifecycleEventObserver lifecycleEventObserver) {
        mFullLifecycleObserver = fullLifecycleObserver;
        mLifecycleEventObserver = lifecycleEventObserver;
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
        switch (event) {
            case ON_CREATE:
                mFullLifecycleObserver.onCreate(source);
            //....
        }
        if (mLifecycleEventObserver != null) {
            mLifecycleEventObserver.onStateChanged(source, event);
        }
    }
}
```

#### Lifecycle 如何感知生命周期

```java
public class ComponentActivity extends Activity implements LifecycleOwner {

    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
    }
        @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}          
```

```java
public class ReportFragment extends android.app.Fragment {


    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            //在高版本直接观察 Activity 生命周期，
            LifecycleCallbacks.registerIn(activity);
        }
        //再低版本使用无界面的 Fragment 来间接的观察 Activity 的生命周期
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }

    //LifecycleCallbacks.registerIn(activity) 或者 fragment 回调过来
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
      //如果继承 LifecycleRegistryOwner 接口，该接口已经过时了。
        if (activity instanceof LifecycleRegistryOwner) {
            //.....
        }
        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
         			 //分发生命周期事件     
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
}
```

通过上面代码可以看到，如果是高版本，就直接观察 Activity 生命周期的 API ，否则就使用 Fragment 间接的观察 Activity 的生命周期。

最终将生命周期的事件分发到 `dispatch` 方法中，然后通过 activity 获取到 Lifeccyle 的实现类进行分发。

#### Lifecycle 分发生命周期的过程

通过上面代码我们可以知道，当宿主的生命周期发生变化时，就会分发到 `lifecycle.handleLifecycleEvent(Event) 中`，然后再分发给观察者。

整个生命周期的流程如下所示：

![生命周期状态示意图](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202207271450637.svg)

`LifecycleRegistry.java`

```java

private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();

private LifecycleRegistry(@NonNull LifecycleOwner provider, boolean enforceMainThread) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
        mEnforceMainThread = enforceMainThread;
}

//调用到这里
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    moveToState(event.getTargetState());
}

private void moveToState(State next) {
	 if (mState == next) {//如果和现在事件相同
			return;
		}
    //更新事件
		mState = next;
    //如果事件正在更新中
		if (mHandlingEvent || mAddingObserverCounter != 0) {
		mNewEventOccurred = true;
		// we will figure out what to do on upper level.
			return;
		}
		mHandlingEvent = true;
		sync();
		mHandlingEvent = false;
}
private void sync() {
  LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
  //...
  //判断所有观察者是否同步到最新状态
  while (!isSynced()) {
    mNewEventOccurred = false;
    // no need to check eldest for nullability, because isSynced does it for us.
    if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
      //生命周期回退，最终调用 ObserverWithState#dispatchEvent() 分发事件
      backwardPass(lifecycleOwner);
    }
    Map.Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
    if (!mNewEventOccurred && newest != null
        && mState.compareTo(newest.getValue().mState) > 0) {
      //生命周期前进，最终调用 ObserverWithState#dispatchEvent() 分发事件
      forwardPass(lifecycleOwner);
    }
  }
  mNewEventOccurred = false;
}
```

到此，整个流程就完全跑通了

### Lifecycle 的实践

#### 判断当前所处的生命周期

```kotlin
if(lifecycle.currentState.isAtLeast(Lifecycle.State.RESUMED)){
    //..
}
```

`isAtLeast` 比较状态是否大于或者等于指定的状态

#### 配合协程使用

Lifecycle 也提供了对于 Kotlin 协程的支持 `LifecycleCoroutineScope` ，该协程作用域与生命周期相关联，主要有两个特性：

1. 页面关闭时自动取消协程
2. 再宿主离开指定生命周期时挂起，再宿主重新进入生命周期时恢复状态

```kotlin
lifecycleScope.launch { 
    //生命周期处于 resume 的时候执行块
    whenResumed {}
}
lifecycleScope.launchWhenStarted {}
lifecycleScope.launchWhenCreated {}
lifecycleScope.launchWhenResumed {}
```

##### 自动取消协程分析

```kotlin
// 被观察者LifecycleOwner 的扩展属性
public val LifecycleOwner.lifecycleScope: LifecycleCoroutineScope
    get() = lifecycle.coroutineScope
```

`lifecycle.kt`

```kotlin
public val Lifecycle.coroutineScope: LifecycleCoroutineScope
    get() {
        while (true) {
            val existing = mInternalScopeRef.get() as LifecycleCoroutineScopeImpl?
            if (existing != null) {
                return existing
            }
           //SupervisorJob:主从作用域，同级别的协程异常不会影响当前协程
           // Dispatchers.Main 主线程
            val newScope = LifecycleCoroutineScopeImpl(
                this,
                SupervisorJob() + Dispatchers.Main.immediate
            )
            if (mInternalScopeRef.compareAndSet(null, newScope)) {
                //绑定生命周期
                newScope.register()
                return newScope
            }
        }
    }
```

```kotlin
public abstract class LifecycleCoroutineScope internal constructor() : CoroutineScope {
    internal abstract val lifecycle: Lifecycle
    public fun launchWhenCreated(block: suspend CoroutineScope.() -> Unit): Job = launch {
        lifecycle.whenCreated(block)
    }
    public fun launchWhenStarted(block: suspend CoroutineScope.() -> Unit): Job = launch {
        lifecycle.whenStarted(block)
    }

    public fun launchWhenResumed(block: suspend CoroutineScope.() -> Unit): Job = launch {
        lifecycle.whenResumed(block)
    }
}
//实现类
internal class LifecycleCoroutineScopeImpl(
    override val lifecycle: Lifecycle,
    override val coroutineContext: CoroutineContext
) : LifecycleCoroutineScope(), LifecycleEventObserver {
    init {
        // 如果已经销毁，立即取消协程
        if (lifecycle.currentState == Lifecycle.State.DESTROYED) {
            coroutineContext.cancel()
        }
    }

    fun register() {
        //绑定生命周期
        launch(Dispatchers.Main.immediate) {
            if (lifecycle.currentState >= Lifecycle.State.INITIALIZED) {
                lifecycle.addObserver(this@LifecycleCoroutineScopeImpl)
            } else {
                coroutineContext.cancel()
            }
        }
    }

    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        if (lifecycle.currentState <= Lifecycle.State.DESTROYED) {
           //取消携程
            lifecycle.removeObserver(this)
            coroutineContext.cancel()
        }
    }
}
```

可以看到，再获取 `lifecycleScope` 时会进行绑定，添加观察者。

当观察到宿主销毁之后，则取消订阅，关闭协程。

##### 关联指定的生命周期

```kotlin
public fun launchWhenResumed(block: suspend CoroutineScope.() -> Unit): Job = launch {
    lifecycle.whenResumed(block)
}
```

`pausingDispatcher.kt`

```kotlin
public suspend fun <T> Lifecycle.whenResumed(block: suspend CoroutineScope.() -> T): T {
    return whenStateAtLeast(Lifecycle.State.RESUMED, block)
}

public suspend fun <T> Lifecycle.whenStateAtLeast(
    minState: Lifecycle.State,
    block: suspend CoroutineScope.() -> T
): T = withContext(Dispatchers.Main.immediate) {
    val job = coroutineContext[Job] ?: error("when[State] methods should have a parent job")
   //分发器，内部有一个队列，用于支持暂停协程
    val dispatcher = PausingDispatcher()
    val controller =
        LifecycleController(this@whenStateAtLeast, minState, dispatcher.dispatchQueue, job)
    try {
        withContext(dispatcher, block)
    } finally {
        controller.finish()
    }
}
```

`LifecycleController.kt`

```kotlin
@MainThread
internal class LifecycleController(
    private val lifecycle: Lifecycle,
    private val minState: Lifecycle.State,
    private val dispatchQueue: DispatchQueue,
    parentJob: Job
) {
    private val observer = LifecycleEventObserver { source, _ ->
        if (source.lifecycle.currentState == Lifecycle.State.DESTROYED) {
       			//取消协程
            handleDestroy(parentJob)
        } else if (source.lifecycle.currentState < minState) {
           //暂停协程
            dispatchQueue.pause()
        } else {
            //恢复协程
            dispatchQueue.resume()
        }
    }

    init {
        //取消协程
        if (lifecycle.currentState == Lifecycle.State.DESTROYED) {
            handleDestroy(parentJob)
        } else {
            //添加观察
            lifecycle.addObserver(observer)
        }
    }
    @Suppress("NOTHING_TO_INLINE") // avoid unnecessary method
    private inline fun handleDestroy(parentJob: Job) {
        parentJob.cancel()
        finish()
    }
    @MainThread
    fun finish() {
        lifecycle.removeObserver(observer)
        dispatchQueue.finish()
    }
}
```

launchWhenResumed 内部在 `LifecycleContro` 中注册了观察者，最终通过协程的调度器 `PausingDispatcher` 来控制挂起或者恢复协程.

