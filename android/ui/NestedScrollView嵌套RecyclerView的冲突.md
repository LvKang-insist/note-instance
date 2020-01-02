使用NestedScrollView 嵌套 RecyclerView 出现 RecyclerView不能滑动到最底部

布局如下：

```java
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.nuli100.changmao.chebangyang.module.find.video.VideoActivity">
    <NestedScrollView
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
            <FrameLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content">

                <android.support.v7.widget.AppCompatImageView
                    android:id="@+id/activity_a_imageview"
                    android:layout_width="match_parent"
                    android:layout_height="296dp"
                    android:scaleType="fitXY"/>

                <android.support.v7.widget.RecyclerView
                    android:id="@+id/activity_video_recycler"
                    android:layout_width="match_parent"
                    android:layout_height="match_parent"
                    />
        </FrameLayout>

	</NestedScrollView>
</LinearLayout>	
```
​	这样写的好处就是在 RecyclerView 滑动的时候上面的ImageView 也会跟着滑动。

### 解决

#### 1，设置RecyclerView 为不可滑动

​	recyclerView.setNestedScrollingEnabled(false);

### 2，自定义NestedScrollView


​	
	public class MyNestedScrollView extends NestedScrollView {
	private int mDownX;
	private int mDownY;
	private int mTouchSlop;
	
	public MyNestedScrollView(Context context) {
	    super(context);
	    mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
	}
	
	public MyNestedScrollView(Context context, AttributeSet attrs) {
	    super(context, attrs);
	    mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
	}
	
	public MyNestedScrollView(Context context, AttributeSet attrs, int defStyleAttr) {
	    super(context, attrs, defStyleAttr);
	    mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
	}
	
	@Override
	public boolean onInterceptTouchEvent(MotionEvent e) {
	    int action = e.getAction();
	    switch (action) {
	        case MotionEvent.ACTION_DOWN:
	            mDownX = (int) e.getRawX();
	            mDownY = (int) e.getRawY();
	            break;
	        case MotionEvent.ACTION_MOVE:
	            int moveY = (int) e.getRawY();
	            if (Math.abs(moveY - mDownY) > mTouchSlop) {
	                return true;
	            }
	    }
	    return super.onInterceptTouchEvent(e);
	}
	}



### 3,给 MyNestedScrollView 控件添加如下属性

​	android:fillViewport="true"

#### 4，设置 recyclerView 的条目最外层layout的高度为 warp_content 即可。




​	