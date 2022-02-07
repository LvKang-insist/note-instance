

### 前言

Image 是 Flutter 用于显示图像的小组件，它可以加载网络，本地，文件或者内存中的图像，支持 JPEG、PNG、GIF、动画 GIF、WebP、动画 WebP、BMP 和 WBMP 格式。Flutter Image 本身也实现了内存缓存的机制，可以很大的提高图片展示速度等。



### 重温 Image 的打开方式

- `Image.network`

  ```dart
  Image.network("图片地址",fit: BoxFit.cover,width: ,height: 400)
  ```

- `Image.file`

  ```dart
  Image.file(File("本地图片路径"));
  ```

- `Image.asset`

  ```dart
  Image.asset("项目中图片资源，需要在 pubspec.yanl 文件中声明");
  ```

- `Image.memory`

  ```dart
  Image.memory(Uint8List.fromList([]));
  ```

  需要传入一个字节数组

### Flutter 加载 Image 的分辨率

Flutter 可以为当前设备加载合适的分辨率图片，指定不同分辨率的图片分配如下图所示：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210113225930.png" alt="image-20210113225930811" style="zoom: 67%;" />

主资源默认对应 1.0x 的分辨率，大于 1.0 则会去选用 2.0x 下的图片文件。Flutter 中图片必须声明在 `pubspec.yaml` 文件中，具体如下图所示：

```yaml
flutter:
  uses-material-design: true
  assets:
    - images/icon.png
    - images/2.0x/icon.png
    - images/3.0x/icon.png
    - images/4.0x/icon.png
```

`pubspec.yaml` 文件中什么的每一张图片都要和实际文件相对应。相应的，当主资源图片缺少是，会按照分辨率从最高顺序寻找加载。

Flutter 打包应用时，资源会按照 key-value 形式存放在 apk 的 assets/flutter_assets/AssetManifest.josn 文件中，加载资源时会解析文件，选择最合适的文件进行加载显示。具体如下所示：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20220121222606.png" alt="image-20220121222606149" style="zoom:33%;" />

### Flutter.network 源码分析

```dart
Image.network(
  String src, {
  Key? key,
  ...///
}) : image = ResizeImage.resizeIfNeeded(cacheWidth, cacheHeight, NetworkImage(src, scale: scale, headers: headers)),;
```

使用 `Image.network` 创建 Image 对象时，会初始化实例变量 image。

```dart
/// The image to display.
final ImageProvider image;
```

image 实际上是 `ImageProvider` 图片的提供者，它本身是一个抽象类，子类如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20220121223405.png" alt="image-20220121223405333" style="zoom:33%;" />

网络图片加载使用的就是 `NetWorkImage` 类。

我们先来看一下 Image 组件，`Image` 组件本身是一个 `StatefulWidget`，其本身的状态是由 `_ImageState` 来管理的。根据 `State` 的生命周期我们可以知道，首先执行的是 `initState` 方法

#### initState

```dart
@override
void initState() {
  super.initState();
  //添加系统设置的监听，例如屏幕旋转等  
  WidgetsBinding.instance!.addObserver(this);
  //提供对 BuildContext 的非泄漏访问
  _scrollAwareContext = DisposableBuildContext<State<Image>>(this);
}
```

#### didChangeDependencies

`initState()` 执行完成后，就会执行  `didChangeDependencies()` ，如下所示：

```dart
@override
void didChangeDependencies() {
  _updateInvertColors();
  _resolveImage();///解析图片

  if (TickerMode.of(context))
    _listenToStream();
  else
    _stopListeningToStream(keepStreamAlive: true);

  super.didChangeDependencies();
}
```

##### ImageState._resolveImage

```dart
void _resolveImage() {
  //ScrollAwareImageProvider 可以避免在滚动时加载图片
  final ScrollAwareImageProvider provider = ScrollAwareImageProvider<Object>(
    context: _scrollAwareContext,
    imageProvider: widget.image,
  );
  ///创建 ImageStream 对象  
  final ImageStream newStream =
    //这里调用的是 `ImageProvder` 的 `resolve` 方法。如下所示：
    provider.resolve(createLocalImageConfiguration(
      context,
      size: widget.width != null && widget.height != null ? Size(widget.width!, widget.height!) : null,
    ));
  assert(newStream != null);
  ///更新流  
  _updateSourceStream(newStream);
}
```

