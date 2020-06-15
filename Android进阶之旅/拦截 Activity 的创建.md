```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
  
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        //1
        int result = ActivityTaskManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

从 startActivity 开始执行，就会执行到 execStartActivity 方法，

在 execStartActivity 方法中 获取了 ActivityTaskManager 的 service ，然后调用了 service 的 startActivity 方法

```java
/** @hide */
public static IActivityTaskManager getService() {
    return IActivityTaskManagerSingleton.get();
}

@UnsupportedAppUsage(trackingBug = 129726065)
private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
            @Override
            protected IActivityTaskManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                return IActivityTaskManager.Stub.asInterface(b);
            }
        };
```

从 getService 方法可以看出，这个 Service 是通过 AIDL 跨进程的。上面调用的 startActivity 肯定就是服务端的 startActivity 方法了。

而 服务端的 aidl 实现 就是 ActivityTaskManagerService 类了。我们需要去通过动态代理拦截 他的 startActivity 方法，所以我们要找到 ActivityTaskManagerService 的 aidl 接口 也就是  IActivityTaskManager  接口

那么 startActivity 是由谁调用的呢，从上面代码可以看出 getService 里面最终调用的是 Singleton 的 get 方法，下面看一下：

```java
public abstract class Singleton<T> {
    @UnsupportedAppUsage
    private T mInstance;

    protected abstract T create();

    @UnsupportedAppUsage
    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```

get 方法 返回的 T 是一个单例，并且在 创建 singleton的时候传入的是 IActivityTaskManager ，这本身就是一个 aidl 接口。

所以在最终调用 startActivity 的就是 IActivityTaskManager 的实现类  ActivityTaskManagerService  。最终我们如果要调用 startActivity 就需要获取到  get 方法的结果，也就是 mInstance 属性值。

下面看一下拦截的流程：

------

拦截：

​	1，获取 ActivityTaskManager 中的 IActivityTaskManagerSingleton 属性

​	2， 获取 IActivityTaskManagerSingleton 中的 mInstance  属性