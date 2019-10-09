	**在日常开发中，我们经常会使用到 对话框。原来一直使用的就是普通的 dialog。但是最近看到了 DialogFragment。于是就研究了一下，他本是也是一个 fragmen，它具有自己的生命周期，只是内部绑定了一个 dialog。经过几天的使用之后，我对他做了一个 封装。满足了日常的需求。**



## 			下面看一下封装的过程

### 1，BaseFragDialog

```java
public class BaseFragDialog extends DialogFragment {

    public static final Handler HANDLER = new Handler();
    private Object mView;
    private View mRootView;
    private float mAlpha;
    private boolean mAutoDismiss;
    private boolean mCancelable;
    private Window window;
    private int mAnimation;
    private int mGravity;
    private SparseArray<OnListener> mClickArray;
    private SparseArray<String> mSetText;
    private SparseArray<String> mSetImage;

    BaseFragDialog(Object view, float alpha, boolean autoDismiss, boolean cancelable, int animation, int gravity) {
        this.mView = view;
        this.mAlpha = alpha;
        this.mAutoDismiss = autoDismiss;
        this.mCancelable = cancelable;
        this.mAnimation = animation;
        this.mGravity = gravity;
        mClickArray = new SparseArray<>();
        mSetText = new SparseArray<>();
        mSetImage = new SparseArray<>();
    }

    public static BaseFragDialog newInstance(Object view, float alpha,
                                             boolean mAutoDismiss, boolean cancelable,
                                             int animation, int gravity) {
        return new BaseFragDialog(view, alpha, mAutoDismiss, cancelable, animation, gravity);
    }


    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        window = Objects.requireNonNull(getDialog()).getWindow();
        if (window != null) {
            if (mView instanceof Integer) {
                this.mRootView = inflater.inflate((Integer) mView, (ViewGroup) window.findViewById(android.R.id.content), false);
            } else if (mView instanceof View) {
                this.mRootView = (View) mView;
            } else {
                throw new NullPointerException("Not Layout File ");
            }
            create();
        }
        return mRootView;
    }

    public static DialogBuilder<BaseFragDialog> Builder() {
        return new DialogBuilder();
    }

    /**
     * 设置背景遮盖层开关
     */
    public void setBackgroundDimEnabled(boolean enabled) {
        if (window != null) {
            if (enabled) {
                window.addFlags(WindowManager.LayoutParams.FLAG_DIM_BEHIND);
            } else {
                window.clearFlags(WindowManager.LayoutParams.FLAG_DIM_BEHIND);
            }
        }
    }

    /**
     * 设置背景遮盖层的透明度（前提条件是背景遮盖层开关必须是为开启状态）
     */
    public void setBackgroundDimAmount(float dimAmount) {
        if (window != null) {
            window.setDimAmount(dimAmount);
        }
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if (mSetText != null) {
            mSetText.clear();
            mSetText = null;
        }
        if (mClickArray != null) {
            mClickArray.clear();
            mClickArray = null;
        }
        if (mRootView != null) {
            mRootView = null;
        }
    }

    /**
     * 对点击事件进行处理
     */
    private final class ViewOnClick implements View.OnClickListener {
        private final BaseFragDialog dialog;
        private final OnListener listener;

        ViewOnClick(BaseFragDialog dialog, OnListener listener) {
            this.dialog = dialog;
            this.listener = listener;
        }

        @Override
        public void onClick(View view) {
            if (!mAutoDismiss) {
                listener.onClick(dialog, view);
            }
        }
    }

    /**
     * 对事件进行监听，
     */
    public interface OnListener {
        void onClick(BaseFragDialog dialog, View view);
    }

    /**
     * 设置 文本
     *
     * @param Id      id
     * @param strings 内容
     */
    public BaseFragDialog setText(@IdRes int Id, String strings) {
        mSetText.put(Id, strings);
        return this;
    }

    /**
     * 设置 图片
     *
     * @param Id  id
     * @param url 内容
     */
    public BaseFragDialog setImageUrl(@IdRes int Id, String url) {
        mSetImage.put(Id, url);
        return this;
    }

    /**
     * 监听事件
     */
    public BaseFragDialog setListener(int id, OnListener listener) {
        mClickArray.put(id, listener);
        return this;
    }


    private void setLocation() {
        WindowManager.LayoutParams attributes = window.getAttributes();
        attributes.alpha = mAlpha;
        attributes.gravity = mGravity;
        window.setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        window.setAttributes(attributes);
    }

    /**
     * 延时发送，在指定的时间执行
     */
    public void postAtTime(long uptimeMillis, Runnable run) {
        HANDLER.postDelayed(run, uptimeMillis);
    }

    private void create() {
        setLocation();
        initView(mRootView);
        setCancelable(mCancelable);
        window.setWindowAnimations(mAnimation);
        for (int i = 0; i < mSetText.size(); i++) {
            View viewById = mRootView.findViewById(mSetText.keyAt(i));
            if (viewById instanceof AppCompatTextView) {
                ((AppCompatTextView) viewById).setText(mSetText.valueAt(i));
            } else if (viewById instanceof AppCompatButton) {
                ((AppCompatButton) viewById).setText(mSetText.valueAt(i));
            }
        }

        for (int i = 0; i < mClickArray.size(); i++) {
            mRootView.findViewById(mClickArray.keyAt(i))
                    .setOnClickListener(new ViewOnClick(this, mClickArray.valueAt(i)));
        }
        for (int i = 0; i < mSetImage.size(); i++) {
            ImageView image = mRootView.findViewById(mSetImage.keyAt(i));
            Glide.with(this)
                    .load(mSetImage.valueAt(i))
                    .diskCacheStrategy(DiskCacheStrategy.ALL)
                    .into(image);
        }
    }

    /**
     * 空实现，如果dialog的逻辑过于复杂，则可以继承此类，实现此方法。
     * 这个方法可用于绑定 view 进行一些初始化等操作
     */
    public void initView(View view) {

    }
}
```

 	继承自 DialogFragment ，最基础的类，这个类中封装了一些常用的方法，如 setText() 等 可以直接对控件进行赋值，只需要传入对应的 id 即可，不需要手动的 findViewById ，注释已经写在上面了，这个类需要由 DialogBuilder来创建。

