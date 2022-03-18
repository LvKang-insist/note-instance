### Window

Window 是一个窗口的概念，是所有视图的载体，不管是 Activity，Dialog，还是 Toast，他们的视图都是附加在 Window 上面的。例如在桌面显示一个悬浮窗，就需要用到 Window 来实现。`WindowManager` 是访问 Window 的入口。

Window 是一个抽象类，他的实现类是 `PhoneWidow`，Activity 中的 DecorView ，Dialog 中的 View 都是在 PhoneWindow 中创建的。因此 Window 实际是 View 的直接管理者，例如：事件分发机制中，在 Activity 里面收到点击事件后，会首先通过 window 将事件传递到 DecorView，最后再分发到我们的 View 上。Activity 的 SetContentView 在底层也是通过 Window 来完成的。还有 findViewById  也是调用的 window。

> 在我的理解中，上面第一句话中的 window 和 第二句话中的 Window 不是一个东西。
>
> 第一句话中的 Window 是一个窗口，是一个抽象的概念，并不真实存在，他只是以 View 的形式存在。例如通过 WindowManager 添加一个 Window，这个 Window 就是以 View 的形式存在的。
>
> 第二句话中的 Window 指的是一个类，他的实现类是 PhoneWindow，他是用来创建我们页面中所需要的 View 的。所以这个 Window 可以称之为 View 的直接管理者。PhoneWindow 中的 DecorView 最终也是附加到 Window(窗口)上面的。
>
> 因为在最开始的时候经常把二者搞混，Window 即是 View 管理者，也是窗口，显然是不合理的。以上是我的个人理解，如果有感觉不对的，请指出，谢谢！



### Window 和 WindowManager

如果要对 Window 进行添加和删除就需要通过 `WindowManager ` 来操作，具体如下：

 WindowManager 如何添加 Window？

```kotlin
val textView = TextView(this).apply {
    text = "window"
    textSize = 18f
    setTextColor(Color.BLACK)
    setBackgroundColor(Color.WHITE)
}
val parent = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT, 0, 0, PixelFormat.TRANSPARENT
)
parent.type = WindowManager.LayoutParams.TYPE_APPLICATION
parent.flags =
    WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
parent.gravity = Gravity.END or Gravity.BOTTOM
parent.y = 500
parent.x = 100
windowManager.addView(textView, parent)
```

上面这段代码可以添加一个 Window，位置在 (100,500)，这里面比较重要的属性分别是 `type` 和 `flags`。

#### Type 窗口属性

Type 参数表示 Window 的类型，Window 分三种类型，对应着三种层级，如下：

| Window 类型 |  层级范围   | 说明                                                         |
| ----------- | :---------: | ------------------------------------------------------------ |
| 应用 Window |   1 ~ 99    | 对应着一个 Activity                                          |
| 子 Window   | 1000 ~ 1999 | 不能单独存在，需要附属在特定的 Window 之中，<br />例如常见的 PopupDialog，就是子 Window。 |
| 系统 Window | 2000 ~ 2999 | 需要声明权限才能创建的 Window<br />例如 Toast 和 系统状态栏这些都是系统的 Window |

- **子 Window 无法单独存在**，必须依赖父级 Window，例如 Dialog 必须依赖 Activity 的存在
- Window 分层，**在显示时层级高的会覆盖层级低的窗口**

#### Flags窗口的标志

Flags 表示 Window 的属性，它有多选项，通过这些可以通知 Window 显示的特性，例如：

| Floags                | 特性                                                         |
| --------------------- | ------------------------------------------------------------ |
| FLAG_NOT_FOCUSABLE    | 表示 Window 不需要获取焦点，也不需要各种输入事件，<br />此标记通同时启用 FLAG_NOT_TOUCH_MODAL<br />最终事件会直接传递给下层具有焦点的 Window。 |
| FLAG_NOT_TOUCH_MODAL  | 将 Window 区域以外的单击事件传递给底层的 Window，<br />当前 Window 内的单击事件自己处理，<br />一般都要开启此事件，否则其他 Window 无法收到单击事件 |
| FLAG_SHOW_WHEN_LOCKED | 可以将 Window 显示在锁屏的界面上                             |
| FLAG_TURN_SCREEN_ON   | Window 显示时将屏幕点亮                                      |

