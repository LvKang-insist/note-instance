JetPack 的新成员，App Startup 笔记

一般情况下我们初始化 SDK 都会提供一个初始化的 api，我们需要在 Application 进行初始化，如下：

这个对开发者很不友好，有些 SDK 初始化的时候可能比较耗时，可能会造成 APP 启动慢等一系列问题。还有一种做法是通过 ConetntProvider 进行初始化：

```kotlin
public class Sdk1InitializeProvider extends ContentProvider {
    @Override
    public boolean onCreate() {
        Sdk1.init(getContext());
        return true;
    }
    ...
}
```

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="cn.zhengzhengxiaogege.sdk1">
    <application>
        <provider
            android:authorities="${applicationId}.init-provider"
            android:name=".Sdk1InitializeProvider"
            android:exported="false"/>
    </application>
</manifest>
```

如果项目有很多要初始化 SDK ，都放在一个 ContentProvicer 中，就会导致代码量增加，而且会造成性能问题。

APP Startup 可以有效的解决这个问题





### 使用

依赖

```kotlin
// App startup
api "androidx.startup:startup-runtime:1.0.0-alpha01"
```

- 初始化**第一个** SDK 

   - 首先需要建立一个包装类，对 SDK 进行初始化

      ```kotlin
      object LvHttpInit {
          private const val TAG = "LvHttpInit"
          fun init(application: Application) {
              LvHttp.Builder()
                  .setApplication(application)
                  .setBaseUrl("https://api.github.com/")
                  //是否开启缓存
                  .isCache(false)
                  .setService(Api::class.java)
                  //是否打印 log
                  .isLoging(true)
                  //对 Code 异常的处理，可自定义,参考 ResponseData 类
                  .setErrorDispose(ErrorKey.ErrorCode, ErrorValue {
                      Toast.makeText(application, "Code 错误", Toast.LENGTH_SHORT).show()
                  })
                  //全局异常处理，参考 Launch.kt ，可自定义异常处理，参考 ErrorKey 即可
                  .setErrorDispose(ErrorKey.AllEexeption, ErrorValue {
                      it.printStackTrace()
                      Toast.makeText(application, "网络错误", Toast.LENGTH_SHORT).show()
                  })
                  .build()
              Log.e(TAG, "init: 完成")
          }
      }
      ```

   - 接着实现 Initializer 接口，进行初始化

      ```kotlin
      /**
       * 泛型T为待初始化的Sdk对外提供的对象类型
       */
      class LvHttpInitializer : Initializer<LvHttpInit> {
           /**
           * Sdk初始化逻辑写入的地方，
           * 其参数context为Application Context，同时需要返回一个Sdk对外提供的对象实例。
           */
          override fun create(context: Context): LvHttpInit {
              LvHttpInit.init(context.applicationContext as Application)
              return LvHttpInit
          }
       	/**
           * 该方法则需要返回一个列表，这个列表需要给出一个该Sdk依赖的其它Sdk的初始化器，
           * 也就是这个列表决定了哪些sdk会在这个sdk之前初始化，
           * 如果这个sdk是独立的没有依赖与其它的sdk，可以将该方法返回一个空列表，如当前
           */
          override fun dependencies(): MutableList<Class<out Initializer<*>>> {
              return Collections.emptyList()
          }
      }
      ```

   - 最后 注册Provider和Initializer<?>

      我们需要告诉 startup 我们实现了那些 sdk 的初始化器(LvHttpInitializer)，同时 startup 没有提供清单文件，所以 startup 中用到的 provider 需要进行注册，如下：

      ```xml
       <!--     初始化项目-->
              <provider
                  android:name="androidx.startup.InitializationProvider"
                  android:authorities="com.lv.admin.androidx-startup"
                  android:exported="false"
                  tools:node="merge">
                  <meta-data
                      android:name="com.lv.admin.LvHttpInitializer"
                      android:value="@string/androidx_startup" />
              </provider>
      ```

- 初始化第二个 SDK

   - 包装类

      ```kotlin
      object UtilsInit {
          private const val TAG = "UtilsInit"
          fun init(application: Application){
              //初始化 Toast
              ToastUtils.init(application)
      
              //初始化 Log
              XLog.init(
                  LogConfiguration.Builder()
                      .t() //允许打印线程信息
                      .tag("345")
                      .build()
              )
              Log.e(TAG, "init: 完成")
          }
      }
      ```

   - 接着实现 Initializer 接口，进行初始化

      ```kotlin
      class UtilsInitializer : Initializer<UtilsInit> {
          override fun create(context: Context): UtilsInit {
              UtilsInit.init(context.applicationContext as Application)
              return UtilsInit
          }
      
          /**
           * 在 ComponentInitializer 之后初始化 当前类
           */
          override fun dependencies(): MutableList<Class<out Initializer<*>>> {
              val dependencies = arrayListOf<Class<out Initializer<*>>>()
              dependencies.add(ComponentInitializer::class.java)
              return Collections.emptyList()
          }
      
      }
      ```

   - 注册

      ```xml
       <!--        初始化项目-->
              <provider
                  android:name="androidx.startup.InitializationProvider"
                  android:authorities="com.lv.admin.androidx-startup"
                  android:exported="false"
                  tools:node="merge">
                  <meta-data
                      android:name="com.lv.admin.LvHttpInitializer"
                      android:value="@string/androidx_startup" />
                  <meta-data
                      android:name="com.lv.admin.UtilsInitializer"
                      android:value="@string/androidx_startup" />
              </provider>
      ```



### 懒加载



### 优缺点

优点

- 减少多个 sdk 导致 application 和 清单文件需要频繁改动的问题，减少 application 和 清单中的代码量，方便维护
- 提供了所有sdk使用同一个ContentProvider做初始化的能力，减少性能损耗，并精简了sdk的使用流程
- 符合面向对象中类的单一职责原则
- 有效解耦，方便协同开发

缺点

- 会通过反射实例化Initializer<>的实现类，在低版本系统中会有一定的性能损耗
- 导致类文件增多，特别是有大量需要初始化的sdk存在时。
- 版本较低，还没有发行正式版。