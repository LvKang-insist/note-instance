

自定义 绘制的方式是重写 绘制方法。其中最常用的是 onDraw()方法

绘制的关键是 Canvas 的使用

​	Canvas 的绘制类方法：drawXX(),关键参数为Paint

​	Canvas 的辅助类方法：范围裁切 和 几何变换，可以使用不同的绘制方法来控制 遮盖关系

## 绘制类方法：

#### Paint 类中最常用的方法。

- Paint.setStyle(Style style) 设置绘制模
- Paint.setColor(int color) 设置颜色
- Paint.setStrokeWidth(float width) 设置线条宽度
- Paint.setTextSize(float textSize) 设置文字大小
- Paint.setAntiAlias(boolean aa) 设置抗锯齿开关

#### Canvas 中的常用方法,canvas 类下的 draw 打头的方法，例如drawCircle(),drawBitmp()

- Canvas.drawColor(@ColorInt int color) 颜色填充

  例如：`drawColor(Color.BLACK)` 会把整个区域染成纯黑色，覆盖掉原有内容；

  drawColor(Color.parse("#88880000")` 会在原有的绘制效果上加一层半透明的红色遮罩。 

  类似的方法还有 drawRGB 和 drawARGB ,他们只是使用方式不同，作用都是一样的。

- drawCircle(float centerX, float centerY, float radius, Paint paint) 画圆

  前两个参数是圆心的坐标，第三个参数是 圆的半径，单位都是像素第四个就是 Paint 了。

- drawRect(float left, float top, float right, float bottom, Paint paint) 画矩形

  前四个参数分别对应四个坐标，最后是 Paint

- drawPoint(float x, float y, Paint paint) 画点

  x 和 y 是坐标

  点的 大小可以通过paint.setStrokeWidth()来设置，点的形状可以通过 paint .setStrokeCap 来设置

- drawPoints(float[] pts, int offset, int count, Paint paint) / drawPoints(float[] pts, Paint paint) 画点（批量）

- drawOval(float left, float top, float right, float bottom, Paint paint) 画椭圆

  只能绘制 横着或者竖着的椭圆。斜着的需要使用几何变换。四个参数分别对应四个边界的坐标

- drawLine(float startX, float startY, float stopX, float stopY, Paint paint) 画线

  前两个是开始的坐标，然后是终点的坐标。

- drawLines(float[] pts, int offset, int count, Paint paint) / drawLines(float[] pts, Paint paint) 画线（批量）

- drawRoundRect(float left, float top, float right, float bottom, float rx, float ry, Paint paint) 画圆角矩形

  前四个是坐标，rx 和 ry 是圆角的 横向半径和 纵向半径

- drawArc(float left, float top, float right, float bottom, float startAngle, float sweepAngle, boolean useCenter, Paint paint) 绘制弧形或扇形

  `drawArc()` 是使用一个椭圆来描述弧形的。`left`, `top`, `right`, `bottom` 描述的是这个弧形所在的椭圆；`startAngle` 是弧形的起始角度（x 轴的正向，即正右的方向，是 0 度的位置；顺时针为正角度，逆时针为负角度），`sweepAngle` 是弧形划过的角度；`useCenter` 表示是否连接到圆心，如果不连接到圆心，就是弧形，如果连接到圆心，就是扇形。 

- drawPath(Path path, Paint paint) 画自定义图形

  Path 可以描述直线二次曲线三次曲线，圆，椭圆，弧形，圆角矩形。将这些图形结合起来就可以描述出 很多复杂的图形

##### Path 可以描述直线，二次曲线，三次曲线，圆，椭圆，圆角矩形，将这些结合起来，就可以绘制很多复杂的图形。

Path 有两类方法，一类是描述路径的，另一类是辅助的设置或者计算

​	第一类：描述路径

- addCircle(float x, float y, float radius, Direction dir) 添加圆 

  前两个时候 圆心的 坐标，然后就是半径。最后是 路径，有两种，是CW / 顺时针 和 CCW / 逆时针

- xxxTo()——画线（直线或曲线)

- lineTo(float x, float y) / rLineTo(float x, float y) 画直线 

  从当前位置 向目标位置画一条线，x,y 对应的是目标位置的坐标，这两个区别 第一个是绝对坐标，第二个是 相对坐标

  ```java
  path.lineTo(100, 100); // 由当前位置 (0, 0) 向 (100, 100) 画一条直线  
  path.rLineTo(100, 0); // 由当前位置 (100, 100) 向正右方 100 像素的位置画一条直线  
  ```

- quadTo(float x1, float y1, float x2, float y2) / rQuadTo(float dx1, float dy1, float dx2, float dy2) 画二次贝塞尔曲线 

  四个参数对应 控制点 和终点的坐标，第二个参数是 相对坐标。

- cubicTo(float x1, float y1, float x2, float y2, float x3, float y3) / rCubicTo(float x1, float y1, float x2, float y2, float x3, float y3) 画三次贝塞尔曲线 

  和上面的同理

- moveTo(float x, float y) / rMoveTo(float x, float y) 移动到目标位置 

  不论是直线还是贝塞尔曲线，都是以当前位置作为起点，而不能指定起点。但你可以通过 `moveTo(x, y)` 或 `rMoveTo()` 来改变当前位置，从而间接地设置这些方法的起点 

  ```java
     //开始的位置
      path.moveTo(100, 100);
      // 线的末端，分别对应 x 和 y
      path.lineTo(300, 100);
      canvas.drawPath(path, mPaint);
  ```

- arcTo(RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo) / arcTo(float left, float top, float right, float bottom, float startAngle, float sweepAngle, boolean forceMoveTo) / arcTo(RectF oval, float startAngle, float sweepAngle) 画弧形 

  这个方法和 `Canvas.drawArc()` 比起来，少了一个参数 `useCenter`，而多了一个参数 `forceMoveTo` 。

  少了 `useCenter` ，是因为 `arcTo()` 只用来画弧形而不画扇形，所以不再需要 `useCenter` 参数；而多出来的这个 `forceMoveTo` 参数的意思是，绘制是要「抬一下笔移动过去」，还是「直接拖着笔过去」，区别在于是否留下移动的痕迹。

  第二类：辅助的设置或计算

- Path.setFillType(Path.FillType ft) 设置填充方式

- 方法中填入不同的 `FillType` 值，就会有不同的填充效果。`FillType` 的取值有四个：

  - `EVEN_ODD`
  - `WINDING` （默认值）
  - `INVERSE_EVEN_ODD`
  - `INVERSE_WINDING`



### 示例：

Paint ：该类保存了绘制几何 图形，文本 和位图 的样式和颜色信息。也就是说 我们可以使用Paint 保存的样式 和 颜色 还绘制 图形，文本 和 bitmap ，这就是 Paint 的 强大之处。

Paint 的使用

使用 Paint 之前需要初始化

```
mPaint = new Paint();
```

常用的 方法

```java
        mPaint.setColor(Color.BLUE);
        //alpha 的范围是 0-255，是一个 int值
        mPaint.setAlpha(255);

 		//设置画笔的样式
        mPaint.setStyle(Paint.Style.FILL);// 填充内容
        mPaint.setStyle(Paint.Style.STROKE);//描边
        mPaint.setStyle(Paint.Style.FILL_AND_STROKE); //填充内容并 描边

 		//设置画笔的宽度
        mPaint.setStrokeWidth(50);
        
        //设置画笔的线帽,共三种
        mPaint.setStrokeCap(Paint.Cap.BUTT);// 默认
        mPaint.setStrokeCap(Paint.Cap.ROUND); // 圆形
        mPaint.setStrokeCap(Paint.Cap.SQUARE); //直线
        
         //路线
        Path path = new Path();
        //开始的位置
        path.moveTo(100, 100);
        // 线的末端，分别对应 x 和 y
        path.lineTo(300, 100);
        
        
        //线的 连接处
        mPaint.setStrokeJoin(Paint.Join.BEVEL); //直线
        mPaint.setStrokeJoin(Paint.Join.ROUND);//圆弧
        mPaint.setStrokeJoin(Paint.Join.MITER);//锐角
        
	    //防锯齿 损失性能
        mPaint.setAntiAlias(true);
        //抖动处理 损失性能
        mPaint.setDither(true);

		 //Shader ,着色器，用于绘制颜色，他和直接设置颜色的区别就是 ，着色器 设置的是一个颜色方案，或者是一套规则。
        //当设置了 Shader 之后，Paint 在绘制图形和位置时 就不使用 setColor 设置的颜色了
        //在绘制里面使用 Shader ，不是直接使用这个类，而是用他的几个子类：linearGradient ,RadiaGradient
        //SweepGradient ,BitmapShader ,ComposeShader 这几个。
        //LinearGradient ：线性渐变,设置两个点的两种颜色，两个点的中级则为渐变。
        Shader shader = new LinearGradient(100,100,500,500,Color.RED,Color.BLUE, Shader.TileMode.CLAMP);
        paint.setShader(shader);
        canvas.drawCircle(300,300,300,paint);
// 更多的用法可查看这篇 文章：https://hencoder.com/ui-1-2/

```

三种 线帽 的样子：

```java
@Override
public void draw(Canvas canvas) {
    super.draw(canvas);
    mPaint = new Paint();
    mPaint.setColor(Color.BLUE);
    //alpha 的范围是 0-255，是一个 int值
    mPaint.setAlpha(255);
    //画笔的样式
    mPaint.setStyle(Paint.Style.FILL_AND_STROKE); //填充内容并 描边
    //设置画笔的宽度
    mPaint.setStrokeWidth(50);
    //设置画笔的线帽
    mPaint.setStrokeCap(Paint.Cap.SQUARE); // 圆形
    //路线
    Path path = new Path();
    //开始的位置
    path.moveTo(100, 100);
    // 线的末端，分别对应 x 和 y
    path.lineTo(300, 100);
    canvas.drawPath(path, mPaint);

    //重置
    mPaint.reset();
    mPaint.setColor(Color.RED);
    mPaint.setStyle(Paint.Style.FILL_AND_STROKE);
    mPaint.setStrokeWidth(50);
    mPaint.setStrokeCap(Paint.Cap.ROUND); //圆形
    Path path1 = new Path();
    path1.moveTo(100,200);
    path1.lineTo(300,200);
    canvas.drawPath(path1,mPaint);

    //重置
    mPaint.reset();
    mPaint.setColor(Color.BLACK);
    mPaint.setStyle(Paint.Style.FILL_AND_STROKE);
    mPaint.setStrokeWidth(50);
    mPaint.setStrokeCap(Paint.Cap.BUTT);//没有
    Path path2 = new Path();
    path2.moveTo(100,300);
    path2.lineTo(300,300);
    canvas.drawPath(path2,mPaint);
}
```

![1558861017409](F:\笔记\android\自定义View 系列\assets\1558861017409.png)

以上就是三种 键帽的对比

下面看一下 线 连接处 和 填充/描边 的对比

```java
@Override
public void draw(Canvas canvas) {
    super.draw(canvas);
    mPaint = new Paint();
    mPaint.setColor(Color.BLUE);
    //alpha 的范围是 0-255，是一个 int值
    mPaint.setAlpha(255);
    //画笔的样式
    mPaint.setStyle(Paint.Style.STROKE); //描边
    //设置画笔的宽度
    mPaint.setStrokeWidth(50);
    //设置画笔的线帽
    mPaint.setStrokeCap(Paint.Cap.BUTT); // 默认
    //直线
    mPaint.setStrokeJoin(Paint.Join.BEVEL); //直线
    //路线
    Path path = new Path();
    //开始的位置
    path.moveTo(100, 100);
    // 线的末端，分别对应 x 和 y
    path.lineTo(300, 100);
    path.lineTo(100, 300);
    //关闭这条线，如果线尾不等于第一点，则补上
    path.close();
    canvas.drawPath(path, mPaint);

    //重置
    mPaint.reset();
    mPaint.setColor(Color.RED);
    mPaint.setStyle(Paint.Style.FILL_AND_STROKE);//描边 加 填充
    mPaint.setStrokeWidth(50);
    mPaint.setStrokeCap(Paint.Cap.BUTT);
    mPaint.setStrokeJoin(Paint.Join.ROUND);//圆弧
    Path path1 = new Path();
    path1.moveTo(100,400);
    path1.lineTo(300,400);
    path1.lineTo(100,600);
    path1.close();
    canvas.drawPath(path1,mPaint);

    //重置
    mPaint.reset();
    mPaint.setColor(Color.BLACK);
    mPaint.setStyle(Paint.Style.STROKE);
    mPaint.setStrokeWidth(50);
    mPaint.setStrokeCap(Paint.Cap.BUTT);
    mPaint.setStrokeJoin(Paint.Join.MITER);//锐角
    Path path2 = new Path();
    path2.moveTo(100,700);
    path2.lineTo(300,700);
    path2.lineTo(100,900);
    path2.close();
    canvas.drawPath(path2,mPaint);

}
```

![1558862731724](F:\笔记\android\自定义View 系列\assets\1558862731724.png)

如上所示，第二个使用了 填充

绘制文本

```java
//设置字符之间的间距
mPaint.setLetterSpacing(0.2f);
//设置文本 删除线
mPaint.setStrikeThruText(true);
//是否设置下划线
mPaint.setUnderlineText(true);
//设置文本大小
mPaint.setTextSize(50);
//设置字体类型
mPaint.setTypeface(Typeface.defaultFromStyle(Typeface.NORMAL));// 常规
mPaint.setTypeface(Typeface.defaultFromStyle(Typeface.BOLD));// 粗体
mPaint.setTypeface(Typeface.defaultFromStyle(Typeface.ITALIC));// 斜体
mPaint.setTypeface(Typeface.defaultFromStyle(Typeface.BOLD_ITALIC));// 粗斜体

 // 使用 Paint.setTextSkewX() 来让文字倾斜
mPaint.setTextSkewX(0.5f);

//加载自定义字体
Typeface.create(familyName,style);

//文字倾斜
mPaint.setTextSkewX(-0.25f);//文字倾斜默认为0，官方推荐的 -0.25f 是斜体

//文本对齐方式
mPaint.setTextAlign(Paint.Align.LEFT); // 左对齐
mPaint.setTextAlign(Paint.Align.CENTER); //居中
mPaint.setTextAlign(Paint.Align.RIGHT); //右对齐
```

```java
@RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
@Override
public void draw(Canvas canvas) {
    super.draw(canvas);
    mPaint = new Paint();
    mPaint.setColor(Color.BLUE);
    mPaint.setStyle(Paint.Style.STROKE); //描边
    mPaint.setStrokeCap(Paint.Cap.BUTT); //线帽：直线
    mPaint.setStrokeJoin(Paint.Join.BEVEL); // 连接处：直线
    
    // 使用 Paint.setFakeBoldText() 来加粗文字
    mPaint.setFakeBoldText(true);

    int baselineX = 100;
    //设置文本大小
    mPaint.setTextSize(50);
    //设置字体类型
    mPaint.setTypeface(Typeface.defaultFromStyle(Typeface.NORMAL));// 常规
    //文本对齐方式
    mPaint.setTextAlign(Paint.Align.LEFT);
    //文本
    Paint.FontMetrics fontMetrics = mPaint.getFontMetrics();
    //获取 基线的坐标,这样文字会位于 自定义View 的中间
    float baseLineY = getHeight() / 2 + ((fontMetrics.bottom - fontMetrics.top) / 2 - fontMetrics.bottom);
    // 开始 的 x ,y
    canvas.drawText("我是自定义文本", baselineX, baseLineY, mPaint);
}
```



### 辅助类方法：范围裁切 和 几何变换

### 范围裁切

​	1，clipRect()

```java
 Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
 Bitmap bitmap= BitmapFactory.decodeResource(getResources(), R.drawable.maps);
 
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        int left = (getWidth() - bitmap.getWidth()) / 2;
        int top = (getHeight() - bitmap.getHeight()) / 2;
        //保存状态
        canvas.save();
        //裁切 一个矩形
        canvas.clipRect(left+50,top+50,left+300,top+200);
        //设置图片的位置
        canvas.drawBitmap(bitmap,left,top,paint);
        //恢复绘制范围
        canvas.restore();
    }