#### WindowManager

WindowManager 所提供的功能很简单，常用的只有三个方法，即添加 View，更新View，和删除 View。

这三个方法定义在 ViewManager 接口中，而 WindowManager 继承了 ViewManager

```java
public interface ViewManager{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

```java
public interface WindowManager extends ViewManager 
```

由此看来 WindowManager 操作 Window 的过程更像是在操作 Window 中的 View，我们平常简单的那种可以拖动的 Window 效果其实是很好实现的，只需要修改 LayoutParams 中的 x，y 值就可以改变 Window 的位置。首先给 View 设置 onTouchListener，然后在 onTouch 方法中不断的更新 View 的位置即可。



### Window 内部机制

Window 是一个抽象的概念，每一个 Window 都对应着一个 VIew 和一个 ViewRootImpl。

Window 和 View 通过 `ViewRootImpl ` 来建立联系，因此 Window 并不是实际存在的，它是以 View 的形式存在。这个从 WindowManager 的定义就可以看出，提供的三个方法都是针对 View。这说明 View 才是 Window 的实体。

在实际开发中无法直接访问 Window，对 Window 访问必须通过 `WindowManager`

#### Window 的添加过程

Window 的添加需要通过 `WindowManager ` 的 `addView` 来实现，WindowManager 是一个接口，他的真正实现是 `WindowManageImpl`。如下：

```java
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }
    
        @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
```

可以看到 WindowManagerImpl 并没有直接实现 Window 三大操作，而是全部交给了 `WindowManagerGlobal` 来处理。

##### WindowManagerGlobal.addView

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    //检测参数是否合法
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        //.... 
    }

    ViewRootImpl root;
    View panelParentView = null;

	//创建 ViewRootImpl，并赋值给 root
    root = new ViewRootImpl(view.getContext(), display);
	
    //设置 View 的params
    view.setLayoutParams(wparams);
	
    //将 view，RootRootImpl，wparams 添加到列表中
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);

    
    try {
        //调用 ViewRootImpl 来更新界面并完成 Window 的添加过程
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        // BadTokenException or InvalidDisplayException, clean up.
        if (index >= 0) {
            removeViewLocked(index, true);
        }
        throw e;
    }
}
```

上面代码中，创建了 ViewRootImpl，然后将 view，RootRootImpl，wparams 添加到列表中。最后通过 ViewRootImpl 来完成添加 Window 的过程。

这些列表的定义如下：

