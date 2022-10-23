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

从 WindowManager 到 WMS 的具体流畅如下所示：

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

```java
public int addWindow(Session session, IWindow client, int seq,
        LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
        Rect outContentInsets, Rect outStableInsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
        InsetsState outInsetsState, InsetsSourceControl[] outActiveControls,
        int requestUserId) {
    Arrays.fill(outActiveControls, null);
    int[] appOp = new int[1];
    final boolean isRoundedCornerOverlay = (attrs.privateFlags
            & PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY) != 0;
    int res = mPolicy.checkAddPermission(attrs.type, isRoundedCornerOverlay, attrs.packageName,
            appOp);
    if (res != WindowManagerGlobal.ADD_OKAY) {
        return res;
    }

    WindowState parentWindow = null;
    final int callingUid = Binder.getCallingUid();
    final int callingPid = Binder.getCallingPid();
    final long origId = Binder.clearCallingIdentity();
    final int type = attrs.type;

    synchronized (mGlobalLock) {
        if (!mDisplayReady) {
            throw new IllegalStateException("Display has not been initialialized");
        }

        final DisplayContent displayContent = getDisplayContentOrCreate(displayId, attrs.token);

        if (displayContent == null) {
            ProtoLog.w(WM_ERROR, "Attempted to add window to a display that does "
                    + "not exist: %d. Aborting.", displayId);
            return WindowManagerGlobal.ADD_INVALID_DISPLAY;
        }
        if (!displayContent.hasAccess(session.mUid)) {
            ProtoLog.w(WM_ERROR,
                    "Attempted to add window to a display for which the application "
                            + "does not have access: %d.  Aborting.", displayId);
            return WindowManagerGlobal.ADD_INVALID_DISPLAY;
        }

        if (mWindowMap.containsKey(client.asBinder())) {
            ProtoLog.w(WM_ERROR, "Window %s is already added", client);
            return WindowManagerGlobal.ADD_DUPLICATE_ADD;
        }

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

        if (type == TYPE_PRIVATE_PRESENTATION && !displayContent.isPrivate()) {
            ProtoLog.w(WM_ERROR,
                    "Attempted to add private presentation window to a non-private display.  "
                            + "Aborting.");
            return WindowManagerGlobal.ADD_PERMISSION_DENIED;
        }

        if (type == TYPE_PRESENTATION && !displayContent.getDisplay().isPublicPresentation()) {
            ProtoLog.w(WM_ERROR,
                    "Attempted to add presentation window to a non-suitable display.  "
                            + "Aborting.");
            return WindowManagerGlobal.ADD_INVALID_DISPLAY;
        }

        int userId = UserHandle.getUserId(session.mUid);
        if (requestUserId != userId) {
            try {
                mAmInternal.handleIncomingUser(callingPid, callingUid, requestUserId,
                        false /*allowAll*/, ALLOW_NON_FULL, null, null);
            } catch (Exception exp) {
                ProtoLog.w(WM_ERROR, "Trying to add window with invalid user=%d",
                        requestUserId);
                return WindowManagerGlobal.ADD_INVALID_USER;
            }
            // It's fine to use this userId
            userId = requestUserId;
        }

        ActivityRecord activity = null;
        final boolean hasParent = parentWindow != null;
        // Use existing parent window token for child windows since they go in the same token
        // as there parent window so we can apply the same policy on them.
        WindowToken token = displayContent.getWindowToken(
                hasParent ? parentWindow.mAttrs.token : attrs.token);
        // If this is a child window, we want to apply the same type checking rules as the
        // parent window type.
        final int rootType = hasParent ? parentWindow.mAttrs.type : type;

        boolean addToastWindowRequiresToken = false;

        if (token == null) {
            if (!unprivilegedAppCanCreateTokenWith(parentWindow, callingUid, type,
                    rootType, attrs.token, attrs.packageName)) {
                return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
            }
            final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
            token = new WindowToken(this, binder, type, false, displayContent,
                    session.mCanAddInternalSystemWindow, isRoundedCornerOverlay);
        } else if (rootType >= FIRST_APPLICATION_WINDOW
                && rootType <= LAST_APPLICATION_WINDOW) {
            activity = token.asActivityRecord();
            if (activity == null) {
                ProtoLog.w(WM_ERROR, "Attempted to add window with non-application token "
                        + ".%s Aborting.", token);
                return WindowManagerGlobal.ADD_NOT_APP_TOKEN;
            } else if (activity.getParent() == null) {
                ProtoLog.w(WM_ERROR, "Attempted to add window with exiting application token "
                        + ".%s Aborting.", token);
                return WindowManagerGlobal.ADD_APP_EXITING;
            } else if (type == TYPE_APPLICATION_STARTING && activity.startingWindow != null) {
                ProtoLog.w(WM_ERROR,
                        "Attempted to add starting window to token with already existing"
                                + " starting window");
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }
        } else if (rootType == TYPE_INPUT_METHOD) {
            if (token.windowType != TYPE_INPUT_METHOD) {
                ProtoLog.w(WM_ERROR, "Attempted to add input method window with bad token "
                        + "%s.  Aborting.", attrs.token);
                return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
            }
        } else if (rootType == TYPE_VOICE_INTERACTION) {
            if (token.windowType != TYPE_VOICE_INTERACTION) {
                ProtoLog.w(WM_ERROR, "Attempted to add voice interaction window with bad token "
                        + "%s.  Aborting.", attrs.token);
                return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
            }
        } else if (rootType == TYPE_WALLPAPER) {
            if (token.windowType != TYPE_WALLPAPER) {
                ProtoLog.w(WM_ERROR, "Attempted to add wallpaper window with bad token "
                        + "%s.  Aborting.", attrs.token);
                return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
            }
        } else if (rootType == TYPE_ACCESSIBILITY_OVERLAY) {
            if (token.windowType != TYPE_ACCESSIBILITY_OVERLAY) {
                ProtoLog.w(WM_ERROR,
                        "Attempted to add Accessibility overlay window with bad token "
                                + "%s.  Aborting.", attrs.token);
                return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
            }
        } else if (type == TYPE_TOAST) {
            // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
            addToastWindowRequiresToken = doesAddToastWindowRequireToken(attrs.packageName,
                    callingUid, parentWindow);
            if (addToastWindowRequiresToken && token.windowType != TYPE_TOAST) {
                ProtoLog.w(WM_ERROR, "Attempted to add a toast window with bad token "
                        + "%s.  Aborting.", attrs.token);
                return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
            }
        } else if (type == TYPE_QS_DIALOG) {
            if (token.windowType != TYPE_QS_DIALOG) {
                ProtoLog.w(WM_ERROR, "Attempted to add QS dialog window with bad token "
                        + "%s.  Aborting.", attrs.token);
                return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
            }
        } else if (token.asActivityRecord() != null) {
            ProtoLog.w(WM_ERROR, "Non-null activity for system window of rootType=%d",
                    rootType);
            // It is not valid to use an app token with other system types; we will
            // instead make a new token for it (as if null had been passed in for the token).
            attrs.token = null;
            token = new WindowToken(this, client.asBinder(), type, false, displayContent,
                    session.mCanAddInternalSystemWindow);
        }

        final WindowState win = new WindowState(this, session, client, token, parentWindow,
                appOp[0], seq, attrs, viewVisibility, session.mUid, userId,
                session.mCanAddInternalSystemWindow);
        if (win.mDeathRecipient == null) {
            // Client has apparently died, so there is no reason to
            // continue.
            ProtoLog.w(WM_ERROR, "Adding window client %s"
                    + " that is dead, aborting.", client.asBinder());
            return WindowManagerGlobal.ADD_APP_EXITING;
        }

        if (win.getDisplayContent() == null) {
            ProtoLog.w(WM_ERROR, "Adding window to Display that has been removed.");
            return WindowManagerGlobal.ADD_INVALID_DISPLAY;
        }

        final DisplayPolicy displayPolicy = displayContent.getDisplayPolicy();
        displayPolicy.adjustWindowParamsLw(win, win.mAttrs, callingPid, callingUid);

        res = displayPolicy.validateAddingWindowLw(attrs, callingPid, callingUid);
        if (res != WindowManagerGlobal.ADD_OKAY) {
            return res;
        }

        final boolean openInputChannels = (outInputChannel != null
                && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
        if  (openInputChannels) {
            win.openInputChannel(outInputChannel);
        }

        // If adding a toast requires a token for this app we always schedule hiding
        // toast windows to make sure they don't stick around longer then necessary.
        // We hide instead of remove such windows as apps aren't prepared to handle
        // windows being removed under them.
        //
        // If the app is older it can add toasts without a token and hence overlay
        // other apps. To be maximally compatible with these apps we will hide the
        // window after the toast timeout only if the focused window is from another
        // UID, otherwise we allow unlimited duration. When a UID looses focus we
        // schedule hiding all of its toast windows.
        if (type == TYPE_TOAST) {
            if (!displayContent.canAddToastWindowForUid(callingUid)) {
                ProtoLog.w(WM_ERROR, "Adding more than one toast window for UID at a time.");
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }
            // Make sure this happens before we moved focus as one can make the
            // toast focusable to force it not being hidden after the timeout.
            // Focusable toasts are always timed out to prevent a focused app to
            // show a focusable toasts while it has focus which will be kept on
            // the screen after the activity goes away.
            if (addToastWindowRequiresToken
                    || (attrs.flags & LayoutParams.FLAG_NOT_FOCUSABLE) == 0
                    || displayContent.mCurrentFocus == null
                    || displayContent.mCurrentFocus.mOwnerUid != callingUid) {
                mH.sendMessageDelayed(
                        mH.obtainMessage(H.WINDOW_HIDE_TIMEOUT, win),
                        win.mAttrs.hideTimeoutMilliseconds);
            }
        }

        // From now on, no exceptions or errors allowed!

        res = WindowManagerGlobal.ADD_OKAY;

        if (mUseBLAST) {
            res |= WindowManagerGlobal.ADD_FLAG_USE_BLAST;
        }
        if (sEnableTripleBuffering) {
            res |= WindowManagerGlobal.ADD_FLAG_USE_TRIPLE_BUFFERING;
        }

        if (displayContent.mCurrentFocus == null) {
            displayContent.mWinAddedSinceNullFocus.add(win);
        }

        if (excludeWindowTypeFromTapOutTask(type)) {
            displayContent.mTapExcludedWindows.add(win);
        }

        win.attach();
        mWindowMap.put(client.asBinder(), win);
        win.initAppOpsState();

        final boolean suspended = mPmInternal.isPackageSuspended(win.getOwningPackage(),
                UserHandle.getUserId(win.getOwningUid()));
        win.setHiddenWhileSuspended(suspended);

        final boolean hideSystemAlertWindows = !mHidingNonSystemOverlayWindows.isEmpty();
        win.setForceHideNonSystemOverlayWindowIfNeeded(hideSystemAlertWindows);

        final ActivityRecord tokenActivity = token.asActivityRecord();
        if (type == TYPE_APPLICATION_STARTING && tokenActivity != null) {
            tokenActivity.startingWindow = win;
            ProtoLog.v(WM_DEBUG_STARTING_WINDOW, "addWindow: %s startingWindow=%s",
                    activity, win);
        }

        boolean imMayMove = true;

        win.mToken.addWindow(win);
        displayPolicy.addWindowLw(win, attrs);
        if (type == TYPE_INPUT_METHOD) {
            displayContent.setInputMethodWindowLocked(win);
            imMayMove = false;
        } else if (type == TYPE_INPUT_METHOD_DIALOG) {
            displayContent.computeImeTarget(true /* updateImeTarget */);
            imMayMove = false;
        } else {
            if (type == TYPE_WALLPAPER) {
                displayContent.mWallpaperController.clearLastWallpaperTimeoutTime();
                displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
            } else if ((attrs.flags & FLAG_SHOW_WALLPAPER) != 0) {
                displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
            } else if (displayContent.mWallpaperController.isBelowWallpaperTarget(win)) {
                // If there is currently a wallpaper being shown, and
                // the base layer of the new window is below the current
                // layer of the target window, then adjust the wallpaper.
                // This is to avoid a new window being placed between the
                // wallpaper and its target.
                displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
            }
        }

        final WindowStateAnimator winAnimator = win.mWinAnimator;
        winAnimator.mEnterAnimationPending = true;
        winAnimator.mEnteringAnimation = true;
        // Check if we need to prepare a transition for replacing window first.
        if (activity != null && activity.isVisible()
                && !prepareWindowReplacementTransition(activity)) {
            // If not, check if need to set up a dummy transition during display freeze
            // so that the unfreeze wait for the apps to draw. This might be needed if
            // the app is relaunching.
            prepareNoneTransitionForRelaunching(activity);
        }

        if (displayPolicy.getLayoutHint(win.mAttrs, token, outFrame, outContentInsets,
                outStableInsets, outDisplayCutout)) {
            res |= WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_SYSTEM_BARS;
        }
        outInsetsState.set(win.getInsetsState(), win.isClientLocal());

        if (mInTouchMode) {
            res |= WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE;
        }
        if (win.mActivityRecord == null || win.mActivityRecord.isClientVisible()) {
            res |= WindowManagerGlobal.ADD_FLAG_APP_VISIBLE;
        }

        displayContent.getInputMonitor().setUpdateInputWindowsNeededLw();

        boolean focusChanged = false;
        if (win.canReceiveKeys()) {
            focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                    false /*updateInputWindows*/);
            if (focusChanged) {
                imMayMove = false;
            }
        }

        if (imMayMove) {
            displayContent.computeImeTarget(true /* updateImeTarget */);
        }

        // Don't do layout here, the window must call
        // relayout to be displayed, so we'll do it there.
        win.getParent().assignChildLayers();

        if (focusChanged) {
            displayContent.getInputMonitor().setInputFocusLw(displayContent.mCurrentFocus,
                    false /*updateInputWindows*/);
        }
        displayContent.getInputMonitor().updateInputWindowsLw(false /*force*/);

        ProtoLog.v(WM_DEBUG_ADD_REMOVE, "addWindow: New client %s"
                + ": window=%s Callers=%s", client.asBinder(), win, Debug.getCallers(5));

        if (win.isVisibleOrAdding() && displayContent.updateOrientation()) {
            displayContent.sendNewConfiguration();
        }

        getInsetsSourceControls(win, outActiveControls);
    }

    Binder.restoreCallingIdentity(origId);

    return res;
}
```