### 2，DilalogBuilder

```java
public class DialogBuilder<T> {

    private Object view;
    private float mAlpha = 1;
    private boolean mAutoDismiss = false;
    private boolean mCancelable = true;
    private int mAnimation = 0;
    private int mGravity;

    public Class<T> tClass;

    public DialogBuilder() {
    }

    public DialogBuilder(Class<T> tClass) {
        this.tClass = tClass;
    }

    /**
     * 设置布局资源，可以为 ID，也可以是 View
     */
    public DialogBuilder<T> setContentView(Object view) {
        this.view = view;
        return this;
    }

    /**
     * 设置透明度透明度
     *
     * @param alpha 从 0 - 1
     */
    public DialogBuilder<T> setAlpha(float alpha) {
        this.mAlpha = alpha;
        return this;
    }

    /**
     * 若为 true 所有的点击事件都不起作用，否则相反
     */
    public DialogBuilder<T> setAutoDismiss(boolean autoDismiss) {
        this.mAutoDismiss = autoDismiss;
        return this;
    }

    /**
     * 若为 false，对话框不可取消
     */
    public DialogBuilder<T> setCancelable(boolean cancelable) {
        this.mCancelable = cancelable;
        return this;
    }

    /**
     * 设置动画
     */
    public DialogBuilder<T> setAnimation(@StyleRes int animation) {
        this.mAnimation = animation;
        return this;
    }

    /**
     * 设置对话框位置
     */
    public DialogBuilder<T> setGravity(int gravity) {
        this.mGravity = gravity;
        return this;
    }

    /**
     * @return 对话框的实例
     */
    public T build() {
        if (tClass != null) {
            try {
                Constructor<T> constructor = tClass.getConstructor(
                        Object.class, float.class, boolean.class, boolean.class, int.class,int.class);
                return constructor.newInstance(view, mAlpha, mAutoDismiss, mCancelable,mAnimation,mGravity);
            } catch (Exception e) {
                throw new RuntimeException("创建 "+tClass.getName()+" 失败，原因可能是构造参数有问题："+e.getMessage());
            }
        } else {
            return (T) BaseFragDialog.newInstance(view, mAlpha, mAutoDismiss, mCancelable,mAnimation,mGravity);
        }
    }
}
```