##### ImageProvder.resolve

```dart
ImageStream resolve(ImageConfiguration configuration) {
  assert(configuration != null);
  //创建 ImageStream
  final ImageStream stream = createStream(configuration);
  // Load the key (potentially asynchronously), set up an error handling zone,
  // and call resolveStreamForKey.
  _createErrorHandlerAndKey(
    configuration,
    (T key, ImageErrorListener errorHandler) {
      ///尝试为 Stream 设置 ImageStreamCompleter  
      resolveStreamForKey(configuration, stream, key, errorHandler);
    },
    (T? key, Object exception, StackTrace? stack) async {
     
    },
  );
  return stream;
}
ImageStream createStream(ImageConfiguration configuration) {
   return ImageStream();
}
```

上面代码中创建了 ImageStream，并设置 ImageStreamCompleter 回调。

##### resolveStreamForKey

```dart
void resolveStreamForKey(ImageConfiguration configuration, ImageStream stream, T key, ImageErrorListener handleError) {
  ///如果不为空，则尝试创建 ImageStreamCompleter
  if (stream.completer != null) {
    ///存入缓存  
    final ImageStreamCompleter? completer = PaintingBinding.instance!.imageCache!.putIfAbsent(
      key,
      () => stream.completer!,
      onError: handleError,
    );
    assert(identical(completer, stream.completer));
    return;
  }
  /// 存入缓存
  final ImageStreamCompleter? completer = PaintingBinding.instance!.imageCache!.putIfAbsent(
    key,
    ///此 closure 会调用 ImageProvider.load 方法  、
    ///注意 load 方法的第二个参数为 PaintingBinding.instance!.instantiateImageCodec
    () => load(key, PaintingBinding.instance!.instantiateImageCodec),
    onError: handleError,
  );
  if (completer != null) {
    stream.setCompleter(completer);
  }
}
```

上面代码中尝试为创建的 ImageStream 设置一个 ImageStreamCompleter 实例

##### ImageCache.putIfAbsent

```dart
ImageStreamCompleter? putIfAbsent(Object key, ImageStreamCompleter Function() loader, { ImageErrorListener? onError }) {

  ImageStreamCompleter? result = _pendingImages[key]?.completer;
  // 如果是第一次加载，result == null
  if (result != null) {
    return result;
  }

  // 如果是第一次加载，此处 image == null  
  final _CachedImage? image = _cache.remove(key);
  if (image != null) {
    // 保证此 ImageStream 存活，存入活跃的 map 中  
    _trackLiveImage(
      key,
      image.completer,
      image.sizeBytes,
    );
    // 缓存此 Image  
    _cache[key] = image;
    return image.completer;
  }

  final _LiveImage? liveImage = _liveImages[key];
  // 如果是第一次加载，liveImage == null  
  if (liveImage != null) {
    // 此 _LiveImage 的流可能已经完成，具体条件为 sizeBytes 不为空
    // 如果未完成，则会释放 _CachedImage 创建的 aliveHandler  
    _touch(
      key,
      _CachedImage(
        liveImage.completer,
        sizeBytes: liveImage.sizeBytes,
      ),
      timelineTask,
    );
    return liveImage.completer;
  }

  try {
    // 如果缓存中没有，则会调用 ImageProvider.load 方法
    result = loader();
    // 保证流不被 dispose  
    _trackLiveImage(key, result, null);
  } catch (error, stackTrace) {
    if (!kReleaseMode) {
   
    if (onError != null) {
      onError(error, stackTrace);
      return null;
    } else {
      rethrow;
    }
  }


  bool listenedOnce = false;
      
  _PendingImage? untrackedPendingImage;
  void listener(ImageInfo? info, bool syncCall) {
    int? sizeBytes;
    if (info != null) {
      sizeBytes = info.sizeBytes;
      // 每一个 Listener 都会造成 ImageInfo.image 引用计数 +1，如果不释放就会造成 image 无法被释放。
      // 释放对此 _Image 的处理  
      info.dispose();
    }
    // 活跃计数 +1  
    final _CachedImage image = _CachedImage(
      result!,
      sizeBytes: sizeBytes,
    );
	// 活跃计数 +1，也可能无视
    _trackLiveImage(key, result, sizeBytes);

    // Only touch if the cache was enabled when resolve was initially called.
    if (untrackedPendingImage == null) {
      _touch(key, image, listenerTask);
    } else {
      // 直接释放图片  
      image.dispose();
    }

    final _PendingImage? pendingImage = untrackedPendingImage ?? _pendingImages.remove(key);
    if (pendingImage != null) {
      /// 移除加载中的图片监听，次数如果是最后一个，则 _LiveImage 也会被释放  
      pendingImage.removeListener();
    }
    listenedOnce = true;
  }

  final ImageStreamListener streamListener = ImageStreamListener(listener);
  if (maximumSize > 0 && maximumSizeBytes > 0) {
    /// 存入加载中的 map  
    _pendingImages[key] = _PendingImage(result, streamListener);
  } else {
    /// 未设置缓存也会调用一个 field 保存，防止前面存入 _LiveImage 导致的内存泄漏   
    untrackedPendingImage = _PendingImage(result, streamListener);
  }
  // 走到这里，为 ImageProvider.load 方法返回的 completer 注册监听
  result.addListener(streamListener);

  return result;
}
```

