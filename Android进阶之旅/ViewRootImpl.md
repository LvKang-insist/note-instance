### 简介

ViewRootImpl 是 View 的最高层级，是所有 View 的根。ViewRootImpl 实现了 View 和 WindowManager 之间所需要的协议。ViewRootImpl 的创建过程是从 `WindowManagerImpl` 中开始的。View 的测量，布局，绘制以及上屏，都是从 ViewRootImpl 中开始的。

我们通过一张图来认识一下它：

![image-20220317223445177](C:\Users\345\Desktop\202203172234346.png)



- Window

  我们知道界面中所有的元素都是由 View 构成的，View 是依附于 Window 上面的。Window 只是一个抽象概念，把界面抽象成一个 窗口，也可以抽象成一个 View。

- ViewManange

  一个接口，内部定义了三个方法，用来对 VIew 的添加，更新和删除。

  ```java
  public interface ViewManager
  {
      public void addView(View view, ViewGroup.LayoutParams params);
      public void updateViewLayout(View view, ViewGroup.LayoutParams params);
      public void removeView(View view);
  }
  ```

- WindowManager

  也是一个接口，继承自 ViewManager，在应用程序中，通过 WindowManager 来管理 Window，将 View 附加到 Window 上。他有一个实现类 `WindowManagerImpl`。

- WindowManagerImpl

  `WindowManager` 的实现类，`WindowManagerImpl` 中的内部方法实现都是通过代理类 `WindowManagerGlobal` 来完成。

- WindowManagerGlobal

  `WindowManagerGlobal` 是一个单例，也就是说一个进程中只有一个 `WindowManagerGlobal`对象，他服务与所有页面的 View。

- ViewParent

  一个接口，定义了将成为 View 父级类的职责。

- ViewRootImpl

  视图层次结构的顶部。一个 Window 对应着一个 ViewRootImpl 和 一个 VIew。这个 View 就是被 `ViewRootImpl` 操作的。

> 一个小栗子，我们都只 Actiivty 中 会创建一个 Window 对象。 `setContentView` 方法中的 View 最终也会被添加到 Window 对象中的 `DecorView` 中，也就是说一个 Window 中对应着一个 View。这个 View 是被  `RootViewImpl` 操作的。
>
>  WindowManager 就是入口。通过 WindowManager 的 addView 添加一个 Window(也可以理解为 View)，然后就会创建一个 ViewRootImpl，来对 view 进行操作，最后将 View 渲染到屏幕的窗口上。
>
> 例如 Activity 中，在 onresume 执行完成后，就会获取 Window 中的 DecorView，然后通过 WindowManager 把 DecorView 添加到窗口上，这个过程中是由 RootViewImpl 来完成测量，绘制，等操作的。

> [如果对 Window，WindowManager 不太熟悉可以先看一下这篇文章](https://juejin.cn/post/7076274407416528909)

### RootViewImpl 的初始化

View 的三大流程都是通过 `RootViewImpl` 来完成的，在 `ActivityThread` 中，当 Activity 对象被创建完毕后，在 `onResume` 后，就会通过 WindowManager 将 DecorView 添加到窗口上，在这个过程中会创建 `ViewRootImpl`：

#### ActivityThread.handleResumeActivity

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {

    // .....
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    final Activity a = r.activity;

    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        //设置窗口类型为应用类型
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
  
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                //添加 decor 到 window 中
                wm.addView(decor, l);
            } else {
                a.onWindowAttributesChanged(l);
            }
        }
    } 
    //....
}
```

#### WindowManagerImpl.addView

```java
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