​	通过此类构建 BaseFragDialog ，可以设置一些关于对话框的常用属性。

​	通过上面这两个类就可以展示出一个 dialog 了。使用如最上面的 2 。但是如果对话框中的 View 或者 事件非常多，逻辑非常复杂，使用这种就会显得代码非常的长，而且逻辑也不太好控制。

------

## 使用：

### 	使用 一：直接进行使用

布局文件：

```java
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="center"
    app:cardBackgroundColor="#ffffff"
    app:cardCornerRadius="15dp"
    app:cardElevation="0px">
    <LinearLayout
        android:layout_width="260dp"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:paddingTop="20dp">

        <www.testdemo.com.utils.SmartTextView
            android:id="@+id/tv_dialog_title"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center_horizontal"
            android:textColor="#333333"
            android:textSize="16sp"
            android:textStyle="bold"
            tools:text="标题" />

        <TextView
            android:id="@+id/tv_dialog_message"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="10dp"
            android:layout_marginLeft="15dp"
            android:layout_marginRight="15dp"
            android:layout_marginTop="10dp"
            android:gravity="center"
            android:lineSpacingExtra="4dp"
            android:textColor="#333333"
            android:textSize="14sp"
            tools:text="内容" />

        <View
            android:layout_width="match_parent"
            android:layout_height="1dp"
            android:layout_marginTop="10dp"
            android:background="#ECECEC" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="48dp"
            android:orientation="horizontal">

            <www.testdemo.com.utils.SmartTextView
                android:id="@+id/tv_dialog_cancel"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_weight="1"
                android:background="@drawable/selector_transparent"
                android:focusable="true"
                android:gravity="center"
                android:text="取消"
                android:textColor="#F44336"
                android:textSize="14sp" />

            <View
                android:id="@+id/v_message_line"
                android:layout_width="1dp"
                android:layout_height="match_parent"
                android:background="#ECECEC" />

            <TextView
                android:id="@+id/tv_dialog_confirm"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_weight="1"
                android:background="@drawable/selector_transparent"
                android:focusable="true"
                android:gravity="center"
                android:text="确定"
                android:textColor="#007AFF"
                android:textSize="14sp" />
        </LinearLayout>

    </LinearLayout>
</android.support.v7.widget.CardView>
```

TextView 的按压效果

```
<!-- 透明背景按压效果样式 -->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 按压状态 -->
    <item android:drawable="@color/black5" android:state_pressed="true" />
    <!-- 选中状态 -->
    <item android:drawable="@color/black5" android:state_selected="true" />
    <!-- 焦点状态 -->
    <item android:drawable="@color/black5" android:state_focused="true" />
    <!-- 默认状态 -->
    <item android:drawable="@color/transparent" />
</selector>
```

调用如下

```java
 BaseFragDialog.Builder()
                .setContentView(R.layout.dialog)
                .setCancelable(false)// 对话框不可取消
                .setAnimation(R.style.BottomAnimStyle)//动画
                .setGravity(Gravity.CENTER)//位置
                .setAutoDismiss(false) //是否禁用点击事件
                .build()
                .setText(R.id.tv_dialog_title, "我是标题")
                .setText(R.id.tv_dialog_message, "我是内容")
                .setListener(R.id.tv_dialog_cancel, new BaseFragDialog.OnListener() {
                    @Override
                    public void onClick(BaseFragDialog dialog, View view) {
                        dialog.dismiss();
                    }
                })
                .setListener(R.id.tv_dialog_confirm, new BaseFragDialog.OnListener() {
                    @Override
                    public void onClick(BaseFragDialog dialog, View view) {
                        dialog.dismiss();
                    }
                })
                .show(getSupportFragmentManager(), "");
```