尝试将请求放入全局缓存 ImageCache 并设置监听

##### ImageProvider.load

```dart
@override
ImageStreamCompleter load(image_provider.NetworkImage key, image_provider.DecoderCallback decode) {
  /// 创建异步加载的事件流控制器
  final StreamController<ImageChunkEvent> chunkEvents =
      StreamController<ImageChunkEvent>();	
  return MultiFrameImageStreamCompleter(
    /// 异步加载的流
    chunkEvents: chunkEvents.stream,
    /// 异步加载方法  
    codec: _loadAsync(key as NetworkImage, decode, chunkEvents),
    scale: key.scale,
    debugLabel: key.url,
    informationCollector: _imageStreamInformationCollector(key),
  );
}
```

load 方法为抽象方法，这里的 load 方法Wie 实现类 NetWorkImage。

##### NetworkImage._loadAsync

```dart
Future<ui.Codec> _loadAsync(
  NetworkImage key,
  image_provider.DecoderCallback decode,
  StreamController<ImageChunkEvent> chunkEvents,
) {
  assert(key == this);

  final Uri resolved = Uri.base.resolve(key.url);
  // 此 API 仅存在于 Web 引擎实现中，不包含在 Flutter 的分析器摘要中。
  return ui.webOnlyInstantiateImageCodecFromUrl(// ignore: undefined_function, avoid_dynamic_calls
    resolved,
    chunkCallback: (int bytes, int total) {
      /// 发送数据流 event  
      chunkEvents.add(ImageChunkEvent(cumulativeBytesLoaded: bytes, expectedTotalBytes: total));
    },
  ) as Future<ui.Codec>;
}
```

此方法是记载图片源数据的方法，不同的数据源会有不同的逻辑。本质上都是获取到图片的原始字节数据，然后通过 `DecoderCallback` 来处理原始数据返回。

##### ImageState.__updateSourceStream

```dart
void _updateSourceStream(ImageStream newStream) {
  if (_imageStream?.key == newStream.key)
    return;

  if (_isListeningToStream)///初始为 false
    _imageStream!.removeListener(_getListener());

  if (!widget.gaplessPlayback)// 当 ImageProvider 改变事发后还显示旧图片，默认为 true
    setState(() { _replaceImage(info: null); }); // 将 ImageInfo 置空

  setState(() {
    _loadingProgress = null;
    _frameNumber = null;
    _wasSynchronouslyLoaded = false;
  });

  _imageStream = newStream; // 保存当前的 ImageStream
  if (_isListeningToStream) /// 初始为 false
    _imageStream!.addListener(_getListener());
}
```

