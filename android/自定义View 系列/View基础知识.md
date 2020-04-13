### 什么是 View

​	View 是Android 中所有控件的基类，不管是 Button 和 ListView ，他们都是 View 的子类。所以 View 是控件的一种抽象，它代表了一个空间。

​	 还有 ViewGroup ，他内部

可以包含多个控件，但自己也是继承自 View。

### View 的位置参数

![image-20200410102003918](View%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86.assets/image-20200410102003918.png)

```
width = right - left
height = bottom - top
```

这四个参数可根据对应的 get 方法获取

Android 3.0 开始，View 增加了几个额外的参数：x，y，translationX，translationY。其中 x，y是 View 左上角的坐标，而 translationX ，translationY 是 View 左上角 相对于父容器的偏移量。这几个参数也是相对于父容器的左边，并 translationX ，translationY 的默认值是 0。

```
x = left + translationX
y = top + translationY
```

### TouchSlop

​	TouchSlop 是系统所能识别出被认为是滑动最小的距离。在手机上滑动是，滑动的距离小于这个常量，那么系统认为你没有滑动。不同的设备的值可能不同。

```
ViewConfiguration.get(this).scaledDoubleTapSlop
```

​	通过如上方式可获取这个常量。

​	当处理滑动是，可以用这个常量做一写过滤，如果小于这个值，则不认为在滑动。

```kotlin
class MyLinearLayout : LinearLayoutCompat {

    constructor(context: Context) : this(context, null)

    constructor(context: Context, attributeSet: AttributeSet?) : this(context, attributeSet, 0)

    constructor(context: Context, attributeSet: AttributeSet?, def: Int) : super(context, attributeSet,def)

    private var velocity: VelocityTracker = VelocityTracker.obtain()
    
    @SuppressLint("ClickableViewAccessibility")
    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                velocity.addMovement(event)
            }
            MotionEvent.ACTION_MOVE -> {
                velocity.addMovement(event)
            }
            MotionEvent.ACTION_UP -> {
                velocity.computeCurrentVelocity(1000)
                Log.e("--------", velocity.xVelocity.toString())
                Log.e("--------", velocity.yVelocity.toString())
            }
        }
        return true
    }
}
```

​	滑动的速度可以为负值，当从左往右划时为正。从右往左时为负

​	速度的计算可以使用如下公式来表示： 速度 = (终点位置 - 起点位置) / 时间段

​	需啊哟 使用 clear 和 recycler 方法重置并回收内存

### GestureDetector 手势检测

​	可检测 单击，滑动，长安，双击等行为。

​	**[打开方式](https://blog.csdn.net/baidu_40389775/article/details/94459776)**

### Scroller

​	详见同级文件夹中 Scroller.md

### 其他的滑动方式

- scrollTo / scrollBy
- 使用动画
- 改变布局参数：例如改变 MarginLayoutParams

### 弹性滑动

- 使用 Scroller
- 通过属性动画
- 使用延时策略

