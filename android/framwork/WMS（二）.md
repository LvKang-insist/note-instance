### 前言

上篇文章中分析了 WindowManager 与 WMS 交互以及 WMS 启动 和 添加 Window 的过程，在 `addWindow` 方法的最后将 `WindowState` 添加到了 `WindowToken` 中，从上篇文章中，我们也知道 Token 是可以复用的，并且会把相同组件的窗口(WindowState) 集合在一起，方便管理。

接着在复习一下 `DisplayContent` ，隶属于同一个 `DisplayContent` 的窗口会被显示到同一个屏幕中。`DisplayContent` 是在 addWindow 的时候通过App端传入的 token(IBinder对象) 和 `displayId`  从 `RootWindowContainer` 中获取的，如果获取不到，就会在 `RootWindowContainer` 中的方法重新创建一个。在 DisplayContent 的构造方法中，会通过 WMS 将自身添加到 `WMS` 的 mRoot (RootWindowContainer) 中去 。大致的流程如下所示：

![image-20221109114830813](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202211091148856.png)

其中 `RootWindowContainer` 是负责管理 DisplayContent 。

通过上篇文章我们知道了 WindowState 与窗口一一对应，内部保存了窗口的所有参数，WindowState 用来描述窗口的状态。这篇文章就从 `WindowState` 入手，继续往下研究。

### WindowState

在 addWIndow 的前半部分中，WMS 为窗口创建了描述窗口状态的 WindowState ，接下来就会确定窗口的显示次序。手机屏幕是以左上角为原点，往右是 X 轴方向，往下是 Y 轴方向。为了方便窗口的显示次序，屏幕被拓展成为了一个三维的空间，即多了一个 Z 轴，其垂直于屏幕。多个窗口会按照顺序排布在这个虚拟的 Z 轴上，因此显示的次序也被称为 Z序(Z-Order)。

#### 构造方法

```java
WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
        WindowState parentWindow, int appOp, int seq, WindowManager.LayoutParams a,
        int viewVisibility, int ownerId, boolean ownerCanAddInternalSystemWindow,
        PowerManagerWrapper powerManagerWrapper) {
    super(service);
    ......
    
    //如果是子窗口  
    if (mAttrs.type >= FIRST_SUB_WINDOW && mAttrs.type <= LAST_SUB_WINDOW) {
        //通过父窗口获取当前窗口的主序
        mBaseLayer = mPolicy.getWindowLayerLw(parentWindow)
                * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
        mSubLayer = mPolicy.getSubWindowLayerFromTypeLw(a.type);
				......
    } else {
        mBaseLayer = mPolicy.getWindowLayerLw(this)
                * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
        mSubLayer = 0;
        ......
    }
		......
}
```

mBaseLayouter 和 mSubLayer 分别描述了窗口的显示次序分为 `主次序和子次序`。主次序面向的是一个窗口组，用于描述窗口以及子窗口在所有窗口中显示的位置。子次序用来标志一个窗口在一组窗口中的位置。

- 主序越大，则窗口及其子窗口显示的位置就越靠前。
- 子序越大，则子窗口相对于其他兄弟窗口的位置就越靠前。对于父窗口而言，其主序取决于类型，其子序保持为 0，而子窗口的主序与父窗口一样，子窗口的子序则取决于类型。

在上面代码中，对于 Activity 来说，主序都是一样的，那么如何保证他们的 Z-Order 呢，其实 Activity 的顺序是由 AMS 保证的，这个顺序确定了，WMS 端 Activity 窗口的顺序也就确定了，接下来的次序也就方便了，