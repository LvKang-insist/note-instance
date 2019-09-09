添加依赖

```java
api 'me.yokeyword:fragmentation:0.10.1'
```



fragmentation 常用于 单Activity 和 多Fragment 的场景

使用如下：

1，Activity 继承 SupportActivity

```java
/**
 * Copyright (C)
 * 文件名称: ProxyActivity
 * 创建人: 345
 * 创建时间: 2019/4/15 20:18
 * 描述:  Activity 继承自 SupportActivity
 *        Fragmentation 提供的 SupportActivity
 */
public abstract class ProxyActivity extends SupportActivity {

    /**
     * @return 返回根 Delegate
     */
    public abstract LatteDelegate setRootDelegate();

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        initContainer(savedInstanceState);
    }

    /**
     * 初始化 视图
     * @param savedInstanceState 保存的数据
     */
    private void initContainer(@Nullable Bundle savedInstanceState) {
        @SuppressLint("RestrictedApi") final ContentFrameLayout container = new ContentFrameLayout(this);
        //设置这个视图的Id
        container.setId(R.id.delegate_container);
        setContentView(container);
        if (savedInstanceState == null){
            //SupportActivity 独有的一个方法
            //加载根Fragment 即Activity的第一个Fragment ，或者Fragment内的第一个Fragment
            //即 加载碎片
            loadRootFragment(R.id.delegate_container,setRootDelegate());
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        System.gc();
        System.runFinalization();
    }
}
```

2,Fragment 继承 SupportFragment 或者 SwipeBackFragment：

```java
/**
 * Copyright (C)
 * 文件名称: ProxyActivity
 * 创建人: 345
 * 创建时间: 2019/4/15 20:18
 * 描述:  Activity 继承自 SupportActivity
 *        Fragmentation 提供的 SupportActivity
 */
public abstract class ProxyActivity extends SupportActivity {

    /**
     * @return 返回根 Delegate
     */
    public abstract LatteDelegate setRootDelegate();

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        initContainer(savedInstanceState);
    }

    /**
     * 初始化 视图
     * @param savedInstanceState 保存的数据
     */
    private void initContainer(@Nullable Bundle savedInstanceState) {
        @SuppressLint("RestrictedApi") final ContentFrameLayout container = new ContentFrameLayout(this);
        //设置这个视图的Id
        container.setId(R.id.delegate_container);
        setContentView(container);
        if (savedInstanceState == null){
            //SupportActivity 独有的一个方法
            //加载根Fragment 即Activity的第一个Fragment ，或者Fragment内的第一个Fragment
            //即 加载碎片
            loadRootFragment(R.id.delegate_container,setRootDelegate());
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        System.gc();
        System.runFinalization();
    }
}
```



```java
// 启动模式

//standard是默认的启动模式，在不进行显式指定的情况下，所有的活动都会自动使用这种启动模式。对于standard模式的活动，系统不会在乎这个活动是否已经在返回栈中存在，每次启动的时候都会创建一个该活动的实例。
public static final int STANDARD = 0;
//栈顶复用，当使用这个模式后 启动碎片的时候发现栈顶如果是当前碎片，则直接使用
public static final int SINGLETOP = 1;
//栈内复用，当站内有该碎片时直接复用
public static final int SINGLETASK = 2;
```

常用的方法：

```java
 start(new LauncherScrollDelegate()); //启动目标Fragment
//启动 Fragment，并且不添加到 回退栈
 replaceFragment(new LauncherScrollDelegate(),false);
 //启动目标 Fragment，并pop 当前Fragment
startWithPop(new LauncherScrollDelegate());
pop();//出栈

	/**
     * 加载根Fragment, 即Activity内的第一个Fragment 或 Fragment内的第一个子Fragment
     *
     * @param containerId 容器id
     * @param toFragment  目标Fragment
     */
    @Override
    public void loadRootFragment(int containerId, SupportFragment toFragment) {
        mFragmentationDelegate.loadRootTransaction(getChildFragmentManager(), containerId, 				toFragment);
    }	

	/**
     * 以replace方式加载根Fragment
     */
    @Override
    public void replaceLoadRootFragment(int containerId, SupportFragment toFragment, boolean addToBack) {
        mFragmentationDelegate.replaceLoadRootTransaction(getChildFragmentManager(), containerId, 			toFragment, addToBack);
    }	


	/**
     * @return 位于栈顶的Fragment
     */
    @Override
    public SupportFragment getTopFragment() {
        return mFragmentationDelegate.getTopFragment(getFragmentManager());
    }	

 	/**
     * @return 位于当前Fragment的前一个Fragment
     */
    @Override
    public SupportFragment getPreFragment() {
        return mFragmentationDelegate.getPreFragment(this);
    }

	/**
     * 出栈
     */
    @Override
    public void pop() {
        mFragmentationDelegate.back(getFragmentManager());
    }


```