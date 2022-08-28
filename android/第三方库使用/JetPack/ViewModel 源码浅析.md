### 前言

ViewMode 是我们日常开发中最常用的组件之一，也是实现 MVVM 模式不可缺少的一环，这篇文章将从使用到源码分析以及常见的一些知识点来分析一下 ViewModel



### 了解 ViewModel

ViewModel 旨在注重生命周期的方式存储和管理界面的相关数据，ViewModel 类可以再发生旋转等配置更改后继续留存。

一般 ViewModel 配合 LiveData / Flow 实现数据驱动，由于 Activity 存在因配置改变而重建的机制，就会造成页面的数据丢失，例如网络数据已经其他数据等，而 ViewModel 可以应对 Activity 应配置而改变的场景，再重建的过程中恢复数据，从而降低用户体验受损。

ViewModel 生命周期如下:

![ ViewModel 随着 Activity 状态的改变而经历的生命周期。](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202207251807778.png)

上图说明了 Activity 经历屏幕旋转而后结束的各种生命周期状态，旁边显示的就是 ViewModel 的生命周期了。

#### ViewModel 的使用

```kotlin
class MyViewModel : ViewModel() {
    private val users: MutableLiveData<List<User>> by lazy {
        MutableLiveData<List<User>>().also {
            loadUsers()
        }
    }

    fun getUsers(): LiveData<List<User>> {
        return users
    }

    private fun loadUsers() {
        // Do an asynchronous operation to fetch users.
    }
}
```

```kotlin
val model: MyViewModel by viewModels()
model.getUsers().observe(this, Observer<List<User>>{ users ->
   // update UI
})
```

#### ViewModel 的创建方式

- 方式1：通过 ViewModelProvider 创建

  ```kotlin
  ViewModelProvider(this).get(WorkViewModel::class.java)
  ```

  也可以使用带工厂的创建方式

  ```kotlin
  ViewModelProvider(this, WorkViewModelFactory()).get(WorkViewModel::class.java)
  
  class WorkViewModelFactory() : ViewModelProvider.Factory {
  
      private val repository = WorkRepository()
  
      override fun <T : ViewModel> create(modelClass: Class<T>): T {
          return WorkViewModel(repository) as T
      }
  }
  ```

- 方式2：使用 Kotlin by 委托属性，实际上也是使用了 ViewModelProvider

  ```kotlin
  private val viewModel by viewModels<UserViewModel>()
  ```

- 方式3：使用 Hilt 进行注入

### ViewModel 源码分析

Viewmodel 创建的方法最终都是通过 `ViewModelProvider` 来完成的，他可以理解为创建 `ViewModel` 的工具类，在创建的时候需要两个参数：

- ViewModelStoreOwner

  对应着 Activity / Fragment 等持有 Viewmode 的宿主，他们内部通过 ViewModelStore 维持一个 ViewModel 的映射表，ViewModelStore 是实现 ViewModel 作用域和数据恢复的关键。

- Factory

  对于于创建 ViewModel 的工厂，如果没有传采用默认的 `NewInstanceFactory` 工厂反射创建 VIewModel 的实例。

创建完 ViewModelProvider 工具类后，就可以调用 get 方法来创建 ViewModel 的实例。get 方法会先从映射表 ViewModelStore 中读取缓存，若没有命中，则通过 VIewModel 的工厂创建实例在缓存到映射表中。

#### ViewModelProvider

```kotlin
//使用默认的工厂创建 ViewModel
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
		this(owner.getViewModelStore(), ...NewInstanceFactory.getInstance());
}

//指定工厂
public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
		this(owner.getViewModelStore(), factory);
}
//记录宿主的 viewmodelStore 和 factory
public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
		mFactory = factory;
		mViewModelStore = store;
}
```

```java
private static final String DEFAULT_KEY =
            "androidx.lifecycle.ViewModelProvider.DefaultKey";

@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    //使用 Default_key + 类名作为缓存的 key
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

/** 通常是 fragment 使用*/
@SuppressWarnings("unchecked")
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
  //先从 viewModelStore 中获取缓存  
  ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
        if (mFactory instanceof OnRequeryFactory) {
            ((OnRequeryFactory) mFactory).onRequery(viewModel);
        }
        return (T) viewModel;
    } 
    //使用 factory 创建 ViewModel
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
    } else {
        viewModel = mFactory.create(modelClass);
    }
   //存储到 viewModelStore 中
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}
```

