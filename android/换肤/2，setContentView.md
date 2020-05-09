实现换肤

1，每次打开的都是新皮肤

2，换肤后所有 activity 中的 view 都要换肤

3，每次进入 app 也要换肤

解决方案

1，每个activity 中把需要换肤的 View 找出来，然后调用代码进行换肤（非常死板）

2，获取 activity 中的根布局，然后通过不断的循环获取子 View，通过 tag 进行换肤

3，拦截 View 的创建，目前是比较好的。

### Activity 中的 setContentView

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

setContentView 方法如下所示，调用的是 window 中的 setContentView，但是 window 中的只是一个抽象方法：

```java
public abstract void setContentView(View view, ViewGroup.LayoutParams params);
```

而在 window 类的最开始也说了，window 唯一的实现类是 PhoneWindow。所以这里调用的是 PhoneWindow 中的 setContentView，如下：

```java
@Override
public void setContentView(int layoutResID) {
   	//父容器如果为 null 则进行创建
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        //将传入的资源Id 加载到 mContentParent 上
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}

private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            // return new DecorView(context, featureId, this, getAttributes());
            //最终会创建一个 DecorView 赋值给 mDecor
            mDecor = generateDecor(-1);
            //.....
        } else {
            mDecor.setWindow(this);
        }
    	//mContentParent是一个	VeiwGroup
        if (mContentParent == null) {
            //最终会根据当前窗口样式加载一个资源文件，并且加载到 mDecor 中，
            //并返回资源文件中 id 为： @android:id/content 的ViewGroup，类型为 FrameLayout
            mContentParent = generateLayout(mDecor);
        }
    }


protected ViewGroup generateLayout(DecorView decor) {
        // 获取自当前主题的数据
        TypedArray a = getWindowStyle();
		//.......
    
        // 装饰 decor，判断当前窗口的属性，将符合条件的布局添加到 decor

        int layoutResource;
        int features = getLocalFeatures();
      
        else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
           
            layoutResource = R.layout.screen_progress;
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
                //加载资源
                layoutResource = R.layout.screen_custom_title;
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
                layoutResource = res.resourceId;
                layoutResource = R.layout.screen_title;
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            layoutResource = R.layout.screen_simple;
        }

        mDecor.startChanging();
    	//调用 mDecor 的方法，将刚才找到的系统布局文件加载到 DecorView 中
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    	// ID_ANDROID_CONTENT 这个ID 就是资源文件中的 Id
    	//点开findViewById ，就可以看到里面用的 View 是 DecorView 
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }
		//......
        mDecor.finishChanging();

        return contentParent;
    }
```

在 setContentView 方法中，调用了 installDecor()，下面分析一下这个方法。

首先会创建一个 DecorView ，这是一个 继承子 Fragment 的View 。也就是说 Window 类中包含了一个 DecorView。

接着就会判断 mContentParent 是否为 null，如果为 null，就会调用 generateLayout 去创建，mContentParent  是一个 ViewGroup。在 generateLayout中会调用系统的资源，判断系统当前的窗口模式。然后加载对应的布局。最终就会将这个资源文件加载到 DecorView 中。并且会调用 findViewById 找到一个 ViewGroup，并返回，**点开findViewById ，就可以看到里面用的 View 是 DecorView** 。至于加载的是那个 id，如下所示：

一般情况下，加载的资源layout中都有会 framelayout 这个 View，并且可以看到 id 为 @android:id/content。你可以复制布局名称然后全局搜索查看一下这个布局。

布局如下：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

到这里，installDecor()方法中比较重要的地方已经看完了。梳理一下：

在 setContentView() 中创建 DecorView ，接着根据系统窗口的类型获取到一个资源 layout。接着讲这个资源文件 加载到 DecorView 中，并 通过findViewById 获取了 资源文件中 id 为 @android:id/content 的控件，将其强转为 ViewGroup 并返回。

<img src="2%EF%BC%8CsetContentView.assets/image-20200508173225686.png" alt="image-20200508173225686" style="zoom:67%;" />

在 setContentView 方法中，调用完 installDecor()方法后，往下还有非常重要的一句话

```java
mLayoutInflater.inflate(layoutResID, mContentParent);
```

这个 layoutResID 就是 调用 setContentView 时传入的。这里将这个资源加载到了 mContentParent 上面，通过上面的分析我们可以知道 contentParent 就是 DecorView 中 id 为 @android:id/content 的 Framelayout 布局。

