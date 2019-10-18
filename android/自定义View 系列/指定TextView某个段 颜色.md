​		如果我们要改变 TextView 中某一个字 ，或者某一段字的颜色，那样会非常麻烦，改变颜色。但是如何你的 TextView 只是需要展示一下，不需要绑定控件等，使用那种方法就会非常麻烦，下面我们自定义一个CustomTextView来实现这种效果。

### 1，指定需要的属性

​	在 res/values 下创建 attrs.xml 文件

```java
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- 自定义View的名字 -->
    <declare-styleable name="CustomTextView">

        <!-- 以下是所用到的属性-->

        <!-- 文字-string -->
        <attr name="novateText" format="string"/>
        <!-- 文字大小-dimension -->
        <attr name="novateTextSize" format="dimension"/>
        <!-- 文字长度-->
        <attr name="novateMaxLength" format="integer"/>
        <!-- 文字颜色-color-->
        <attr name="novateColor" format="color"/>

        <!--指定位置的颜色-->
        <attr name="TextColor" format="color"/>
        <!--要改变颜色的文字-->
        <attr name="colorText" format="string"/>
        <!--指定改变颜色位置-->
        <attr name="startPosInteger" format="integer"/>
        <attr name="endPosInteger" format="integer"/>

    </declare-styleable>
</resources>
```

### 2，自定义View

```java
public class CustomTextView extends android.support.v7.widget.AppCompatTextView {

    // 文字
    private String mText;
    // 文字默认大小 15sp
    private int mTextSize = 15;
    // 文字默认颜色
    private int mTextColor = Color.BLACK;
    //要改变颜色的文字
    private String mColorText = null;
    // 初始化画笔
    private TextPaint mPaint;
    private Rect wRect;
    private Rect hRect;

    private int startPos = -1;
    private int endPos = -1;
    private int colorPos = Color.RED;

    /**
     * 这种调用第1个构造方法
     * TextView tv = new TextView(this)：
     */
    public CustomTextView(Context context) {
        this(context, null);
    }

    /**
     * 这种调用第2个构造方法
     * <com.novate.test.customview.MyTextView
     * ......>
     */
    public CustomTextView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CustomTextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);

        // 获取自定义属性
        TypedArray typedArray = context.obtainStyledAttributes(attrs, R.styleable.CustomTextView);
        // 获取文字
        mText = typedArray.getString(R.styleable.CustomTextView_novateText);
        //获取设置颜色的文字
        mColorText = typedArray.getString(R.styleable.CustomTextView_colorText);
        // 获取文字大小
        mTextSize = typedArray.getDimensionPixelSize(R.styleable.CustomTextView_novateTextSize, sp2px(mTextSize));
        // 获取文字颜色
        typedArray.getColor(R.styleable.CustomTextView_novateColor, mTextColor);
        startPos = typedArray.getInt(R.styleable.CustomTextView_startPosInteger, -1);
        endPos = typedArray.getInt(R.styleable.CustomTextView_endPosInteger, -1);
        colorPos = typedArray.getInt(R.styleable.CustomTextView_TextColor, Color.RED);
        // 释放资源
        typedArray.recycle();

        // 创建画笔
        mPaint = new TextPaint();
        // 设置抗锯齿，让文字比较清晰，同时文字也会变得圆滑
        mPaint.setAntiAlias(true);
        // 设置文字大小
        mPaint.setTextSize(mTextSize);
        // 设置画笔颜色
        mPaint.setColor(mTextColor);

    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        // 如果在布局文件中设置的宽高都是固定值[比如100dp、200dp等]，就用下边方式直接获取宽高
        int width = MeasureSpec.getSize(widthMeasureSpec);
        int height = MeasureSpec.getSize(heightMeasureSpec);

        //  T如果调用者在布局文件给自定义MyTextView的宽高属性设置wrap_content，
        // 就需要通过下边的 widthMode和heightMode来获取宽高
        // 获取宽高模式
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);

        // 如果在布局中设置宽高都是wrap_content[对应AT_MOST]，必须用mode计算
        if (widthMode == MeasureSpec.AT_MOST) {
            // 文字宽度 与字体大小和长度有关
            if (wRect == null) {
                wRect = new Rect();
            }
            // 获取文本区域 param1__测量的文字 param2__从位置0开始 param3__到文字长度
            mPaint.getTextBounds(mText, 0, mText.length(), wRect);
            // 文字宽度 getPaddingLeft宽度 getPaddingRight高度 写这两个原因是为了在布局文件中设置padding属性起作用
            width = wRect.width() + getPaddingLeft() + getPaddingRight();
        }

        if (heightMode == MeasureSpec.AT_MOST) {
            if (hRect == null) {
                hRect = new Rect();
            }
            mPaint.getTextBounds(mText, 0, mText.length(), hRect);
            height = hRect.height() + getPaddingTop() + getPaddingBottom();
        }

        // 给文字设置宽高
        setMeasuredDimension(width, height);
    }

    /**
     * 绘制文字
     */
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        if (mColorText != null && !mColorText.isEmpty()) {
            int start = mText.indexOf(mColorText);
            int end = start + mColorText.length();
            startToEnd(canvas, start, end, colorPos);
        } else if (startPos > -1 && endPos > -1) {
            startToEnd(canvas, startPos, endPos, colorPos);
        } else if (startPos > -1) {
            startToEnd(canvas, startPos, startPos + 1, colorPos);
        } else {
            int baseLine = getBaseLine();
            mPaint.setColor(mTextColor);
            canvas.drawText(mText, getPaddingLeft(), baseLine, mPaint);
        }
    }

    /**
     * 指定位置的文字变为指定的颜色
     *
     * @param canvas   画布
     * @param startPos 开始的位置
     * @param endPos   结束的位置
     * @param colorPos 指定的颜色
     */
    private void startToEnd(Canvas canvas, int startPos, int endPos, int colorPos) {

        int baseLine = getBaseLine();
        int paddingLeft = getPaddingLeft();

        String start = mText.substring(0, startPos);
        String red = mText.substring(startPos, endPos);
        String end = mText.substring(endPos, mText.length());
        canvas.drawText(start, paddingLeft, baseLine, mPaint);

        mPaint.setColor(colorPos);
        float redLength = Layout.getDesiredWidth(start, mPaint) + paddingLeft;
        canvas.drawText(red, redLength, baseLine, mPaint);

        mPaint.setColor(mTextColor);
        float endLength = Layout.getDesiredWidth(red, mPaint) + Layout.getDesiredWidth(start, mPaint) + paddingLeft;
        canvas.drawText(end, endLength, baseLine, mPaint);
    }

    /**
     * 开始的位置，可单独使用
     *
     * @param startPos 开始的位置，未设置结束位置则结束位置为 startPos + 1
     */
    public void startPos(int startPos) {
        this.startPos = startPos;
        invalidate();
    }

    /**
     * 结束位置
     *
     * @param endPos 结束位置，必须大于开始位置
     */
    public void endPos(int endPos) {
        this.endPos = endPos;
        invalidate();
    }

    /**
     * 设置要改变颜色的文字，和设置 位置 选择其一即可
     *
     * @param colorText 改变颜色的文字
     */
    public void setColorText(String colorText) {
        this.mColorText = colorText;
        invalidate();
    }

    /**
     * 设置普通文字的颜色
     *
     * @param color 颜色
     */
    public void novateColor(@ColorInt int color) {
        this.mTextColor = color;
        invalidate();
    }

    /**
     * 设置 text
     * @param mText text
     */
    public void novateText(String mText) {
        this.mText = mText;
        invalidate();
    }

    /**
     * 设置 text size
     * @param mTextSize size
     */
    public void novateTextSize(int mTextSize){
        this.mTextSize = mTextSize;
        invalidate();
    }

    /**
     * 获取基线
     */
    private int getBaseLine() {
        Paint.FontMetricsInt fontMetricsInt = mPaint.getFontMetricsInt();
        int dy = (fontMetricsInt.bottom - fontMetricsInt.top) / 2 - fontMetricsInt.bottom;
        return getHeight() / 2 + dy;
    }

    /**
     * sp转为px
     */
    private int sp2px(int sp) {
        return (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, sp,
                getResources().getDisplayMetrics());
    }

}

```