```

原图：

![1559300388291](F:\笔记\android\自定义View 系列\assets\1559300388291.png)

裁切后：

![1559300412485](F:\笔记\android\自定义View 系列\assets\1559300412485.png)

​	2,clipPath()

​	其实 和clipRect 用法一样，只是把参数换成了 path ，所以能裁切的形状更多一些

```java
    Paint paint = new Paint();
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.maps);
    Point point1 = new Point(200, 200);
    Point point2 = new Point(600, 200);
    Path path1 = new Path();
    Path path2 = new Path();

 	@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        //保存状态
        canvas.save();
        //添加一个圆
        path1.addCircle(200+point1.x,200+point1.y,150, Path.Direction.CW);
        //裁剪圆
        canvas.clipPath(path1);
        //中间两个参数为 图片距离左边和顶部的位置
        canvas.drawBitmap(bitmap,point1.x,point1.y,paint);
        canvas.restore();
        
        canvas.save();
        //画在圆外面
        path2.setFillType(Path.FillType.INVERSE_WINDING);
        path2.addCircle(200+point2.x,200+point2.y,150, Path.Direction.CW);
        canvas.clipPath(path2);
        canvas.drawBitmap(bitmap, point2.x, point2.y, paint);
        canvas.restore();
    }
```

原图同上，裁切后：

![1559300677840](F:\笔记\android\自定义View 系列\assets\1559300677840.png)

### 几何变换

​	几何变换大概分为三类

​	1，使用Canvas 来做常见的 二维变换

​	2，使用Matrix 来做常见和不常见的二维变换

​	3，使用Camera 来做 三维变换

1,二维变换

​	平移：

```java
Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
Bitmap bitmap= BitmapFactory.decodeResource(getResources(), R.drawable.maps);
Point point1 = new Point(200, 200);
Point point2 = new Point(600, 200);

  @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.drawBitmap(bitmap, point1.x, point1.y, paint);
        canvas.save();
        //平移 dx 表示横向，dy 表示纵向
        canvas.translate(200,50);
        canvas.drawBitmap(bitmap, point2.x, point2.y, paint);
        canvas.restore();
    }
