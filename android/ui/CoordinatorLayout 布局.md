### 前言

CoordinatorLayout 翻译过来就是协调者布局，用于实现自定义的嵌套滑动效果， 主要的特征就是让 childView 来协调使用，比如在移动 A View 的时候让 B 和 C View 发生改变，**主要作用就是管理子 View 之间的联动交互。**事实上，它本身并没要太大的作用，重要的是来配合 behavior 来使用。

### 关键类

#### CoordinatorLayout.Behavior

behavior 意为行为，主要的表现形式就是配合使用，该行为实现了可以对子视图进行一个或者多个的交互，包括拖动，滑动，甩动等其他手势。例如 A 在移动的过程中 B 想跟着 A 一起移动，就需要在 XML 中给 B 添加 behavior 属性。

```
public static abstract class Behavior<V extends View> {
    
    //当解析 layout 完成时候调用 View.onAttacheToWindow()，紧接着调用此方法
    public void onAttachedToLayoutParams(@NonNull CoordinatorLayout.LayoutParams params)
    
    //销毁时调用
    public void onDetachedFromLayoutParams() 

    //当 CoordinatorLayout.onInterceptTouchEvent 事件的时候调用 
    public boolean onInterceptTouchEvent(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull MotionEvent ev)

    //在 CoordinatorLayout.onTouchEvent 事件后调用
    public boolean onTouchEvent(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull MotionEvent ev)

    //设置背景色 
    @ColorInt
    public int getScrimColor(@NonNull CoordinatorLayout parent, @NonNull V child)

    //设置不透明度
    @FloatRange(from = 0, to = 1)
    public float getScrimOpacity(@NonNull CoordinatorLayout parent, @NonNull V child)

    /**
     * 
     */
    public boolean blocksInteractionBelow(@NonNull CoordinatorLayout parent, @NonNull V child) {
        return getScrimOpacity(parent, child) > 0.f;
    }

    /**
     * 需要依赖的 View
     * @param child 当前 View
     * @param dependency 需要依赖的 View
     */
    public boolean layoutDependsOn(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull View dependency)

    //依赖的 View 在发生变化的时候调用
    
    public boolean onDependentViewChanged(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull View dependency)

    //依赖的 View 移除的时候调用
    public void onDependentViewRemoved(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull View dependency) 

    //CoordinatorLayout.onMeasureChild 的时候调用
    public boolean onMeasureChild(@NonNull CoordinatorLayout parent, @NonNull V child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed)


    //CoordinatorLayout.layout 的时候调用
    public boolean onLayoutChild(@NonNull CoordinatorLayout parent, @NonNull V child,
            int layoutDirection) 
            
    //estedScrollingChild.startNestedScroll 时调用
    @Deprecated
    public boolean onStartNestedScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View directTargetChild, @NonNull View target,
            @ScrollAxis int axes) 

    //NestedScrollingChild2.startNestedScroll 时调用
    public boolean onStartNestedScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View directTargetChild, @NonNull View target,
            @ScrollAxis int axes, @NestedScrollType int type)

    //NestedScrollingChild.startNestedScroll = true 时调用
    @Deprecated
    public void onNestedScrollAccepted(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View directTargetChild, @NonNull View target,
            @ScrollAxis int axes)

    //NestedScrollingChild2.startNestedScroll() = true的时候调用
    public void onNestedScrollAccepted(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View directTargetChild, @NonNull View target,
            @ScrollAxis int axes, @NestedScrollType int type)

    //NestedScrollingChild.stopNestedScroll() 调用的时候执行
    @Deprecated
    public void onStopNestedScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target) {
        // Do nothing
    }

    //NestedScrollingChild2.stopNestedScroll() 调用的时候执行
    public void onStopNestedScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, @NestedScrollType int type)

    //NestedScrollingChild.dispatchNestedScroll() 调用的时候执行
    @Deprecated
    public void onNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull V child,
            @NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed,
            int dyUnconsumed)
            
    //NestedScrollingChild2.dispatchNestedScroll() 调用的时候执行
    @Deprecated
    public void onNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull V child,
            @NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed,
            int dyUnconsumed, @NestedScrollType int type)

    //NestedScrollingChild3.dispatchNestedScroll() 调用的时候执行
    public void onNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull V child,
            @NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed,
            int dyUnconsumed, @NestedScrollType int type, @NonNull int[] consumed)

    //NestedScrollingChild.dispatchNestedPreScroll() 调用的时候执行
    @Deprecated
    public void onNestedPreScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, int dx, int dy, @NonNull int[] consumed) {
        // Do nothing
    }

    //NestedScrollingChild2.dispatchNestedPreScroll() 调用的时候执行
    public void onNestedPreScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, int dx, int dy, @NonNull int[] consumed,
            @NestedScrollType int type)
    }

    //NestedScrollingChild.dispatchNestedFling() 调用的时候执行
    public boolean onNestedFling(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, float velocityX, float velocityY,
            boolean consumed) {
        return false;
    }

    /**
     * NestedScrollingChild2.dispatchNestedFling() 调用的时候执行
     */
    public boolean onNestedPreFling(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, float velocityX, float velocityY) {
        return false;
    }

    /**
     * 恢复状态
     */
    public void onRestoreInstanceState(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull Parcelable state) {
        // no-op
    }

    /**
     * 保存状态
     */
    @Nullable
    public Parcelable onSaveInstanceState(@NonNull CoordinatorLayout parent, @NonNull V child) {
        return BaseSavedState.EMPTY_STATE;
    }
}
```