还有动画文件

```java
 <!-- 底部弹出动画 -->
    <style name="BottomAnimStyle" parent="android:Animation">
        <item name="android:windowEnterAnimation">@anim/dialog_bottom_in</item>
        <item name="android:windowExitAnimation">@anim/dialog_bottom_out</item>
    </style>
```

效果如图：

![sadf](assets\sadf.gif)

### 使用二：自定义继承自BaseFragDialog

​	如果你的对话框中逻辑比较复杂，可以看一下这种方式，例如：日期对话框

```java
public class DateDialog extends BaseFragDialog implements View.OnClickListener {

    private TextView tv1;
    private TextView tv2;

    public DateDialog(Object view, float alpha, boolean autoDismiss, boolean cancelable, int animation, int gravity) {
        super(view, alpha, autoDismiss, cancelable, animation, gravity);
    }

    @Override
    public void initView(View view) {
        tv1 = view.findViewById(R.id.tv_dialog_cancel);
        tv2 = view.findViewById(R.id.tv_dialog_confirm);
        tv1.setOnClickListener(this);
        tv2.setOnClickListener(this);

        TextView title = view.findViewById(R.id.tv_dialog_title);
        TextView message = view.findViewById(R.id.tv_dialog_message);
        title.setText("我是日期对话框");
        message.setText("我是时间戳："+ System.currentTimeMillis());
    }

    @Override
    public void onClick(View view) {
        switch (view.getId()) {
            case R.id.tv_dialog_cancel:
                Toast.makeText(getContext(), "取消", Toast.LENGTH_SHORT).show();
                dismiss();
                break;
            case R.id.tv_dialog_confirm:
                Toast.makeText(getContext(), "成功", Toast.LENGTH_SHORT).show();
                dismiss();
                break;
        }
    }

    /**
     * 继承的类必须写此方法，泛型为当前类
     */
    public static DialogBuilder<DateDialog> DateBuilder() {
        return new DialogBuilder<>(DateDialog.class);
    }   
}
```

​	直接继承 BaseFragDialog ，将逻辑直接写在里面，如果需要传入数据的话直接写 set 方法即可，然后在 initView()中进行使用。

​	调用如下

```java
DateDialog.DateBuilder()
                .setContentView(R.layout.dialog)//设置 view
                .setCancelable(false) //对话框不可取消
                .setAlpha(1) // 透明度
                .setAutoDismiss(true) //是否禁用所有的点击事件
                .setGravity(Gravity.CENTER) //对话框的位置
                .setAnimation(R.style.BottomAnimStyle) // 添加动画
                .build()
                .show(getSupportFragmentManager(),"d1");
```

布局，动画使用一中的一样

效果如图

![sadf](assets\sadf-1570627768436.gif)

### 使用三：实现 提示框，如成功，失败，警告，加载中等。

布局如下：

```xml
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="center"
    android:gravity="center"
    android:orientation="vertical"
    app:cardBackgroundColor="#dc000000"
    app:cardCornerRadius="15dp"
    app:cardElevation="0px">

    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:minWidth="110dp"
        android:minHeight="110dp"
        android:orientation="vertical">

        <ImageView
            android:id="@+id/iv_toast_icon"
            android:layout_width="60dp"
            android:layout_height="60dp"
            android:layout_marginTop="6dp"
            android:layout_marginBottom="6dp"
            android:src="@drawable/ic_dialog_warning" />

        <www.testdemo.com.utils.ProgressView
            android:id="@+id/pw_progress"
            android:layout_width="60dp"
            android:layout_height="60dp"
            app:barColor="@android:color/white"
            app:barWidth="2dp"
            app:fillRadius="false"
            app:linearProgress="true"
            app:progressIndeterminate="true"
            android:visibility="gone"
            />

        <TextView
            android:id="@+id/tv_toast_message"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginLeft="16dp"
            android:layout_marginRight="16dp"
            android:layout_marginBottom="6dp"
            android:maxLines="3"
            android:textColor="@color/white"
            android:textSize="14sp"
            tools:text="提示语" />
    </LinearLayout>

</android.support.v7.widget.CardView>
```