```

​	移动前：

​	![1559301136712](F:\笔记\android\自定义View 系列\assets\1559301136712.png)

​	移动后：

​	![1559301221560](F:\笔记\android\自定义View 系列\assets\1559301221560.png)

​	缩放：

```java
canvas.save();
//缩放，前两个是缩放的倍数，后两个是缩放的轴心
canvas.scale(1,0.5f,(float)(point2.x+(bitmap.getWidth()/2)),(float)(point2.y+(bitmap.getHeight()/2)));
canvas.drawBitmap(bitmap, point2.x, point2.y, paint);
canvas.restore();
```

​	旋转：

```java
canvas.save();
//第一个参数 为 旋转的度数，后面是 轴心
canvas.rotate(60,(float)(point2.x+(bitmap.getWidth()/2)),(float)(point2.y+(bitmap.getHeight()/2)));
canvas.drawBitmap(bitmap, point2.x, point2.y, paint);
canvas.restore();
```

​	错切：

```java
canvas.save();
// 参数为 x 和 y 轴的错切系数
//通俗一点 x 就是左右旋转，y 就是上下旋转
canvas.skew(0,0.5f);
canvas.drawBitmap(bitmap, point1.x, point1.y, paint);
canvas.restore();

canvas.save();
canvas.skew(-0.5f,0);
canvas.drawBitmap(bitmap, point2.x, point2.y, paint);
canvas.restore();
```

![1559302903305](F:\笔记\android\自定义View 系列\assets\1559302903305.png)

2，使用Matrix 来做 变换

​	Matrix 常见的变换方式

​	1,创建 Matrix 对象

​	2,调用 Matrix 的pre/postTranslate/Rotate/Scale/Skew()方法来设置几何变换;

​	3,使用Canvas.setMatrix(matrix) 或 Canvas.concat(matrix) 把几何变换应用到 Canvas .

```java
//移动
matrix.postTranslate(200,50);
//旋转
matrix.postRotate(60,(float)(point2.x+(bitmap.getWidth()/2)),(float)(point2.y+(bitmap.getHeight()/2)));
canvas.drawBitmap(bitmap, point1.x, point1.y, paint);

