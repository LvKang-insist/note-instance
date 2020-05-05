IOC

​	**控制反转，英文缩写为 IOC，一般分为依赖注入和依赖查找。在android 中主要体现在 反射+注解**

下面实现一个 使用反射+注解的形式取代 findViewById ，进行 findViewById 和 事件的注入

1，首先是需要用到的注解

```kotlin
/**
 * Target：FIELD:作用于字段
 * Retention：RUNTIME:作用于运行时
 * findviewByid
 */
@Target(AnnotationTarget.FIELD)
@Retention(AnnotationRetention.RUNTIME)
annotation class ViewById(val value: Int) {
}
```



```kotlin
/**
 * Target：FUNCTION:作用于函数，方法
 * Retention：RUNTIME:作用于运行时
 * 点击事件
 */
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class OnClick(val value: IntArray) {
}
```

```kotlin
/**
 * Target：FIELD:作用于函数，方法
 * Retention：RUNTIME:作用于运行时
 * 检测网络
 */
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class CheckNet() {
}
```



2，辅助类，进行 findViewById

```kotlin
/**
 * findViewById 的辅助类
 */
class ViewFinder {

    private var mActivity: Activity? = null
    private var mView: View? = null

    constructor(activity: Activity) {
        this.mActivity = activity
    }

    constructor(view: View) {
        this.mView = view
    }

    fun findViewById(viewId: Int): View? {
        return mActivity?.findViewById(viewId) ?: mView?.findViewById(viewId)!!
    }
}
```

3，要使用的类

```kotlin
object ViewUtils {

    @JvmStatic
    fun inject(activity: Activity) {
        inject(ViewFinder(activity), activity)
    }

    @JvmStatic
    fun inject(view: View) {
        inject(ViewFinder(view), view)
    }

    @JvmStatic
    fun inject(view: View, any: Any) {
        inject(ViewFinder(view), any)
    }

    //兼容 ,any 反射要执行的类
    private fun inject(finder: ViewFinder, any: Any) {
        injectFiled(finder, any)
        injectEvent(finder, any)
    }

    /**
     * 事件注入
     */
    private fun injectEvent(finder: ViewFinder, any: Any) {
        val clazz = any::class.java
        val methods = clazz.declaredMethods
        methods.forEach {
            //找到被注解的方法
            val onClick = it.getAnnotation(OnClick::class.java)
            if (onClick != null) {
                val value = onClick.value
                value.forEach { id ->
                    val view = finder.findViewById(id) ?: throw NullPointerException("获取 view 失败")

                    //扩展，检测网络
                    val isCheckNet = it.getAnnotation(CheckNet::class.java) != null

                    view.setOnClickListener(DeclareOnClickListener(it, any, isCheckNet))
                }
            }
        }
    }

    private class DeclareOnClickListener(
        val method: Method,
        val any: Any,
        val checkNet: Boolean
    ) : View.OnClickListener {
        override fun onClick(v: View?) {
            //是否需要检测网络
            if (checkNet) {
                if (!netWorkAvailable(v!!.context)) {
                    Toast.makeText(v.context, "亲，您的网络不太给力", Toast.LENGTH_LONG).show()
                    return
                }
            }

            //反射执行点击事件
            method.isAccessible = true
            try {
                method.invoke(any, v)
            } catch (e: Exception) {
                try {
                    method.invoke(any)
                } catch (e: Exception) {
                    e.printStackTrace()
                }
            }
        }
    }

    /**
     * 注入属性
     */
    private fun injectFiled(finder: ViewFinder, any: Any) {
        //遍历所有的属性
        val clazz = any::class.java
        val fields = clazz.declaredFields
        //获取 viewById 中的 value 值
        fields.forEach {
            val viewById = it.getAnnotation(ViewById::class.java)
            if (viewById != null) {
                //获取注解的id值
                val id = viewById.value
                //findViewByiD 找到 View
                val view = finder.findViewById(id) ?: throw NullPointerException("获取 view 失败")
                it.isAccessible = true
                //动态的注入找到View
                it.set(any, view)
            }
        }
    }

    fun netWorkAvailable(context: Context): Boolean {
        val connectivityManager =
            context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val networkInfo = connectivityManager.activeNetworkInfo
        if (networkInfo != null && networkInfo.isAvailable) {
            return true
        }
        return false
    }
}
```

4，使用如下：

```kotlin
class MainActivity : AppCompatActivity() {

    @ViewById(R.id.test_tv)
    private lateinit var mTextTv: AppCompatTextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        ViewUtils.inject(this)

    }


    @OnClick([R.id.test_tv, R.id.test_iv])
    @CheckNet
    private fun onClick() {
        Toast.makeText(this, "-----", Toast.LENGTH_LONG).show()
    }
}
```