ProgressView：[点击查看详情](https://github.com/Todd-Davies/ProgressWheel)

继承 BaseFragmetnDialog

```java
public class ToastDialog extends BaseFragDialog {

    private Type mType;
    private String mMessage;
    private ImageView icon;
    private TextView message;
    private long mUptimeMillis;
    private ProgressView progressView;
    
    public ToastDialog(Object view, float alpha, boolean autoDismiss, boolean cancelable, int animation, int gravity) {
        super(view, alpha, autoDismiss, cancelable, animation, gravity);
    }

    public static DialogBuilder<ToastDialog> ToastBuilder() {
        return new DialogBuilder<>(ToastDialog.class);
    }

    @Override
    public void initView(View view) {
        icon = view.findViewById(R.id.iv_toast_icon);
        message = view.findViewById(R.id.tv_toast_message);
        progressView = view.findViewById(R.id.pw_progress);
        if (mType == null) {
            throw new NullPointerException("没有设置类型，请调用 setType() 设置类型");
        }
        crate();
        //取消背景遮罩
        setBackgroundDimEnabled(false);
    }

    private void crate() {
        icon.setVisibility(View.VISIBLE);
        progressView.setVisibility(View.GONE);
        switch (mType) {
            case FINISH:
                icon.setImageResource(R.drawable.ic_dialog_finish);
                break;
            case ERROR:
                icon.setImageResource(R.drawable.ic_dialog_error);
                break;
            case WARN:
                icon.setImageResource(R.drawable.ic_dialog_warning);
                break;
            case LOADING:
                icon.setVisibility(View.GONE);
                progressView.setVisibility(View.VISIBLE);
                break;
        }
        if (mMessage == null) {
            message.setVisibility(View.GONE);
        } else {
            message.setVisibility(View.VISIBLE);
            message.setText(mMessage);
        }
        if (mUptimeMillis > 0) {
            postAtTime(mUptimeMillis, new Runnable() {
                @Override
                public void run() {
                    dismiss();
                }
            });
        }
    }

    /**
     * 设置显示类型
     *
     * @param type 类型
     */
    public ToastDialog setType(Type type) {
        this.mType = type;
        return this;
    }

    /**
     * 设置消息内容
     */
    public ToastDialog setMessage(String message) {
        this.mMessage = message;
        return this;
    }

    /**
     * 延时关闭
     *
     * @param uptimeMillis 时间，毫秒为单位
     */
    public ToastDialog postAtTime(long uptimeMillis) {
        this.mUptimeMillis = uptimeMillis;
        return this;
    }

    /**
     * 显示的类型
     */
    public enum Type {
        // 完成，错误，警告,  加载中
        FINISH, ERROR, WARN ,LOADING
    }
}
```



图：

![ic_dialog_error](ic_dialog_error.png)

![ic_dialog_finish](ic_dialog_finish.png)

![ic_dialog_warning](ic_dialog_warning.png)

使用如下：

```java
 ToastDialog.ToastBuilder()
                .setContentView(R.layout.dialog_toast)
                .setCancelable(false)
                .build()
                .setType(ToastDialog.Type.LOADING)
                .setMessage("加载中...")
                .show(getSupportFragmentManager(),"toast");
```

效果如图：

![sadf](assets\sadf-1570628082817.gif)

![1570628235468](assets\1570628235468.png)

![1570628315027](assets\1570628315027.png)

![1570628360754](assets\1570628360754.png)



 

------

​	以上就是 简单的封装和使用，其实里面还可以添加很多东西，这就看自己的使用了，如果需要某些配置，直接在 BaseFragDialog 中添加即可。

​	是不是非常简单呢！