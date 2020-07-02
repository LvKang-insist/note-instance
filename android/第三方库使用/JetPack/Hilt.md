## Jetpck Dagger-Hilt

### 什么是 IOC

IOC ：控制反转，在 android 中的体现就是依赖注入。

例如：类中有很多成员变量，如果需要用到里面的变量，传统的做法是通过 new 创建对象

但是在 IOC 中，不需要 new，只需要给变量加个注解，在用到的时候就会将对象注入。

### Hilt 是什么

Hilt 是 android 中的依赖注入库，他减少了在项目中进行手动依赖，依赖注入库的出现节省了 Android 开发者的大量时间

Hilt 的目的就是帮助 android 开发者在 开发中更好的使用 Dagger，代理了很多复杂的初始配置，大大的降低了开发成本

### Hilt 常用的注解的含义

- @HiltAndroidApp

  所有使用 Hilt 的 app 必须使用一个使用 @HiltAndroidApp 注解的 Application

  @HiltAndroidApp 将会触发 Hilt 的代码生成，作为程序依赖项容器的基类

  生成的 Hilt 依附于 Application 的生命周期，他是 App 的父组件，提供访问其他组件的依赖

  在 Application 中配置好后，就可以使用 Hilt 提供的组件了；组件包含 Application，Activity，Fragment，View，Service 等。

- @HiltAndroidApp

  创建一个依赖容器，该容器遵循 Android 的生命周期类，目前支持的类型是: Activity, Fragment, View, Service, BroadcastReceiver.

- @Inject

  字段注入，注意字段不能是 private 的

- @Module

  常用于创建依赖的对象（如，Okhttp，Retrofit 等），使用 @Module 注解的类，需啊哟使用 @InstallIn 注解指定 module 的范围

- @InstallIn

  使用 @Module 注入的类，需要使用 @InstallIn 注解指定 module 的范围。

  例如使用 @InstallIn(ActivityComponent::class) 注解的 module 会绑定到 activity 的生命周期上。

- @Provides

  常用于被 @Module 注解标记类的内部方法上。并提供依赖项对象。

-  @EntryPoint

  Hilt 支持最常见的 Android 类 Application、Activity、Fragment、View、Service、BroadcastReceiver 等等，但是您可能需要在Hilt 不支持的类中执行依赖注入，在这种情况下可以使用 @EntryPoint 注解进行创建，Hilt 会提供相应的依赖。

  

### Hilt 中的组件(Compenent)

使用 @Module 注解的类，需要使用 @Installin 注解来指定 module 的范围。

例如 @InstallIn(ApplicationComponent::class) 注解的 Module 就会绑定到 Application 的生命周期上。

Hilt 提供了以下组件来绑定依赖与对应 Android 类的活动范围

| Hilt 组件                 | 对应 Android 类活动的范围                 |
| ------------------------- | ----------------------------------------- |
| ApplicationComponent      | Application                               |
| ActivityRetainedComponent | ViewModel                                 |
| ActivityComponent         | Activity                                  |
| FragmentComponent         | Fragment                                  |
| ViewComponent             | View                                      |
| ViewWithFragmentComponent | View annotated with @WithFragmentBindings |
| ServiceComponent          | Service                                   |

> Hilt 没有为 broadcast receivers 提供组件，因为 Hilt 直接进从 ApplicationComponent 中注入 broadcast receivers。

### Hilt 中组件的生命周期

Hilt 会根据相应的 Android 类生命周期自动创建和销毁组件的实例，对应关系如下：

| Hilt 提供的组件           | 创建对应的生命周期     | 结束对应的生命周期      | 作用范围               |
| ------------------------- | ---------------------- | ----------------------- | ---------------------- |
| ApplicationComponent      | Application#onCreate() | Application#onDestroy() | @Singleton             |
| ActivityRetainedComponent | Activity#onCreate()    | Activity#onDestroy()    | @ActivityRetainedScope |
| ActivityComponent         | Activity#onCreate()    | Activity#onDestroy()    | @ActivityScoped        |
| FragmentComponent         | Fragment#onAttach()    | Fragment#onDestroy()    | @FragmentScoped        |
| ViewComponent             | View#super()           | View destroyed          | @ViewScoped            |
| ViewWithFragmentComponent | View#super()           | View destroyed          | @ViewScoped            |
| ServiceComponent          | Service#onCreate()     | View destroyed          | @ViewScoped            |

### 如何使用 Hilt

```java
buildscript {
    dependencies {
        //hilt
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.28-alpha'
    }
}
```

