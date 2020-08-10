```
/**
 * dp 转像素
 */
fun dp2px(dp: Float): Float {
    return TypedValue.applyDimension(
        TypedValue.COMPLEX_UNIT_DIP,
        dp,
        Resources.getSystem().displayMetrics
    )
}

fun getAvatar(resources: Resources, width: Int): Bitmap {
    val options = BitmapFactory.Options()
    //设置 true，就只会取到宽高
    options.inJustDecodeBounds = true
    //拿到宽高
    BitmapFactory.decodeResource(resources, R.drawable.abc, options)
    //使用宽高，重新获取图片，对性能有一定好处
    options.inJustDecodeBounds = false
    options.inDensity = options.outWidth
    options.inTargetDensity = width

    return BitmapFactory.decodeResource(resources, R.drawable.abc, options)
}

fun getZForCamera(): Float {
    return -8f * Resources.getSystem().displayMetrics.density
}
```

### 裁切

```kotlin
/**
 * 裁切
 */
class CustomView(context: Context?, attrs: AttributeSet?) : View(context, attrs) {

    val paint = Paint()

    val camera = Camera()

    var bitmapWidth: Int = dp2px(300f).toInt()

    var mLeft = 100f
    var mTop = 100f


    /**
     * 上面翻起的度数
     */
    var topFlip = 0f
        set(value) {
            field = value
            invalidate()
        }

    /**
     * 下面翻起的度数
     */
    var bottomFlip = 0f
        set(value) {
            field = value
            invalidate()
        }

    /**
     * 旋转角度
     */
    var flipRotation = 0f
        set(value) {
            field = value
            invalidate()
        }

    val bitmap = getAvatar(resources, bitmapWidth)

    init {
        camera.setLocation(0f, 0f, getZForCamera())
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)


        canvas.save()
        //上半部分
        canvas.translate((mLeft + (bitmapWidth / 2)), (mTop + (bitmapWidth / 2).toFloat()))
        canvas.rotate(-flipRotation)
        camera.save()
        camera.rotateX(topFlip)
        camera.applyToCanvas(canvas)
        camera.restore()
        //切割
        canvas.clipRect(-bitmapWidth, -bitmapWidth, bitmapWidth, 0)
        canvas.rotate(flipRotation)
        //移动到左上角
        canvas.translate(-(mLeft + (bitmapWidth / 2)), -(mTop + (bitmapWidth / 2)))
        //绘制
        canvas.drawBitmap(bitmap, mLeft, mTop, paint)
        canvas.restore()

        //下半部分
        canvas.translate((mLeft + (bitmapWidth / 2)), mTop + (bitmapWidth / 2).toFloat())
//        //旋转回来
        canvas.rotate(-flipRotation)

        camera.save()
        //X 轴旋转
        camera.rotateX(bottomFlip)
//        投影
        camera.applyToCanvas(canvas)
        camera.restore()
        //切割
        canvas.clipRect(-bitmapWidth, 0, bitmapWidth, bitmapWidth)
        //旋转
        canvas.rotate(flipRotation)
        //移动到左上角
        canvas.translate(
            -(mLeft + (bitmapWidth / 2).toFloat()),
            -(mTop + (bitmapWidth / 2).toFloat())
        )
        //绘制
        canvas.drawBitmap(bitmap, mLeft, mTop, paint)
    }
}
```

### 绘制刻度

