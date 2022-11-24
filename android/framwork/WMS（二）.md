### 前言

上篇文章中分析了 WindowManager 与 WMS 交互以及 WMS 启动 和 添加 Window 的过程，在 `addWindow` 方法的最后将 `WindowState` 添加到了 `WindowToken` 中，从上篇文章中，我们也知道 Token 是可以复用的，并且会把相同组件的窗口(WindowState) 集合在一起方便管理。并且也知道了隶属于同一个 `DsiplayContent` 的窗口会被显示在同一块屏幕中，`DisplayContent` 可以起到一个隔离的效果。

在上篇文章中对 `DisplayContent` 描述的不是特别清楚，只知道隶属于 `DsiplayContent` 的窗口会被显示在同一块屏幕中，但是不知道是如何进行绑定的。在后面几天重新看了一下源码，发现 DisplayContent 和 WindowToken 是进行绑定的。在创建 `WindowToken` 的时候会传入 DisplayContent ，接着调用 `onDisplayChanged` 方法将 Windowtoken 添加进去，从而达到绑定的效果。而 WindowToken 中保存的是 相同组件的 WindowState ，所以这三者刚好相连接了起来。

那么 WMS 是如何确定窗口的显示次序以及如何布局进行展示呢，下面我们接着看

### 窗口显示次序

在 addWIndow 的前半部分中，WMS 为窗口创建了描述窗口状态的 WindowState ，接下来就会确定窗口的显示次序。手机屏幕是以左上角为原点，往右是 X 轴方向，往下是 Y 轴方向。为了方便窗口的显示次序，屏幕被拓展成为了一个三维的空间，即多了一个 Z 轴，其垂直于屏幕。多个窗口会按照顺序排布在这个虚拟的 Z 轴上，因此显示的次序也被称为 Z序(Z-Order)。

在 `WindowState` 中有三个 int 值 mBaseLayer + mSubLayer + mLayer 来标志窗口所处的位置，前两个主要确定窗口的位置，mLayer 才是正在 Z 轴的值。

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

接着就会将 WindowState 添加到 mToken 中去做进一步调整

```java
win.mToken.addWindow(win);
```

```java
void addWindow(final WindowState win) {
    if (DEBUG_FOCUS) Slog.d(TAG_WM,
            "addWindow: win=" + win + " Callers=" + Debug.getCallers(5));

    if (win.isChildWindow()) {
        // Child windows are added to their parent windows.
        return;
    }
    if (!mChildren.contains(win)) {
        if (DEBUG_ADD_REMOVE) Slog.v(TAG_WM, "Adding " + win + " to " + this);
        addChild(win, mWindowComparator);
        mWmService.mWindowsChanged = true;
        // TODO: Should we also be setting layout needed here and other places?
    }
}
private final Comparator<WindowState> mWindowComparator =
            (WindowState newWindow, WindowState existingWindow) -> {
        final WindowToken token = WindowToken.this;
        if (newWindow.mToken != token) {
            throw new IllegalArgumentException("newWindow=" + newWindow
                    + " is not a child of token=" + token);
        }

        if (existingWindow.mToken != token) {
            throw new IllegalArgumentException("existingWindow=" + existingWindow
                    + " is not a child of token=" + token);
        }

        return isFirstChildWindowGreaterThanSecond(newWindow, existingWindow) ? 1 : -1;
};
protected boolean isFirstChildWindowGreaterThanSecond(WindowState newWindow,
                                                      WindowState existingWindow) {
   // New window is considered greater if it has a higher or equal base layer.
   return newWindow.mBaseLayer >= existingWindow.mBaseLayer;
}
```

可以看出来上面都是非子窗口的逻辑，在添加的时候会根据 mBaseLayer 去不断地对比，直到找到一个大于等于的层级添加到上面。

通过上篇文章和上面的分析，我们可以知道，在创建了 `WindowToken` 之后，