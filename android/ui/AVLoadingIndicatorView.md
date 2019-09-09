首先添加依赖

```java
//Loading 依赖
api 'com.wang.avi:library:2.1.3'
```

使用方式一

​	以布局的方式使用

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="400dp"
        android:gravity="center"
        android:background="#888">

        <com.wang.avi.AVLoadingIndicatorView
            android:id="@+id/av"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:visibility="visible"
            app:indicatorName="BallClipRotatePulseIndicator"
            />
    </LinearLayout>

</LinearLayout>
```

这样就可以将 Loader显示出来。

使用方式2：

​	对他进行封装，不使用布局，以代码的形式全部实现

1，以反射的方式 拿到 动画的类：

```java
public final class LoaderCreator{

    //进行缓存
    private static final WeakHashMap<String,Indicator> LOADING_MAP = new WeakHashMap<>();

    static AVLoadingIndicatorView creawte(String type, Context context){
        final AVLoadingIndicatorView avLoadingIndicatorView = new AVLoadingIndicatorView(context);
        if (LOADING_MAP.get(type) == null){
            final Indicator indicator = getIndicator(type);
            LOADING_MAP.put(type,indicator);
        }
        //拿到该 样式
        avLoadingIndicatorView.setIndicator(LOADING_MAP.get(type));
        return avLoadingIndicatorView;
    }

    private static Indicator getIndicator(String name){
        if (name == null || name.isEmpty()){
            return null;
        }
        final StringBuilder drawbleClassName = new StringBuilder();
        //如果类名不包含 . 就说明传入的是一个 整个的类名
        if (!name.contains(".")){
            //拿到 包名
            final String defultPackageName = AVLoadingIndicatorView.class.getPackage().getName();
            drawbleClassName.append(defultPackageName)
                    .append(".indicators")
                    .append(".");
        }
        drawbleClassName.append(name);
        try {
            //通过反射 拿到给定的类没
            final Class<?> drawableClass = Class.forName(drawbleClassName.toString());
            //创建 该对象并返回
            return (Indicator) drawableClass.newInstance();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

2，设置工具类，拿到屏幕可用的宽高

```java
public class DimenUtil {
    /**
     * @return 可用显示大小的绝对宽度(以像素为单位)
     */
    public static int getScreenWidth(){
        final Resources resources = Latte.getApplication().getResources();
        final DisplayMetrics dm = resources.getDisplayMetrics();

        return dm.widthPixels;
    }
    /**
     * @return 可用显示大小的绝对高度(以像素为单位)
     */
    public static int getScreenHeight(){
        final Resources resources = Latte.getApplication().getResources();
        final DisplayMetrics dm = resources.getDisplayMetrics();
        return dm.heightPixels;
    }
}
```

3，设置 style 样式

```java
<resources>
    <style name="dialog" parent="@android:style/Theme.Dialog">
        <!--去掉边框-->
        <item name="android:windowFrame">@null</item>
        <!--悬浮-->
        <item name="android:windowIsFloating">true</item>
        <!--半透明-->
        <item name="android:windowIsTranslucent">false</item>
        <!--不需要标题-->
        <item name="android:windowNoTitle">true</item>
        <!--背景透明-->
        <item name="android:windowBackground">@android:color/transparent</item>
        <!--允许模糊-->
        <item name="android:backgroundDimEnabled">true</item>
        <!--全屏幕-->
        <item name="android:windowFullscreen">true</item>
    </style>
</resources>
```

4，加载Loading ，设置适配屏幕居中，创建缓存，以便重复使用。



```java
public class LatteLoader {

    //缩放比，让load根据屏幕的大小 来调整大小。
    private static final int LOADER_SIZE_SCALE = 8;

    //屏幕的偏移量
    private static final int LOADER_OFFSET_SCALT = 10;

    private static final ArrayList<AppCompatDialog> LOADERS = new ArrayList<>();

    private static final String DEFULT_LOADER = LoaderStyle.BallClipRotatePulseIndicator.name();

    public static void showLoading(Context context,Enum<LoaderStyle> type){
        showLoading(context,type.name());
    }

    public static void showLoading(Context context,String type){
        // 使用 dialog 来承载 Loading .
        // 尽量 使用 v7包下的东西，这样兼容性比较好
        final AppCompatDialog dialog = new AppCompatDialog(context,R.style.dialog);
        //通过creawte方法 设置 样式 并返回对象
        final AVLoadingIndicatorView avLoadingIndicatorView = LoaderCreator.creawte(type,context);
        dialog.setContentView(avLoadingIndicatorView);

        int deviceWidth = DimenUtil.getScreenWidth();
        int deviceHeight = DimenUtil.getScreenHeight();

        final Window dialogWindow = dialog.getWindow();
        if (dialogWindow != null){
            //设置 dialog 的属性
            WindowManager.LayoutParams lp = dialogWindow.getAttributes();
            lp.alpha = 0.4f;
            lp.width = deviceWidth/LOADER_SIZE_SCALE;
            lp.height = deviceHeight/LOADER_SIZE_SCALE;
            //偏移量，会将上面的 height 个给覆盖掉
            lp.height = lp.height+deviceHeight/ LOADER_OFFSET_SCALT;
            lp.gravity = Gravity.CENTER;
        }
        LOADERS.add(dialog);
        dialog.show();
    }

    public static void showLoading(Context context){
         showLoading(context,DEFULT_LOADER);
    }
    public static void stopLoading(){
        for (AppCompatDialog dialog : LOADERS){
            if (dialog.isShowing()){
                //  dismiss() 只是单纯的消失掉 dialog 而 cancel 会有一些回调，所以使用cancel
                dialog.cancel();
            }
        }
    }
}
```

5，创建枚举类，每个对象都是一个 动画类，通过前面的反射可拿到具体的类

```java
public enum LoaderStyle {
    BallPulseIndicator,
    BallGridPulseIndicator,
    BallClipRotateIndicator,
    BallClipRotatePulseIndicator,
    SquareSpinIndicator,
    BallClipRotateMultipleIndicator,
    BallPulseRiseIndicator,
    BallRotateIndicator,
    CubeTransitionIndicator,
    BallZigZagIndicator,
    BallZigZagDeflectIndicator,
    BallTrianglePathIndicator,
    BallScaleIndicator,
    LineScaleIndicator,
    LineScalePartyIndicator,
    BallPulseSyncIndicator,
    BallBeatIndicator,
    LineScalePulseOutIndicator,
    LineScalePulseOutRapidIndicator,
    BallScaleRippleIndicator,
    BallScaleRippleMultipleIndicator,
    BallSpinFadeLoaderIndicator,
    LineSpinFadeLoaderIndicator,
    TriangleSkewSpinIndicator,
    PacmanIndicator,
    BallGridBeatIndicator,
    SemiCircleSpinIndicator
 }
```



6.使用

```java
LatteLoader.showLoading(CONTEXT,LOADER_STYLE);//开启 Loading

LatteLoader.stopLoading(); //关闭loadding
```