canvas.save();
canvas.concat(matrix);
canvas.drawBitmap(bitmap, point2.x, point2.y, paint);
canvas.restore();
```

把Matrix 应用到 Canvas 有两个方法：Canvas.setMatrix(matrix) 和 Canvas.concat(matrix)

1. `Canvas.setMatrix(matrix)`：用 `Matrix` 直接替换 `Canvas` 当前的变换矩阵，即抛弃 `Canvas` 当前的变换，改用 `Matrix` 的变换（注：根据下面评论里以及我在微信公众号中收到的反馈，不同的系统中 `setMatrix(matrix)` 的行为可能不一致，所以还是尽量用 `concat(matrix)` 吧）；
2. `Canvas.concat(matrix)`：用 `Canvas` 当前的变换矩阵和 `Matrix` 相乘，即基于 `Canvas` 当前的变换，叠加上 `Matrix` 中的变换。

3, 使用Camera 来做 三维变换

​	Camera 的三维变换有三类：旋转，平移，移动相机

​	1，三维旋转：

​		`Camera.rotate*()` 一共有四个方法： `rotateX(deg)` `rotateY(deg)` `rotateZ(deg)` `rotate(x, y, z)`。 

```java
Camera camera = new Camera();
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        canvas.save();
        camera.save();
        //旋转Camera 的三维空间
        camera.rotateX(30);
        //将旋转投影到 Canvas
        camera.applyToCanvas(canvas);
        camera.restore();
        canvas.drawBitmap(bitmap, point1.x, point1.y, paint);
        canvas.restore();

        canvas.save();
        camera.save();
        //旋转Camera 的三维空间
        camera.rotateY(30);
        //将旋转投影到 Canvas
        camera.applyToCanvas(canvas);
        canvas.drawBitmap(bitmap, point2.x, point2.y, paint);
        canvas.restore();
        camera.restore();
    }
