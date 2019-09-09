

View 的主要工作流程 是指 measure，layout，draw 这三大过程，即布局，测量，和绘制。measure 确定 View 的测量宽高，layout 确定 View 的最终 宽 / 高 和四个顶点的位置，二draw 则将View 绘制到屏幕上。

measure 过程：

​	如果是一个原始的View ，那么通过 measure 方法就完成了 测量的过程，如果是一个 ViewGroup ，除了 完成自己的测量意外，还会去遍历调用所有的子元素的 measure 方法，各个子元素 再去递归执行这个过程。下面针对两种情况分别进行讨论：

​	1，View 的 measure 过程

​	View 的 measure 的过程 由其measure 方法来完成，measure 方法是一个final 类型的方法，这意味着子类不能重写 此方法。在View 的 measure 方法中 会去调用 View 的onMeasure 方法。因此只需要看 onMeasure 的实现即可，View 的 onMeasure 方法如下 所示：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

setMeasuredDimensionrt 方法会设置 View 的宽和高 的测量值,因此我们朱啊哟看getDefaultSize 这个方法 即可。

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

这个 方法中的 逻辑非常简单，对于我们来说，只需要 看 AT_MOST,EXACTLY 这两种 情况。简单的理解一下，其实 getDefaultSize 返回的大小 就是  measureSpec 中的 specSize, 而这个specSize 就是View 测量后的大小。这里多次提到测量后的大小，是因为 最终 确定的大小 还是在 layout 阶段 确定的。所以这里必须加上区分。但是机会所有情况下View 的测量 大小 和最终 大小 是相等的。

至于UNSPECIFED 这种情况，一般用于系统内部的测量过程。在 这种情况下，View 的大小 为 getDefaultSize 的第一个参数 size。即宽和高 分别为：getSuggestedMinimumWidth / getSuggestedMinimumHeight 这两个方法的 返回值，看一下源码：

```
 protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }

protected int getSuggestedMinimumHeight() {
    return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
}
```

从 getSuggestedMinimumWidth 来看，如果 View 没有设置背景，那么他的宽度为 mMinWidth ，而mMinWidth 对应于 android minWidth： 这个属性的值，因此 View 的宽度即为 minWidth 属性所指定 的值。如果这个值不指定，那么minWidth 的值默认为 0 ，如果 View 指定了 背景，则View 的宽度 为 max(mMinWidth, mBackground.getMinimumWidth()。mMinWidth 的含义我们知道了，那么 mBackground.getMinmumWidth() 是什么呢。我们看一下Drawable 的 getMinimumWidth 方法，如下所示：

```java
public int getMinimumWidth() {
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
```

可以看出 getMinimumWidth 返回的就就是Drawable 的原始高度，前提是有原始高度，否则就返回 0 。



这里总结一下 getSuggestedMinimumWidth 的逻辑：

​	如果View 没有设置 背景，则返回 android:minWidth 这个属性所指定的大小，这个值可以为0。如果View设置了背景，  则返回 mMinWidth 和 mBackground.getMinimumWidth() 这两者的最大值。getSuggestedMinimumWidth() 和 getSuggestedMinimumHeight() 的返回值 就是 View 在 UNSPECIFIEED 情况下的测量的 宽 和 高。



从 getDefaultSize 方法的实现来看，View 的宽 / 高 有specSize 来确定，所以我们可以得出如下结论：直接继承 View 的自定义 控件 需要重写 onMeasure 方法并设置 wrap_content 时 的自身大小，否则在布局中 使用 wrap_content 就相当于 使用 match_parent 。为什么呢？。从上述代码我们知道。如果 View 在布局中使用 wrap_content ，那么他的 specMode 是AT_MOST 模式，在这种 模式下 ，他的宽高 等于 等于 specSize。