```kotlin
/**
 * 绘制刻度
 */
class DashBoardView : View {

    /**
     * 开口弧度
     */
    val angle = 120

    /**
     * 半径
     */
    val radius = dp2px(150f)

    /**
     * 表圈宽度
     */
    val arcWidth = dp2px(5f)

    /**
     * 指针长度
     */
    val length = dp2px(100f)

    /**
     * 指针位置
     */
    val pointerPos = 5

    /**
     * 指针宽度
     */
    val pointerWidth = dp2px(4f)

    /**
     * 刻度数量
     */
    val scaleCount = 20

    /**
     * 刻度宽度
     */
    val scaleWidth = dp2px(2f)


    val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    val dash = Path()

    var dashPath: PathDashPathEffect? = null

    constructor(context: Context?) : super(context)
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)
    constructor(context: Context?, attrs: AttributeSet?, defStyleAttr: Int) : super(
        context,
        attrs,
        defStyleAttr
    )

    init {
        paint.style = Paint.Style.STROKE
        paint.strokeWidth = dp2px(2f)
        paint.strokeWidth = dp2px(8f)
        // dash 一个矩形，顺时针画
        dash.addRect(0f, 0f, dp2px(2f), dp2px(10f), Path.Direction.CW)

        //计算长度
        val arc = Path()
        arc.addArc(
            width / 2 - radius, height / 2 - radius,
            width / 2 + radius, height / 2 + radius,
            90 + angle / 2f, 360f - angle
        )
        val pathMeasure = PathMeasure(arc, false)
        val length = pathMeasure.length - dp2px(2f)
        /**
         * PathDashPathEffect：用指定的形状在路径上画横线
         * 1，指定的形状
         * 2，横线之间的距离
         * 3，距离第一个刻度空多少。
         */
        dashPath =
            PathDashPathEffect(dash, length / (scaleCount - 1), 0f, PathDashPathEffect.Style.ROTATE)
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        //设置弧度的宽度
        paint.strokeWidth = arcWidth
        //画弧度
        canvas.drawArc(
            width / 2 - radius, height / 2 - radius,
            width / 2 + radius, height / 2 + radius,
            90 + angle / 2f, 360f - angle,
            false, paint
        )

        //刻度的宽度
        paint.strokeWidth = scaleWidth
        //画刻度
        paint.pathEffect = dashPath
        //画弧度
        canvas.drawArc(
            width / 2 - radius, height / 2 - radius,
            width / 2 + radius, height / 2 + radius,
            90 + angle / 2f, 360f - angle,
            false, paint
        )
        paint.pathEffect = null

        // 指针的宽度
        paint.strokeWidth = pointerWidth
        //画指针
        canvas.drawLine(
            (width / 2).toFloat(), (height / 2).toFloat(),
            (Math.cos(Math.toRadians(getAngleFromMark(pointerPos - 1).toDouble())) * length).toFloat() + width / 2,
            (Math.sin(Math.toRadians(getAngleFromMark(pointerPos - 1).toDouble())) * length).toFloat() + height / 2,
            paint
        )
    }


    fun getAngleFromMark(mark: Int): Int {
        return (90 + angle.toFloat() / 2 + (360 - angle.toFloat()) / (scaleCount - 1) * mark).toInt()
    }
}
```

### 绘制头像

```kotlin
/**
 * 绘制头像
 */
class AvatarView : View {

    val paint = Paint()

    val padding = dp2px(50f)

    val xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_IN)

    val saveYser = RectF()

    var bitmap: Bitmap = getAvatar(resources, dp2px(400f).toInt())

    constructor(context: Context?) : super(context)
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)
    constructor(context: Context?, attrs: AttributeSet?, defStyleAttr: Int) : super(
        context,
        attrs,
        defStyleAttr
    )

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        saveYser.set(padding, padding, width - padding, width - padding)
    }


    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        //抠出一个 方块
        val saved = canvas.saveLayer(saveYser, paint)
        //在抠出的方块中画圆
        canvas.drawOval(padding, padding, width - padding, width - padding, paint)
        //保留覆盖的图，丢弃剩余的图
        paint.setXfermode(xfermode)
        //画个方形图片，结果就是保留这个圆，并且丢弃没有覆盖的地方
        canvas.drawBitmap(bitmap, padding, padding, paint)

        //恢复 xfermode，保证后面绘制不会有问题
        paint.setXfermode(null)
        // 将扣出来的方法放回去
        canvas.restoreToCount(saved)
    }


}
```

### 自定义 Drawable

```kotlin
/**
 * @name MeshDrawable
 * @package com.standalone.core.ui
 * @author 345 QQ:1831712732
 * @time 2020/7/8 21:08
 * @description 网眼 Drawable
 */
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

### 绘制多行文本

```kotlin
/**
 * 绘制多行文本
 */
class LinsTextView : View {


    val text = "Hilt 是 Android 的依赖注入库，是基于 Dagger 。可以说 Hilt 是专门为 Andorid 打造的。" +
            "Hilt 创建了一组标准的 组件和作用域。这些组件会自动集成到 Android 程序中的生命周期中。在使用的时候可以指定使用的范围，事情作用在对应的生命周期当中。" +
            "版权声明：本文为CSDN博主「345丶」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。" +
            "原文链接：https://blog.csdn.net/baidu_40389775/article/details/107095700"

    val paint = Paint(Paint.ANTI_ALIAS_FLAG)

    var bitmap: Bitmap