#### WindowManagerGlobal.addView

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    //检查参数是否合法
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
  	//.....
    
    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
   
		//创建 ViewRootImpl
        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);
		
        //将 Window 所对应的 View，ViewRootImp，params 顺序添加到列表中，这一步是为了方便更新和删除 View
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // do this last because it fires off messages to start doing things
        try {
            //把 Window 对应的 View 设置给创建的 ViewRootImpl
            //通过 ViewRootImpl 来更新界面并添加到 WIndow中。
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

### ViewRootImpl  对 View 进行操作



#### ViewRootImpl.setView

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            attrs = mWindowAttributes;
   
      		//请求布局，执行 View 的绘制方法
            requestLayout();
            if ((mWindowAttributes.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                mInputChannel = new InputChannel();
            }
            mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                    & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
                //将 Window 添加到屏幕上，mWindowSession 实现了 IWindowSession接口，是 Session 的 Binder 对象。
                // addToDisplay 是一次 AIDL 的过程，通知 WindowManagerServer 添加 Window
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                        mTempInsets);
                setFrame(mTmpFrame);
            }
			//设置当前 View 的 Parent
            view.assignParent(this);
        }
    }
}
```

#### ViewRootImpl.requestLayout

```java
//请求布局
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
//检测当前线程，如果不是主线程，直接抛出异常
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
            "Only the original thread that created a view hierarchy can touch its views.");
    }
}


final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}


```

在上面代码中，调用 `requestLayout` 请求布局，最终会执行到  `performTraversals` 方法中。在这个方法中会调用 `View` 的 `measure()` ，`layout()` ，`draw()` 方法。

#### ViewRootImpl. performTraversals

```java
private void performTraversals() {
    // cache mView since it is used so much below...
    final View host = mView;

	//想要展示 Window 的宽高
    int desiredWindowWidth;
    int desiredWindowHeight;
	//第一次执行
    if (mFirst) {
		//....
    
   		//将窗口信息附加到 View 上。
        host.dispatchAttachedToWindow(mAttachInfo, 0);
		//....
    } else {
        desiredWindowWidth = frame.width();
        desiredWindowHeight = frame.height();
        if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
            if (DEBUG_ORIENTATION) Log.v(mTag, "View " + host + " resized to: " + frame);
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            windowSizeMayChange = true;
        }
    }

	//开始进行布局准备
    boolean windowShouldResize = layoutRequested && windowSizeMayChange
        && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
            || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.width() < desiredWindowWidth && frame.width() != mWidth)
            || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.height() < desiredWindowHeight && frame.height() != mHeight));
    windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;
    windowShouldResize |= mActivityRelaunched;
    final boolean computesInternalInsets =
            mAttachInfo.mTreeObserver.hasComputeInternalInsetsListeners()
            || mAttachInfo.mHasNonEmptyGivenInternalInsets;

    if (mFirst || windowShouldResize || insetsChanged ||
            viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        mForceNextWindowRelayout = false;
		//....
        if (!mStopped || mReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                    (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                    || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                    updatedConfiguration) {
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        
                if (measureAgain) {
					//View 的测量
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }

                layoutRequested = true;
            }
        }
    }
	//..........
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    boolean triggerGlobalLayoutListener = didLayout
            || mAttachInfo.mRecomputeGlobalAttributes;
    if (didLayout) {
        // View 的布局
        performLayout(lp, mWidth, mHeight);
		//.....
    }

	//......
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    if (!cancelDraw) {
      	// View 的绘制
        performDraw();
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }
    }

    mIsInTraversal = false;
}
```

### View 的测量

`ViewRootImpl` 调用 `performMeasure` 执行 Window 对应的 View 测量。

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

在 `performMeasure` 方法中就会执行 View 的 measure 方法，在其中会计算一下约束信息，然后就会调用 onMeasure 方法，

### View 的布局

`ViewRootImpl` 调用 `performLayout` 执行 Window 对应 View 的布局

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    mScrollMayChange = true;
    mInLayout = true;


    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
    try {
        // 调用 layout 方法进行布局
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

        mInLayout = false;
        int numViewsRequestingLayout = mLayoutRequesters.size();
        if (numViewsRequestingLayout > 0) {
      
            ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                    false);
            if (validLayoutRequesters != null) {
   
                mHandlingLayoutInLayoutRequest = true;

                // Process fresh layout requests, then measure and layout
                int numValidRequests = validLayoutRequesters.size();
                for (int i = 0; i < numValidRequests; ++i) {
                    final View view = validLayoutRequesters.get(i);
                    Log.w("View", "requestLayout() improperly called by " + view +
                            " during layout: running second layout pass");
                    // 请求对该 View 进行布局，该操作会导致此 View 被重新测量，布局，绘制
                    view.requestLayout();
                }
                measureHierarchy(host, lp, mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);
                mInLayout = true;
                host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                mHandlingLayoutInLayoutRequest = false;

                // Check the valid requests again, this time without checking/clearing the
                // layout flags, since requests happening during the second pass get noop'd
                validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                if (validLayoutRequesters != null) {
                    final ArrayList<View> finalRequesters = validLayoutRequesters;
                    // Post second-pass requests to the next frame
                    getRunQueue().post(new Runnable() {
                        @Override
                        public void run() {
                            int numValidRequests = finalRequesters.size();
                            for (int i = 0; i < numValidRequests; ++i) {
                                final View view = finalRequesters.get(i);
                                // 请求对该 View 进行布局，该操作会导致此 View 被重新测量，布局，绘制
                                view.requestLayout();
                            }
                        }
                    });
                }
            }

        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    mInLayout = false;
}
```

