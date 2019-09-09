Lifecycler 被包含在 support 及之后的包中，如果我们的依赖支持库在 

26.1.0 以上，则不需要额外导入 Lifecycle 库

如果版本下小于 26.1.0 ，就需要单独导入 Lifecycle库：

```
  implementation "android.arch.lifecycle:runtime:1.1.1"
```

如果项目迁移到了 AndroidX，可以用下面的方式引入

```
    implementation "androidx.lifecycle:lifecycle-runtime:2.0.0"
```

如果是 26.1.0 以上，则就是

```
    implementation 'com.android.support:appcompat-v7:28.0.0'
```

使用

1，创建 MyObserver 实现 LifecycleObserver 接口

```java
public class MyObserver implements LifecycleObserver {
    private static final String TAG = "MyObserver";

    //该注解表示方法需要监听指定的生命周期时间
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void connectListener(){
        Log.e(TAG, "connectListener:  onResume" );
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void disconnectListener(){
        Log.e(TAG, "disconnectListener: onPause");
    }
}
```

2, 让 Activity 继承自 AppCompatActivity

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //将 Lifecycle 对象和LifecycleObserver 对象进行绑定
        getLifecycle().addObserver(new MyObserver());

    }
/*

```

只需要一行代码，就可以完成了。

3，如果 Activity 继承的 普通的ivity 呢？使用方法如下

```java
public class MainActivity extends Activity implements LifecycleOwner {

    private LifecycleRegistry mLifecycleRegistry;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mLifecycleRegistry = new LifecycleRegistry(this);
        getLifecycle().addObserver(new MyObserver());
        mLifecycleRegistry.markState(Lifecycle.State.CREATED);
    }

    @Override
    protected void onResume() {
        super.onResume();
        mLifecycleRegistry.markState(Lifecycle.State.RESUMED);
    }

    @Override
    protected void onPause() {
        super.onPause();
        mLifecycleRegistry.markState(Lifecycle.State.RESUMED);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mLifecycleRegistry.markState(Lifecycle.State.DESTROYED);
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}

```

继承自 Activity 后就需要自己创建 Lifecycle 对象，这就需要自己实现 LifecycleOwner 接口，滨海自己进行事件的分发了。

4，Lifecycle 提供了 查询当前组件所处的生命周期的方法：

```
lifecycle.getCurrentState().isAtLeast(STARTED)
```

5，在碎片中使用 Lifecycle ，

​	和在 Activty 中使用没有太大区别，只不过有些生命周期没有，比如 onDestroyView。