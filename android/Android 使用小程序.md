### 前言

对于小程序这个词，对于普通的用户而言，确实是一个非常方便快捷的东西，不需要下载app就可以直接使用应用，使用完了可以随时离开，不需要关心安装太多的应用。但是对于 Android 开发者而言，在小程序刚开始的时候，传出了非常大的地震，例如 "小程序时代崛起，App 即将被消灭" 等等，但是这么长时间过去了， App 依然好好的，因为 `小程序` 目前之恩能够针对那些使用低频率、服务类型、轻量级的 App ，因此，小程序只能被视为一个简单的轻应用，在功能和体验上并不能和原生的相媲美。

虽然小程序并不能取代 App，但是小程序天生就具有它独特的优势，就是使用方便并且非常轻量，随着这几年的发展，小程序的技术也在飞速提升，越来越多的应用都集成了小程序框架，现在更多的产品会首选小程序来完成，因为足够轻量，使用足够方便，推广页容易，也更容易提升用户数量，并且小程序是支持多个平台的，从动态化的角度而言，小程序也绝对是首选。

所以，如何选择一个合适的小程序框架是一个非常重要的事情，故此，本篇文章将重点介绍 FinClip小程序容器框架   ，并快速的集成到 App 中。



### 小程序原生应用的区别

小程序本身就是一种更轻量的应用程序，与传统的应用程序不同，它无需安装，支持跨平台，可以内嵌到 原生的 App 中。目前的主流 App 都具备了这种能力，例如微信，支付宝和抖音等等。

在我看来，小程序就像是 App 的助手，通过 App 这个平台来使用小程序，从而达到更多的内容，吸引更多的用户，同时也增加了用户的沉淀。

小程序具有的优势：

- 具备跨平台能力，一套代码可以在 Android 和 IOS 中运行
- 远超 H5 的体验，有丰富的组件支持，可以获得更多的系统权限
- 相比较原生，小程序的开发难度较低，通常使用的是 vue 等语言，比较方便上手。
- 易于推广，推广成本比较低，并且因为轻量的特性，用户留存也比较好
- ........

### Finclip是什么?

![image-20230218185421313](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202302181950624.png)

FinClip 是凡泰极客推出的一个小程序容器，可以让 App 具有运行小程序能力的一种技术，任何一个 App 都可以通过 FinClip 小程序来获得运行小程序的能力，它具有如下特性：

1. 兼容微信小程序语法与登录体系，微信的小程序代码可以直接在 FinClip 中运行。
2. 轻量的小程序 SDK ，App 集成核心 SKD 后的打包体积不会超过 3MB。
3. 完善的开发工具，通过 FinClip 的 IDE 可以完成小程序的设计，调试，预览和开发等一系列操作。
4. 面向业务的生命周期管理。
5. 小程序一键转为 App。
6. 支持 Android，IOS，Flutter，Reatct Native， Windows等



### FinClip 组成与工作部分

Finclip 分别由 云侧，端侧与开发者工具组成：


<img src="https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202302181950302.png" alt="image-20230218194707648" style="zoom:50%;" />

