## View 和 ViewGroup 的区别

View 是所有控件的 基类，我们平常用 所用的 TextView 和 ImageView 等，都是继承自View 。

ViewGroup 可以理解为 View 的组合，他里面可以包含多个 View 以及ViewGroup，而他包含的 ViewGroup 又可以包含View 和 ViewGroup，以此类推，形成一个树

![1559636481091](F:\笔记\android\自定义View 系列\assets\1559636481091.png)

#### ViewGroup 的作用：

​	ViewGroup 相当于一个容器，他的主要作用为 给子View 计算出建议的宽高 和 测量模式，决定子View 的位置，为什么只是建议宽和高呢，因为 子View 的宽和高 都可以设置为 wrap_content 。

#### View 的作用

​	根据测量的模式和 ViewGroup 给出建议的宽和高，计算出自己的宽和高，同时还有一个非常重要的 作用就是：在ViewGroup 内为其指定的区域绘制出自己的形态。

#### ViewGroup 和 LayoutParams 之间的关系

​	当在LinearLayout中写childView的时候，可以写layout_gravity，layout_weight属性；在RelativeLayout中的childView有layout_centerInParent属性，却没有layout_gravity，layout_weight，这是为什么呢？这是因为每个ViewGroup需要指定一个LayoutParams，用于确定支持childView支持哪些属性，比如LinearLayout指定LinearLayout.LayoutParams等。如果大家去看LinearLayout的源码，会发现其内部定义了LinearLayout.LayoutParams，在此类中，你可以发现weight和gravity的身影。

例子：

需求：自定义一个 ViewGroup ，内部可以传入 0 到 4 个 childView ，分别 依次显示在 四个角	

```java
public class CustomContainer extends ViewGroup {

    public CustomContainer(Context context) {
        this(context, null);
    }

    public CustomContainer(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CustomContainer(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    //只需要 ViewGroup 能够支持margin 即可，那么直接使用 MarginLayoutParams
    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new MarginLayoutParams(getContext(), attrs);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        //计算出所有的childView 的宽和高
        measureChildren(widthMeasureSpec, heightMeasureSpec);

        //如果是 wrap_content 设置的宽和高
        int width = 0;
        int height = 0;

        int cCount = getChildCount();
        int cWidth = 0;
        int cHeight = 0;
        MarginLayoutParams cParams = null;

        //用于计算左边两个childView 的高度
        int lHeight = 0;
        //用于计算 右边两个childView 的高度，最终高度 区两者的最大值
        int rHeight = 0;

        //用于计算 上面两个 childView 的宽度
        int tWidth = 0;
        //用于计算 下面两个 childView 的宽度，最终宽度器取两者中的最大值
        int bWidth = 0;

        //根据 childView 计算出ViewGroup的宽和高，以及设置的margin 计算容器的 宽和高，主要容器是 warp_content时
        for (int i = 0; i < cCount; i++) {
            View childView = getChildAt(i);
            cWidth = childView.getMeasuredWidth();
            cHeight = childView.getMeasuredHeight();
            cParams = (MarginLayoutParams) childView.getLayoutParams();
            //宽
            if (i == 0 || i == 1) {
                tWidth += cWidth + cParams.leftMargin + cParams.rightMargin;
            }
            if (i == 2 || i == 3) {
                bWidth += cWidth + cParams.leftMargin + cParams.rightMargin;
            }
            //高
            if (i == 0 || i == 2) {
                lHeight += cHeight + cParams.topMargin + cParams.bottomMargin;
            }
            if (i == 1 || i == 3) {
                rHeight += cHeight + cParams.topMargin + cParams.bottomMargin;
            }

        }
        width = Math.max(tWidth, bWidth);
        height = Math.max(lHeight, rHeight);
        /**
         * 如果是 wrap_content 设置为我们计算的值
         * 否则 直接设置为 父容器计算的值
         */
        setMeasuredDimension((widthMode == MeasureSpec.EXACTLY) ? widthSize : width,
                (heightMode == MeasureSpec.EXACTLY )? heightSize : height);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int cCount = getChildCount();
        int cWidth = 0;
        int cHeight = 0;
        MarginLayoutParams cParams = null;

        /**
         * 遍历 所有的childView ，根据childView的宽以及 margin，然后分别将 0,1,2,3 位置的View
         * 分别设置在四个角。
         */
        for (int i = 0; i < cCount; i++) {
            View childView = getChildAt(i);
            cWidth = childView.getMeasuredWidth();
            cHeight = childView.getMeasuredHeight();
            cParams = (MarginLayoutParams) childView.getLayoutParams();

            int cl = 0, ct = 0, cr = 0, cb = 0;

            switch (i) {
                case 0:
                    cl = cParams.leftMargin;
                    ct = cParams.topMargin;
                    break;
                case 1:
                    cl = getWidth() - cWidth- cParams.leftMargin - cParams.rightMargin;
                    ct = cParams.topMargin;
                    break;
                case 2:
                    cl = cParams.leftMargin;
                    ct = getHeight() - cHeight - cParams.bottomMargin;
                    break;
                case 3:
                    cl = getWidth() - cWidth - cParams.leftMargin - cParams.rightMargin;
                    ct = getHeight() - cHeight - cParams.bottomMargin;
                    break;
                default:
                    break;
            }
            cr = cl+cWidth;
            cb = cHeight +ct;
            childView.layout(cl,ct,cr,cb);
        }
    }
}
```
使用如下：

布局一：宽高 为固定值：

```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <com.admin.view_core.viewGroup.CustomContainer
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:background="#999999">

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#ff4444"
            android:text="0"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#00ff00"
            android:text="1"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#0000ff"
            android:text="2"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#88ffff"
            android:text="3"
            android:textColor="#ffffff"
            android:textSize="20sp" />
    </com.admin.view_core.viewGroup.CustomContainer>
</RelativeLayout>
```

![1560310224510](F:\笔记\android\自定义View 系列\assets\1560310224510.png)

布局二：宽高为 wrap_content 

```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <com.admin.view_core.viewGroup.CustomContainer
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="#999999">

        <TextView
            android:layout_width="150dp"
            android:layout_height="150dp"
            android:background="#ff4444"
            android:text="0"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#00ff00"
            android:text="1"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#0000ff"
            android:text="2"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#88ffff"
            android:text="3"
            android:textColor="#ffffff"
            android:textSize="20sp" />
    </com.admin.view_core.viewGroup.CustomContainer>
</RelativeLayout>
```

![1560310357454](F:\笔记\android\自定义View 系列\assets\1560310357454.png)

布局三：宽高为 match_parent 

```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <com.admin.view_core.viewGroup.CustomContainer
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#999999">

        <TextView
            android:layout_width="150dp"
            android:layout_height="150dp"
            android:background="#ff4444"
            android:text="0"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#00ff00"
            android:text="1"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#0000ff"
            android:text="2"
            android:textColor="#ffffff"
            android:textSize="20sp" />

        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#88ffff"
            android:text="3"
            android:textColor="#ffffff"
            android:textSize="20sp" />
    </com.admin.view_core.viewGroup.CustomContainer>
</RelativeLayout>
```

![1560310428952](F:\笔记\android\自定义View 系列\assets\1560310428952.png)