<img src="2%EF%BC%8CsetContentView.assets/image-20200509112319150.png" alt="image-20200509112319150" style="zoom: 50%;" />

最终大致的逻辑如上图

我们可以做一个测试，看一下我们分析的有没有问题：

```java
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
//        setContentView(layout())

        val view = LayoutInflater.from(this).inflate(layout(), null)
        val decorView = window.decorView
        val frameLayout = decorView.findViewById<FrameLayout>(android.R.id.content)
        frameLayout.addView(view)

}
```

这是一个 activity 的 onCreate 方法，我注释掉了 setContentView 方法，并直接加载了一个布局。然后调用 window 中的 decorView。获取到其中的 frameLayout，然后将  布局添加进去。

最终运行显示的效果是正常的。

下面给一张图，清楚的展示了布局加载的流程

![image-20200509142841196](2%EF%BC%8CsetContentView.assets/image-20200509142841196.png)

### AppCompatActivity 中的 setContentView

其实相比于 Activity 的 setContentView 还是有一些区别。主要是为了兼容低版本的一些东西。

接下来就看一下源码吧

```java
@Override
public void setContentView(View view, ViewGroup.LayoutParams params) {
    getDelegate().setContentView(view, params);
}
@NonNull
public AppCompatDelegate getDelegate() {
    if (mDelegate == null) {
        mDelegate = AppCompatDelegate.create(this, this);
    }
    return mDelegate;
}
public static AppCompatDelegate create(Activity activity, AppCompatCallback callback) {
        return new AppCompatDelegateImpl(activity, activity.getWindow(), callback);
}
```

这里调用的 setContentView 也是一个抽象方法，最终调用的是 AppCompatDelegateImpl 中的

```java
@Override
public void setContentView(View v) {
    ensureSubDecor();
    ViewGroup contentParent = (ViewGroup) 		        			 	mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    contentParent.addView(v);
    mOriginalWindowCallback.onContentChanged();
}
private void ensureSubDecor() {
        if (!mSubDecorInstalled) {
            mSubDecor = createSubDecor();
        }
}
private ViewGroup createSubDecor() {
        TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);

        final LayoutInflater inflater = LayoutInflater.from(mContext);
    	//空的 ViewGroup
        ViewGroup subDecor = null;
    
    	//根据一系列的判断，最后加载一个 layout 到 ViewGroup 上
        if (!mWindowNoTitle) {
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_dialog_title_material, null);
            } else if (mHasActionBar) {
              
                subDecor = (ViewGroup) LayoutInflater.from(themedContext)
                        .inflate(R.layout.abc_screen_toolbar, null);
        } else {
            if (mOverlayActionMode) {
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_screen_simple_overlay_action_mode, null);
            } else {
                subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
            }
        }
    
    //获取 subDecor 中的id
     final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
                R.id.action_bar_activity_content);
		
    //这里获取的是 Window 中的 DecorView 
        final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
        if (windowContentView != null) {
            //设置一个 空的 Id
            windowContentView.setId(View.NO_ID);
            //给 contentView 设置一个新的 id
            contentView.setId(android.R.id.content);
        }

		//最终还是调用了 window 中的 setContentView 方法
        // Now set the Window's content view with the decor
        mWindow.setContentView(subDecor);

        return subDecor;
    }
```

看流程，可以发现最终还是调用的 window 中的 setContentView，但是在这里有多了一层

他首先会创建一个ViewGroup，然后根据一系列的判断添加一个layout。最后调用 window 的 setContentView 。接着就会将我们传入的布局 添加到这个 ViewGroup 中，相比于 Activity。他中间多了一个 ViewGroup。

### AppCompatActivity 的兼容性

一个小例子：新建一个 activity，在布局中创建一个 ImageView。

```java
<ImageView
    android:id="@+id/test_iv"
    android:layout_width="match_parent"
    android:layout_height="200dp"
    android:layout_marginTop="50dp"
    android:src="@drawable/image"/>
```

接着让 activity 首先继承 Activity ，最后继承 AppCompatActivity，然后打印 ImageView

```
android.widget.ImageView
androidx.appcompat.widget.AppCompatImageView
```

发现如果是继承自 AppcompatActivity，则 iamgeView 最终创建的是 AppCompatImageView ，具有兼容性的 ImageView。这个是为啥呢，下面分析一下源码：

---

源码分析：

