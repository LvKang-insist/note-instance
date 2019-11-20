### Tinker基本介绍

​		**Tinker 是微信官方的 Andriod 热补丁解决方案，它支持动态下发代码，so库以及资源，让应用在不需要安装的情况下实现更新，当然，你也可以使用 Thinker 来更新你的插件。**

#### 它主要包含以下几部分：

​		1，gradle 编译插件 tinker-patch-gradle-plugin：主要用于在 as 中直接完成 patch 文件的生成

​		2，核心 sdk 库 tinker-android-lib ：核心库，为应用层提供的 api 

​		3，非 gradle 编译用户的命令行版本，tinker-path-cli.jar ：为 eclipse 做的一个工具

#### 为什么使用 Tinker

​		当前市面上 热修复的解决方案很多，但是他们都有一些无法解决的问题，但是 Tinker 的功能是比较全面的。

|             | Tinker | QZone | AndFix | Robust |
| ----------- | :----: | :---: | :----: | :----: |
| 类替换      |  yes   |  yes  |   no   |   no   |
| So 替换     |  yes   |  no   |   no   |   no   |
| 资源替换    |  yes   |  yes  |   no   |   no   |
| 全平台支持  |  yes   |  yes  |  yes   |  yes   |
| 即时生效    |   no   |  no   |  yes   |  yes   |
| 性能损耗    |  较小  | 较大  |  较小  |  较小  |
| 补丁包大小  |  较小  | 较大  |  一般  |  一般  |
| 开发透明    |  yes   |  yes  |   no   |   no   |
| 复杂度      |  较低  | 较低  |  复杂  |  复杂  |
| gradle 支持 |  yes   |  no   |   no   |   no   |
| Rom 体积    |  较大  | 较小  |  较小  |  较小  |
| 成功率      |  较高  | 较高  |  一般  |  最高  |



### Tinker 执行原理及流程

​		基于 android 原生的 ClassLoader ，开发了自己的 ClassLoader，通过自定义的 ClassLoader去加载 patch 文件中的字节码

​		基于 android 原生的 aapt，开发了自己的 aapt ，加载 资源文件

​		基于 Dex 文件的格式，研发了 DexDiff 算法，通过 DexDiff 算法比较两个 apk 文件中的差异 

### 简单的使用 Tinker 

#### 	1,在项目的gradle.properties 中添加

```java
# tinker版本号 ,控制版本,以下版本已经兼容 9.0
TINKER_VERSION=1.9.14
TINKERPATCH_VERSION=1.2.14
```

#### 			2,在项目的 gradle中添加：

```
classpath("com.tencent.tinker:tinker-patch-gradle-plugin:${TINKER_VERSION}")
```

#### 			3,在 app 中的 gradle 中添加：

```java
   compileOnly("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    annotationProcessor("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    // tinker 的核心 sdk 库
    implementation("com.tinkerpatch.sdk:tinkerpatch-android-sdk:${TINKERPATCH_VERSION}") { changing = true }
```

#### 	4,接着进行初始化，新建一个类用于管理 tinker 的初始化

```java
/**
 * 对 TinkerManager api 做一层封装
 */
public class TinkerManager {

    /**
     * 是否初始化
     */
    private static boolean isInstalled = false;

    private static ApplicationLike mApplike;

    /**
     * 完成 Tinker初始化
     *
     * @param applicationLike
     */
    public static void installTinker(ApplicationLike applicationLike) {
        mApplike = applicationLike;
        if (isInstalled) {
            return;
        }
        //完成 tinker 初始化
        TinkerInstaller.install(mApplike);
        isInstalled = true;
    }

    public static void loadPatch(String path) {
        if (Tinker.isTinkerInstalled()) {
            TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), path);
        }
    }

    /**
     * 通过 ApplicationLike 获取 Context
     */
    private static Context getApplicationContext() {
        if (mApplike != null) {
            return mApplike.getApplication().getApplicationContext();
        }
        return null;
    }

}
```

#### 5,自定义 application 继承自 ApplicationLike

```java
//通过 DefaultLifeCycle 注解来生成我们程序中需要用到的 Application
@DefaultLifeCycle(application = ".MyTinkerApplication",
        flags = ShareConstants.TINKER_ENABLE_ALL,
        loadVerifyFlag = false)
public class CustomTinkerLike extends ApplicationLike {

	public CustomTinkerLike(Application application, int tinkerFlags, boolean tinkerLoadVerifyFlag, long applicationStartElapsedTime, long applicationStartMillisTime, Intent tinkerResultIntent) {
        super(application, tinkerFlags, tinkerLoadVerifyFlag, applicationStartElapsedTime, applicationStartMillisTime, tinkerResultIntent);
    }

    @Override
    public void onBaseContextAttached(Context base) {
        super.onBaseContextAttached(base);
        //初始化
        TinkerManager.installTinker(this);
    }
}
```