```
private final ArrayList<View> mViews = new ArrayList<View>();
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
private final ArrayList<WindowManager.LayoutParams> mParams =
        new ArrayList<WindowManager.LayoutParams>();
private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

- mViews 中是所有 Window 对应的 View
- mRoots 中是所有 Window 对应的 ViewRootImpl
- mParams 存储的是所有 Window 所对应的布局参数
- 而 mDyingViews 中是哪些真在被删除的 View，或者说是已经调用 RemoveView 但是删除操作没有完成的 Window 对象。

##### ViewRootImpl.setView

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;

            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            requestLayout();

            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                        mTempInsets);
                setFrame(mTmpFrame);
            }
            //.....
        }
    }
}

@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

这个方法首先会调用 `requestLayout` 来进行一次刷新请求，其中 `scheduleTraversals()` 是 View 绘制的入口

`requestLayout` 调用之后，调用了 `mWindowSession.addToDisplay` 方法，来完成最终的 `Window` 的添加过程。

在上面代码中，`mWindowSession` 的类型是 `IWindowSession`，他是一个 Binder 对象，真正的实现是 `Session`，也就是 Window 的添加过程是一次 IPC 调用。

在 `Session` 内部会通过 `WindoweManagerServer` 来实现 Window 的添加，如下所示：

```java
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
 Rect outStableInsets, Rect outOutsets, DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel, InsetsState outInsetsState) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel,outInsetsState);
}
```

如此一来，Window 的添加过程就交给了 WindowManagerServer 去处理。`WMS` 会为其分配 `Surface`，确定窗口显示的次序，最终通过 `SurfaceFlinger` 将这些 `Surface` 绘制到屏幕上。

##### 梳理一下流程

1. 首先调用的是 `WindowManagerImpl.addView()`

   在 addView 中将实现委托给了 `WindowManagerGlobal.addView()`

2.  `WindowManagerGlobal.addView()`

   在 addView 中创建了 `ViewRootImpl` 赋值给了  root 。然后将 view，params，root 全部存入了各自的列表中。

   最后调用了 `ViewRootImpl.setView()`

3. `ViewRootImpl.setView()`

   在 setView 中通过调用 `requestLayout` 完成刷新的请求，接着会通过 `IWindowSession` 来完成最终的 `Window` 添加的过程，`IWindowSession` 是一个 Binder 对象，真正的实现类是 `Session`，也就是说 Window 的添加过程试一次 IPC 的调用。

   在 Session 中会通过 `WindowManageServer` 来实现 Window 的添加。



####  Window 的更新过程

这里直接从 `WindowManagerGlobal` 开始：

##### WindowManagerGlobal.updateViewLayout

```java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }
	
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
	//将更新的参数设置到 view 中
    view.setLayoutParams(wparams);

    synchronized (mLock) {
        //获取到 view 在列表中的索引
        int index = findViewLocked(view, true);
        //拿到 view 对应的 ViewRootImpl
        ViewRootImpl root = mRoots.get(index);
        //从参数列表中移除旧的参数
        mParams.remove(index);
        //将新的参数添加到指定的位置中
        mParams.add(index, wparams);
        //调用 ViewRootImpl.setLayoutPrams 对参数进行更新
        root.setLayoutParams(wparams, false);
    }
}
```

##### ViewRootImpl.setLayoutPrams

在 setLayoutPrams 方法中，最终调用了 `scheduleTraversals` 方法来对 View 重新策略，布局，重绘。

除了 View 本身的重绘外，ViewRootImpl 还会通过 WindowSession 来更新 Window 视图，这个过程是由 `WindowManagerServer` 的 `relayoutWindow`来实现的，这同样也是一个 IPC 过程。



#### Window 的删除过程

Window 的删除过程和添加过程都一样，都是先通过 WindowManagerImpl 后，在进一步通过 WindowManagerGlobal 来实现的：

##### WindowManagerGlobal.removeView

```java
@UnsupportedAppUsage
public void removeView(View view, boolean immediate) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }

    synchronized (mLock) {
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }

        throw new IllegalStateException("Calling with view " + view
                + " but the ViewAncestor is attached to " + curView);
    }
}
```

上面代码中，找到在 views 列表中的索引，然后调用了 `removeViewLocked` 来做进一步的删除

```java
private void removeViewLocked(int index, boolean immediate) {
    ViewRootImpl root = mRoots.get(index);
    View view = root.getView();

    if (view != null) {
        InputMethodManager imm = view.getContext().getSystemService(InputMethodManager.class);
        if (imm != null) {
            imm.windowDismissed(mViews.get(index).getWindowToken());
        }
    }
    boolean deferred = root.die(immediate);
    if (view != null) {
        view.assignParent(null);
        if (deferred) {
            mDyingViews.add(view);
        }
    }
}
```

removeViewLocked 是通过 ViewRootImpl 来完成删除操作的。在 WindowManager 中提供了两种删除接口 `removeView` 和 `removeViewImmedialte` 分别是异步删除和同步删除。

一般不会使用 `removeViewImmedialte` 来删除 Window，以免发生意外错误。

所以这里使用的是 异步的删除情况，采用的是 die 方法。die 方法只是发送了一个请求删除的消息就立刻返回了，这个时候 View 并没有完成删除操作，所以最后会将其添加到 mDyingViews 列表中。

`die` 如下所示：

```java
boolean die(boolean immediate) {
    // Make sure we do execute immediately if we are in the middle of a traversal or the damage
    // done by dispatchDetachedFromWindow will cause havoc on return.
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }

    if (!mIsDrawing) {
        destroyHardwareRenderer();
    } else {
        Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                "  window=" + this + ", title=" + mWindowAttributes.getTitle());
    }
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}
```

这个方法里面做了判断，如果是异步删除就会发送一个 MSG_DIE 的消息，ViewRootImpl 中的 handler 会收到这个消息，并调用 doDie 方法，这就是这两种删除方式的区别。

```java
void doDie() {
    checkThread();
    if (LOCAL_LOGV) Log.v(mTag, "DIE in " + this + " of " + mSurface);
    synchronized (this) {
        if (mRemoved) {
            return;
        }
        mRemoved = true;
        if (mAdded) {
            //真正删除的逻辑是在此方法中
            dispatchDetachedFromWindow();
        }
		//....
        mAdded = false;
    }
    WindowManagerGlobal.getInstance().doRemoveView(this);
}
```

### Window 的创建过程

通过上面的分析可以得出，View 不能单独存在，必须依附在 Window 上面，因此有视图的地方就有 Window。这些视图包括 ：Activity，Dialog，Toast，PopupWindow 等等。

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //.....
        if (activity != null) {
            
            appContext.setOuterContext(activity);
            
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);

            //....
        }
    return activity;
}
```