​	注释都写在上面 了，下面看一下使用

### 3，使用

####  	先看一下最普通的使用：

```
<com.testdemo.www.svg.CustomTextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:novateColor="#000000"
        app:novateText="姓名*：张三" />
```

![1571367335229](%E6%8C%87%E5%AE%9ATextView%E7%9A%84%E9%A2%9C%E8%89%B2.assets/1571367335229.png)

#### 	设置指定文字的颜色

```
 <com.testdemo.www.svg.CustomTextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:novateColor="#000000"
        app:colorText="名*"
        app:novateText="姓名*：张三" />
```

![1571367455689](%E6%8C%87%E5%AE%9ATextView%E7%9A%84%E9%A2%9C%E8%89%B2.assets/1571367455689.png)

设置某一个字的颜色

```java
<com.testdemo.www.svg.CustomTextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:novateColor="#000000"
        app:novateText="姓名*：张三"
        app:startPosInteger="2" />
```

![1571367682272](%E6%8C%87%E5%AE%9ATextView%E7%9A%84%E9%A2%9C%E8%89%B2.assets/1571367682272.png)

设置某一段文字的颜色

```java
<com.testdemo.www.svg.CustomTextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:novateColor="#000000"
        app:novateText="姓名*：张三"
        app:startPosInteger="2" 
        app:endPosInteger="4"/>
```

![1571367661267](%E6%8C%87%E5%AE%9ATextView%E7%9A%84%E9%A2%9C%E8%89%B2.assets/1571367661267.png)

设置指定文字的颜色

```java
<com.testdemo.www.svg.CustomTextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:novateColor="#000000"
        app:novateText="姓名*：张三"
        app:startPosInteger="2"
        app:endPosInteger="4"
        app:textColor="#0037ff"/>
```

![1571368109072](%E6%8C%87%E5%AE%9ATextView%E7%9A%84%E9%A2%9C%E8%89%B2.assets/1571368109072.png)

------

以上就是具体的使用了，如果需要什么功能，直接在 CustomTextView 中添加就行了。如有错误，还请指出