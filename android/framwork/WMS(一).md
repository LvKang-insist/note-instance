### 前言

前段时间分析了 Window 的添加、更新和删除流程，也知晓了 Activity 、Dialog 和 Toast 中 Window 的创建过程，今天就接着上篇文章，看一下 WMS 的创建 以及WindowManager 添加 WIndow 后 WMS 是怎样进行操作的。[上篇文章点这里直达](https://juejin.cn/post/7076274407416528909#heading-25)；

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

Android 系统在启动的时候，会启动两个重要的进程，一个是 Aygote 进程，两一个是由 Zygote 进程 fork 出来的 system_server 进程，SystemServer 会启动我们在系统中所需要的一系列 Service。

```java
//#SystemServer.java
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

上面的和 WMS 的 main 方法类似，WindowManagerPolicy (简称 WMP) 是一个接口，init 的具体实现在 **PhoneWindowManager**(PWM) 中，并且通过上面代码我们知道，init 方法运行在 `android.ui`线程中。因此他的线程优先级要高于 `android.display` 线程，必须等 init 方法执行完成后，`android.display`线程才会被唤醒从而继续执行下面的代码。

WindowManagerPolicy  用来定义一个窗口策略所要遵循的通用规范，，并提供了 WindowManager 所有的特定 UI 行为
他的具体实现类为 PhoneWindowManager

在上面的文章中，一共提供了三个线程，分别是 `system_server`，`android.display` ，`android.ui`，他们之间的关系如下图所示：

<img src="https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202210261651764.png" style="zoom: 67%;" />

`system_server` 线程中会调用 main 方法，mian 方法中会创建 WMS，创建的过程实在 `android.display` 线程中，他的优先级会高一些，创建完成后才会唤醒处于 `system_server` 线程。

WMS 创建完成后会调用 onInitReady 中的 initPolicy 方法，该方法中会调用 PWM 的 init() 方法，init 方法调用完成之后就会唤醒 `system_server` 线程。init 方法运行中 `android.ui` 线程中。

之后就会接着执行 `system_server` 中的代码，例如 displayReady 等。

### WIMS 窗口管理

Window 的操作有两大部分，一部分是 WindowManager 来处理，一部分是 WMS 来处理，如下图所示：

![image-20221022193438766](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202210221934815.png)

- WindowManager 中，通过 WindowManagerGlobal 创建 ViewRootImpl ，也就是 View 的根。在 ViewRootImpl 中完成对 View 的绘制等操作，然后通过 IPC 获取到 Session ，最终通过 WMS 来进行处理。在理解 **[Window 和 WindowManager 这篇文章中](https://juejin.cn/post/7076274407416528909#heading-0)**，讲解了 WIndow 的创建，添加和删除部分，有兴趣的可以看一下。
- WMS 来处理后面的部分，具体见下文

#### WindowManager -> WMS

我们都知道 Window 的添加最后是通过 ViewRootImpl.addTodisplay 方法来完成的，我们先来看一下：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
    synchronized (this) {
        if (mView == null) {
         	....
            try {
                res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mDisplayCutout, inputChannel,
                        mTempInsets, mTempControls);
            }
            ...
    }
}
```

这里调用了 mWindowSession.addToDisplayAsUser 来完成最后的添加，我们先看看 mWindowSession 是个啥东西。

```java
public ViewRootImpl(Context context, Display display) {
    this(context, display, WindowManagerGlobal.getWindowSession(),
            false /* useSfChoreographer */);
}
```

```java
//WindowManagerGlobal.java
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
     			....
                IWindowManager windowManager = getWindowManagerService();
                //通过 windowManager 创建一个 IWindowSession 
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        });
            }
        return sWindowSession;
    }
}
public static IWindowManager getWindowManagerService() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowManagerService == null) {
                // 通过 AIDL 获取 IWindowManager 
                sWindowManagerService = IWindowManager.Stub.asInterface(
                    //获取 WindowManagerService 的 IBinder 
                    ServiceManager.getService("window"));
			//...
            }
            return sWindowManagerService;
        }
    }
// WindowManagerService.java    
@Override
public IWindowSession openSession(IWindowSessionCallback callback) {
    return new Session(this, callback);//传入了 this，将 Session 就会持有 WindowManagerService 的引用
}
    
```

mWindowSession 是 IwindowSession 的对象，实现类是 Session。

上面代码中，在 VIewRootImpl 初始化的时候，通过IPC 获取到 IWindowManager，然后通过 IPC 的调用创建了 Session 对象。

接着我们来看一下 `mWindowSession.addToDisplayAsUser` 方法，也就是在 Session 类中

```java
@Override
public int addToDisplayAsUser(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, int userId, Rect outFrame,
        Rect outContentInsets, Rect outStableInsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
        InsetsState outInsetsState, InsetsSourceControl[] outActiveControls) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
            outContentInsets, outStableInsets, outDisplayCutout, outInputChannel,
            outInsetsState, outActiveControls, userId);
}
```

可以看到，最终是通过 mService 来完成添加的，

需要注意的是，WMS 并不关系 View 的具体内容，他只关心各个应用显示的界面大小，层级值等，这些数据到包含在 WindowManager.LayoutParams 中。也就是上面的 atrs 属性。

注意 `addWindow` 的第二个参数是一个 IWindow 类型，这是 App 暴露给 WMS 的抽象实例，在 ViewRootImp 中实例化，与 ViewRootImpl 一一对应，同事也是 WMS 向 App 端发送消息的 Binder 通道。

从 WindowManager 到 WMS 的具体流程如下所示：

<img src="https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202210221957806.png" alt="image-20221022195732745" style="zoom: 67%;" />

#### WMS 重要成员

```java

final WindowManagerPolicy mPolicy;
final IActivityManager mActivityManager;
final ActivityManagerInternal mAmInternal;
final AppOpsManager mAppOps;
final DisplaySettings mDisplaySettings;
...
final ArraySet<Session> mSessions = new ArraySet<>();
final WindowHashMap mWindowMap = new WindowHashMap();
final ArrayList<AppWindowToken> mFinishedStarting = new ArrayList<>();
final ArrayList<AppWindowToken> mFinishedEarlyAnim = new ArrayList<>();
final ArrayList<AppWindowToken> mWindowReplacementTimeouts = new ArrayList<>();
final ArrayList<WindowState> mResizingWindows = new ArrayList<>();
final ArrayList<WindowState> mPendingRemove = new ArrayList<>();
WindowState[] mPendingRemoveTmp = new WindowState[20];
final ArrayList<WindowState> mDestroySurface = new ArrayList<>();
final ArrayList<WindowState> mDestroyPreservedSurface = new ArrayList<>();
...
final H mH = new H();
...
final WindowAnimator mAnimator;
...
 final InputManagerService mInputManager
```

- WindowManagerPolicy mPolicy

  窗口管理策略的接口类，用来定义一个窗口策略所要遵循的通用规范，并提供了 WindowManager 所有的特定 UI 行为
  他的具体实现类为 PhoneWindowManager，这个实现类在 WMS 创建时被创建。WMP 允许定制窗口层级和特殊窗口类型以及关键字的调度和布局

- ArraySet<Session> mSessions

  ArraySet 类型的变量，元素类型为 Session，他主要用于进程间的通信。并且每个应用程序都会对应一个 Session，WMS 保存这些 Session 用来记录所有向 WMS 提出窗口管理服务的客户端

- WindowHashMap mWindowMap

  继承自 HashMap，它限制 key 为 IBinder，value 的值类型为 WindowState，WindowState 用于保存窗口信息，在 WMS 中用来描述一个窗口。所以 mWindowMap 就是用来保存 WMS 中各种窗口的集合。

- ArrayList<AppWindowToken> mFinishedStarting

  元素类型是 AppWindowToken，他是 WindowToken 的子类，想要了解 mFinishedStarting 的含义，需要先了解 WindowToken 是什么，WindowToken 主要由两个作用：

  1. 可以理解为窗口令牌，当应用程序想要向 WMS 申请创建一个窗口，则需要向 WMS  出示有效的 WindowToken。AppWindowToken 作为 WindowToken 的子类，主要用来描述应用程序的 WindowToken 结构。
  2. WindowToken 会将相同组件（例如 Activity）的窗口（WindowState）集合在一起，方便管理。

  mFinishedStarting 就是用于存储已经完成启动的应用窗口程序窗口（比如Activity） 的 AppWindowToken 的列表。

  除了 mFinishedStarting，还有类似的 mFinishedEarlyAnim和 mWindowReplacementTimeouts 

  其中 mFinishedEarlyAnim 存储了已经完成窗口绘制并且不需要再展示任何已经保存 surface 的应用程序窗口的 AppWindowToken

  mWindowReplacementTimeout 存储了等待更换的应用程序窗口的 AppWindowToken，如果更换不及时，旧窗口就需要被处理

- ArrayList<WindowState> mResizingWindow

  用来存储正在调整大小的窗口列表。与 mResizingWindows 类似的还有个 mPendingRemove，mDestorySurface 和 mDestoryPreservedSurface 等等。

  其中 mPendingRemove 是在内存耗尽时设置的，里面存有需要强制删除的窗口。

  mDestorySurface 里面存有需要被 Destory 的 Surface ，mDestoryPreservedSurface 里面存有窗口需要保存的等等销毁的 Surface ，为什么窗口需要保存这些 Surface ？这是因为当窗口经历 Surface 变化时，窗口需要一直保持旧 Surface，直到新 Surface 的第一帧绘制完成。

- WindowAnimator mAnimator

  WindowAnimator 类型的变量，用于管理窗口的动画以及特效动画

- H mH

  继承自 Hnadler 类，用于将任务加入到主线程的消息队列中，这样代码逻辑就会在主线程执行

- InputManagerService mINputManager

  IMS ,输入系统的管理者。IMS 会对触摸事件进行处理，他会寻找一个最合适的窗口来处理触摸反馈信息，WMS 是窗口管理者，因此 WMS 理所应当的成为了输入系统的中转站，WMS 包含了 IMS 的引用不足为怪。

#### WMS 添加 Window

##### Part 1

```java
public int addWindow(Session session, IWindow client, int seq,
        LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
        Rect outContentInsets, Rect outStableInsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
        InsetsState outInsetsState, InsetsSourceControl[] outActiveControls,
        int requestUserId) {

    int[] appOp = new int[1];
    //1
    int res = mPolicy.checkAddPermission(attrs.type, isRoundedCornerOverlay, attrs.packageName,
            appOp);
    if (res != WindowManagerGlobal.ADD_OKAY) {
        return res;
    }

    synchronized (mGlobalLock) {
        //2
        final DisplayContent displayContent = getDisplayContentOrCreate(displayId, attrs.token);
        ...
				//3
        if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
            parentWindow = windowForClientLocked(null, attrs.token, false);
            if (parentWindow == null) {
                ProtoLog.w(WM_ERROR, "Attempted to add window with token that is not a window: "
                        + "%s.  Aborting.", attrs.token);
                return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
            }
            if (parentWindow.mAttrs.type >= FIRST_SUB_WINDOW
                    && parentWindow.mAttrs.type <= LAST_SUB_WINDOW) {
                ProtoLog.w(WM_ERROR, "Attempted to add window with token that is a sub-window: "
                        + "%s.  Aborting.", attrs.token);
                return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
            }
        }
        ...		
}
```

WMS 的 `addWindow` 方法返回的是 `addWindow` 的各种状态，例如 添加成功，失败，无效的 display 等，这些状态定义在 WindowManagerGloabl 中 。

注释1 处调用了 checkAddPermission 方法来检查权限，`mPolicy` 的实现类是 `PhoneWindowManager`。

注释2 通过  displayId 来获得 Window 要添加到那个 DisplayContent，如果没有找到，则返回 `WindowManagerGlobal.ADD_INVALID_DISPLAY` 状态。**其中DisplayContent 用来描述一块屏幕**。

如下面代码，从 mRoot（RootWindowContainer） 对应的 DisplayContent，如果没有，则创建一个再返回，RootWindowContainer 是用来管理 DisplayContent 的。

- ```java
    private DisplayContent getDisplayContentOrCreate(int displayId, IBinder token) {
        if (token != null) {
            final WindowToken wToken = mRoot.getWindowToken(token);
            if (wToken != null) {
                return wToken.getDisplayContent();
            }
        }
        DisplayContent displayContent = mRoot.getDisplayContent(displayId);
        // Create an instance if possible instead of waiting for the ActivityManagerService to drive
        // the creation.
        if (displayContent == null) {
            final Display display = mDisplayManager.getDisplay(displayId);
    
            if (display != null) {
                displayContent = mRoot.createDisplayContent(display, null /* controller */);
            }
        }
    
        return displayContent;
    }
    ```

注释3 判断 type 的窗口类型(100 - 1999)，如果是子类型，必须要有父窗口，并且父窗口不能是子窗口类型

##### Part 2

```java
 				ActivityRecord activity = null;
        final boolean hasParent = parentWindow != null;
        //1
        WindowToken token = displayContent.getWindowToken(
                hasParent ? parentWindow.mAttrs.token : attrs.token);
        final int rootType = hasParent ? parentWindow.mAttrs.type : type;

        boolean addToastWindowRequiresToken = false;

			  //2
        if (token == null) {
        				...
           			//3
                token = new WindowToken(this, binder, type, false, displayContent,
                        session.mCanAddInternalSystemWindow, isRoundedCornerOverlay);
          			//4
            } else if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {							  //5	
                atoken = token.asAppWindowToken();
                if (atoken == null) {
                    return WindowManagerGlobal.ADD_NOT_APP_TOKEN;
                } 
          			....
            } else if (rootType == TYPE_INPUT_METHOD) {
                if (token.windowType != TYPE_INPUT_METHOD) {
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } 
						.....
```

注释 1 处通过 displayContent 的 getWindowToken 方法得到父窗口的 WindowToken 或者是当前窗口的 WindowToken。RootType 也同样如此。

注释2处如果 token 等于 null，并且不是应用窗口或者是其他类型的窗口，则窗口就是系统类型的了(例如 Toast)，就进行隐式创建 WindowToken，这说明我们添加窗口时是可以不向 WMS 提供 WindowToken 的，WindowToken 的隐式和显式创建是需要区分的，第四个参数 false 表示隐式创建。一般系统窗口都不需要添加 token，WMS 会隐式创建。例如 Toast 类型的窗口。

注释3创建了 WindowToken 对象，当该对象创建出来之后就说明有一个新的 Window 诞生了，**在 WindowToken 构造方法中，会调用 `onDisplayChanged` 将 token 添加到 `DisplayContent` 中。**

接着就是 token 不为空的情况，会在注释 4 处判断是否位 `应用窗口`，如果是 应用窗口，就会讲 WindowToken 转换为针对于应用程序窗口的 AppWindowToken，然后再继续进行判断。

##### part3

```java
//1
final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
          //2  
					 if (win.mDeathRecipient == null) {
                return WindowManagerGlobal.ADD_APP_EXITING;
            }
						//3
            if (win.getDisplayContent() == null) {
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }

            final DisplayPolicy displayPolicy = displayContent.getDisplayPolicy();
						//4
            displayPolicy.adjustWindowParamsLw(win, win.mAttrs, Binder.getCallingPid(),
                    Binder.getCallingUid());
            win.setShowToOwnerOnlyLocked(mPolicy.checkShowToOwnerOnly(attrs));
						//5
            res = displayPolicy.prepareAddWindowLw(win, attrs);
            ....
						//6
            mWindowMap.put(client.asBinder(), win);
					
            boolean imMayMove = true;
						//7，添加窗口
            win.mToken.addWindow(win);
            
            ...
```

在注释1 处创建了`WindowState`，保存了窗口的所有状态信息(例如 WMS ,Session，WindowToken等)，在 WMS 中它代表一个窗口。WindowState 与窗口时一一对应的关系

在注释2和注释3处判断请求添加窗口的客户端是否已经死亡，如果死亡则不会执行下面逻辑。

注释 4处调用了 adjustWindowParamsLw 方法，这里会根据窗口的 type 类型对窗口的 LayoutParams 的一些成员变量进行修改。源码注释信息为 **清理来自客户端的布局参数。允许策略 做一些事情，比如确保特定类型的窗口不能 输入焦点**

注释 5处调用了 `prepareAddWindowLw` 方法用于准备将窗口添加到系统中

注释 6处将 WindowState 添加到 mWindowMap 中，mWindowMap 是各种窗口的集合。

注释 7 处将 WindowState 添加到该 WindowState 对应的 WindowToken 中（实际上就是保存在 WindowToken 的父类 WindowContainer），这样 WindowToken 就包含了相同组件的 WindowState。

##### 相关类

- IWindow

    App 端暴露给 WMS 的抽象实例，在 ViewRootImpl 中实例化，与 ViewRootImpl 一一对应，也是 WMS 和 APP 端交互的通道

- RootWindowContainer

    设备窗口层次结构的根，再 WMS 构造方法中被创建。负责管理 DisplayContent。

- WindowState

    WMS 端的窗口令牌，与窗口一一对应，是 WMS 管理窗口的重要依据，内部保存了窗口的所有状态信息

- DisplayContent

    如果说 WindowToken 按照窗口之间的逻辑将其分组，那么 DisplayContent 则根据窗口的显示位置将其分组。隶属于同一个 DisplayContent 的窗口会被显示在同一个屏幕中，每一个 DisplayContent 都对应一个唯一的 ID，在添加窗口的时候通过指定这个 ID 决定将被显示在那个屏幕中。

    DisplayContent 有一个隔离的概念，处于不同 DisplayContent 的两个窗口在布局，显示顺序以及动画处理上不会有任何的耦合。因此，就这几个方面来说，DisplayContent 就像是一个孤岛，所有这些操作都可以在内部执行。因此这个本来属于 WMS 全局操作的东西，变成了 DisplayContent 内部的操作了。

    另外，DisplayContent 由 RootWindowContainer 来管理，再添加窗口的最开始，就会根据传入的参数获取 DisplayContent 

- WindowToken

    WindowToken 主要有两个作用

    1. 可以理解为窗口令牌，当应用程序想要向 WMS 申请创建一个窗口，则需要向 WMS  出示有效的 WindowToken。并且窗口类型必须与所持有的 WindowToken 的类型一致。

        从上面的代码中可以看到，在创建系统类型窗口时不需要提供有效的 Token，WMS 会隐式的创建一个 WindowToken，看起来谁都可以添加这个系统窗口，但是在 addWindow 方法一开始就调用 `mPolicy.checkAddPermission` 来检查权限，她要求客户端必须拥有 INTERNAL_SYSTEM_WINDOW 或者 SYSTEM_ALERT_WINDOW 权限才可以创建系统类型窗口。

    2. WindowToken 会将相同组件（例如 Activity）的窗口（WindowState）集合在一起，方便管理。
    
        至于为什么说会集合在一起，因为有些窗口时复用的同一个 token，例如 Activity 和 Dialog 就是复用的同一个 AppToken，Activity 中的 PopWindow 复用的是一个 IWindow 类型 Token，Toast 系统类型的窗口也可以看成 null，就算不是 null，WMS 也会强制创建一个隐式 token。
    
    **构造方法**
    
    ```java
    WindowToken(WindowManagerService service, IBinder _token, int type, boolean persistOnEmpty,
            DisplayContent dc, boolean ownerCanManageAppTokens, boolean roundedCornerOverlay) {
        super(service);
        token = _token;
        windowType = type;
        mPersistOnEmpty = persistOnEmpty;
        mOwnerCanManageAppTokens = ownerCanManageAppTokens;
        mRoundedCornerOverlay = roundedCornerOverlay;
        onDisplayChanged(dc);
    }
    @Override
    void onDisplayChanged(DisplayContent dc) {
      dc.reParentWindowToken(this);
      super.onDisplayChanged(dc);
    }
    ##DisplayContent.java
    void reParentWindowToken(WindowToken token) {
      final DisplayContent prevDc = token.getDisplayContent();
      if (prevDc == this) {
        return;
      }
      if (prevDc != null) {
        if (prevDc.mTokenMap.remove(token.token) != null && token.asAppWindowToken() == null) {
          token.getParent().removeChild(token);
        }
        if (prevDc.mLastFocus == mCurrentFocus) {
          prevDc.mLastFocus = null;
        }
      }
      addWindowToken(token.token, token);
    }
      private void addWindowToken(IBinder binder, WindowToken token) {
        final DisplayContent dc = mWmService.mRoot.getWindowTokenDisplay(token);
        if (dc != null) {
          throw new IllegalArgumentException
        }
        if (binder == null) {
          throw new IllegalArgumentException
        }
        if (token == null) {
          throw new IllegalArgumentException(
        }
        mTokenMap.put(binder, token);
        if (token.asAppWindowToken() == null) {
          // Add non-app token to container hierarchy on the display. App tokens are added through
          // the parent container managing them (e.g. Tasks).
          switch (token.windowType) {
            case TYPE_WALLPAPER:
              mBelowAppWindowsContainers.addChild(token);
              break;
            case TYPE_INPUT_METHOD:
            case TYPE_INPUT_METHOD_DIALOG:
              mImeWindowsContainers.addChild(token);
              break;
            default:
              mAboveAppWindowsContainers.addChild(token);
              break;
          }
        }
    }
    ```
    
    在构造方法中调用 onDisplayChanged 对窗口进行更新，在 reParentWindowToken 中，token 已经绑定过当前 DisplayContent 并且是当前的，就没必要进行添加。然后就会从 mTokenMap 中进行移除，接着讲 token 从 父 WindowContainer 中移除出来。
    
    `addWindowToken` 中 将 token 添加到 `mTonkenMap` 中

### 总结一下子

通过上面的流程，App 到 WMS 注册窗口的流程就完了，WMS 为窗口创建了用来描述状态的 WindowState，接下来就会为新建的窗口显示次序，然后再去申请 Surface，才算是真正的分配了窗口。接下来还有一部分，后面再慢慢搞吧。

这里对 WMS 的 addWindow  流程做一个总结 ：

1. 首先检查权限

2. 接着从 mRoot(RootWindowContainer)中获取 DisplayContent ，如果没有就会根据 `displayId` 创建一个新的。DisplayContent

3. 接着就是 type 类型的判断，如果是子类型，就必须要获取到他的父窗口，

4. 接着使用 DisplayContent 获取当前或者父窗口获取 token，如果为 null 就排除一下子窗口和其他的窗口，剩下的就是可以不用携带 token 的窗口，WMS 会隐式的创建窗口 token。如果不等于 null 就判断是应用窗口就将 token 转为 AppWindowToken，后面还有一大堆窗口判断，只要是不满足就直接 return。

5. 类型啥的判断完成后，就会创建 WindowState，并且传入 WMS、IWindow、token 等。WindowState 里面保存了窗口的所有信息。WindowState 与窗口一一对应。

6. 接着就执行调用了 WindowState 的 attache 、initAppOpsState 等方法，这些先暂时不说。

    WindowState 创建完成后就会被添加到 mWindowMap 中，可以 IWindow 的 Binder 为 key，WindowState 为 value 添加进去。

7. 最后就是 `win.mToken.addWindow(win)` ，这里的 mToken 就是上面 **第三步** 获取的 token，然后将 WindowState 添加到 WindowToken 中。因为 WindowToken 是可以复用的，所以这里的关系就是，每个 WindowToken 都会保存对应的 WindowState，而每个 WindowState 也都会都持有 WindowToekn。

### 通过本文，你应该了解到

- WindowManager 如何与 WMS 交互
- WindowToken 到底是个啥东西，在什么情况下需要传，在啥情况下不用手动传，它的作用是什么。
- DisplayContent 是用来干啥的，他是被谁管理，他自己又在管理者谁。本篇文章到最后也没太看懂如何使用 DisplayContent，所以到目前为止只需要知道他的作用就行了。
- RootWindowContainer 到目前为止干了什么事。
- 了解 WindowState 是个啥东西，与 WindowToken 的关系是咋样的。
- WindowContainer 类，上面 `WindowToken`、`WindowState`、`RootWindowContainer`、`DisplayContent` 都是继承自 WindowContainer，至于这里为啥这么写，我也不太懂，慢慢再看呗。
- ...

### 最后

到这里这篇文章也写完了，但是 WMS 的窗口管理 还有一部分没写，原因就是因为我也还没看懂，所以就先写到这里，等后面看懂了，学会了再写一篇。

文章中写的也不一定全部是正确的，我本人也是自己学自己写，然后再慢慢的梳理，如果那些错了，或者是有任何问题可在下方评论，或者是直接私信我。

### 参考资料

[深入理解 Android 卷3 第四章](https://www.kancloud.cn/alex_wsc/android-deep3/416239)

[WMS 窗口管理](https://www.jianshu.com/p/e00898609874)

[皇叔解析WMS](https://juejin.cn/post/6844903502464942088)