    val cutWith = floatArrayOf()

    constructor(context: Context?) : super(context)

    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)


    constructor(context: Context?, attrs: AttributeSet?, defStyleAttr: Int) : super(
        context,
        attrs,
        defStyleAttr
    )

    init {
        paint.textSize = dp2px(15f)
        bitmap = getAvatar(dp2px(100f))
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        canvas.drawBitmap(bitmap, width - dp2px(100f), 100f, paint)

        /**
         * 2，是否正向绘制
         * 3，View 的宽度
         * 4，拿到截取的宽度
         * return 第一行的位置
         */
        var index = paint.breakText(text, true, width.toFloat(), cutWith)
        //绘制第一行
        canvas.drawText(text, 0, index, 0f, 50f, paint)
        //绘制第二行
        var oldIndex = index
        index = paint.breakText(
            text, index, text.length, true,
            (width - bitmap.width).toFloat(), cutWith
        )
        canvas.drawText(text, oldIndex, oldIndex + index, 0f, (50 + paint.fontSpacing), paint)

        //绘制第三行
        oldIndex = index
        index = paint.breakText(
            text, oldIndex, text.length, true,
            (width - bitmap.width).toFloat(), cutWith
        )
        canvas.drawText(text, oldIndex, oldIndex + index, 0f, (50 + (paint.fontSpacing * 2)), paint)

    }


    fun getAvatar(width: Float): Bitmap {
        val options = BitmapFactory.Options()
        //设置 true，就只会取到宽高
        options.inJustDecodeBounds = true
        //拿到宽高
        BitmapFactory.decodeResource(resources, R.drawable.avatar, options)
        //使用宽高，重新获取图片，对性能有一定好处
        options.inJustDecodeBounds = false
        options.inDensity = options.outWidth
        options.inTargetDensity = width.toInt()
        return BitmapFactory.decodeResource(resources, R.drawable.avatar, options)
    }
}
```

### MaterialEditText

```kotlin
class MaterialEditText(context: Context?, attrs: AttributeSet?) :
    AppCompatEditText(context, attrs) {

    val paint = Paint(Paint.ANTI_ALIAS_FLAG)

    //文字高度
    private val mTextSize = dp2px(12f)

    //文字和输入框间距
    private val mTextMargin = dp2px(8f)

    //垂直偏移
    private val mTextVerticalOffset = dp2px(22f)

    //纵向偏移
    private val mTextHorizontalOffset = dp2px(5f)

    //文字动画偏移量
    private val mTextAnimationOffset = dp2px(16f)

    private var isFloatingLabel: Boolean = true

    val backgroundPadding = Rect()

    val animator: ObjectAnimator by lazy {
        ObjectAnimator.ofFloat(
            this@MaterialEditText, "floatingLabelFaction", 0f, 1f
        )
    }

    //动画是否显示
    private var floatingLabelShown = false

    private var floatingLabelFaction = 0f
        set(value) {
            field = value
            invalidate()
        }

    init {
        val typeArray = context?.obtainStyledAttributes(attrs, R.styleable.MaterialEditText)
        isFloatingLabel = typeArray!!.getBoolean(R.styleable.MaterialEditText_floatingLabel, true)
        typeArray.recycle()
        init()
    }



    fun init() {

        background.getPadding(backgroundPadding)
        onFloatingLabelChange()
        paint.textSize = mTextSize

        addChangeListener()
    }

    private fun addChangeListener() {
        addTextChangedListener(object : TextWatcher {
            override fun afterTextChanged(s: Editable?) = Unit
            override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) =
                Unit

            override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {
                if (isFloatingLabel) {
                    if (floatingLabelShown && TextUtils.isEmpty(s)) {
                        animator.reverse()
                        floatingLabelShown = false

                    } else if (!floatingLabelShown && !TextUtils.isEmpty(s)) {
                        floatingLabelShown = true
                        animator.start()
                    }
                }
            }
        })
    }


    fun isFloatingLabel(isFloatingLabel: Boolean) {
        if (this.isFloatingLabel != isFloatingLabel) {
            this.isFloatingLabel = isFloatingLabel
            onFloatingLabelChange()
        }
    }

    private fun onFloatingLabelChange() {
        if (isFloatingLabel) {
            setPadding(
                paddingLeft, backgroundPadding.top + (mTextSize + mTextMargin).toInt(),
                paddingRight, paddingBottom
            )
        } else {
            setPadding(
                paddingLeft, backgroundPadding.top,
                paddingRight, paddingBottom
            )
        }
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        paint.alpha = (floatingLabelFaction * 0xff).toInt()

        //计算偏移量
        val extraOffset = mTextAnimationOffset * (1 - floatingLabelFaction)
        //绘制提示信息
        canvas.drawText(
            hint as String, mTextHorizontalOffset,
            mTextVerticalOffset + (-extraOffset),
            paint
        )

    }
}
```

### 绘制饼图

```kotlin
/**'
 *
 * 绘制饼图
 */
