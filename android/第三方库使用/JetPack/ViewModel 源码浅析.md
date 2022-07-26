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

ViewModel 的宿主是 ViewModelStoreOwner 接口的实现类，例如 Activity