##### ImageState._listenToStream

```dart
void _listenToStream() {
  if (_isListeningToStream) //初始为 false
    return;
  ///为流增加监听，每个监听的 ImageInfo 为 Completer 中的 clone
  _imageStream!.addListener(_getListener());
  _completerHandle?.dispose();
  _completerHandle = null;

  _isListeningToStream = true;
}
```

##### ImageState._getListener

```dart
ImageStreamListener _getListener({bool recreateListener = false}) {
  if(_imageStreamListener == null || recreateListener) {
    _lastException = null;
    _lastStack = null;
    /// 创建 ImageStreamListener  
    _imageStreamListener = ImageStreamListener(
      /// 处理 ImageInfo 回调  
      _handleImageFrame,
      /// 字节流回调  
      onChunk: widget.loadingBuilder == null ? null : _handleImageChunk,
      ///错误回调  
      onError: widget.errorBuilder != null || kDebugMode
          ? (Object error, StackTrace? stackTrace) {
              setState(() {
                _lastException = error;
                _lastStack = stackTrace;
              });
              assert(() {
                if (widget.errorBuilder == null)
                  throw error; // Ensures the error message is printed to the console.
                return true;
              }());
            }
          : null,
    );
  }
  return _imageStreamListener!;
}
```

创建 ImageStream 的 Listener。

##### ImageState._handleImageFrame

Listener 中处理 ImageInfo 回调的部分

```dart
void _handleImageFrame(ImageInfo imageInfo, bool synchronousCall) {
  setState(() {
    ///图片加载完成，刷新 Image 组件，此 ImageInfo 中持有的 image 为原始数据的 clone  
    _replaceImage(info: imageInfo);
    _loadingProgress = null;
    _lastException = null;
    _lastStack = null;
    _frameNumber = _frameNumber == null ? 0 : _frameNumber! + 1;
    _wasSynchronouslyLoaded = _wasSynchronouslyLoaded | synchronousCall;
  });
}
```

#### build

```dart
Widget build(BuildContext context) {
  if (_lastException != null) {
    if (widget.errorBuilder != null)
      return widget.errorBuilder!(context, _lastException!, _lastStack);
    if (kDebugMode)
      return _debugBuildErrorWidget(context, _lastException!);
  }
  // 使用  RawImage 展示 _imageInfo?.image，如果 image 为空，则 RawImage 的大小为 Size(0,0)
  // 如果加载完成，则会被刷新和展示  
  Widget result = RawImage(
    // Do not clone the image, because RawImage is a stateless wrapper.
    // The image will be disposed by this state object when it is not needed
    // anymore, such as when it is unmounted or when the image stream pushes
    // a new image.
    image: _imageInfo?.image, //解码后的图片数据
    debugImageLabel: _imageInfo?.debugLabel,
    width: widget.width,
    height: widget.height,
    scale: _imageInfo?.scale ?? 1.0,
    color: widget.color,
    opacity: widget.opacity,
    colorBlendMode: widget.colorBlendMode,
    fit: widget.fit,
    alignment: widget.alignment,
    repeat: widget.repeat,
    centerSlice: widget.centerSlice,
    matchTextDirection: widget.matchTextDirection,
    invertColors: _invertColors,
    isAntiAlias: widget.isAntiAlias,
    filterQuality: widget.filterQuality,
  );

  if (!widget.excludeFromSemantics) {
    result = Semantics(
      container: widget.semanticLabel != null,
      image: true,
      label: widget.semanticLabel ?? '',
      child: result,
    );
  }

  if (widget.frameBuilder != null)
    result = widget.frameBuilder!(context, result, _frameNumber, _wasSynchronouslyLoaded);

  if (widget.loadingBuilder != null)
    result = widget.loadingBuilder!(context, result, _loadingProgress);

  return result;
}
```