class PieChart : View {

    private val radius = dp2px(150f).toInt()
    private val length = dp2px(20f).toInt()
    private val index = 2

    val angles = arrayOf(60, 100, 120, 90)
    val colors = arrayOf(
        Color.parseColor("#81FF6F"),
        Color.parseColor("#FF2A31"),
        Color.parseColor("#F0FF2C"),
        Color.parseColor("#001AFF")
    )


    val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    val bounds = RectF()

    constructor(context: Context?) : super(context)
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)
    constructor(context: Context?, attrs: AttributeSet?, defStyleAttr: Int) : super(
        context,
        attrs,
        defStyleAttr
    )


    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        bounds.set(
            (width / 2 - radius).toFloat(), (height / 2 - radius).toFloat(),
            (width / 2 + radius).toFloat(), (height / 2 + radius).toFloat()
        )
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        canvas.drawArc(bounds, 0f, 60f, true, paint)

        var currentAge = 0f
        for (i in 0..3) {
            paint.color = colors[i]
            canvas.save()
            if (i == index) {
                canvas.translate(
                    (Math.cos(Math.toRadians(currentAge + angles[i].toDouble() / 2)) * length).toFloat(),
                    (Math.sin(Math.toRadians(currentAge + angles[i].toDouble() / 2)) * length).toFloat()
                )
            }
            canvas.drawArc(bounds, currentAge, angles[i].toFloat(), true, paint)
            canvas.restore()
            currentAge += angles[i].toFloat()
        }
    }

}
```

###  绘制文本，位置

```kotlin
/**
 * 绘制文本，位置
 */
class SportView : View {

    private val strokeWidth = dp2px(10F)
    private val radius = dp2px(150F)

    val rect = Rect()

    private var mScrollPos = 0f
    private var mProcess: Int = 0
    val fontMetrics = Paint.FontMetrics()

    val paint = Paint(Paint.ANTI_ALIAS_FLAG)

    constructor(context: Context?) : super(context)

    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)

    constructor(context: Context?, attrs: AttributeSet?, defStyleAttr: Int) : super(
        context,
        attrs,
        defStyleAttr
    )

    init {
        paint.textSize = dp2px(80f)
        //设置字体
//        paint.setTypeface(Typeface.createFromFile(""))
        paint.typeface = Typeface.DEFAULT
        //文字位置
        paint.textAlign = Paint.Align.CENTER
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec)
//        resolveSize()
    }

    /* */
    /**
     * 0 .. 360 之间
     */
    fun setScrollPos(scrollPos: Float) {
        this.mScrollPos = scrollPos
        invalidate()
    }

    fun getScrollPos(): Float {
        return this.mScrollPos
    }

    fun setProcess(process: Int) {
        this.mProcess = process
        invalidate()
    }

    fun getProcess(): Int {
        return mProcess
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        paint.textAlign = Paint.Align.CENTER
        paint.getFontMetrics(fontMetrics)
        paint.textSize = dp2px(50f)
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        //圆环
        paint.style = Paint.Style.STROKE
        paint.color = Color.BLACK
        paint.strokeWidth = strokeWidth
        canvas.drawCircle(
            (width / 2).toFloat(), (height / 2).toFloat(),
            radius, paint
        )
        //绘制进度条
        paint.color = Color.BLUE
        paint.strokeCap = Paint.Cap.ROUND
        canvas.drawArc(
            width / 2 - radius, height / 2 - radius,
            width / 2 + radius, height / 2 + radius,
            -90f, mScrollPos, false, paint
        )

        //绘制文字
        paint.color = Color.RED
        paint.style = Paint.Style.FILL
        val offset = (fontMetrics.ascent + fontMetrics.descent) / 2
        canvas.drawText("$mProcess%", (width / 2).toFloat(), (height / 2).toFloat() - offset, paint)
    }

}
```