首先在 AppCompatActivity 的 onCreate 方法中 调用了一个非常重要的方法，如下：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    final AppCompatDelegate delegate = getDelegate();
    //安装View 的工厂，这里的 delegate 就是 AppCompatDelegateImpl 
    delegate.installViewFactory();
    delegate.onCreate(savedInstanceState);
    super.onCreate(savedInstanceState);
}
```

AppCompatDelegateImpl 实现了一个接口 LayoutInflater.Factory2

```java
@Override
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
        //把 LayoutInflater 的 Factory 设置为 this，也就是说创建 View 就会掉自己的 onCreateView 方法
        //如果没看懂就看一下 LayoutInflater 的源码，LayoutInflater.from(mContext) 其实是一个单例
        //如果设置了 Factory，那么每次创建 View 时都会先执行 onCreateView  方法
        LayoutInflaterCompat.setFactory2(layoutInflater, this);
    } else {
        if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImpl)) {
            Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                    + " so we can not install AppCompat's");
        }
    }
}
```

LayoutInflater.from(mContext) 其实是一个单例。他会将 LayoutInflater 的 Factory 设置为 this。

当我们在使用这种 ： LayoutInflater.from(this).inflate(layout(), parent) 代码时，就会调用到  AppCompatDelegateImpl 类中 实现 LayoutInflater.Factory2的 onCreateView 方法，

```java
public abstract class LayoutInflater {
	@UnsupportedAppUsage(trackingBug = 122360734)
	@Nullable
	public final View tryCreateView(@Nullable View parent, @NonNull String name,
	    @NonNull Context context,
	    @NonNull AttributeSet attrs) {
 	   if (name.equals(TAG_1995)) {
    	    // Let's party like it's 1995!
   	 }

  	  View view;
  	  if (mFactory2 != null) {
        //这里，LayoutInflater.from(this).inflate(layout(), null)，如果使用 AppCompatActivity ，就会给 mFactory2 设置一个值，最终这里就会调用到 AppCompatDelegateImpl 中的 onCreateView 中。
  	      view = mFactory2.onCreateView(parent, name, context, attrs);
 	   } else if (mFactory != null) {
 	       view = mFactory.onCreateView(name, context, attrs);
    	} else {
    	    view = null;
   		 }

  	  if (view == null && mPrivateFactory != null) {
  	      view = mPrivateFactory.onCreateView(parent, name, context, attrs);
 	   }
		return view;
	}
}
```

通过上面可以看到 在使用 LayoutInflater.from(this).inflate(layout(), null) 时，是如何调用到  AppCompatDelegateImpl 中的 onCreateView 中的。

接着看一下 onCreateView

```java
/**
 * From {@link LayoutInflater.Factory2}.
 */
@Override
public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    return createView(parent, name, context, attrs);
}

 @Override
    public View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs) {
    //.......

        return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext, IS_PRE_LOLLIPOP,     true,    VectorEnabledTintResources.shouldBeUsed() 
        );
    }
```

```java
final View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs, boolean inheritContext,
        boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
    final Context originalContext = context;
	//.......

    View view = null;
	//在这里进行了替换
    switch (name) {
        case "TextView":
            view = createTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageView":
            view = createImageView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Button":
            view = createButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "EditText":
            view = createEditText(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Spinner":
            view = createSpinner(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageButton":
            view = createImageButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "CheckBox":
            view = createCheckBox(context, attrs);
            verifyNotNull(view, name);
            break;
        case "RadioButton":
            view = createRadioButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "CheckedTextView":
            view = createCheckedTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "AutoCompleteTextView":
            view = createAutoCompleteTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "MultiAutoCompleteTextView":
            view = createMultiAutoCompleteTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "RatingBar":
            view = createRatingBar(context, attrs);
            verifyNotNull(view, name);
            break;
        case "SeekBar":
            view = createSeekBar(context, attrs);
            verifyNotNull(view, name);
            break;
        default:
            // The fallback that allows extending class to take over view inflation
            // for other tags. Note that we don't check that the result is not-null.
            // That allows the custom inflater path to fall back on the default one
            // later in this method.
            view = createView(context, name, attrs);
    }

 
    return view;
}

//替换后的 AppComptImageView
@NonNull
protected AppCompatImageView createImageView(Context context, AttributeSet attrs) {
    return new AppCompatImageView(context, attrs);
}
```

最终将替换后的 View 进行返回

这里比较重要的是 View 的创建首先会走 mFactory2，然后才会走 mFactory，只要不会 null，就会执行 Factory 的 onCreateView 方法。否则最后就会走 系统的 createView 方法。

