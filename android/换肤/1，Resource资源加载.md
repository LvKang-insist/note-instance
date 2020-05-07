**了解资源的加载过程，实现加载皮肤文件中的资源文件**

### 资源加载：

​	imageView 布局中的 src 图片是怎么加载的呢？

```
mResources.loadDrawable(value, value.resourceId, density, mTheme)
```

​	其实都是通过 Resource 进行加载的

​	既然资源的加载是通过 Resource 类，如果想要获取另一个 apk 中的资源文件，那么自己实例化一个 Resource 进行加载可以吗？

- 首先需要搞清楚 Resource 是怎样实例化的

  ```java
  @Override
  public Resources getResources() {
      if (mResources == null && VectorEnabledTintResources.shouldBeUsed()) {
          mResources = new VectorEnabledTintResources(this, super.getResources());
      }
      return mResources == null ? super.getResources() : mResources;
  }
  ```

  ​	在 AppcompatActivity 中，有一个 getResources 方法用来获取 Resource。如果 mResource 为 null，则调用的就是 super.Resources()。

  ```java
  @Override
  public Resources getResources() {
      return getResourcesInternal();
  }
  
  private Resources getResourcesInternal() {
      if (mResources == null) {
          if (mOverrideConfiguration == null) {
              //1
              mResources = super.getResources();
          } else {
              final Context resContext = createConfigurationContext(mOverrideConfiguration);
              //2
              mResources = resContext.getResources();
          }
      }
      return mResources;
  }
  ```

  上面的 1 和 2 最终都调用的是 Context 的 getResource 方法，但是 Context 的 getResource 是一个抽象方法，我们必须要找到他的实现类。

  ```java
  //返回应用程序包的Resources实例
  public abstract Resources getResources();
  ```

  他的实现类其实就是 **ContextImpl** ,这个类在 as 上面是看不到的，[需要从源码中查看](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/app/ContextImpl.java#)

  ```java
  @Override
      public Resources getResources() {
          return mResources;
      }
  ```

  最终是调用的上面的这个方法，至于 mResource 是怎样实例化的，如下：

  ```java
   private static Resources createResources(IBinder activityToken, LoadedApk pi, String splitName,
              int displayId, Configuration overrideConfig, CompatibilityInfo compatInfo) {
          final String[] splitResDirs;
          final ClassLoader classLoader;
          try {
              splitResDirs = pi.getSplitPaths(splitName);
              classLoader = pi.getSplitClassLoader(splitName);
          } catch (NameNotFoundException e) {
              throw new RuntimeException(e);
          }
          return ResourcesManager.getInstance().getResources(activityToken,
                  pi.getResDir(),
                  splitResDirs,
                  pi.getOverlayDirs(),
                  pi.getApplicationInfo().sharedLibraryFiles,
                  displayId,
                  overrideConfig,
                  compatInfo,
                  classLoader);
      }
  ```

  创建 Resource 都会走到 createResources 方法。

  可以看到最终调用的是 **ResourcesManager** 的 getResource 方法，

  ```java
  public @Nullable Resources getResources(@Nullable IBinder activityToken,
          @Nullable String resDir,
          @Nullable String[] splitResDirs,
          @Nullable String[] overlayDirs,
          @Nullable String[] libDirs,
          int displayId,
          @Nullable Configuration overrideConfig,
          @NonNull CompatibilityInfo compatInfo,
          @Nullable ClassLoader classLoader) {
      try {
          Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "ResourcesManager#getResources");
          final ResourcesKey key = new ResourcesKey(
                  resDir,
                  splitResDirs,
                  overlayDirs,
                  libDirs,
                  displayId,
                  overrideConfig != null ? new Configuration(overrideConfig) : null, // Copy
                  compatInfo);
          classLoader = classLoader != null ? classLoader : ClassLoader.getSystemClassLoader();
          return getOrCreateResources(activityToken, key, classLoader);
      } finally {
          Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
      }
  }
  ```

  其中 参数 resDir 就是 apk 存放的路径。在这个方法中将 参数整合到 ResourceKey 中，然后调用了 getOrCreateResources 方法

  ```java
  private @Nullable Resources getOrCreateResources(@Nullable IBinder activityToken,
          @NonNull ResourcesKey key, @NonNull ClassLoader classLoader) {
     ynchronized (this) {
                 //从缓存中获取 ResourcesImpl
               ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
              if (resourcesImpl != null) {
                      if (DEBUG) {
                          Slog.d(TAG, "- using existing impl=" + resourcesImpl);
                      }
                      //1
                      return getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                              resourcesImpl, key.mCompatInfo);
                  }
              } else {
                  // Clean up any dead references so they don't pile up.
                  ArrayUtils.unstableRemoveIf(mResourceReferences, sEmptyReferencePredicate);
  
                  // Not tied to an Activity, find a shared Resources that has the right ResourcesImpl
                  ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
                  if (resourcesImpl != null) {
                      if (DEBUG) {
                          Slog.d(TAG, "- using existing impl=" + resourcesImpl);
                      }
                      //2
                      return getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
                  }
  
                  // We will create the ResourcesImpl object outside of holding this lock.
              }
  
       		//如果没有找到则创建
              ResourcesImpl resourcesImpl = createResourcesImpl(key);
              if (resourcesImpl == null) {
                  return null;
              }
  
              // 加入缓存
              mResourceImpls.put(key, new WeakReference<>(resourcesImpl));
  
              final Resources resources;
              if (activityToken != null) {
                  //3
                  resources = getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                          resourcesImpl, key.mCompatInfo);
              } else {
                  //4
                  resources = getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
              }
              return resources;
          }
  }
  ```

  注意看 标注 的 1,2,3,4 ，这四个位置调用的是两个方法，这两个方法最终都会通过下面这句话创建 Resource

  ```java
  // 创建一个新的资源引用并使用现有的ResourcesImpl对象
  Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
          : new Resources(classLoader);
  resources.setImpl(impl);
  ```

  到这里 Resources 就创建成功了

  我们只要清楚 Resource 是怎样进行实例化的就好了，是怎么 new 出来的。

- Resources 的构造函数

  ```java
  
  @Deprecated
  public Resources(AssetManager assets, DisplayMetrics metrics, Configuration config) {
      this(null);
      mResourcesImpl = new ResourcesImpl(assets, metrics, config, new DisplayAdjustments());
  }
  
  @UnsupportedAppUsage
  public Resources(@Nullable ClassLoader classLoader) {
      mClassLoader = classLoader == null ? ClassLoader.getSystemClassLoader() : classLoader;
  }
  ```

  Resources 构造方法，由于源码的 版本不同，new Resource 的时候构造可能也会有不同。看起来有些不同，但是都差不多，在 new Resources 的下一行，调用了 setImpl 方法，传入的 ResourcesImpl 其实是通过 createResourcesImpl 方法进行创建的：

  ```java
  private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key) {
      final DisplayAdjustments daj = new DisplayAdjustments(key.mOverrideConfiguration);
      daj.setCompatibilityInfo(key.mCompatInfo);
  
      final AssetManager assets = createAssetManager(key);
    	//.....
      final DisplayMetrics dm = getDisplayMetrics(key.mDisplayId, daj);
      final Configuration config = generateConfig(key, dm);
      final ResourcesImpl impl = new ResourcesImpl(assets, dm, config, daj);
  
      return impl;
  }
  ```

  通过上面的 方法去创建 ResourceImpl，其实和第一个构造方法中创建的差不多。

- ResourceImpl

  构造：new ResourcesImpl(assets, dm, config, daj);

  - assets： AssetManager 资源管理，创建如下，通过 Builder 创建 AssetManager 对象，并且传入了各种路径，在有些版本的源码中 AssetManager 是直接 new 出来的，并调用了 addAssetPath 方法传入 apk 路径

  ```java
  protected @Nullable AssetManager createAssetManager(@NonNull final ResourcesKey key) {
      final AssetManager.Builder builder = new AssetManager.Builder();
  
      // resDir can be null if the 'android' package is creating a new Resources object.
      // This is fine, since each AssetManager automatically loads the 'android' package
      // already.
      if (key.mResDir != null) {
          	//mResDir:apk的目录
              builder.addApkAssets(loadApkAssets(key.mResDir, false /*sharedLib*/, false /*overlay*/));
      }
  
      if (key.mSplitResDirs != null) {
          for (final String splitResDir : key.mSplitResDirs) {
                  builder.addApkAssets(loadApkAssets(splitResDir, false /*sharedLib*/,
                          false /*overlay*/));
          }
      }
  
      if (key.mOverlayDirs != null) {
          for (final String idmapPath : key.mOverlayDirs) {
                  builder.addApkAssets(loadApkAssets(idmapPath, false /*sharedLib*/,
                          true /*overlay*/));
          }
      }
  
      if (key.mLibDirs != null) {
          for (final String libDir : key.mLibDirs) {
              if (libDir.endsWith(".apk")) {
             
                      builder.addApkAssets(loadApkAssets(libDir, true /*sharedLib*/,
                              false /*overlay*/));
              }
          }
      }
  
      return builder.build();
  }
  ```

  资源加载总结：所有的资源加载都是通过Resource ，Resource -> 的构建对象时直接 new 的对象，-> 其中有一个很重要的参数 AssetManager，这个类是时 Resource核心的实例。

  最终是通过 AssetManager 获取。

### 通过自己创建 Resources 加载皮肤文件中的资源文件：

1，了解皮肤文件

​	皮肤文件其实就是一个 apk，将资源文件添加到项目中，然后生成一个 apk，则这个apk就是皮肤文件

2，通过 Resources 获取皮肤文件中的资源文件，并加载

```java
//获取项目中的 resources
val superRes = resources
//创反射创建 AssetManager，构造是隐藏的，无法直接创建
val assetManager = AssetManager::class.java.newInstance()
//反射得到方法，添加本地下载好的资源apk目录，apk 的后缀名可随便更改，这里改为了 .skin
val method =
    AssetManager::class.java.getDeclaredMethod("addAssetPath", String::class.java)
method.invoke(
    assetManager,
    "${Environment.getExternalStorageDirectory().absolutePath}${File.separator}red.skin"
)

//创建一个 Resource，具体的做法在源码中有，上面已经介绍过了。
//创建的参数等在源码中都有，直接照搬即可
val resources = Resources(assetManager, superRes.displayMetrics, superRes.configuration)
//1，资源名称，资源类型，包名
val drawableId = resources.getIdentifier("image_src", "drawable", "com.qs.redskin")
 //获取 资源
val drawable = resources.getDrawable(drawableId)

mImageView.setImageDrawable(drawable)
```

通过上面几步就可以加载 皮肤中的 资源文件了。**注意要申请 存储权限**



以上只是一个简单的 Demo