为什么需要继承 ApplicationLike，而不直接在 Application 中初始化呢？

​		因为 ApplicationLike 需要对 Application 的生命周期进行监听，	所以他通过 ApplicationLike 进行代理。通过这个代理可以完成对 Application 的生命周期监听，然后在不同的生命周期做一些别的工作，因为Tinker 初始化非常复杂，所用用 ApplicationLike 进行了代理，这样使用就非常简单了！

​		注意上面的注解：**通过这个注解可以生成 需要在程序中进行添加的 application**，

```java
public class MyTinkerApplication extends TinkerApplication {

    public MyTinkerApplication() {
        super(15, "com.testdemo.www.tinker.CustomTinkerLike", "com.tencent.tinker.loader.TinkerLoader", false);
    }
}
```

​		上面这个就是通过注解生成的。我们需要将他添加的 AndroidManifest 中。



#### 6,配置 tinker

```java
//buildDir 代表的是 app 目录下 build 文件夹，
// 如果创建成果，他会在 build 文件夹中创建 bakApk文件夹，存放 old.apk
def bakPath = file("${buildDir}/bakApk") //指定基准文件存放位置

ext {
    tinkerEnable = true
    tinkerID = "1.0"
    tinkerOldApkPath = "${bakPath}/"
    tinkerApplyMappingPath = "${bakPath}/"
    tinkerApplyResourcePath = "${bakPath}/"
}

//是否启用 tinker
def buildWithTinker() {
    return ext.tinkerEnable
}
// old 路径
def getOldApkPath() {
    return ext.tinkerOldApkPath
}
// 混淆文件路径
def getApplyMappingPath() {
    return ext.tinkerApplyMappingPath
}
// 资源文件路径
def getApplyResourceMappingPath() {
    return ext.tinkerApplyResourcePath
}
// id
def getTinkerIdValue() {
    return ext.tinkerID
}

if (buildWithTinker()) {
    //启用 tinker
    apply plugin: 'com.tencent.tinker.patch'

    //所有 tinker 相关的参数配置
    tinkerPatch() {
        oldApk = getOldApkPath() // old.apk 路径
        ignoreWarning = false //不忽略警告，如果警告取消生成patch
        useSign = true  // 强制 patch 文件使用签名
        tinkerEnable = buildWithTinker() //指示是否启用 tinker
        buildConfig() {
            applyMapping = getApplyMappingPath() // 指定old.apk 打包时所使用的的混淆文件
            applyResourceMapping = getApplyResourceMappingPath() // 指定 old.apk 资源文件
            tinkerId = getTinkerIdValue() //指定 TinkerId
            keepDexApply = false //一般置为 false，true：生成patch 的时候会根据 dex 文件的分包去动态的编译 patch 文件
        }

        dex() {
            dexMode = "jar"  //jar 是配到14以下，会将dex压缩为jar文件，然后进行处理，体积小row只能在14以上使用，直接对 dex 文件处理
            pattern = ["classes*.dex", "assets/secondary-dex-?.jar"]//指定 dex 文件位于哪些牡蛎
            loader = ["com.testdemo.www.tinker.MyTinkerApplication"] //指定加载patch文件时所用到的类
        }

        //工程中的 jar 和 so
        lib {
            pattern = ["libs/*/*.so"]
        }

        res { //指定 tinker 可以修改的资源文件路径
            pattern = ["res/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]
            ignoreChange = ["assets/sampple_meta.txt"] //制定不受影响的资源路径，即时修改了这个资源文件，也不会被打入 patch
            largeModSize = 100 //资源修改大小默认值
        }
//-----------------必须项配置完成-------------------------------------
        //patch的介绍
        packageConfig {
            //补丁加载完成后通过 key 可以拿到这些 value
            configField("patchMessage", "fix 1.0 version's bugs")
            configField("patchVersion", "1.0")
        }

        //使用压缩
        sevenZip {
            zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
        }
    }

    //判断一下当前的配置中是否配置了多渠道
    List<String> flavors = new ArrayList<>()
    project.android.productFlavors.each { flavor ->
        flavors.add(flavor.name)
    }
    boolean hasFlavors = flavors.size() > 0

    /**
     * 复制基准包和其它必须文件到指定目录
     */
    android.applicationVariants.all { variant ->
        /**
         * task type, you want to bak
         */
        def taskName = variant.name
        def date = new Date().format("MMdd-HH-mm-ss")

        tasks.all {
            if ("assemble${taskName.capitalize()}".equalsIgnoreCase(it.name)) {
                it.doLast {
                    copy {
                        def fileNamePrefix = "${project.name}-${variant.baseName}"
                        def newFileNamePrefix = hasFlavors ? "${fileNamePrefix}" : "${fileNamePrefix}-${date}"

                        def destPath = hasFlavors ? file("${bakPath}/${project.name}-${date}/${variant.flavorName}") : bakPath
                        from variant.outputs[0].outputFile
                        into destPath
                        rename { String fileName ->
                            fileName.replace("${fileNamePrefix}.apk", "${newFileNamePrefix}.apk")
                        }

                        from "${buildDir}/outputs/mapping/${variant.dirName}/mapping.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("mapping.txt", "${newFileNamePrefix}-mapping.txt")
                        }

                        from "${buildDir}/intermediates/symbols/${variant.dirName}/R.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("R.txt", "${newFileNamePrefix}-R.txt")
                        }
                    }
                }
            }
        }
    }

    project.afterEvaluate {
        if (hasFlavors) {
            task(tinkerPatchAllFlavorRelease) {
                group = 'tinker'
                def originOldPath = getTinkerBuildFlavorDirectory()
                for (String flavor : flavors) {
                    def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Release")
                    dependsOn tinkerTask
                    def preAssembleTask = tasks.getByName("process${flavor.capitalize()}ReleaseManifest")
                    preAssembleTask.doFirst {
                        String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 15)
                        project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release.apk"
                        project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-mapping.txt"
                        project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-R.txt"
                    }
                }
            }

            task(tinkerPatchAllFlavorDebug) {
                group = 'tinker'
                def originOldPath = getTinkerBuildFlavorDirectory()
                for (String flavor : flavors) {
                    def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Debug")
                    dependsOn tinkerTask
                    def preAssembleTask = tasks.getByName("process${flavor.capitalize()}DebugManifest")
                    preAssembleTask.doFirst {
                        String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 13)
                        project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug.apk"
                        project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-mapping.txt"
                        project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-R.txt"
                    }

                }
            }
        }
    }
}
```

