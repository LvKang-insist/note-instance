	是一个用于处理滚动效果的工具类。其实**任何一个控件都是可以滚动的**，因为 VIew 类中有 scrollTo 和 scrollBy 方法。

​	先看一个效果

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">
    <androidx.appcompat.widget.AppCompatButton
        android:id="@+id/scroll_to"
        android:layout_width="100dp"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:padding="5dp"
        android:text="Scroll_To"
        android:textColor="#000000"
        android:textSize="15sp"
        tools:ignore="HardcodedText" />

    <androidx.appcompat.widget.AppCompatButton
        android:id="@+id/scroll_by"
        android:layout_width="100dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="15dp"
        android:gravity="center"
        android:padding="5dp"
        android:text="Scroll_By"
        android:textColor="#000000"
        android:textSize="15sp"
        tools:ignore="HardcodedText" />
</LinearLayout>
```

```kotlin
scroll_to.setOnClickListener {
    //layout 是父布局LinearLayout
    layout.scrollTo(-60, -100)
}
scroll_by.setOnClickListener {
    layout.scrollBy(-60, -100)
}
```

为什么调用的是 LinearLayout 的 scrollBy 方法，不管是 scrollTo 还是 scrollBy 方法，滚动的都是该 View 内部的内容，而 LinearLayout 中内容就是 两个 Button。如果调用 Button 的 scroll 方法，结果一定不是你想看到的。

效果如下：

![Video_20200410_024544_465](Scroller.assets/Video_20200410_024544_465.gif)

可以看到 点击 scrollTo 是，两个按钮会往下一点，再次点击没有反应了

点击 scrollBy 时往下移动，不停点击就会一直移动。

区别：scrollTo 方法是放 View 相对于初始位置以为某段距离，由于初始位置不会变，虽有不管点多少此都会是同一个位置。而 scrollBy 是相对于当前位置移动某段距离，所以才可以一直往右下方移动。

从图中可以看出没有任何滑动的痕迹，效果就是跳跃式的。通过这个方法是很难完成 ViewPager 类似的效果，因此要借助另外一个关键的工具，也就是 Scroller 。

Scroller 的 用法可以分为以下几个步骤：

1，创建 Scroller 的实例

2，调用 startScroll() 方法来初始化滚动数据并刷新界面

3，重写 computeScroll 方法，并在其内部完成平滑滚动的逻辑

------

下面就按照上述的步骤通过模拟一个 ViewPager 来理解以下 Scroller 的用法

