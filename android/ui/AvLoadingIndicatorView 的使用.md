AVLoadingIndicatorView 是非常炫酷的加载动画，使用起来也非常方法。今天对他进行一个封装。

1，添加依赖

```java
//Loading 依赖
api 'com.wang.avi:library:2.1.3'
```

2, 设置一些加载的样式，这些在 github 上面都可以找到

```java
public enum  LoaderStyle {
    /**
     *  加载样式
     */
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

3, 通过反射获取到 需要使用 加载动画

```java
public class LoaderCreator {
    //进行缓存
    private static final WeakHashMap<String,Indicator> LOADING_MAP = new WeakHashMap<>();

    static AVLoadingIndicatorView creawte(String type, Context context){
        final AVLoadingIndicatorView avLoadingIndicatorView = new AVLoadingIndicatorView(context);
        if (LOADING_MAP.get(type) == null){
            //根据传入的 type 进行反射 获取动画 
            final Indicator indicator = getIndicator(type);
            LOADING_MAP.put(type,indicator);
        }
        //设置 这个动画
        avLoadingIndicatorView.setIndicator(LOADING_MAP.get(type));
        return avLoadingIndicatorView;
    }

    private static Indicator getIndicator(String name){
        if (name == null || name.isEmpty()){
            return null;
        }
        final StringBuilder drawbleClassName = new StringBuilder();
        //如果类名不包含 . 就说明传入的是一个 整个的类名，否则就是包名
        if (!name.contains(".")){
            // 拼装一个完整的路径，这个路径就是 动画类所在的位置。
            final String defultPackageName = AVLoadingIndicatorView.class.getPackage().getName();
            drawbleClassName.append(defultPackageName)
                    .append(".indicators")
                    .append(".");
        }
        drawbleClassName.append(name);
        try {
            //通过反射 拿到给定的类
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

4,设置样式。我们要将loading 显示在 dialog 上面,在style.xml 中添加如下代码：

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

5,创建 一个测量的类。用于适应屏幕

```java
public class DimenUtil {
    /**
     * @return 可用显示大小的绝对宽度(以像素为单位)
     */
    public static int getScreenWidth(){
        //这里的 getAppContext 是获取Context
        final Resources resources = Coke.getAppContext().getResources();
        final DisplayMetrics dm = resources.getDisplayMetrics();

        return dm.widthPixels;
    }
    /**
     * @return 可用显示大小的绝对高度(以像素为单位)
     */
    public static int getScreenHeight(){
        final Resources resources = Coke.getAppContext().getResources();
        final DisplayMetrics dm = resources.getDisplayMetrics();
        return dm.heightPixels;
    }
}
```

5, 最后一步，设置 AVLoading 

```java
public class CokeLoading {
    //缩放比，让load根据屏幕的大小 来调整大小。
    private static final int LOADER_SIZE_SCALE = 8;

    //屏幕的偏移量
    private static final int LOADER_OFFSET_SCALT = 10;

    private static final ArrayList<AppCompatDialog> LOADERS = new ArrayList<>();

    private static final String DEFULT_LOADER = LoaderStyle.BallClipRotatePulseIndicator.name();

    public static void showLoading(Context context, Enum<LoaderStyle> type) {
        showLoading(context, type.name());
    }

    public static void showLoading(Context context, String type) {
        // 使用 dialog 来承载 Loading .
        // 尽量 使用 v7包下的东西，这样兼容性比较好
        final AppCompatDialog dialog = new AppCompatDialog(context, R.style.dialog);
        //通过creawte方法 设置 样式 并返回对象
        final AVLoadingIndicatorView avLoadingIndicatorView = LoaderCreator.creawte(type, context);
        dialog.setContentView(avLoadingIndicatorView);

        int deviceWidth = DimenUtil.getScreenWidth();
        int deviceHeight = DimenUtil.getScreenHeight();

        final Window dialogWindow = dialog.getWindow();
        if (dialogWindow != null) {
            //设置 dialog 的属性
            final WindowManager.LayoutParams lp = dialogWindow.getAttributes();
            lp.alpha = 0.4f;
            lp.width = deviceWidth / LOADER_SIZE_SCALE;
            lp.height = deviceHeight / LOADER_SIZE_SCALE;
            //偏移量，会将上面的 height 个给覆盖掉
            lp.height = lp.height + deviceHeight / LOADER_OFFSET_SCALT;
            lp.gravity = Gravity.CENTER;
        }
        LOADERS.add(dialog);
        dialog.show();
    }

    public static void showLoading(Context context) {
         //自定义 一个默认的 加载动画
        showLoading(context, DEFULT_LOADER);
    }

    public static void stopLoading() {
        for (AppCompatDialog dialog : LOADERS) {
            if (dialog != null) {
                if (dialog.isShowing()) {
                    //  dismiss() 只是单纯的消失掉 dialog 而 cancel 会有一些回调，所以使用cancel
                    dialog.cancel();
                }
            }
        }
    }
}
```

7,使用如下：

```java
//使用 指定的动画
CokeLoading.showLoading(getContext(),LoaderStyle.SemiCircleSpinIndicator);
//使用默认的
CokeLoading.showLoading(getContext());

//关闭如下
 CokeLoading.stopLoading();
```

效果如下：

![1558061688465](F:\笔记\android\杂\assets\1558061688465.png)

![1558061850087](F:\笔记\android\杂\assets\1558061850087.png)

到这里就已经封装完成了。是不是使用起来非常 方便呢。赶紧试一试吧！