### RawImage

`Image` 控件其实只是负责图片的获取和逻辑处理，真正的绘制图片的地方是 RawImage。

 `RawImage` 继承自 `LeafRenderObjectWidget` 。

```dart
class RawImage extends LeafRenderObjectWidget
```

通过 `RenderImage` 来渲染组件。

```dart
  RenderImage createRenderObject(BuildContext context) {
    assert((!matchTextDirection && alignment is Alignment) || debugCheckHasDirectionality(context));
    assert(
      image?.debugGetOpenHandleStackTraces()?.isNotEmpty ?? true,
      'Creator of a RawImage disposed of the image when the RawImage still '
      'needed it.',
    );
    return RenderImage(
      image: image?.clone(),
      debugImageLabel: debugImageLabel,
      width: width,
      height: height,
      scale: scale,
      color: color,
      opacity: opacity,
      colorBlendMode: colorBlendMode,
      fit: fit,
      alignment: alignment,
      ///.
    );
  }
```

RenderImage 继承自 `RenderBox`，因此它需要提供自身的 `Size`, 具体在 `performLayout` 中。

```dart
@override
void performLayout() {
  size = _sizeForConstraints(constraints);
}
```

```dart
Size _sizeForConstraints(BoxConstraints constraints) {
  // Folds the given |width| and |height| into |constraints| so they can all
  // be treated uniformly.
  constraints = BoxConstraints.tightFor(
    width: _width,
    height: _height,
  ).enforce(constraints);

  if (_image == null)
    return constraints.smallest;

  return constraints.constrainSizeAndAttemptToPreserveAspectRatio(Size(
    _image!.width.toDouble() / _scale,
    _image!.height.toDouble() / _scale,
  ));
}
```

可以看到上面 _image == null 的时候返回的约束大小是 smalleset，也就是 Size(0,0)。

RenderImage 的绘制逻辑在 paint 方法中。

```dart
@override
void paint(PaintingContext context, Offset offset) {
  if (_image == null)
    return;
  _resolve();
  assert(_resolvedAlignment != null);
  assert(_flipHorizontally != null);
  paintImage(
    canvas: context.canvas,
    rect: offset & size,
    image: _image!,
    debugImageLabel: debugImageLabel,
    scale: _scale,
    opacity: _opacity?.value ?? 1.0,
    colorFilter: _colorFilter,
    fit: _fit,
    alignment: _resolvedAlignment!,
    centerSlice: _centerSlice,
    repeat: _repeat,
    flipHorizontally: _flipHorizontally!,
    invertColors: invertColors,
    filterQuality: _filterQuality,
    isAntiAlias: _isAntiAlias,
  );
}
```

最后通过 paintImage 来进行实际的绘制。



### ImageCache

通过上文的了解，我们知道通过 `ImageProvider` 加载的图片都会有一份内存中的缓存，这是一个全局的图片缓存，`ImageCache` 的初始化是在 `binding.dart` 文件中的：

```dart
mixin PaintingBinding on BindingBase, ServicesBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    ///初始化图片缓存   
    _imageCache = createImageCache();
    shaderWarmUp?.execute();
  }
}  
```

我们可以通过继承的方式来替换这个全局的 ImageCache ，不过一般我们不需要这么做。

```dart
class ImageCache {
  final Map<Object, _PendingImage> _pendingImages = <Object, _PendingImage>{};
  final Map<Object, _CachedImage> _cache = <Object, _CachedImage>{};
  final Map<Object, _LiveImage> _liveImages = <Object, _LiveImage>{};
```

ImageCache 的三种缓存：

- _liveImages
- CacheImage
- PendingImage              

### 小结 

Image 的流程比较长，单实际上就是 `获取图片资源-> 解码 —> 绘制` 的过程

[这里借用一张图](https://juejin.cn/post/6940549272320344071#heading-7)，来看一下整个调用流程

![image-20210317094330466](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b4e950be4adc4a748ec1f7a69960cff2~tplv-k3u1fbpfcp-watermark.awebp)

