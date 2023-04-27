### 前言

相信绝大部分人都使用过 `view.post`这个方法，且使用场景基本上都是用来获取 `view` 的一些属性数据，并且我们也都知道，该方法会使用 `handler` 发送一个消息，并且该消息被回调执行的时候 `view 是已经绘制完成的`，今天我们来聊一聊它内部的一些细节。

### View.post

```java
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }
    getRunQueue().post(action);
    return true;
}
```

代码看起来非常清楚明了，主要可以分为两部分

- 如果 `attachInfo` 不为 null ，则直接获取它的 `handler` 将 `action` 发送出去
- 否则就调用 getRunQueue.post ，并传入 `action`，看名字好像是一个可运行的队列

下面我们来分别看一下这两者都干了什么

### AttachInfo

```java
/**
 * 将视图附加到其父视图时提供的一组信息 窗口
 */
final static class AttachInfo {
		//......
    @UnsupportedAppUsage
    final Handler mHandler;

    AttachInfo(IWindowSession session, IWindow window, Display display,
            ViewRootImpl viewRootImpl, Handler handler, Callbacks effectPlayer,
            Context context) {
        mSession = session;
        mWindow = window;
        mWindowToken = window.asBinder();
        mDisplay = display;
        mViewRootImpl = viewRootImpl;
        mHandler = handler;
        mRootCallbacks = effectPlayer;
        mTreeObserver = new ViewTreeObserver(context);
    }
}
```

根据该类的注释信息可以看出来这个类是用来保存窗口信息的，并且熟悉 `View` 添加流程的同学应该清楚，该类是在 `WindowManager.addView ` 中创建 ViewRootImpl 的时候在 `ViewRootImpl` 的构造方法中创建的：

```java
public ViewRootImpl(@UiContext Context context, Display display, IWindowSession session,
        boolean useSfChoreographer) {
		//.....
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
            context);

}
```

ViewRootImpl 是所有 View 的顶层，测量，布局绘制都是从该类中开始的。

WindowManager 创建完 ViewRootImpl 后会调用他的 setView 方法

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
    synchronized (this) {
        if (mView == null) {
						//......
            mAttachInfo.mDisplayState = mDisplay.getState();
            mSoftInputMode = attrs.softInputMode;
            mWindowAttributesChanged = true;
            mAttachInfo.mRootView = view;
            mAttachInfo.mScalingRequired = mTranslator != null;
            mAttachInfo.mApplicationScale =
                    mTranslator == null ? 1.0f : mTranslator.applicationScale;
            if (panelParentView != null) {
                mAttachInfo.mPanelParentWindowToken
                        = panelParentView.getApplicationWindowToken();
            }
            mAdded = true;
            int res; /* = WindowManagerImpl.ADD_OKAY; */

            // 请求布局
            requestLayout();
           
            try {
                mOrigWindowType = mWindowAttributes.type;
                //调用 WMS 添加窗口
                res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), userId,
                        mInsetsController.getRequestedVisibilities(), inputChannel, mTempInsets,
                        mTempControls);    
            } catch (RemoteException e) {
        }
    }
}
```

上面代码中先是对 mAttachInfo 进行各种赋值操作，接着 `requestLayout` ，View 的测量绘制布局都是从该方法中开始的，最后调用系统服务添加窗口，我们需要关心的就是 `requestLayout`

```
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();//检测线程
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
       //发送同步屏障,立即执行 mTraversalRunnable 任务 
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

