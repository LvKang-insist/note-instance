### Bitmap 是什么？

翻译过来就是 位图。位图就是一个图片完整的像素数据。

例如一个纯蓝色的图片是 640*640 的，那么对应的就是 0000ff ，该图片在保存的时候可以进行一些优化。但是如果要显示出来，那么他就是个 640 * 640  个 0000ff。

**bitmap 会把每个像素的颜色信息全部存储起来。Bitmap 中并没有别的什么信息，只有所有的像素信息和尺寸信息**

### Drawable 是什么？

Drawable 事实上并不是一个图。Drawable 确切点来说就像是一个抽象版的且内容更加专注的 View。相比于普通的 View ，Drawable 只负责绘制。

```kotlin
class DrawableView(context: Context?, attrs: AttributeSet?) : View(context, attrs) {
    val drawable: Drawable

    init {
        drawable = ColorDrawable(Color.RED)
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        //设置 drawable 的四个边
        drawable.setBounds(0, 0, width, height)
        drawable.draw(canvas)
    }
}
```

上面创建了一个 ColorDrawable。并且设置了四个边。然后将自己的内容会在 canvas 上面。

一般情况下，我们的理解都是 canvas.draw(xxxx)。那为什么这里是 drawable.draw(xxx) 呢？

Drawable 其实就是一个小型且更加专注的 View，这个 View 只负责绘制。

**Drawable 并不是一个被绘制的对象**。像 Bitmap 等都是被 Canvas 绘制的对象。而 Drawable 更像是一个 View，他是拿着这个 canvas 进行绘制，他持有这个 canvas，他想怎么绘制就按照自己的规则绘制即可。

**Drawable 内部存储的是一个绘制的规则，这个规则可以是一个 bitmap，可以是一个 color，甚至可以是一个抽象的灵活的描述。**

### Bitmap 和 Drawable 的相互转换

按照上面的概念，bitmap 和 drawable 根本就不是一个东西，他们之间就不存在转这一个说法，因为一个完整的像素信息怎么可能被转换成一个绘制规则呢！

那么怎么转呢，我只能把 Bitmap 包在一个 Drawable 里面，然后这个 Drawable 绘制规则就是把这个 bitmap 原封原样的给绘制出来。你只能转换到这个程度了，如下：

```kotlin
val bitmap: Bitmap = xxx
val bitmapDrawable = BitmapDrawable(resources, bitmap)
```

如果要将 drawable 转为 bitmap 呢？

```kotlin
private void drawableToBitamp(Drawable drawable){
    int w = drawable.getIntrinsicWidth();
    int h = drawable.getIntrinsicHeight();
    System.out.println("Drawable转Bitmap");
    Bitmap.Config config =  drawable.getOpacity() != PixelFormat.OPAQUE ? Bitmap.Config.ARGB_8888
    : Bitmap.Config.RGB_565;
    bitmap = Bitmap.createBitmap(w,h,config);
    //注意，下面三行代码要用到，否在在View或者surfaceview里的canvas.drawBitmap会看不到图
    Canvas canvas = new Canvas(bitmap);   
    drawable.setBounds(0, 0, w, h);   
    drawable.draw(canvas);
}
```

根据 drawable 的宽高创建一个 bitmap。然后根据 bitmap 创建一个 canvas。最后使用 draw 方法进行绘制即可。

根据这个过程也可以看出来，这个转换实际上就是进行了一次绘制。

### 自定义 Drawable

``` kotlin
class MeshDrawable : Drawable() {

    val paint = Paint(Paint.ANTI_ALIAS_FLAG)

    /**
     * 间隔
     */
    val interval = dp2px(80f).toInt()

    init {
        paint.color = Color.RED
        paint.strokeWidth = dp2px(3f)
    }

    override fun draw(canvas: Canvas) {
        var i = 0
        while (i < bounds.right) {
            i += interval
            var j = 0
            while (j < bounds.bottom) {
                j += interval
                canvas.drawLine(
                    bounds.left.toFloat(), j.toFloat(),
                    bounds.right.toFloat(), j.toFloat(), paint
                )
                canvas.drawLine(
                    i.toFloat(), bounds.top.toFloat(),
                    i.toFloat(), bounds.bottom.toFloat(), paint
                )
            }
        }
    }

    /**
     * 设置透明度
     */
    override fun setAlpha(alpha: Int) {
        paint.alpha = alpha
    }

    override fun getAlpha(): Int {
        return paint.alpha
    }

    /**
     * 设置颜色过滤
     */
    override fun setColorFilter(colorFilter: ColorFilter?) {
        paint.colorFilter = colorFilter
    }

    /**
     * 该图像是完全透明，还是部分透明，或者是完全不透明
     * TRANSPARENT：透明
     * OPAQUE:不透明   0xff:255
     * TRANSLUCENT：半透明
     */
    override fun getOpacity(): Int {
        if (paint.alpha == 0) {
            return PixelFormat.TRANSPARENT
        } else if (paint.alpha == 0xff) {
            return PixelFormat.OPAQUE
        } else {
            return PixelFormat.TRANSLUCENT
        }
    }
}
```

如上所示，即自定义 Drawable。

但是日常开发中会使用到自定义 Drawable 吗，实际上是基本不会使用。因为 Drawable 有很多子类，如 BitmapDrawable，ColorDrawable 等。所以我们一般情况下必要去自定义 Drawable；

如果你项目中自定义View 的时候用到了同一个 view，那么你就可以创建一个自定义的 Drawable。例如 股票图中的 蜡烛条，如果你自定义股票图，那么就需要使用很多个 蜡烛条，这个时候就可以使用 自定义 Drawable，在使用的时候直接用即可。

### 自定义 Bitmap

没有人会自定义 Bitmap 的，Bitmap 里面只有尺寸和像素信息，根本就没有啥需要自定义的地方。并且 Bitmap 是final 的，根本无法被继承。所以 bitmap 不可以自定义。