注意 ext 中：

 tinkerEnable = true   //是否启用 tinker
 tinkerID = "1.0"  //  id  ,线上的版本 id 和 补丁包的 tinkerID 必须相等

[详细的说明](https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97)

#### **7,进行测试，打包**

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private static final String FILE_END = ".apk";
    private String mPatchDir;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		//创建路径
        mPatchDir = getExternalCacheDir().getAbsolutePath() + "/tpatch/";
        File file = new File(mPatchDir);
        file.mkdir();
        findViewById(R.id.btn).setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
    	//加载补丁包
        TinkerManager.loadPatch(getPatchName());
    }
	//拼装一个路径
    private String getPatchName() {
        return mPatchDir.concat("tinker").concat(FILE_END);
    }

}
```

看一下布局

```java
<androidx.appcompat.widget.LinearLayoutCompat xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <androidx.appcompat.widget.AppCompatButton
        android:id="@+id/btn"
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:layout_marginTop="20dp"
        android:text="加载 PATCH 包"
        android:textSize="20sp"
        tools:ignore="HardcodedText" />

</androidx.appcompat.widget.LinearLayoutCompat>
```

接着我们就可以进行打包了，注意不能是 debug 。是需要签名的。打包完成后会在 build 文件下生成 bakApk 文件夹，里面就是打包的 apk。

![1574064429434](6%EF%BC%8CThinker%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1574064429434.png)

然后把这个apk安装到手机上即可。

#### 8，创建补丁文件

创建补丁文件的时候需要线上的 apk。所以在这里 我们将刚才打包的 apk 名字复制下来，然后放在build.gradle 中的 ext 中，如下所示：

![1574064629024](6%EF%BC%8CThinker%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1574064629024.png)

这里的路径就是 build 中 bakApk 中的文件，第一个是 apk，第二个是混淆路径，因为我没有使用混淆，所以在上面打包后就没有混淆文件。第三个对应资源文件。注意这里一定要填正确，否则会导致不能生成补丁文件。弄完之后同步一下。

接着就可以修复bug了。这里我们进行模拟一下。

```
<androidx.appcompat.widget.LinearLayoutCompat xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <androidx.appcompat.widget.AppCompatButton
        android:id="@+id/btn"
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:layout_marginTop="20dp"
        android:text="加载 PATCH 包"
        android:textSize="20sp"
        tools:ignore="HardcodedText" />

    <TextView
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:gravity="center"
        android:text="我是被修复的bug"
        android:textSize="25sp" />