在 Activity 的 attach 方法中，系统会创建 Activity 所属的 Window，并未其设置回调接口，由于 Activity 实现了 `Window` 的 `Callback` 接口，因此当 `Window` 接受到外接的状态改变时就会回调 Activity 中的方法。

Callback 中的方法有很多，但是有些我们是非常熟悉的，例如 `dispatchTouchEvent`,`onAttachedToWdindow` 等等。



#### Activity 的 Window 创建过程

Activity 的创建过程比较复杂，最终会通过 ActivityThread 中的 `performLaunchActivity()` 来完成整个启动过程，在这个方法中会通过类加载器创建 Activity 的实例对象，并调用其 attach 方法为其关联所需的环境变量。

```java
##Activity.attach
	//创建 PhoneWindow
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
	//设置 window 的回调
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
```

通过上面代码可以知道在 attach 方法中，创建了 Window，并设置了 callback。

由于 Activity 的视图是通过 setContentView 方法提供的，我们直接看 setContentView 即可：

```java
##Activity
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

```java
##PhoneWindow
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        // 1，创建 DecorView
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        //2 添加 activity 的布局文件
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        //3
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

在上面代码中，如果没有 DecorView 就创建它，一般来说它内部包含了标题栏和内容栏，但是这个会随着主题的改变而发生改变。但是不管怎么样，内容栏是一定存在的，并且内容栏有固定的 id `content`，完整的 id 是 `android.R.id.content` 。

注释1：:通过 `generateDecor` 创建了 DecorView，接着会调用 `generateLayout` 来加载具体的布局文件到 DecorView 中，这个要加载的布局就和系统版本以及定义的主题有关了。加载完之后就会将内容区域的 View 返回出来，也就是 `mContentParent`

注释2：将 activity 需要显示的布局添加到 `mcontentParent` 中。

注释3：由于 activity 实现了 window 的callback 接口，这里表示 activity 的布局文件已经被添加到 decorView 的 mParentView 中了，于是通知 onContentChanged 。



经过上面三个步骤，`DecorView` 已经初始完成，Activity 的布局文件以及加载到了 `DecorView` 的 `mParentView` 中了，但是这个时候 DecorView 还没有被 WindowManager 正式添加到 Window 中。

在 ActivityThread 的 handlerResumeActivity 中，会调用 activity 的 onResume 方法，接着就会将 DecorView 添加到 Window 中

```java
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
	//..... 
    
    //调用 activity 的 onResume 方法
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    
    final Activity a = r.activity;

    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        if (r.mPreserveWindow) {
            a.mWindowAdded = true;
            r.mPreserveWindow = false;
            ViewRootImpl impl = decor.getViewRootImpl();
            if (impl != null) {
                impl.notifyChildRebuilt();
            }
        }
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                //DecorView 完成了添加和显示的过程
                wm.addView(decor, l);
            } else {
                a.onWindowAttributesChanged(l);
            }
        }
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }
    //..........
}
```