#### NewInstanceFactory

```java
public static class NewInstanceFactory implements Factory {
    private static NewInstanceFactory sInstance;

    @NonNull
    static NewInstanceFactory getInstance() {
        if (sInstance == null) {
            sInstance = new NewInstanceFactory();
        }
        return sInstance;
    }

 
    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
       //反射创建 ViewModel
        try {
            return modelClass.newInstance();
        }....
    }
}
```

#### by viewModels

`ActivityViewModelLazy`

```kotlin
@MainThread
public inline fun <reified VM : ViewModel> ComponentActivity.viewModels(
    noinline factoryProducer: (() -> Factory)? = null
): Lazy<VM> {
    val factoryPromise = factoryProducer ?: {
        defaultViewModelProviderFactory
    }

    return ViewModelLazy(VM::class, { viewModelStore }, factoryPromise)
}
```

`ViewModelLazy`

```kotlin
public class ViewModelLazy<VM : ViewModel> (
    private val viewModelClass: KClass<VM>,
    private val storeProducer: () -> ViewModelStore,
    private val factoryProducer: () -> ViewModelProvider.Factory
) : Lazy<VM> {
    private var cached: VM? = null

    override val value: VM
        get() {
            val viewModel = cached
            //如果第一次调用 by viewModels，则先初始化再返回
            return if (viewModel == null) {
                val factory = factoryProducer()
                val store = storeProducer()
                //最终是通过 ViewModelProvider 来创建
                ViewModelProvider(store, factory).get(viewModelClass.java).also {
                    cached = it
                }
            } else {
                //否则直接返回
                viewModel
            }
        }

    override fun isInitialized(): Boolean = cached != null
}
```

#### ViewModelStoreOwner

ViewModel 的宿主是 ViewModelStoreOwner 接口的实现类，例如 ComponentActivity，Fragment 等

```kotlin
public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}
```

该接口的实现的责任就是在配置期间保留拥有的 ViewModelStore，并在销毁的时候

此接口实现的责任是在配置更改期间保留拥有的 ViewModelStore 并在此范围将被销毁的时候调用 `ViewModelStore.clear()`

##### ComponentActivity

```kotlin
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        ContextAware,
        LifecycleOwner,
        ViewModelStoreOwner.... {
          
   static final class NonConfigurationInstances {
        Object custom;
        ViewModelStore viewModelStore;
    }      

    //viewmodel 的存储容器
    private ViewModelStore mViewModelStore;
    //创建 viewmodel 的工厂      
    private ViewModelProvider.Factory mDefaultFactory;
          
       @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        //.....
        ensureViewModelStore();
        return mViewModelStore;
    }
          
    void ensureViewModelStore() {
        if (mViewModelStore == null) {
            //先从配置文件中获取，看能不能获取到 
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                mViewModelStore = nc.viewModelStore;
            }
            //如果没有获取到则重新创建 ViewModelStore
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
    }
    //重建时保存 viewModelStore
    // ViweModelStore 会被封装为 NonConfigurationInstances 类，然后保存在 NonConfigurationInstances 类的 Object activity 属性中。
   //前一个 NonConfigurationInstances 是 ComponentActivity 中定义的，后一个是 Activity 类中定义的，不是同一个类，不要搞混了哟!       
    public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();
        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }
        if (viewModelStore == null && custom == null) {
            return null;
        }

        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        nci.viewModelStore = viewModelStore;
        return nci;
    }              
}
```

上面代码中的 `NonConfigurationInstances` 是一个配置文件的实例，当 activity 重建时，最终会调用到 `onRetainNonConfigurationInstance()` 方法中对 `viewModelStore` 进行缓存。所以上面才是先尝试从配置文件中获取，最后再创建新的 ViewModelStore。

##### Fragment

```kotlin
@NonNull
@Override
public ViewModelStore getViewModelStore() {
	return mFragmentManager.getViewModelStore(this);
}
```



```kotlin
//fragment 中 ViewModel 的映射
private final HashMap<String, ViewModelStore> mViewModelStores = new HashMap<>();

@NonNull
ViewModelStore getViewModelStore(@NonNull Fragment f) {
    return mNonConfig.getViewModelStore(f);
}

 @NonNull
 ViewModelStore getViewModelStore(@NonNull Fragment f) {
 		ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
 		if (viewModelStore == null) {
       viewModelStore = new ViewModelStore();
       mViewModelStores.put(f.mWho, viewModelStore);
    }
    return viewModelStore;
 }
```