```

​	2,Camera.translate(float x, float y, float z) 移动

​	它的使用方式和 `Camera.rotate*()` 相同 

​	4,Camera.setLocation(x, y, z) 设置虚拟相机的位置

​	注意，这个方法参数的单位不是像素，而是 inch ，尺寸

​	在 `Camera` 中，相机的默认位置是 (0, 0, -8)（英寸）。8 x 72 = 576，所以它的默认位置是 (0, 0, -576)（像素）。 

​	如果绘制的内容过大，当它翻转起来的时候，就有可能出现图像投影过大的「糊脸」效果。而且由于换算单位被写死成了 72 像素，而不是和设备 dpi 相关的，所以在像素越大的手机上，这种「糊脸」效果会越明显。

 	而使用 `setLocation()` 方法来把相机往后移动，就可以修复这种问题。 

```
camera.setLocation(0, 0, newZ);  
```

​	

​	旋转效果：

```java
public class Practice13CameraRotateHittingFaceView extends View {
    Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    Bitmap bitmap;
    Point point = new Point(200, 50);
    Camera camera = new Camera();
    Matrix matrix = new Matrix();
    int degree;
    ObjectAnimator animator = ObjectAnimator.ofInt(this, "degree", 0, 360);

    public Practice13CameraRotateHittingFaceView(Context context) {
        super(context);
    }

