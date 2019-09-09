Android 里面的绘制都是按 顺序的，先绘制的内容会被后绘制的盖住。比如你先画一个 圆，在画一个方块，这个圆就会被盖住。

1,super.onDraw() 前 或者 后？

​	一般我们自定义绘制 ，全部都是直接继承 View 类，然后重写他的 onDraw() 方法，把绘制的代码写在里面

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

   ......绘制内容
 }
```

​	这是自定义绘制的基本状态，继承View ，在onDraw 中完成自定义绘制。
	其实绘制代码写在 super.onDraw 的上面或者下面 都无所谓，甚至可以将这句话删掉。效果都是一样的，因为在 View 的 onDraw 中 是空实现。

```java
/**
     * Implement this to do your drawing.
     *
     * @param canvas the canvas on which the background will be drawn
     */
    protected void onDraw(Canvas canvas) {
    }
```

2,写在 super.onDraw 下面

​	如果你继承的 不是View ，而是一个ImageView 。这时由于绘制代码绘制原有内容绘制结束后执行，所以就会覆盖原来控件的内容。

3，写在 super.onDraw 上面

​	如果继承的是一个 TextView ，将你需要绘制的东西 写在 super.onDraw 上面，这样绘制的内容就会被控件原有的内容覆盖掉。

​	比如：你可以在文字的下面绘制一个 纯色的背景。

4，dispathDraw(): 绘制字View 的方法

​	如果你继承了一个 LinearLayout ，重写了 onDraw , 然后在 onDraw 中绘制了一些内容，但是 运行程序后会发现 绘制的内容看不见。

​	我学的时候 看到这里也懵了。我没有添加子View 呀。为啥会被覆盖呢。别急，文章的最后面会解答你的疑惑。

​	解决：只有让他在绘制子View 之后执行就好了，重写 dispatchDraw() 并在 super.dispatchDraw() 的下面写上自己的绘制代码，就可以了

```java
 @Override
    protected void dispatchDraw(Canvas canvas) {
        super.dispatchDraw(canvas);
        paint.setColor(Color.parseColor("#ED6F99"));
        paint.setStyle(Paint.Style.FILL);

        canvas.drawCircle(100,100,30,paint);
        canvas.drawCircle(300,300,50,paint);
        canvas.drawCircle(300,500,80,paint);
        canvas.drawCircle(700,300,20,paint);
    }
```

5,绘制过程：

​	一般来说，一个完整的绘制过程会依次绘制一下几个内容：

​	1，背景

​	2，主体（onDraw）

​	3，子View (dispatchDraw)

​	4，滑动边缘渐变 和 滑动条

​	5，前景

​	一般来说，一个View 或者 ViewGroup 的绘制不会将这几项 全部包括，但是必然 逃不出这几项，并且会 遵守这个 顺序。例如一个 LinearLayout 只有背景 和 子View ，那么他会先绘制 背景在绘制子 View 。一个ImageView 有主题，有可能会加上一层半透明的前景 作为 遮罩，那么他的前景 也会在主体之后进行绘制。注意：前景 是在 android6.0 之后加入的，之前其实也有，只不过只支持FrameLayout。直到 6.0 才加入到了 View 里面。

​	第一步背景 ，他的绘制在一个叫 drawBackground() 的方法中，但是这个方法是 private 的，不能重写。如果需要设置背景，只能通过 API 去设置。

​	第二步 绘制主体，在onDraw()中 绘制

​	第三步 绘制子View ，在dispatchDraw 中绘制

​	第四 、 五 两步：这两个部分被放在了 onDrawForeground 方法里，是可以重新写的。

​	![1559356547066](F:\笔记\android\自定义View 系列\assets\1559356547066.png)

6，onDrawForeground

​	首先，再说一遍，这个方法是 API 23 才引入的，所以在重写这个方法的时候要确认你的 `minSdk` 达到了 23，不然低版本的手机装上你的软件会没有效果。 

7，draw 总调度方法

​	draw 是绘制的总调度方法，一个View 的绘制过程都发生在 draw 方法里，如下所示：

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
    int saveCount;

    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        drawAutofilledHighlight(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);

        if (debugDraw()) {
            debugDrawFocus(canvas);
        }

        // we're done...
        return;
    }	
```

​	从上面可以看出首先 是绘制背景。然后就是 onDraw ,dispatchDraw , ondrawForeground , 这三个方法被依次调用。所以你也可以重写 这个 draw 放入 来做自定义的调度。

![1559357019285](F:\笔记\android\自定义View 系列\assets\1559357019285.png)

8，写在 super.draw 的下面

​	由于 draw 是总调度方法，如果绘制代码写在下面，那么这段代码将会在 其他所有的 绘制完成后执行。也就是说，他绘制的内容会盖住 其他原有的 内容。

9，写在 super.draw 上面

​	如果写在上面，这段代码会在 其他所有绘制之前执行。包括背景，背景也会 盖住他。



注意：关于绘制，有两点需要注意，

​	1，出于效率的考虑，ViewGroup 默认会绕过 draw 方法，转而直接执行 dispatchDraw ，以此来简化 绘制流程。比如自定义了 一个ViewGroup 的子类。并且需要他在 dispatchDraw 以外的任何一个绘制方法内绘制内容，你需要调用 View.setWillNotDraw(false) 方法来切换到完整的绘制流程。

​	2，有的时候，一段绘制代码写在不同的绘制方法中效果是一样的，这时你可以选一个自己喜欢或者习惯的绘制方法来重写。但有一个例外：如果绘制代码既可以写在 `onDraw()` 里，也可以写在其他绘制方法里，那么优先写在 `onDraw()` ，因为 Android 有相关的优化，可以在不需要重绘的时候自动跳过 `onDraw()` 的重复执行，以提升开发效率。享受这种优化的只有 `onDraw()` 一个方法 

总结：

![1559357683483](F:\笔记\android\自定义View 系列\assets\1559357683483.png)



参考自：<https://hencoder.com/ui-1-5/> 