### 关于 ViewModel 的一些问题

1. #### ViewModel 如何实现不同的作用域

   在使用 ViewModelProvider 时，需要传入一个 `ViewModelStoreOwner` 接口，这个接口的 `getViewModelStore` 会返回对应的 ViewModelStore 实例。

   对于 Activity 来说，ViewModelStore 是直接保存在成员变量中的。

   对于 Fragment 来说， ViewModelstore 是间接的存储在 FragmentManagerViewModel 中的 map 中。

   这样就实现了不同的 activity 或者 fragment 分别对应不同的 ViewModelStore 实例，进而区分不同的作用域

2. #### 为什么 Activity 可以再重建后恢复 viewMdoel

   当 Activity 因为配置而发生重建时，我们可以将页面上的数据分为两类：

   1. 配置数据，例如窗口大小，主题资源等，当配置发生改变后，需要重新读取这些配置，因此这些数据在配置改变后就失去了意义，也就没有存在的价值

   2. 非配置数据，这些数据就是一些用户自己的信息，以及页面上显示的数据，这些数据和配置没有关系，如果丢失掉就会造成比较大的用户体验。

   说以，Activity 再重建时支持恢复非配置的数据，整个过程如下：

   1. 重建时保存数据

      Activity 再重建时会调用 `retainNonConfigurationInstances` 方法，在里面会获取需要保存的数据，例如 fragment ，activity 等数据，最后打包为 `NonConfigurationInstances` 类，保存在 `ActivityClientRecord` 中。

      ```kotlin
      NonConfigurationInstances retainNonConfigurationInstances() {
         //activity 中的非配置数据，例如 viewmodelStore 
         //该方法需要子类实现 ，例如 ComponentActivity 
         Object activity = onRetainNonConfigurationInstance();
          //fragment 中的非配置数据
          FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();
          // .....
      		//构建 NonConfigurationInstances
          NonConfigurationInstances nci = new NonConfigurationInstances();
          nci.activity = activity;
          nci.fragments = fragments;
          return nci;
      }
      ```

   2. 恢复数据

      Activity 重新启动的最后，会通过 ActivityThread 中的 performLaunchActivity() 来完成整个启动过程，再这个方法中会通过类加载器来创建 Activity 对象，并调用 attach 方法为其关联所需要的一些信息。

      我们需要关注的就是 `attach` 方法：

      ```java
      final void attach(NonConfigurationInstances lastNonConfigurationInstances //.. ) {
        	//.....
          mLastNonConfigurationInstances = lastNonConfigurationInstances;
      }
      ```

      最后，数据被保存在了 Activity . mLastNonConfigurationInstances 成员变量中。

   3. 获取数据

      这个我们之前已经分析过了，我们简单回顾一下

      ```java
      void ensureViewModelStore() {
          if (mViewModelStore == null) {
             //获取之前保存的数据
              NonConfigurationInstances nc =
                      (NonConfigurationInstances) getLastNonConfigurationInstance();
              if (nc != null) {
                  // Restore the ViewModelStore from NonConfigurationInstances
                  mViewModelStore = nc.viewModelStore;
              }
              if (mViewModelStore == null) {
                  mViewModelStore = new ViewModelStore();
              }
          }
      }
      
      @Nullable
      public Object getLastNonConfigurationInstance() {
        //mLastNonConfigurationInstances 就是 attach 中保存的
        return mLastNonConfigurationInstances != null
          ? mLastNonConfigurationInstances.activity : null;
      }
      
      ```

   至此，就完成了 ViewModel 的数据恢复了。

