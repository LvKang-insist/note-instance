##### 每个app都有一个application实例，如果我们没有继承他，app就会创建一个默认的实例。
##### application有这个和app一样的长的生命周期，当app开启的时候，application的实例就会创建，app销毁的时候也会随之销毁。下面我们看一下他的使用方法。

```
public class Myapplication extends Application {

    final String TAG = "MyAppcliation";

    @Override
    public void onCreate() {

        /**
         *在应用程序启动之前，在
         创建任何其他应用程序*对象之前调用。实现应该尽可能快
         *（例如使用状态的延迟初始化），因为
         在此函数中花费的时间直接影响
         在进程中启动*第一个活动，服务或接收器的性能。
         *如果重写此方法，请务必调用super.onCreate（）。
         */
        //这个函数是当程序刚开始的时候就会被调用，在程序刚开始的时候执行

        Log.e(TAG, "onCreate: ");
        super.onCreate();
    }

    @Override
    public void onTerminate() {
        Log.e(TAG, "onTerminate: ");
        super.onTerminate();
    }

    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        Log.e(TAG, "onConfigurationChanged: ");
        super.onConfigurationChanged(newConfig);
    }

    @Override
    public void onLowMemory() {
        Log.e(TAG, "onLowMemory: ");
        super.onLowMemory();
    }

    @Override
    public void onTrimMemory(int level) {
        Log.e(TAG, "onTrimMemory: ");
        super.onTrimMemory(level);
    }
}

```
1，首先，onCreate方法在Appliaction创建的时候调用，一般用于初始化一些东西，在这里不应该做过多的任务，如果任务过多就会直接影响我们第一个activity/service。如果你要重写这个方法必须调用super.onCreate()。

2，onTerminate ：这个方法在程序结束的时候会调用，但是这个方法只用于在Android仿真机测试的时候，在android产品机上是不会调用的，所以这个方法并没有什么用。

3，onConfigurationChanged：重写此方法可以监听App一些配置信息的改变事件（如屏幕旋转）。当配置改变时会调用这个方法，这Manifest文件下的Activity标签里面配置 android:configChanges 相应的属性，会是activity配置在改变时不会冲洗，只会执行onConfigurationChanged()方法，如   android:configChanges="keyboardHidden|orientation|screenSize"可以是activity旋转是不重启.

4,onLowMemory:这个方法的作用是监听系统整体内存较低的时刻，当系统内存比较低时 会调用这个方法。

5，onTrimMemory：通知 应用程序 当前内存使用情况（以内存级别进行识别）




---
应用场景

从这个类的方法可以看出，Application类的应用场景有：
- 初始化 应用程序，如全局的对象，环境配置等。
- 数据共享，数据缓存，设置全局共享变量，方法等。
- 获取应用程序当前的内存使用情况，意识释放资源，从而避免被系统杀死。
- 监听应用程序配置信息的改变，如屏幕旋转等。
- 监听应用程序内所有Activity生命周期




---
具体使用：
1，继承Application类

```
public class Myapplication extends Application {

    final String TAG = "MyApplication";


    public String  My(){
        return TAG;
    }
}
```
2，在配置中定义Application的子类

```
<application
        ......
        android:name=".Myapplication"
        tools:ignore="GoogleAppIndexingWarning">
        
</application>
```
3,使用自定义的Application类实例


```

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Myapplication application = (Myapplication) getApplication();
        Log.e("onCreate", "onCreate: ;application.My()");
        ......
```
结果如下：

```
E/onCreate: onCreate: ;MyApplication
```


以上就是Application的生命周期和简单的介绍。

> 如有错误，还请指出。