#### Dialog 的 Window 创建过程

- 创建 Window

  Dialog 中创建 `Window` 是在其构造方法中完成，具体如下：

  ```java
  Dialog(@UiContext @NonNull Context context, @StyleRes int themeResId,
          boolean createContextThemeWrapper) {
      //...
      //获取 WindowManager
      mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
  	//创建 Window
      final Window w = new PhoneWindow(mContext);
      mWindow = w;
      //设置 Callback
      w.setCallback(this);
      w.setOnWindowDismissedCallback(this);
      w.setOnWindowSwipeDismissedCallback(() -> {
          if (mCancelable) {
              cancel();
          }
      });
      w.setWindowManager(mWindowManager, null, null);
      w.setGravity(Gravity.CENTER);
      mListenersHandler = new ListenersHandler(this);
  }
  ```

- 初始化 DecorView，将 dialog 的视图添加到 DecorView 中

  ```java
  public void setContentView(@LayoutRes int layoutResID) {
      mWindow.setContentView(layoutResID);
  }
  ```

  这个和 activity 的类似，都是通过 Window 去添加指定的布局文件

- 将 DecorView 添加到 Window 中显示

  ```java
  public void show() {
  	//...
      mDecor = mWindow.getDecorView();
      mWindowManager.addView(mDecor, l);
  	//发送回调消息
      sendShowMessage();
  }
  ```

从上面三个步骤可以发现，Dialog 的 Window 创建和 Activity 的 Window 创建很类似，二者几乎没有什么区别。

当 dialog 关闭时，它会通过 WindowManager来移除 DecorView， `	mWindowManager.removeViewImmediate(mDecor)` 。



普通的 Dialog 有一个特殊的地方，就是必须采用 Activity 的 Context，如果采用 Application 的 Context，就会报错：

```tex
Caused by: android.view.WindowManager$BadTokenException: Unable to add window -- token null is not valid; is your activity running?

```

错误信息很明确，是没有 Token 导致的，而 Token 一般只有 Activity 拥有，所以这里只需要用 Activity 作为 Context 即可。

另外，系统 Window 比较特殊，他可以不需要 Token，我们可以将 Dialog 的 Window Type 修改为系统类型就可以了，如下所示：

```java
val dialog = Dialog(application)
    dialog.setContentView(textView)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        dialog.window?.setType(WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY)
    }else{
        dialog.window?.setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT)
    }
dialog.show()
```

需要注意的是需要申请悬浮窗权限



#### Toast 的 Window 创建过程

Toast 也是基于 Window 来实现的，但是他的工作过程有些复杂。在 Toast 的内部有两类 IPC 的过程，第一类是 Toast 访问 `NotificationManagerService` 过程。第二类是 `NotificationManagerServer` 回调 Toast 里的 TN 接口。下面将 `NotificationManagerService` 简称为 NMS。

Toast 属于系统 Window，内部视图有两种定义方式，一种是系统默认的，另一种是通过 setView 方法来指定一个 View(setView 方法在 android 11 以后已经废弃了，不会再展示自定义视图)，他们都对应 Toast 的一个内部成员 `mNextView`。

##### Toast.show()

Toast 提供了 show 和 cancel 分别用于显示和隐藏 Toast，它们的内部是一个 IPC 的过程，实现如下：

```java
public void show() {
    INotificationManager service = getService();
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    tn.mNextView = mNextView;
    final int displayId = mContext.getDisplayId();
    try {
        if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
            if (mNextView != null) {
                service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
            } else {
                // ...
            }
        }
    } 
    //....
}
public void cancel() {

    try {
        getService().cancelToast(mContext.getOpPackageName(), mToken);
    } catch (RemoteException e) {
        // Empty
    }
    //....
}

static private INotificationManager getService() {
        if (sService != null) {
            return sService;
        }
        sService = INotificationManager.Stub.asInterface(
                ServiceManager.getService(Context.NOTIFICATION_SERVICE));
        return sService;
    }
```