3. #### Activity 重建的过程

   再 Activity 重建时，系统会执行 Relaunch 重建过程。在这个过程中通过 `ActivityClientRecord` 来完成信息传递，并销毁 Activity，紧接着马上重建同一个 Activity。

   这些操作都是在 ActivityThread 中完成的：

   ```java
   private void handleRelaunchActivityInner(ActivityClientRecord r //...) {
       final Intent customIntent = r.activity.mIntent;
       //处理 onPause
       performPauseActivity(r, false, reason, null /* pendingActions */);
       //处理 onStop
       callActivityOnStop(r, true /* saveState */, reason);
       //1
       handleDestroyActivity(r.token, false, configChanges, true, reason);
   		//2
       handleLaunchActivity(r, pendingActions, customIntent);
   }
   //1
   public void handleDestroyActivity(IBinder token, boolean finishing,//...) {
       ActivityClientRecord r = performDestroyActivity(token, finishing,
                   configChanges, getNonConfigInstance, reason);
   }      
   ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
               int configChanges, boolean getNonConfigInstance, String reason) {
           ActivityClientRecord r = mActivities.get(token);
           if (r != null) {
               if (getNonConfigInstance) {
                 //调用 activity 的 retainNonConfigurationInstances 方法
                 r.lastNonConfigurationInstances
                               = r.activity.retainNonConfigurationInstances();
              }
           }
           return r;
   }
   //2
   public Activity handleLaunchActivity(ActivityClientRecord r,
               PendingTransactionActions pendingActions, Intent customIntent) {
           final Activity a = performLaunchActivity(r, customIntent);
           return a;
   }
   private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
          
     activity = mInstrumentation.newActivity(
                       cl, component.getClassName(), r.intent);
     //...
     //传递缓存数据以及其他数据
     activity.attach(appContext, this, getInstrumentation(), r.token,
                           r.ident, app, r.intent, r.activityInfo, title, r.parent,
                           r.embeddedID, r.lastNonConfigurationInstances, config,
                           r.referrer, r.voiceInteractor, window, r.configCallback,
                           r.assistToken);
     //....
     return activity;
   }
   ```

   上面的代码主要可以分为两部分：

   第一处：再处理 `onDestory` 逻辑时，调用 `retainNonConfigurationInstances()` 方法获取非配置数据，并临时保存在 `ActivityClientRecord` 上。

   第二处：再 Launch 新 activity 的时候通过 `attach` 方法将数据传到新 activity 中即可

   至此旧的 `Activity` 数据已经被传递到新的 `Activity` 中了。

4. #### ViewModel 的数据在什么时候才会清除

   ViewModel 的数据会在 Activity 非配置变化销毁时清除，具体分为三种情况

   1. 直接调用 finish 或者按返回键退出
   2. 异常退出 Activity，例如内存不足
   3. 强制退出应用

   前两种都属于非配置变更触发的，再 Activity中存在一个 Lifecycle 的监听，当 Activity 进入 Destory 状态时，如果 Activity 不处于配置重建阶段，将调用 `viewModelStore.clear()` 清除 viewmodel 数据。

   ```java
   public ComponentActivity() {
       Lifecycle lifecycle = getLifecycle();
       getLifecycle().addObserver(new LifecycleEventObserver() {
           @Override
           public void onStateChanged(@NonNull LifecycleOwner source,
                   @NonNull Lifecycle.Event event) {
               if (event == Lifecycle.Event.ON_DESTROY) {
                   // Clear out the available context
                   mContextAwareHelper.clearAvailableContext();
                   //是否处于配置变更引起的重建
                   if (!isChangingConfigurations()) {
                       getViewModelStore().clear();
                   }
               }
           }
       });
   }
   ```

5. #### ViewModel 和 onSaveInstanceState 对比

   这两种都是对数据恢复的机制，但是他们针对的场景不同，导致他们的实现原理也不同，进而优缺点也不同

   viewModel：使用常见针对于配置变更中的非配置数据恢复，由于数据是直接存储在内存中的，所以他的读取速度非常快，并且支持存储大数据，但是会收到内存空间的限制

   onSaveInstanceState：针对于应用被系统回收后重建时的数据恢复，由于应用进程坑会在这个过程中消亡，所以不能存在内存中，只能进行持久化存储，并且这种方式的数据传递是通过 Bundle 传递的，会受到 Binder 事务缓冲区的大小限制，只能存储小规模数据。

   这里借用[一张大佬的图](https://juejin.cn/post/7076274407416528909#heading-16)，来看一下具体的优缺点：

   ![https://juejin.cn/post/7121998366103306254#heading-12](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202207261524464.awebp)

### 总结

到这里，ViewModel 就整个分析完了，如果有任何问题可直接留言评论，谢谢！

### 参考

- [官方文档](https://developer.android.google.cn/topic/libraries/architecture/viewmodel?hl=zh_cn)
- [彭旭锐(为什么 Activity 都重建了 ViewModel 还存在？)](https://juejin.cn/post/7121998366103306254#heading-13)