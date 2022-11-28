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

通过上篇文章和上面的分析，我们可以知道，在创建了 `WindowToken` 之后，会将 WindowToken 添加到 `DisplayConent` 中，当调用 addWindow 要添加 WindowState 的时候，会将 WindowState 添加到对应的层级。

### 层级的二次计算

通过上面的计算，只是大致确定了显示的顺序，但是实际上，还需要再进行一次计算，将每个窗口的主序以及他们窗口列表中重新计算最终显示的次序。还有一下情况需要特殊出列，例如输入框或者对话框都需要进行特殊的调整。

```kotlin
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
    } else if ((attrs.flags&FLAG_SHOW_WALLPAPER) != 0) {
        displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
    } else if (displayContent.mWallpaperController.isBelowWallpaperTarget(win)) {
        displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
    }
}
......
if (imMayMove) {
    displayContent.computeImeTarget(true /* updateImeTarget */);
}

// Don't do layout here, the window must call
// relayout to be displayed, so we'll do it there.
win.getParent().assignChildLayers();
.....
```

在最开始是针对各种类型的窗口做一个处理。接着就能看到两个核心的方法 `computeImeTarget()` 和 `assignChildLayers()`。

#### computeImeTarget

```java
WindowState computeImeTarget(boolean updateImeTarget) {
    //1 如果 WMS 中没有输入法窗口，
    if (mInputMethodWindow == null) {
        if (updateImeTarget) {
            setInputMethodTarget(null, mInputMethodTargetWaitingAnim);
        }
        return null;
    }

  
    final WindowState curTarget = mInputMethodTarget;
    //2是否应该计算新的IME目标，不需要直接返回 curTarget
    if (!canUpdateImeTarget()) {
        return curTarget;
    }

    mUpdateImeTarget = updateImeTarget;
    WindowState target = getWindow(mComputeImeTargetPredicate);
  	//3
    if (target != null && target.mAttrs.type == TYPE_APPLICATION_STARTING) {
        final AppWindowToken token = target.mAppToken;
        if (token != null) {
            final WindowState betterTarget = token.getImeTargetBelowWindow(target);
            if (betterTarget != null) {
                target = betterTarget;
            }
        }
    }
    //4
    if (curTarget != null && !curTarget.mRemoved && curTarget.isDisplayedLw()
            && curTarget.isClosing()) {
        return curTarget;
    }
    //5
    if (target == null) {
        if (updateImeTarget) {
            setInputMethodTarget(null, mInputMethodTargetWaitingAnim);
        }
        return null;
    }
    //6
    if (updateImeTarget) {
        AppWindowToken token = curTarget == null ? null : curTarget.mAppToken;
        if (token != null) {
            WindowState highestTarget = null;
            if (token.isSelfAnimating()) {
                highestTarget = token.getHighestAnimLayerWindow(curTarget);
            }

            if (highestTarget != null) {
                if (mAppTransition.isTransitionSet()) {
                    setInputMethodTarget(highestTarget, true);
                    return highestTarget;
                }
            }
        };
        setInputMethodTarget(target, false);
    }
    return target;
}
private void setInputMethodTarget(WindowState target, boolean targetWaitingAnim) {
  if (target == mInputMethodTarget && mInputMethodTargetWaitingAnim == targetWaitingAnim) {
    return;
  }
  mInputMethodTarget = target;
  mInputMethodTargetWaitingAnim = targetWaitingAnim;
  assignWindowLayers(false /* setLayoutNeeded */);
  mInsetsStateController.onImeTargetChanged(target);
  updateImeParent();
}
void assignWindowLayers(boolean setLayoutNeeded) {
  assignChildLayers(getPendingTransaction());
  if (setLayoutNeeded) {
    setLayoutNeeded();
  }
  scheduleAnimation();
}
```

在上面代码中，curTarget 是 WMS 的 mInputMethodTarget，也就意味着是 WMS 预定的输入法岑寂，代表当前输入法窗口。 target 是 DisplayContent 对自己的孩子进行搜索，找到下一个顶部的输入法窗口 WIndow。

1. 最开始判断 WMS 中没有 输入法(IMEWindow)，此时还没有获取 DisplayContent 中的输入法窗口，所以直接使用 `mInputMethodTargetWaitingAnim` 作为新输入法的窗口动画。
2. 判断是否需要计算新的 IME，不需要知己返回 WMS 中的窗口
3. 接着通过 getWindow 找到顶层能够成为输入法弹窗的 DisplayContent 窗口，在 `getImeTargetBelowWindow` 中遍历，找到窗口。
4. 一个特殊的情况-如果最后一个目标的窗口正在退出的过程中，但没有被移除，请保持在最后一个目标上，以避免IME闪烁。
5. 目标输入法为空，则调用 `setInputMethodTarget`
6. 输入法动画播放，根据 `updateImeTarget` 来判断是否要处理动画 ，因为是需要做动画，所以需要找到当前输入法的弹框下，最高层级的可见窗口。

这个方法中好多地方都调用了另一个比较核心的方法 `setInputMethodTarget` 去设置当前输入法的弹窗目标。然后调用 ` assignWindowLayers` 执行窗口动画。

#### assignChildLayers

```java
void assignChildLayers(Transaction t) {
    int layer = 0;

    // We use two passes as a way to promote children which
    // need Z-boosting to the end of the list.
    for (int j = 0; j < mChildren.size(); ++j) {
        final WindowContainer wc = mChildren.get(j);
        wc.assignChildLayers(t);
        if (!wc.needsZBoost()) {
            wc.assignLayer(t, layer++);
        }
    }
    for (int j = 0; j < mChildren.size(); ++j) {
        final WindowContainer wc = mChildren.get(j);
        if (wc.needsZBoost()) {
            wc.assignLayer(t, layer++);
        }
    }
}
```

首先对整个 WindowContainer 的子窗体做了一次调整，接着打开了 neesdZBoost 标志位的窗口在添加到上面。这样就分离了需要做动画的层级，以及普通的层级，保证了做动画的窗口一定要在普通窗口之上。

到这里窗口的显示层级就已经计算的差不多了，在 addWindow 中也没有其他的和显示次序相关的代码了，那么窗口如何布局并且展示出来的呢，接下来我们来看一下窗口的布局

### 窗口的布局