通过上面代码可以看出，在 layout 布局期间，有可能会对 View 进行 `requestLayout` 重新进行测量，布局，绘制。

### View 的绘制

`ViewRootImpl` 通过调用 `performDraw` 执行 Window 对应 View 的绘制。

```java
private void performDraw() {
    if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
        return;
    } else if (mView == null) {
        return;
    }

    final boolean fullRedrawNeeded =
            mFullRedrawNeeded || mReportNextDraw || mNextDrawUseBlastSync;
    mFullRedrawNeeded = false;

    mIsDrawing = true;
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");

    boolean usingAsyncReport = addFrameCompleteCallbackIfNeeded();
    addFrameCallbackIfNeeded();

    try {
        //绘制
        boolean canUseAsync = draw(fullRedrawNeeded);
        if (usingAsyncReport && !canUseAsync) {
            mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
            usingAsyncReport = false;
        }
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
	//...
}
```

```java
private boolean draw(boolean fullRedrawNeeded) {
    //........
 
	//通知 View 上注册的绘制事件
    mAttachInfo.mTreeObserver.dispatchOnDraw();

    boolean useAsyncReport = false;
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty || mNextDrawUseBlastSync) {
        if (isHardwareEnabled()) {
          	//调用 Window 对应 View 的 invalidate 方法
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
          	// 绘制 View
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                    scalingRequired, dirty, surfaceInsets)) {
                return false;
            }
        }
    }

    if (animating) {
        mFullRedrawNeeded = true;
        scheduleTraversals();
    }
    return useAsyncReport;
}
void invalidate() {
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        scheduleTraversals();
    }
}
```

```java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty, Rect surfaceInsets) {

    // Draw with software renderer.
    final Canvas canvas;
    try {
        canvas = mSurface.lockCanvas(dirty);
        // TODO: Do this in native
        canvas.setDensity(mDensity);
    } 

    try {
        if (DEBUG_ORIENTATION || DEBUG_DRAW) {
            Log.v(mTag, "Surface " + surface + " drawing to bitmap w="
                    + canvas.getWidth() + ", h=" + canvas.getHeight());
            //canvas.drawARGB(255, 255, 0, 0);
        }
		// View 绘制
        mView.draw(canvas);

        drawAccessibilityFocusedDrawableIfNeeded(canvas);
    } finally {
        
    }
    return true;
}
```

### 参考资料

> Android 开发艺术探索
>
> [ViewRootImpl的独白](https://blog.csdn.net/stven_king/article/details/78775166)

#### 推荐阅读

- Android | 理解 Window 和 WindowManager



> 如果本文对你有帮助，请点赞支持，谢谢！如果有任何问题，可直接在下方评论，谢谢!

