### 前言

`WindowManagerService` 简称 WMS ，是系统的核心服务，主要分为四大部分，风别是 `窗口管理`，`窗口动画`,`输入系统中转站`,`Surface 管理` 。

### 了解 WMS

WMS 的职责很多，主要的就是下面这几点：

- 窗口管理：WMS是窗口的管理者，负责窗口的启动，添加和删除，另外窗口的大小也时有 WMS 管理的，管理窗口的核心成员有`DisplayContent`,`WindowToken` 和 `WindowState`
- 窗口动画：窗口间进行切换时，使用窗口动画可以更好看一些，窗口动画由 WMS 动画子系统来负责，动画的管理系统为 WindowAnimator
- 输入系统的中转站：通过对窗口触摸而产生的触摸事件，`InputManagerServer(IMS)` 会对触摸事件进行处理，他会寻找一个最合适的窗口来处理触摸反馈信息，WMS 是窗口的管理者，因此理所当然的就成为了输入系统的中转站。
- Surface 管理：窗口并不具备绘制的功能，因此每个窗口都需要有一个块 Surface 来供自己绘制，为每个窗口分配 Surface 是由 WMS 来完成的。

WMS 的职责可以总结为下图：

![2828107-9ddff2816fe1805e](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202210191446431.webp)

### WMS 的启动

WMS 是在 `SystemServer` 内部启动的

```java
private void startOtherServices() {
 	  //.....
  
  	//1
    WindowManagerService wm = null;
    InputManagerService inputManager = null;

		//2
    traceBeginAndSlog("StartInputManagerService");
    inputManager = new InputManagerService(context);
    traceEnd();

    traceBeginAndSlog("StartWindowManagerService");
    //WMS needs sensor service ready
    ConcurrentUtils.waitForFutureNoInterrupt(mSensorServiceStart, START_SENSOR_SERVICE);
    mSensorServiceStart = null;
    //3
    wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
            new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
  	// WMS 初创建完成后调用，后面会讲到
    wm.onInitReady();
  
    //4
    ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
    //5
    ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
            /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
    traceEnd();
		//...
    try {
        //6 
        wm.displayReady();
    } catch (Throwable e) {
        reportWtf("making display ready", e);
    }
    traceEnd();

    
    try {
        //7
        wm.systemReady();
    } catch (Throwable e) {
        reportWtf("making Window Manager Service ready", e);
    }
    traceEnd();
		///...
}
```

startOtherServer 用于启动其他服务，大概有70多个，上面列出了 WMS 和 IMS 的启动逻辑。

在注释1的地方声明了 WMS 和 IMS， 这里值列出了两个，其实非常多。

注释2处创建了 IMS ，

注释 3 调用了 WMS 的 main 方法，并且传入了 IMS，因为 WMS 是 IMS 的中转站。观察  WindowManagerService.main 方法可以知道他是运行在 SystemServer 的 run 方法中，换句话说就是运行在 `system_server` 线程中。

注释4和5处将 WMS 和 IMS 注册到 ServerManager 里面，这样客户端想要使用 WMS 就需要先去 ServiceManager 中查询信息，然后与 WMS 所在的进程建立通信，这样客户端就可以使用 WMS 了。

注释 6 用来初始化显示信息，注释 7 处用来通知 WMS 系统初始化工作已经完成，内部调用了 WindowManagerPolicy 的 systemReady 方法。

接着，我们看看 WMS 的 maain 方法：

```java
public static WindowManagerService main(final Context context, final InputManagerService im,
        final boolean showBootMsgs, final boolean onlyCore, WindowManagerPolicy policy,
        ActivityTaskManagerService atm) {
    return main(context, im, showBootMsgs, onlyCore, policy, atm,
            SurfaceControl.Transaction::new, Surface::new, SurfaceControl.Builder::new);
}

public static WindowManagerService main(final Context context, final InputManagerService im,final boolean showBootMsgs, final boolean onlyCore, WindowManagerPolicy policy,ActivityTaskManagerService atm, Supplier<SurfaceControl.Transaction> transactionFactory,Supplier<Surface> surfaceFactory,
            Function<SurfaceSession, SurfaceControl.Builder> surfaceControlFactory) {
    	
        DisplayThread.getHandler().runWithScissors(() ->
                sInstance = new WindowManagerService(context, im, showBootMsgs, onlyCore, policy,
                        atm, transactionFactory, surfaceFactory, surfaceControlFactory), 0);
        return sInstance;
    }

```