</androidx.appcompat.widget.LinearLayoutCompat>
```

修改了一下布局，添加了一个 TextView。

接着就需要使用 tinker 插件生成补丁了。如下

![123](6%EF%BC%8CThinker%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/123.png)

点击 tinkerPatchRelease 即可生成 补丁包，如果出现 IO 异常，可以删除 build 文件夹中除过 bakApk 文件以外的所有文件，可能是有冲突。

或者是 出现 config 的错误：则在 gradle 的 android 包下添加如下：

```java
 signingConfigs {
        config {
            storeFile file('C:\\Users\\Administrator\\Desktop\\345\\Project\\Tinker\\keystore.jks')
            storePassword '123456789'
            keyAlias = 'key0'
            keyPassword '123456789'
        }
    }
```

因为使用 tinker 插件打包的时候也需要 key 和 密码等。

最后生成的结果如下

![456](6%EF%BC%8CThinker%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/456.png)

#### 9，加载补丁文件，修复bug

其中 patch_signed.apk 就是我们需要的补丁包。我们需要将补丁包复制到的我们程序中定义的路径中：

![1574066373751](6%EF%BC%8CThinker%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1574066373751.png)

这里我用的电脑直接复制到手机中了

然后点击加载 **加载 PATCH 包** 接着程序会直接退出。当你再次打开的时候就会发现你添加的 TextView 已经显示出来了，最终的结果如下：

![Screenshot_2019-11-18-16-40-17-039_com.testdemo.w](6%EF%BC%8CThinker%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/Screenshot_2019-11-18-16-40-17-039_com.testdemo.w.jpg)



​	经过上面几步，我们就已经完成了一个从本地加载的热修复





### 在项目中使用 Tinker 

​		当然了，我们不可能一直从本地加载补丁文件。所以我们需要对加载 补丁文件进行修改一下。

​		新建一个 服务。在服务中我们会进行请求服务器是否有新的补丁，如果有补丁就下载到指定的目录中，然后进行加载补丁文件。

```java
public class TinkerService extends Service {

    /**
     * 文件后缀名
     */
    private static final String FILE_END = ".apk";
    /**
     * 下载 patch 文件信息
     */
    private static final int DOWNLOAD_PATCH = 0x01;
    /**
     * 检查是否有 patch 更新
     */
    private static final int UPDATE_PATCH = 0x02;


    /**
     * patch 要保存的文件夹
     */
    private String mPatchFileDir;
    /**
     * patch 文件保存路径
     */
    private String mFilePath;

    /**
     * 服务器 patch 的信息
     */
    private BasePatch mBasePatchInfo;

    @SuppressLint("HandlerLeak")
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case UPDATE_PATCH:
                    checkPatchInfo();
                    break;
                case DOWNLOAD_PATCH:
                    downloadPatch();
                    break;
            }
        }
    };

    private void downloadPatch() {
		//下载补丁文件
		mFilePath = mPatchFileDir.concat("tinker")
                        .concat(FILE_END);
		//.....下载完成，进行加载
         TinkerManager.loadPatch(mFilePath);
         //加载完成后终止服务
         stopSelf();
    }


    @Override
    public void onCreate() {
        super.onCreate();
        init();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        //检查是否有 patch 更新
        mHandler.sendEmptyMessage(UPDATE_PATCH);
        return START_NOT_STICKY;
    }

    private void init() {
        mPatchFileDir = getExternalCacheDir().getAbsolutePath() + "/tPatch/";
        File patchFileDir = new File(mPatchFileDir);
        try {
            if (!patchFileDir.exists()) {
                //文件夹不存在则创建
                patchFileDir.mkdir();
            }
        } catch (Exception e) {
            e.printStackTrace();
            //无法创建文件，终止服务
            stopSelf();
        }

    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    /**
     * 检查是否有更新
     */
    private void checkPatchInfo() {
		//.....网络请求获取是否更新补丁文件，不更新就终止服务
		//下载补丁文件
        mHandler.sendEmptyMessage(DOWNLOAD_PATCH);
    }
}

```



### Tinker 高级用法

- Tinker 如何支持多渠道打包
- 如何自定义 Tinker 行为
- 要注意的问题





### Tiner 源码

​		