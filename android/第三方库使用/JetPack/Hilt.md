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

### Hilt 中的组件

| Hilt 组件                 | 对应 Android 类活动的范围                 |
| ------------------------- | ----------------------------------------- |
| ApplicationComponent      | Application                               |
| ActivityRetainedComponent | ViewModel                                 |
| ActivityComponent         | Activity                                  |
| FragmentComponent         | Fragment                                  |
| ViewComponent             | View                                      |
| ViewWithFragmentComponent | View annotated with @WithFragmentBindings |
| ServiceComponent          | Service                                   |

Hilt 没有为 broadcast receivers 提供组件，因为 Hilt 直接进从 ApplicationComponent 中注入 broadcast receivers。

Hilt 会根据相应的 Android 类生命周期自动创建和销毁组件的实例，对应关系如下：

| Hilt 提供的组件           | 创建对应的生命周期     | 创建对应的生命周期      |
| ------------------------- | ---------------------- | ----------------------- |
| ApplicationComponent      | Application#onCreate() | Application#onDestroy() |
| ActivityRetainedComponent | Activity#onCreate()    | Activity#onDestroy()    |
| ActivityComponent         | Activity#onCreate()    | Activity#onDestroy()    |
| FragmentComponent         | Fragment#onAttach()    | Fragment#onDestroy()    |
| ViewComponent             | View#super()           | View destroyed          |
| ViewWithFragmentComponent | View#super()           | View destroyed          |
| ServiceComponent          | Service#onCreate()     | View destroyed          |

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

### 使用你 Hilt 进行依赖注入