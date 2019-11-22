AndFix基本介绍

​		已经好几年没有维护了，阿里出了一个收费的。这个已经被放弃了。这里只是简单的介绍一下用法

​		AndFix是一个在线修复bug的解决方案，而不是重新发布Android App。它是作为Android库发布的。Andfix是“Android热修复”的缩写。

​		 AndFix支持Android 2.3 - 7.0版本，支持ARM和X86架构，支持Dalvik和ART runtime，支持32位和64位 

​		 AndFix补丁的压缩文件格式是.apatch。它会从你自己的服务器发送到客户端来修复你的应用程序的bug。 

​		AndFix 只用用于方法级别的替换，使用场景有限

AndFix 执行流程及核心原理

![principle.png](5%EF%BC%8CAndFix%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/principle.png) 

​		从图中看到 A 调用 B，B 调用 C，但是 B 出了 bug。出现bug 以后 通过 AndFix 生成一个 patch，这个patch 中包含了要被替换的 B，然后执行的顺序就成为了 A 调用 B' ，B' 调用 C，这样就避免了调用 bug。

​		修复 Bug 的步骤

![1573627242279](5%EF%BC%8CAndFix%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573627242279.png)

使用 AndFix 完成线上bug 修复

​	1，集成 

```java
implementation 'com.alipay.euler:andfix:0.5.0@aar'
```

​	2，AndFix 使用非常简单，只有三个 api [(github)](https://github.com/alibaba/AndFix)，下面对他做一个简单的封装：

```java
public class AndFixManager {
    private static AndFixManager mInstance = null;
    private PatchManager mPatchManger = null;

    public static AndFixManager getInstance() {
        if (mInstance == null)
            synchronized (AndFixManager.class) {
                if (mInstance == null) {
                    mInstance = new AndFixManager();
                }
            }
        return mInstance;
    }

    /**
     * 初始化 AndFix 方法
     *
     * @param context
     */
    public void initPatch(Context context) {
        mPatchManger = new PatchManager(context);
        mPatchManger.init(getVersionName(context));
        mPatchManger.loadPatch();
    }


    /**
     * 加载 path 文件
     *
     * @param path
     */
    public void addPath(String path) {
        if (mPatchManger != null) {
            try {
                mPatchManger.addPatch(path);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }


    public static String getVersionName(Context context) {
        String version = "1.0.0";
        try {
            PackageManager pm = context.getPackageManager();
            PackageInfo packageInfo = pm.getPackageInfo(context.getPackageName(), 0);
            version = packageInfo.versionName;
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        return version;
    }
}
```

​	注意在 Application 中进行初始化：

```
  AndFixManager.getInstance().initPatch(this);
```

​	3，准备一个有 bug 的 apk 并安装到手机

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    /**
     * 文件后缀名
     */
    private static final String FILE_END = ".apatch";
    /**
     * 文件路径
     */
    private String mPatchDir;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);


        findViewById(R.id.btn_one).setOnClickListener(this);
        findViewById(R.id.btn_two).setOnClickListener(this);

        mPatchDir = getExternalCacheDir().getAbsolutePath() + "/apatch/";
        //创建文件夹
        File file = new File(mPatchDir);
        if (file.exists()) {
            file.mkdir();
        }
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.btn_one:
                printLog();
                break;
            case R.id.btn_two:
                AndFixManager.getInstance().addPath(getPatchName());
                break;
        }
    }

    /**
     * bug 方法
     */
    public void printLog() {
        String error = null;
        Log.e("-----", "printLog: " + Integer.valueOf(error));
    }

    /**
     * 构造文件名
     *
     * @return
     */
    private String getPatchName() {
        return mPatchDir.concat("345").concat(FILE_END);
    }

}
```

​		我们创建了两个按钮，按钮 one 执行bug 方法，two 进行修复，注意修复传入的路径。接着打包，主要要 replace 有签名的包，打完包后给它改一个名字 为 old.apk。然后保存在一个文件夹中。

4，分析解决 bug 后，build 一个新的Apk 

​		我们现在找到 bug 了，然后进行修复：	

```
	/**
     * bug 方法
     */
    public void printLog() {
//        String error = null;
//        Log.e("-----", "printLog: " + Integer.valueOf(error));
        Toast.makeText(this, "没有bug", Toast.LENGTH_LONG).show();
    }
```

​		我们修复了 bug，明企鹅弹出了一个提示框，接着进行 replace 打包，保存名字为 new.apk。将他 和 old .apk 放在一起

5，构建 patch 文件

​		[下载构建工具](https://github.com/alibaba/AndFix/blob/master/tools/apkpatch-1.0.3.zip)

​		如下所示：

​		![1573635111925](5%EF%BC%8CAndFix%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573635111925.png)

​		其中 apkpatch.bat 是 window 用到的，apkpatch.sh 是苹果用到的，

​		我创建了一个 output 文件夹用来保存 patch 文件，然后将 old.apk 和 new.apk ,KeyStory.jsk 都放进去了，因为我们要用 apkpath 来比较 old 和 new 的变化。接着我们来看一下怎么生成 patch 文件。



打开 cmd ，进入上面的目录，输入 apkpatch.bat 就可以查看对应命令和解释

![1573635446547](5%EF%BC%8CAndFix%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573635446547.png)

 		apkpatch -f <new> -t <old> -o <output> -k <keystore> -p <***> -a <alias>

​			生成 patch 文件所用到的命令

​		 apkpatch -m <apatch_path...> -k <keystore> -p <***> -a <alias> -e <***>

​			合并多个 patch 文件为 1 个的时候用到的命令

- -f ：没有 bug 的 apk
- -t ：有 bug 的 apk
- -o：patch 文件输出路径
- -k：keyStore 
- -p：密码
- -a：别名
- -e：密码

​		下面我们来看下一生成 patch文件：

​		![1573634282335](5%EF%BC%8CAndFix%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573634282335.png)

​		可以看到他将两者的差异找到了，就是 printLog 方法。接着我们看一下输出的 patch 文件：

​		![1573636194833](5%EF%BC%8CAndFix%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573636194833.png)

​		这就是生成的文件，其中我们只需要用到 .apatch 文件，345是我自己改的哈。。

patch 安装

​		将我们生成的 patch copy 到程序中定义的路径中，就是我们在程序中创建的那个路径。可以进行 copy，也可以进行 adb push ，都可以。只要将 patch 文件弄进去就好了。这里我通过 adb 的方式：

![1573636715508](5%EF%BC%8CAndFix%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573636715508.png)

​	![1573636962771](5%EF%BC%8CAndFix%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573636962771.png)

可以看到已经成功了。



然后打开 app ，点击修复按钮之后重启应用，就会发现bug已经修复了。

​		