- 云端代表 [FinClip](https://www.finclip.com/login/) 小程序管理后台，可以管理小程序的开发，上架等生命周期
- 端侧代表 FinClip 小程序的 SDK ，用于向其提供能够运行小程序的能力3
- 开发者工具 主要用于编写，调试，上传，预览小程序代码

FinClip 小程序 SDK 提供了一套可以运行小程序业务代码的安全沙箱与宿主环境

<img src="https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202302181955785.png" alt="image-20230218195530720" style="zoom:50%;" />

- 安全沙箱负责保护小程序中的业务应用，在安全可信的环境中传输数据，避免第三方的干扰和窃听
- 宿主环境负责执行小程序 JS 业务逻辑代码，并使用 WebView 渲染小程序页面。



### 集成 FinClip

在项目的 build.gradle 中添加 maven 仓库地址，新项目需要在  setting.gradle 中进行添加

```groovy
maven {
    url "https://gradle.finogeeks.club/repository/applet/"
    credentials {
        username "applet"
        password "123321"
    }
}
```

接着在 app 下的 build.gradle 中添加对 finapplet 的依赖：

```groovy
implementation 'com.finogeeks.lib:finapplet:2.39.7' //可查询最新的版本，目前最新为 2.39.7
```

> 需要注意的是 SDK 中的动态库是被加固过的，被加固过的动态库在编译打包时不能被压缩，否则在加载的时候会报错
>
> 因此，需要在 App Model 下的 build.gradle 中增加 doNotStrip 配置
>
> ```
> packagingOptions {
>       // libsdkcore.so、libfin-yuvutil.so是被加固过的，不能被压缩，否则加载动态库时会报错
>       doNotStrip "*/x86/libsdkcore.so"
>       doNotStrip "*/x86_64/libsdkcore.so"
>       doNotStrip "*/armeabi/libsdkcore.so"
>       doNotStrip "*/armeabi-v7a/libsdkcore.so"
>       doNotStrip "*/arm64-v8a/libsdkcore.so"
>       
>       doNotStrip "*/x86/libfin-yuvutil.so"
>       doNotStrip "*/x86_64/libfin-yuvutil.so"
>       doNotStrip "*/armeabi/libfin-yuvutil.so"
>       doNotStrip "*/armeabi-v7a/libfin-yuvutil.so"
>       doNotStrip "*/arm64-v8a/libfin-yuvutil.so"
> }
> ```

完整配置如下：

```kotlin
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-kapt'
}

android {
    compileSdk 31

    defaultConfig {
        applicationId "com.example.demotest"
        minSdk 21
        targetSdk 29
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled true
            debuggable true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
    buildFeatures {
        dataBinding = true
    }

    packagingOptions {
        // libsdkcore.so、libfin-yuvutil.so是被加固过的，不能被压缩，否则加载动态库时会报错
        doNotStrip "*/x86/libsdkcore.so"
        doNotStrip "*/x86_64/libsdkcore.so"
        doNotStrip "*/armeabi/libsdkcore.so"
        doNotStrip "*/armeabi-v7a/libsdkcore.so"
        doNotStrip "*/arm64-v8a/libsdkcore.so"

        doNotStrip "*/x86/libfin-yuvutil.so"
        doNotStrip "*/x86_64/libfin-yuvutil.so"
        doNotStrip "*/armeabi/libfin-yuvutil.so"
        doNotStrip "*/armeabi-v7a/libfin-yuvutil.so"
        doNotStrip "*/arm64-v8a/libfin-yuvutil.so"
    }
}

dependencies {

    implementation 'com.finogeeks.lib:finapplet:2.39.7'

 	//......
}
```

**混淆处理**

```
-keep class com.finogeeks.** {*;}
```

### 初始化

SDK 采用多进程来实现，每个小程序运行在独立的进程中，也就是一个小程序对应一个进程，在初始化的时候需要注意的是：**小程序在创建的时候不需要执行任何的初始化操作，所以需要再 Application 中进行判断，如果是小程序进程·，则不初始化别的组件即可。**

整个初始化如下所示：

```kotlin
class App : Application() {

    override fun onCreate() {
        super.onCreate()
        if(FinAppClient.isFinAppProcess(this)){
            //小程序进程不需要执行任何初始化操作
            return
        }
        initFinApp()
    }

    private fun initFinApp() {
        // 服务器信息集合
        // 服务器信息集合
        val storeConfigs = arrayListOf<FinStoreConfig>()
        val config1 = FinStoreConfig(
            "22LyZEib0gLTQdU3MUauATBwgfnTCJjdr7FCnywmAEM=",   // SDK Key
            "bdfd76cae24d4313",   // SDK Secret
            "https://api.finclip.com",   // 服务器地址
            "",   // 数据上报服务器地址
            "/api/v1/mop/",   // 服务器接口请求路由前缀
            "",
            "MD5"   // 加密方式，国密:SM，md5: MD5（推荐）
        )
        storeConfigs.add(config1)
      val config =  FinAppConfig.Builder()
            .setFinStoreConfigs(storeConfigs)
            .build()

        val callback = object :FinCallback<Any?>{
            override fun onSuccess(p0: Any?) {
                Log.e("---345--->", "SDK 初始化成功");
            }

            override fun onError(p0: Int, p1: String?) {
                Log.e("---345--->", "SDK 初始化失败");
            }

            override fun onProgress(p0: Int, p1: String?) {
            }
        }

        FinAppClient.init(this,config,callback)
    }
}
```

上面中，SDK key 等都是采用的官方例子中的，如果自己有小程序项目的则可以直接使用自己的 key。



### 小程序简单使用

- 直接打开小程序

  ```
  binding.start.setOnClickListener {
      FinAppClient.appletApiManager.startApplet(
          this,
          IFinAppletRequest.Companion.fromAppId("5fa214a29a6a7900019b5cc1"), null
      )
  }
  ```

  滴啊用 startApplet 打开小程序即可，第二个参数为小程序 id。

- 带参数打开小程序

  ```kotlin
  // path为小程序页面路径
  // query为启动参数，内容为"key1=value1&key2=value2 ..."的形式
  FinAppClient.appletApiManager.startApplet(
      this,
      IFinAppletRequest.fromAppId("apiServer", "appId")
          .setStartParams(mapOf(
              "path" to "/pages/index/index",
              "query" to "aaa=test&bbb=123"
          ))
  )
   
  ```

- 二维码打开小程序

  ```kotlin
  /**
   * 通过参数封装对象来启动小程序
   * @param request 参数封装，包括RemoteFinAppletRequest, QrCodeFinAppletRequest, LocalFinAppletRequest, DecryptFinAppletRequest,
   * 建议使用IFinAppletRequest.fromAppId, IFinAppletRequest.fromQrCode, IFinAppletRequest.fromLocal, IFinAppletRequest.fromDecrypt等方法生成对应的request对象。
   */
  fun startApplet(context: Context, request: IFinAppletRequest, callback: FinCallback<String>? = null)
  ```

  将二维码解析出来的内容传递进去即可

最后来看一下打开小程序的效果，由于是gif图，并且进行了压缩，所以不会那么流畅：

![6fd25e5a0594c09f40a86543b040b4b4](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202302191801561.gif)



### 总结

通过上面的代码我们也可以看到，在 Android 项目上接入小程序容器非常容易，使用起来也特别简单。通过我的实际使用，流畅度也非常  nice。

从现在这个环境来看，小程序的发展已经是一种趋势了，而且未来也一片光明，目前主流的各大 App 都早已经在使用小程序了，通过 `FinClip` 普通的 App 也可以将小程序集成到自己的产品中，这点确实非常好。

最为关键的是 `FinClip` 支持了各种平台，我们所熟知的各大平台都可以直接进行使用，当然使用小程序的最终目的也是和自己的产品挂钩，当用户达到一定的体量之后，就需要考虑用户的留存了，小程序在性能和灵活性上取得了较好的平衡，通过充分利用系统 U、线程协作及缓存技术等能够让用户在使用时获得优于 H5、与原生应用近乎一致的交互体验，并且可以根据用户的喜好来推荐对应的小程序，这对用户的留存也非常大。如果你的项目正在考虑使用小程序，那你不妨试一试 `FinClip`，说不定它可以满足你的需求呢！



### 参考

[FinClip官方文档](https://www.finclip.com/mop/document/runtime-sdk/android/api/api-init.html#_1-%E5%88%9D%E5%A7%8B%E5%8C%96)

[小程序的昨日与今天](https://www.finclip.com/blog/wei-shi-yao-yao-shi-yong-xiao-cheng-xu/)

[原来微信小程序已经可以在自己的APP上架运行了](https://mp.weixin.qq.com/s/HW0EUCAecgudlc-4J9aV0A)