    public Practice13CameraRotateHittingFaceView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public Practice13CameraRotateHittingFaceView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    {
        bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.maps);
        Bitmap scaledBitmap = Bitmap.createScaledBitmap(bitmap, bitmap.getWidth() * 2, bitmap.getHeight() * 2, true);
        bitmap.recycle();
        bitmap = scaledBitmap;

        animator.setDuration(5000);
        animator.setInterpolator(new LinearInterpolator());
        animator.setRepeatCount(ValueAnimator.INFINITE);

        DisplayMetrics displayMetrics = getResources().getDisplayMetrics();
        float newZ = - displayMetrics.density * 6;
        camera.setLocation(0, 0, newZ);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        animator.start();
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        animator.end();
    }

    @SuppressWarnings("unused")
    public void setDegree(int degree) {
        this.degree = degree;
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        int bitmapWidth = bitmap.getWidth();
        int bitmapHeight = bitmap.getHeight();
        int centerX = point.x + bitmapWidth / 2;
        int centerY = point.y + bitmapHeight / 2;

        camera.save();
        matrix.reset();
        camera.rotateZ(degree);
        camera.getMatrix(matrix);
        camera.restore();
        matrix.preTranslate(-centerX, -centerY);
        matrix.postTranslate(centerX, centerY);
        canvas.save();
        canvas.concat(matrix);
        canvas.drawBitmap(bitmap, point.x, point.y, paint);
        canvas.restore();
    }
}
```

![ObjectAnimator](F:\笔记\android\自定义View 系列\assets/ObjectAnimator.gif)

![ObjectAnimator](F:\笔记\android\自定义View 系列\assets/ObjectAnimator-1559308278229.gif)

​	翻页效果：

​	

```java
public class Practice14FlipboardView extends View {

    Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    Bitmap bitmap;
    Camera camera = new Camera();
    int degree;
    ObjectAnimator animator = ObjectAnimator.ofInt(this, "degree", 0, 180);