最终会调用到 `doTraversal` 方法中：

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        //开始执行view的绘制流程
        performTraversals();
    }
}
```

```java
private void performTraversals() {
    // cache mView since it is used so much below...
    final View host = mView;

    if (mFirst) {
    		//调用 view 该方法
        host.dispatchAttachedToWindow(mAttachInfo, 0);
    }
    //.........
}
```

```java
#View
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info; 进行赋值
		//...
}
```

通过上面可以看出来最终 `mAttachInfo` 的的赋值是在 `performTraversals` 方法中调用完成的，该方法中也进行了测量布局绘制等操作，如果仔细看源码就会发现 `dispatchAttachedToWindow` 是在测量等操作之前执行的，那为什么 View.post 中还能获取到 View 的宽高等属性呢？

其实这个问题也不是特别难，因为 `performTraversals` 方法也是通过 handler 发送的，在执行 `mTraversalRunnable` 的时候才对 `mAttachInfo` 进行的赋值，然后再执行绘制流程，所以通过 `mAttachInfo`.handler  发送的消息肯定是在 `mTraversalRunnable` 之后执行的，这个时候绘制流程已经结束了，正因为如此，所以才可以获取到 View 的宽高等属性。

#### 小结一下

在 `mAttachInfo` 不为空的情况下会直接使用 handler 发送消息，为什么 `mAttacheInfo` 发送后就可以获取到各种属性数据，主要流程如下所示：

1. View 在创建出来后需要使用 WindowManager.addView 添加到屏幕上，期间会创建 View 的顶层类 ViewRootImpl
2. 在 ViewRootImpl 构造方法中回创建 `mAttachInfo` 
3. 在 ViewRootImpl.setView 中对 `mAttacheInfo` 添加各种数据，并调用 View 的绘制流程，设置同步屏障，使用 handler 发送绘制任务，使得该消息可以再第一时间执行
4. 在绘制流程的最开始的时候将 `mAttachInfo` 传递给 View，这样便是整个流程了
5. 等到 `View.post` 执行的时候，使用 `mattachInfo.handler` 发送的消息肯定会在 View 绘制的任务之后执行

> 如果你对 View 的添加流程和绘制流程不太熟悉，这里推荐两篇文章对你会有一点帮助
>
> [Android | 理解 Window 和 WindowManager](https://juejin.cn/post/7076274407416528909#heading-4) ：里面有 View 的添加流程等
>
> [Android | 理解 ViewRootImpl](https://juejin.cn/post/7077458342007799845) : View 的绘制流程等

### getRunQueue.post

通过 `View.post` 中的代码可以知道如果 `mAttachInfo` 为 null 就会执行 `getRunQueue().post()` 方法，下面我们来看一下这个方法：

```java
private HandlerActionQueue getRunQueue() {
    if (mRunQueue == null) {
        mRunQueue = new HandlerActionQueue();
    }
    return mRunQueue;
}
```

```java
/**
 * 当没有附加处理程序时，用于从视图中排队等待处理的类
 *
 * @hide Exposed for test framework only.
 */
public class HandlerActionQueue {
    private HandlerAction[] mActions;
    private int mCount;
		//......
    private static class HandlerAction {
        final Runnable action;
        final long delay;

        public HandlerAction(Runnable action, long delay) {
            this.action = action;
            this.delay = delay;
        }

        public boolean matches(Runnable otherAction) {
            return otherAction == null && action == null
                    || action != null && action.equals(otherAction);
        }
    }
}
```

该类也比较简单，主要就是对需要处理的任务进行排队等待，我们直接来看 post 方法

```java
public void post(Runnable action) {
    postDelayed(action, 0);
}

public void postDelayed(Runnable action, long delayMillis) {
    final HandlerAction handlerAction = new HandlerAction(action, delayMillis);

    synchronized (this) {
        if (mActions == null) {
            mActions = new HandlerAction[4];
        }
        mActions = GrowingArrayUtils.append(mActions, mCount, handlerAction);
        mCount++;
    }
}
```

```java
#GrowingArrayUtils.java
public static <T> T[] append(T[] array, int currentSize, T element) {
    if (currentSize + 1 > array.length) {
        T[] newArray = (T[]) Array.newInstance(array.getClass().getComponentType(),
                growSize(currentSize));
        System.arraycopy(array, 0, newArray, 0, currentSize);
        array = newArray;
    }
    array[currentSize] = element;
    return array;
}
public static int growSize(int currentSize) {
    return currentSize <= 4 ? 8 : currentSize * 2;
}
```

上面代码中创建了 `HandlerAction` 对象，并且保存到 `mActions` 数组中，默认数组大小等于 4，如果已经满了就会通过反射重新创建一个数组，并将数据迁移过去，每次创建数组的大小都是之前的两倍。

到这里添加到数组之后就没有别的操作了，此时我们需要推测一下这个数组中的任务会在何时被取出来然后在执行，通过上面的分析，我们大致就可以推断出来八成是在 `dispatchAttachedToWindow()` 方法中执行的，我们重新看一下这个方法：

```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
		//....
    // Transfer all pending runnables.
    if (mRunQueue != null) {
      	//传入 handler
        mRunQueue.executeActions(info.mHandler);
        mRunQueue = null;
    }
   //.....
}
```

果不其然，就是在该方法中执行的，在该方法中执行肯定就可以保证任务是在绘制流程之后执行的，我们继续跟进一下执行的方法：

```java
//
public void executeActions(Handler handler) {
    synchronized (this) {
        final HandlerAction[] actions = mActions;
        for (int i = 0, count = mCount; i < count; i++) {
            final HandlerAction handlerAction = actions[i];
            handler.postDelayed(handlerAction.action, handlerAction.delay);
        }

        mActions = null;
        mCount = 0;
    }
}
```

上面遍历数组，将任务取出使用 Handler 发送，最后清理资源就完事了。



### 总结一下

通过上面的分析，其实这个逻辑本身还是非常简单的，但是需要你提前了解 View 的添加流程以及绘制流程和Handler ，了解这些你再去看这个源码就会非常简单。

如果感觉本文对你有点帮助的话请点赞支持一下，多谢啦！