上面通过 DisplayThread 的 getHandler 方法获取到了 DisplayThread 的 Handler 实例。

DisplayThread 是一个单例的前台线程，用来处理需要低延时显示的相关操作，并且只能由 WindowManager，DisplayManager 和 INputManager 试试执行快速操作。

`runWithScissors` 表达式中创建了 WMS 对象。至于 Handler.runWithScissors 方法，**[有兴趣的可以看一下这篇文章](https://juejin.cn/post/7156821621557166087)**。

到这里我们就知道 WMS 是从何处启动的了，下面我们来看一下 WMS 的构造方法

### WMS构造方法

```java
private WindowManagerService(Context context, InputManagerService inputManager,
            boolean showBootMsgs, boolean onlyCore, WindowManagerPolicy policy,
            ActivityTaskManagerService atm, TransactionFactory transactionFactory) {
       ...
     		//1
        mInputManager = inputManager; // Must be before createDisplayContentLocked.
  			//2
  		  mPolicy = policy;
        //3
        mAnimator = new WindowAnimator(this);
  			//4
        mRoot = new RootWindowContainer(this);
				//5
        mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);

				//6
        mActivityManager = ActivityManager.getService();
				//7
        LocalServices.addService(WindowManagerInternal.class, new LocalService());
  		...
    }
```

1. 保存 IMS ，这样就持有了 IMS 的引用。
2. 初始化 WindowManagerPolicy ，它用来定义一个窗口测量所需要遵循的规范。
3. 创建了 WindowAnimator，它用于管理所有的窗口动画。
4. 创建 RootWindowContainer 对象，根窗口容器
5. 获取 DisplayManager 服务
6. 获取 AMS，并持有他的引用
7. 将 LocalService 添加到 LocalServices 中

在上面 WMS的启动中 WMS 创建完成后会调用 `wm.onInitReady` 方法，下面我们来看一下这个方法：

```java
 public void onInitReady() {
   initPolicy();

   //将 WMS 添加到 Watchdog 中，Watchdog 用来监控一下系统关键服务的运行情况。
   //这些被监控的服务都会实现 Watchdog.Monitor 接口，Watchdog 每分钟都会对
   //被监控的服务进行检测，如果被监控的服务出现了死锁，则会杀死 Watchdog 所在的进程
   Watchdog.getInstance().addMonitor(this);
   ......
}

private void initPolicy() {
    UiThread.getHandler().runWithScissors(new Runnable() {
        @Override
        public void run() {
            WindowManagerPolicyThread.set(Thread.currentThread(), Looper.myLooper());
            mPolicy.init(mContext, WindowManagerService.this, WindowManagerService.this);
        }
    }, 0);
}
```

上面的和 WMS 的 main 方法类似，WindowManagerPolicy(简称 WMP) 是一个接口，init 的具体实现在 PhoneWindowManager(PWM) 中，并且通过上面代码我们知道，init 方法运行在 `android.ui`线程中。因此他的线程优先级要高于 `android.display` 线程，必须等 init 方法执行完成后，`android.display`线程才会被唤醒从而继续执行下面的代码。

在上面的文章中，一共提供了三个线程，分别是 `system_server`，`android.display` ，`android.ui`，他们之间的关系如下图所示：

![image-20221021162034412](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202210211620453.png)

`system_server` 线程中会调用 main 方法，mian 方法中会创建 WMS，创建的过程实在 `android.display` 线程中，他的优先级会高一些，创建完成后才会唤醒处于 `system_server` 线程。

WMS 创建完成后会调用 onInitReady 中的 initPolicy 方法，该方法中会调用 PWM 的 init() 方法，init 方法调用完成之后就会唤醒 `system_server` 线程。

之后就会接着执行 `system_server` 中的代码，例如 displayReady 等。