    public Practice14FlipboardView(Context context) {
        super(context);
    }

    public Practice14FlipboardView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public Practice14FlipboardView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    {
        bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.maps);

        animator.setDuration(2500);
        animator.setInterpolator(new LinearInterpolator());
        animator.setRepeatCount(ValueAnimator.INFINITE);
        animator.setRepeatMode(ValueAnimator.REVERSE);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        animator.start();
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        animator.end();
    }

    @SuppressWarnings("unused")
    public void setDegree(int degree) {
        this.degree = degree;
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        int bitmapWidth = bitmap.getWidth();
        int bitmapHeight = bitmap.getHeight();
        int centerX = getWidth() / 2;
        int centerY = getHeight() / 2;
        int x = centerX - bitmapWidth / 2;
        int y = centerY - bitmapHeight / 2;

        canvas.save();
        canvas.clipRect(0,0,getWidth(),centerY);
        canvas.drawBitmap(bitmap,x,y,paint);
        canvas.restore();


        canvas.save();
        if (degree <90){
            //下半部分
            canvas.clipRect(0,centerY,getWidth(),getHeight());
        }else {
            canvas.clipRect(0,0,getWidth(),centerY);
        }
        camera.save();
        camera.rotateX(degree);
        canvas.translate(centerX, centerY);
        camera.applyToCanvas(canvas);
        canvas.translate(-centerX, -centerY);
        camera.restore();

        canvas.drawBitmap(bitmap, x, y, paint);
        canvas.restore();
    }
}

```

![ObjectAnimator](F:\笔记\android\自定义View 系列\assets/ObjectAnimator-1559308369125.gif)



参考自:

​	<https://www.jianshu.com/p/3aa9dc7d3320> 

​	<https://www.jb51.net/article/128264.htm> 

​    	<https://hencoder.com/ui-1-1/> 

​	<https://hencoder.com/ui-1-3/> 