```java
apply plugin: 'kotlin-kapt'
apply plugin: 'com.xiaojinzi.component.plugin'

//hilt
api "com.google.dagger:hilt-android:2.28-alpha"
kapt "com.google.dagger:hilt-android-compiler:2.28-alpha"

```

```kotlin
@HiltAndroidApp
class BaseApplication : Application() {

    override fun onCreate() {
        super.onCreate()
    }
}
```

到这里准备工作就做完了



### 使用 Hilt 进行依赖注入



### Hilt 在 Android 组件中的使用

- 如果使用 @AndroidEntryPoint 注解 Android 类，还必须注解依赖他的 Android 类；

- 如 给 fragment 使用 @AndroidEntryPoint 后，则还需要给 fragmet 依赖的 Activity 依赖 @AndroidEntryPoint ，否则会出现异常
- @AndroidEntryPoint 不能以来在抽象类上
- @AndroidEntryPoint 注解 仅仅支持 ComponentActivity 的子类，例如 Fragment，AppCompatActivity 等等。

```kotlin
@AndroidEntryPoint
class HomeNavigationActivity : BaseLayoutActivity<TestViewModel>() {

    override fun setViewModel(): Class<TestViewModel> =TestViewModel::class.java

    override fun layout(): Int {
        return R.layout.home_navigation
    }

    override fun bindView() {
        val navHost = NavHostFragment.create(R.navigation.home_graph)
        supportFragmentManager
            .beginTransaction()
            .replace(R.id.home_content, navHost)
            .setPrimaryNavigationFragment(navHost)
            .commit()

    }

}
```

```kotLin
// fragment 中使用，需要本身所依赖的 activity 添加注解
@AndroidEntryPoint
class FragmentOne : BaseLayoutFragment<FragOneViewModel>() {
    
    //使用 @Inject 从组件中获取依赖进行注入
    @Inject
    lateinit var hiltTest: HiltTest

    override fun layout(): Int {
        return R.layout.frag_one
    }
    override fun bindView(rootView: View) {
        //对象已经注入，直接调用即可
        one.text = hiltTest.hiltTest()
    }
}
```

### Hilt 和第三方组件的使用

如果需要在项目中注入第三方依赖，可以使用 @Module 注解。使用 @Module 在注解为普通类，在其中创建第三方依赖的对象即可。

- @Module 模块用于向 Hilt 添加绑定，告诉 Hilt 如果提供不同类型的实例。

  使用了 @Module 的类，相当于是一个模块，常用于创建依赖对象（如，Okhttp，Retrofit 等）。

- 使用 @Module 的类，需要使用 #InstallIn 指定此 module 的范围，会绑定到对应 Android 类的生命周期上

- @Providers，常用于被 @Module 注解标记类的内部方法，并提供依赖项对象。

```kotlin
@Module
@InstallIn(ApplicationComponent::class)
object TestModule {
    
    /**
     * 每次都是新的实例
     */
    @Provides
    fun bindHiltTest(): HiltTest {
        XLog.e("--------bindHiltTest----")
        return HiltTest()
    }

    /**
     * 全局复用同一个实例
     */
    @Provides
    @Singleton
    fun bindSingTest(): Test {
        XLog.e("--------bindSingTest----")
        return Test()
    }

}
```

使用如下：

```kotlin
@Inject
lateinit var hiltTest: HiltTest

@Inject
lateinit var hiltTest1: HiltTest

@Inject
lateinit var test1: Test

@Inject
lateinit var test2: Test
```

其中 bindSingTest 只会被调用一次，@SingLeton 相当于是一个单例



### Hilt 和 ViewModel 的使用

使用之前需要在 app.build 下添加一下对 viewModel的支持

```java
implementation 'androidx.hilt:hilt-lifecycle-viewmodel:1.0.0-alpha01'
kapt 'androidx.hilt:hilt-compiler:1.0.0-alpha01'
```

- 通过 @ViewModelInject 注解进行构造注入。
- SavedStateHandle 使用 @Asssisted 注解

```kotlin
class HomeContentViewModel @ViewModelInject  constructor(
    private val response: HomeContentRepository,
    @Assisted val  state: SavedStateHandle
) : ViewModel() {
    
    private val liveData by lazy { MutableLiveData<String>() }

    val testLiveData: LiveData<String> by lazy { liveData }
    
    fun requestBaiDu() {
        launchVmHttp {
            liveData.postValue(response.requestBaidu())
        }
    }
}
```

