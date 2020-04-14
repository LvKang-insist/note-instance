

### VeiwRoot 和 DecorView

​		ViewRoot 对应于 ViewRootImpl 类，他是连接 WindowManager 和 DecorView 的纽带，View 的三大流程都是通过 ViewRoot 来完成的。

​		在 ActivityThread 中，当 Activity 被创建完毕后，会将 DecorView 添加到 Window 中，同时会创建 ViewRootImpl 对象，并将 ViewRootImpl 和 DecorView 建立关联。

​		DecorView ：DecorView 是整个 Window 界面最顶层的 View

​		View 的绘制流程是从 ViewRoot 的 performTraversals 方法开始的，他经过 measure ，layout 和 draw三个过程后最终将一个 View 绘制出来。其中 measure 测量宽高，layout 用来确定 View 在父容器的位置， draw 赋值将 View 绘制到屏幕上。

​		![image-20200413142156876](1%EF%BC%8C%E7%AE%80%E5%8D%95%E7%9A%84%E8%87%AA%E5%AE%9A%E4%B9%89View.assets/image-20200413142156876.png)



### MeasureSpec

​		MeasureSpec 代表一个 32 位的 int 值，高 2为代表 SpecMode，低 30 为代表 SpecSize。

​		SpecMode ：测量模式

​		SpecSize：值在某种测量模式下的规格大小

​		MeasureSpec 将 SpecMode 和 SpecSize 打包成一个 int 值来避免过多的对象内存分配，方法方便操作，其提供了打包 和 解包的方法。这两个都是 int 值，这两个值 就可以打包为一个 MeasureSpec。

​		SpecMode 有三大类，每一类都代表特殊含义，如下所示：

- UNSPECIFIED 

  父容器不对 View 有任何限制，要多大给多大，一般用于系统内部，表示一种测量状态

- EXACTLY 

  父容器检测出 View 所需要的精确值大小，这个时候 View 的最终大小就是 SpecSize 所指定的值，它对应与LayoutParams 中的 match_parent 和具体的数值这两种模式。 

- AT_MOST 

  父容器 指定了一个可用的大小，接 SpecSize，View 的大小不能大于这个值，具体是什么值需要看 View 的具体实现，它对应于 LayoutParams 中的 wrap_content。

#### MeasureSpec 和 LayoutParams 对应关系

​		系统内部使用的是 MeasureSpec 来进程测量，一般情况下我们使用的是 LayoutParams。在 View 测量的时候，系统会将 LayoutParams 在父容器的约束下转换成对应的 MeasureSpec，然后来确定 View 的宽高。MeasureSpec 并不是唯一由 LayoutParams 决定的，LayoutParams 需啊哟和父容器一起才能决定 View 的 MeasureSpec，进而决定 View 的宽高。

​		对于顶级View(即 DecorView) 和普通 View 来说，MeasureSpec 的装过程略有不同，对于 DecorView，器 MeasureSpec 由窗口尺寸和自身的 LayoutParams 来共同决定。对用普通 View，则是由 父容器的 MeasureSpec 和自身的 LayoutParams 来决定。MeasureSpec 一旦确定后， onMeasure 中就可以确定 View 的测试宽/高。

- LayoutParams.MATCH_RARENT：精确模式，大小是窗口大小
- LayoutParams.WRAP_CONTENT：最大模式，大小不确定，但是不能超过窗口的大小

### View 的工作流程

​	View 的主要工作流程 是指 measure，layout，draw 这三大过程，即布局，测量，和绘制。measure 确定 View 的测量宽高，layout 确定 View 的最终 宽 / 高 和四个顶点的位置，二draw 则将View 绘制到屏幕上。

### measure 过程：

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

### layout 过程：

Layout 的作用是 ViweGroup 用来确定子元素的位置，当 ViewGroup 的位置确定后，它在 onLayout 中就会变了所有的子元素并调用其 layout 方法，在 layout 方法中 onLayout 又会被调用。layout 确定 View 本身的位置，而 onLayout 则会确定所有子元素的位置。

```java
int oldL = mLeft;
int oldT = mTop;
int oldB = mBottom;
int oldR = mRight;

boolean changed = isLayoutModeOptical(mParent) ?
        setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
    onLayout(changed, l, t, r, b);
```

在 View 的 layout 方法中，首先会获取四个顶点的位置，顶点的值确定后，View 在父容器的位置也就确定了。接着就会调用 onLayout 方法，这个方法父容器确定子元素的位置。

ViewGroup onLayout ：

```java
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    if (changed && childCount > 0) {
        val childCount = childCount
        for (i in 0 until childCount) {
            val childView = getChildAt(i)
            //给每一个子控件在水平方向上布局
            childView.layout(
                i * childView.measuredWidth, 0,
                (i + 1) * childView.measuredWidth, childView.measuredHeight
            )
        }
        //获取左右边界
        leftBorder = getChildAt(0).left
        rightBorder = getChildAt(childCount - 1).right
    }
}
```

在 ViewGroup 的onLayout 中，获取到子View 的数量，然后调用 View 的 layout 方法，进行布局。

### draw过程

作用是将 View 绘制到屏幕上面，View 的绘制过程遵循如下几步：

1，绘制背景

2，绘制自己

3，绘制 children

4，绘制装饰