从上面代码中可以看出，显示和影藏都需要通过 `NMS` 来实现，由于 `NMS` 运行在系统进程中，所以只通过能跨进程的调用方式来显示和隐藏 Toast。

首先看 Toast 显示的过程，它调用了 NMS 中的 `enqueueToast` 方法，上面的 `INotificationManager` 只是一个 AIDL 接口， 这个接口使用来和 `NMS` 进行通信的，[对 IPC 通信过程不太清楚的同学可以看一下这篇文章](https://blog.csdn.net/baidu_40389775/article/details/105409365?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164749845516782248586352%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=164749845516782248586352&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-2-105409365.nonecase&utm_term=IPC&spm=1018.2226.3001.4450)。

##### NotificationManagerService.enqueueToast

我们来看一下 NMS 的 enqueueToast 方法，这个方法中已经属于别的进程了。调用的时候传了 五个参数，第一个表示当前应用的包名，第二个 token，第三个 tn 表示远程回调，也是一个 IPC 的过程，第四个 时长，第五个是显示的 id

```java
public void enqueueToast(String pkg, IBinder token, ITransientNotification callback,
        int duration, int displayId) {
    enqueueToast(pkg, token, null, callback, duration, displayId, null);
}

private void enqueueToast(String pkg, IBinder token, @Nullable CharSequence text, @Nullable ITransientNotification callback, int duration, int displayId,
@Nullable ITransientNotificationCallback textCallback) {

    synchronized (mToastQueue) {
        int callingPid = Binder.getCallingPid();
        final long callingId = Binder.clearCallingIdentity();
        try {
            ToastRecord record;
            int index = indexOfToastLocked(pkg, token);
			//如果队列中有，就更新它，而不是重新排在末尾
            if (index >= 0) {
                record = mToastQueue.get(index);
                record.update(duration);
            } else {
                int count = 0;
                final int N = mToastQueue.size();
                for (int i = 0; i < N; i++) {
                    final ToastRecord r = mToastQueue.get(i);
                    //对于同一个应用，taost 不能超过 50 个
                    if (r.pkg.equals(pkg)) {
                        count++;
                        if (count >= MAX_PACKAGE_TOASTS) {
                            Slog.e(TAG, "Package has already queued " + count
                                   + " toasts. Not showing more. Package=" + pkg);
                            return;
                        }
                    }
                }

                //创建对应的 ToastRecord
                record = getToastRecord(callingUid, callingPid, pkg, isSystemToast, token,text, callback, duration, windowToken, displayId, textCallback);
                mToastQueue.add(record);
                index = mToastQueue.size() - 1;
                keepProcessAliveForToastIfNeededLocked(callingPid);
            }
            // ==0 表示只有一个 toast了，直接显示，否则就是还有toast，真在进行显示
            if (index == 0) {
                showNextToastLocked(false);
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
}

    private ToastRecord getToastRecord(int uid, int pid, String packageName, boolean isSystemToast,
            IBinder token, @Nullable CharSequence text, @Nullable ITransientNotification callback,
            int duration, Binder windowToken, int displayId,
            @Nullable ITransientNotificationCallback textCallback) {
        if (callback == null) {
            return new TextToastRecord(this, mStatusBar, uid, pid, packageName,
                    isSystemToast, token, text, duration, windowToken, displayId, textCallback);
        } else {
            return new CustomToastRecord(this, uid, pid, packageName,
                    isSystemToast, token, callback, duration, windowToken, displayId);
        }
    }
```

上面代码中对给定应用的 toast 数量进行判断，如果超过 50 条，就直接退出，这是为了防止 DOS ，如果某个应用一直循环弹出 taost 就会导致其他应用无法弹出，这显然是不合理的。

判断完成之后，就会创建 ToastRecord，它分为两种，一种是 `TextToastRecord` ，还有一种是 `CustomToastRecord` 。由于调用 `enqueueToast` 的时候传入了 Tn，所以 `getToastRecord` 返回的是 `CustomToastRecord` 对象。

最后判断只有一个 toast ，就调用 `showNextToastLocked` 显示，否则就是还有好多个 taost 真在显示。

接着我们看一下 `showNextToastLocked`

```java
void showNextToastLocked(boolean lastToastWasTextRecord) {
    ToastRecord record = mToastQueue.get(0);
    while (record != null) {
		//...
        if (tryShowToast(
                record, rateLimitingEnabled, isWithinQuota, isPackageInForeground)) {
            scheduleDurationReachedLocked(record, lastToastWasTextRecord);
            mIsCurrentToastShown = true;
            if (rateLimitingEnabled && !isPackageInForeground) {
                mToastRateLimiter.noteEvent(userId, record.pkg, TOAST_QUOTA_TAG);
            }
            return;
        }

        int index = mToastQueue.indexOf(record);
        if (index >= 0) {
            mToastQueue.remove(index);
        }
        //是否还有剩余的taost需要显示
        record = (mToastQueue.size() > 0) ? mToastQueue.get(0) : null;
    }
}

    private boolean tryShowToast(ToastRecord record, boolean rateLimitingEnabled,
            boolean isWithinQuota, boolean isPackageInForeground) {
		//.....
        return record.show();
    }
```

上面代码中最后调用的是 `record.show()` 这个  `record` 也就是 `CustomToastRecord` 了。

接着我们来看一下他的 show 方法：

```java
@Override
public boolean show() {
    if (DBG) {
        Slog.d(TAG, "Show pkg=" + pkg + " callback=" + callback);
    }
    try {
        callback.show(windowToken);
        return true;
    } catch (RemoteException e) {
        Slog.w(TAG, "Object died trying to show custom toast " + token + " in package "+ pkg);
        mNotificationManager.keepProcessAliveForToastIfNeeded(pid);
        return false;
    }
}
```

可以看到，调用的是 callback 的 show 方法，这个 callback 就是在 `CustomToastRecord` 创建的时候传入的 Tn 了。这里回调到了 `Tn ` 的 show 方法中。

##### Toast# Tn.show

我们接着看：

```java
TN(Context context, String packageName, Binder token, List<Callback> callbacks,
   @Nullable Looper looper) {
    mPresenter = new ToastPresenter(context, accessibilityManager, getService(),packageName);
    mHandler = new Handler(looper, null) {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case SHOW: {
                    IBinder token = (IBinder) msg.obj;
                    handleShow(token);
                    break;
                }
             //.....
            }
        }
    };
}

public void show(IBinder windowToken) {
    if (localLOGV) Log.v(TAG, "SHOW: " + this);
    mHandler.obtainMessage(SHOW, windowToken).sendToTarget();
}

public void handleShow(IBinder windowToken) {
    if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
                         + " mNextView=" + mNextView);
    //...
    if (mView != mNextView) {
        // remove the old view if necessary
        handleHide();
        mView = mNextView;
        mPresenter.show(mView, mToken, windowToken, mDuration, mGravity, mX, mY,
                        mHorizontalMargin, mVerticalMargin,
                        new CallbackBinder(getCallbacks(), mHandler));
    }
}
```

由于 show 方法是被 NMS 夸进程的方式调用的，所以他们运行在 Binder 线程池中，为了切换到 Toast 请求所在的线程，这里使用了 Handler。通过上面代码，我们可以看出，最终是交给 `ToastPresenter` 去处理了

##### ToastPerenter.show

```java
public class ToastPresenter {
//....
    @VisibleForTesting
    public static final int TEXT_TOAST_LAYOUT = R.layout.transient_notification;
    /**
     * Returns the default text toast view for message {@code text}.
     */
    public static View getTextToastView(Context context, CharSequence text) {
        View view = LayoutInflater.from(context).inflate(TEXT_TOAST_LAYOUT, null);
        TextView textView = view.findViewById(com.android.internal.R.id.message);
        textView.setText(text);
        return view;
    }

    //....
    public ToastPresenter(Context context, IAccessibilityManager accessibilityManager, INotificationManager notificationManager, String packageName) {
        mContext = context;
        mResources = context.getResources();
        //获取 WindowManager
        mWindowManager = context.getSystemService(WindowManager.class);
        mNotificationManager = notificationManager;
        mPackageName = packageName;
        mAccessibilityManager = accessibilityManager;
        //创建参数
        mParams = createLayoutParams();
    }
    
    private WindowManager.LayoutParams createLayoutParams() {
        WindowManager.LayoutParams params = new WindowManager.LayoutParams();
        params.height = WindowManager.LayoutParams.WRAP_CONTENT;
        params.width = WindowManager.LayoutParams.WRAP_CONTENT;
        params.format = PixelFormat.TRANSLUCENT;
        params.windowAnimations = R.style.Animation_Toast;
        params.type = WindowManager.LayoutParams.TYPE_TOAST; //TYPE_TOAST：2005
        params.setFitInsetsIgnoringVisibility(true);
        params.setTitle(WINDOW_TITLE);
        params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
        setShowForAllUsersIfApplicable(params, mPackageName);
        return params;
    }
    
     public void show(View view, IBinder token, IBinder windowToken, int duration, int gravity, int xOffset, int yOffset, float horizontalMargin, float verticalMargin, @Nullable ITransientNotificationCallback callback) {
        show(view, token, windowToken, duration, gravity, xOffset, yOffset, horizontalMargin,
                verticalMargin, callback, false /* removeWindowAnimations */);
    }
    
     public void show(View view, IBinder token, IBinder windowToken, int duration, int gravity,int xOffset, int yOffset, float horizontalMargin, float verticalMargin,@Nullable ITransientNotificationCallback callback, boolean removeWindowAnimations) {
	//.....
        addToastView();
        trySendAccessibilityEvent(mView, mPackageName);
        if (callback != null) {
            try {
                //回调
                callback.onToastShown();
            } catch (RemoteException e) {
                Log.w(TAG, "Error calling back " + mPackageName + " to notify onToastShow()", e);
            }
        }
    }
    
    private void addToastView() {
        if (mView.getParent() != null) {
            mWindowManager.removeView(mView);
        }
        try {
            // 将 Toast 视图添加到 Window 中
            mWindowManager.addView(mView, mParams);
        }
    }
}
```

##### 总结一下

通过上面的研究后，突然间发现弹一个 Toast 还是挺麻烦的。主要的就是内部的 IPC 比较绕。

至于说为什么要进行 IPC ，主要就是为了统一管理系统中所有 Toast 的消失与显示，真正显示和消失操作还是在 App 中完成的。

Toast 的窗口类型是 `TYPE_TOAST`，属于系统类型，Toast 有自己的 `token`，不受 Activity 控制。

Toast 通过 WindowManager 将 view 直接添加到了 Window 中，并没有创建 `PhoneWindow` 和 `DecorView`，这点和 Activity 与 Dialog 不同。

最后总结了 Toast 显示的调用流程图，可参考一下：

![image-20220317182308113](https://cdn.jsdelivr.net/gh/LvKang-insist/PicGo/202203171823215.png)



### 最后总结一下

每一个 Window 都对应着一个 View 和 一个 ViewRootImpl 。Window 表示一个窗口的概念，也是一个抽象的概念，它并不是实际存在的，它是以 View 的方式存在的。

WindowManager 是我们访问 Window 的入口，Window 的具体实现位于 `WindowManagerService` 中。`WindowManager` 和 `WindowManagerService` 交互是一个 IPC 的过程，最终的 IPC 是在 `RootViewImpl` 中完成的。



### 参考资料

> Android 开发艺术探索



### 推荐阅读

- [IPC 之 AIDL 跨进程通信](https://blog.csdn.net/baidu_40389775/article/details/105409365?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164749845516782248586352%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=164749845516782248586352&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-2-105409365.nonecase&utm_term=IPC&spm=1018.2226.3001.4450)
- [IPC 之 AIDL 原理](https://blog.csdn.net/baidu_40389775/article/details/105409865?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164757299616782184648526%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=164757299616782184648526&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-105409865.nonecase&utm_term=ipc%5D&spm=1018.2226.3001.4450)



> 如果本文对你有帮助，请点赞支持，谢谢！