- 通过 @Inject 进行注入，在 viewModel 中不需要手动的创建其对象

```kotlin
@ActivityScoped
class HomeContentRepository @Inject constructor() : BaseRepository() {

    suspend fun requestBaidu(): String {
        return LvHttp.createApi(ApiServices::class.java).baidu()
    }
}
```

- 获取 viewModel 的实例

```kotlin
@AndroidEntryPoint
class HomeContentActivity : AppCompatActivity(){

    //生成 ViewModel 的实例
    private val viewModel by viewModels<HomeContentViewModel>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_home_content)


        viewModel.requestBaiDu()
        viewModel.testLiveData.observe(this, Observer {
            ToastUtils.show(it)
        })
}
```

### Hilt 和 Room 的使用

这里需要用到 @Module 注解，使用 @Module 注解的普通类，在其中提供 Room 的实例。并且使用 @InstallIn 来声明 作用范围。

```kotlin
@Module
@InstallIn(ApplicationComponent::class)
object RoomModel {

    /**
     * @Provides：常用于被 @Module 标记类的内部方法，并提供依赖对象
     * @Singleton：提供单例
     */
    @Provides
    @Singleton
    fun provideAppDataBase(application: Application): AppDataBase {
        return Room
            .databaseBuilder(application, AppDataBase::class.java, "knif.db")
            .fallbackToDestructiveMigration()
            .allowMainThreadQueries()
            .build()
    }

    @Provides
    @Singleton
    fun providerUserDao(appDataBase: AppDataBase): UserDao {
        return appDataBase.getUserDao()
    }
}
```

我们给 providerUserDao 使用了 @Provides 注解 和 @Singleton 注解，是为了告诉 Hilt，当使用  UserDao 时需要执行 appDataBase.getUserDao() 。

而在调用  appDataBase.getUserDao()  时需要传入 AppDataBase，这时就会调用上面的方法 provideAppDataBase 了，因为这个方法也是用了  @Provides 注解。

并且这两个方法都是单例，只会调用一次。

**使用如下：**

```kotlin
class FragmentTwo : BaseLayoutFragment<FragTwoViewModel>() {

    @Inject
    lateinit var userDao: UserDao
}
```

到现在为止，就可以在任意地方获取到 UserDao，并且不用手动的创建实例。

### 使用 @Binds 进行接口注入

Binds：必须注释一个抽象函数，抽象函数的返回值是实现的接口。通过添加具有接口实现类型的唯一参数来指定实现。

首先需要一个接口，和一个实现类

```kotlin
interface User {
    fun getName(): String
}
```

```kotlin
class UserImpl @Inject constructor() : User {
    override fun getName(): String {
        return "345"
    }
}
```

接着就需要新建一个 Module。用来实现接口的注入

```kotlin
@Module
@InstallIn(ApplicationComponent::class)
abstract class UserModule {
    @Binds
    abstract fun getUser(userImpl: UserImpl): User
}
```

注意：这个 Module 是抽象的。

使用如下：

```kotlin
@AndroidEntryPoint
class FragmentOne : BaseLayoutFragment<FragOneViewModel>() {

    @Inject
    lateinit var user: User
}
```

### 使用 @Qualifier 提供同一接口，不同的实现

还是上面的 User 接口，有两个不同的实现，如下：

```kotlin
class UserAImpl @Inject constructor() : User {
    override fun getName(): String {
        return "345"
    }
}
```

```kotlin
class UserBImpl @Inject constructor() : User {
    override fun getName(): String {
        return "Lv"
    }
}
```

接着定义两个注解

```kotlin
@Qualifier
annotation class A

@Qualifier
annotation class B
```

然后修改 Module ,在 module 中用来标记相应的依赖。

```kotlin
@Module
@InstallIn(ApplicationComponent::class)
abstract class UserAModule {
    @A
    @Singleton
    @Binds
    abstract fun getUserA(userImpl: UserAImpl): User
}

@Module
@InstallIn(ActivityComponent::class)
abstract class UserBModule {
    @B
    @ActivityScoped
    @Binds
    abstract fun getUserB(userImpl: UserBImpl): User
}
```

这里用了两个不同的 mdule，并且对应两个不同的 component，一个是 application，另一个是 activity

最后使用如下：

```kotlin
@AndroidEntryPoint
class FragmentOne : BaseLayoutFragment<FragOneViewModel>() {

    @A
    @Inject
    lateinit var userA: User

    @B
    @Inject
    lateinit var userB: User
}
```