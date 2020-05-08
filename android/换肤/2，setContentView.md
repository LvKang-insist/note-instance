实现换肤

1，每次打开的都是新皮肤

2，换肤后所有 activity 中的 view 都要换肤

3，每次进入 app 也要换肤

解决方案

1，每个activity 中把需要换肤的 View 找出来，然后调用代码进行换肤（非常死板）

2，获取 activity 中的根布局，然后通过不断的循环获取子 View，通过 tag 进行换肤

3，拦截 View 的创建，目前是比较好的。

### Activity 中的 setContentView

```
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

一般情况下，在 generateLayout 中加载的布局是 else 中的 R.layout.screen_simple，可以看到 id 为 @android:id/content

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

setContentView 还没完，调用完 installDecor()方法后，