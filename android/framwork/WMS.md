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



