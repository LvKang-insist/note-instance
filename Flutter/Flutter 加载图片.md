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

在开始之前，先看一些类，看看便好，等整个流程结束后在回过头看会比较好：

- Image：用来显示图片
- _ImageState: Image 的状态类，处理生命周期，调用加载。
- ImageProvider：图片提供者，用于加载图片，例如 NetWrokImage ，ResizeImage 等。
- ImageStreamCompleter：图片资源的管理类
- ImageStream：`ImageStream` 是一个图片资源的句柄，其中持有者图片资源，加载完毕后的回调和图片资源管理者。而其中的 `ImageStreamCompleter `对象就是图片资源的管理类
- MultiFrameImageStreamCompleter：多帧图片解析器
- ImageStreamListener：监听图片的加载结果，加载完成后会进行回调，然后刷新页面展示图片

下面开始整个流程：

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

我们直接来看 Image 组件，`Image` 组件本身是一个 `StatefulWidget`，其本身的状态是由 `_ImageState` 来管理的。根据 `State` 的生命周期我们可以知道，首先执行的是 `initState` 方法

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

`ImageStream` 是一个图片资源的句柄，其中持有者图片资源，加载完毕后的回调和图片资源管理者。而其中的 `ImageStreamCompleter `对象就是图片资源的管理类

##### resolveStreamForKey

```dart
void resolveStreamForKey(ImageConfiguration configuration, ImageStream stream, T key, ImageErrorListener handleError) {
  ///如果不为空，表示流已经完成了，这里直接存入缓存
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
  /// 如果流没有完成，则存入缓存，并且将加载图片的 load 方法也传入其中。
  final ImageStreamCompleter? completer = PaintingBinding.instance!.imageCache!.putIfAbsent(
    key,
    ///此 closure 会调用 ImageProvider.load 方法  、
    ///注意 load 方法的第二个参数为 PaintingBinding.instance!.instantiateImageCodec
    () => load(key, PaintingBinding.instance!.instantiateImageCodec),
    onError: handleError,
  );
  //最后将 ImageStreamCompleter 设置给 ImageStream 
  if (completer != null) {
    stream.setCompleter(completer);
  }
}
```

上面代码中尝试为创建的 ImageStream 设置一个 ImageStreamCompleter 实例。

`_ImageState` 通过

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
  //  ImageProvider.load 方法返回的 completer 注册监听
  result.addListener(streamListener);
 
  return result;
}
```

尝试将请求放入全局缓存 ImageCache 并设置监听

##### ImageProvider.load

```dart
@override
ImageStreamCompleter load(image_provider.NetworkImage key, image_provider.DecoderCallback decode) {
    // Ownership of this controller is handed off to [_loadAsync]; it is that
    // method's responsibility to close the controller's stream when the image
    // has been loaded or an error is thrown.
    final StreamController<ImageChunkEvent> chunkEvents = StreamController<ImageChunkEvent>();

    return MultiFrameImageStreamCompleter(
      /// 异步加载方法    
      codec: _loadAsync(key as NetworkImage, chunkEvents, decode),
      /// 异步加载监听  
      chunkEvents: chunkEvents.stream,
      scale: key.scale,
      debugLabel: key.url,
      informationCollector: () {
        return <DiagnosticsNode>[
          DiagnosticsProperty<image_provider.ImageProvider>('Image provider', this),
          DiagnosticsProperty<image_provider.NetworkImage>('Image key', key),
        ];
      },
    );
  }
```

load 方法为抽象方法，这里的 load 方法为 实现类 NetWorkImage。这段代码创建了一个 `MultiFrameImageStreamCompleter` 对象并返回， 这是一个多帧图片管理器，表明 Fluter 是支持 GIF 图片的。创建对象的 codec 变量是由 _loadAsync 方法的返回值进行初始化

##### NetworkImage._loadAsync

```dart
 Future<ui.Codec> _loadAsync(
    NetworkImage key,
    StreamController<ImageChunkEvent> chunkEvents,
    image_provider.DecoderCallback decode,
  ) async {
    try {
      assert(key == this);

      final Uri resolved = Uri.base.resolve(key.url);

      final HttpClientRequest request = await _httpClient.getUrl(resolved);

      headers?.forEach((String name, String value) {
        request.headers.add(name, value);
      });
      final HttpClientResponse response = await request.close();
      if (response.statusCode != HttpStatus.ok) {
        await response.drain<List<int>>();
        throw image_provider.NetworkImageLoadException(statusCode: response.statusCode, uri: resolved);
      }

      final Uint8List bytes = await consolidateHttpClientResponseBytes(
        response,
        onBytesReceived: (int cumulative, int? total) {
          chunkEvents.add(ImageChunkEvent(
            cumulativeBytesLoaded: cumulative,
            expectedTotalBytes: total,
          ));
        },
      );
      if (bytes.lengthInBytes == 0)
        throw Exception('NetworkImage is an empty file: $resolved');

      return decode(bytes);
    } catch (e) {
      scheduleMicrotask(() {
        PaintingBinding.instance!.imageCache!.evict(key);
      });
      rethrow;
    } finally {
      chunkEvents.close();
    }
  }
```

此方法是下载图片源数据的操作，不同的数据源会有不同的逻辑。下载完成后根据图片的二进制数据实例化图像解码器对象 Codec，然后返回。接下来我们看一下  MultiFrameImageStreamCompleter 类。

##### MultiFrameImageStreamCompleter

```dart
MultiFrameImageStreamCompleter({
  required Future<ui.Codec> codec,
  required double scale,
  String? debugLabel,
  Stream<ImageChunkEvent>? chunkEvents,
  InformationCollector? informationCollector,
}) : assert(codec != null),
     _informationCollector = informationCollector,
     _scale = scale {
  this.debugLabel = debugLabel;
  codec.then<void>(_handleCodecReady, onError: (Object error, StackTrace stack) {
    reportError(
      context: ErrorDescription('resolving an image codec'),
      exception: error,
      stack: stack,
      informationCollector: informationCollector,
      silent: true,
    );
  });
  if (chunkEvents != null) {
    chunkEvents.listen(reportImageChunkEvent,
      onError: (Object error, StackTrace stack) {
        reportError(
          context: ErrorDescription('loading an image'),
          exception: error,
          stack: stack,
          informationCollector: informationCollector,
          silent: true,
        );
      },
    );
  }
         
 void _handleCodecReady(ui.Codec codec) {
    _codec = codec;
    assert(_codec != null);

    if (hasListeners) {
      _decodeNextFrameAndSchedule();
    }
  }
         
  Future<void> _decodeNextFrameAndSchedule() async {
    // This will be null if we gave it away. If not, it's still ours and it
    // must be disposed of.
    _nextFrame?.image.dispose();
    _nextFrame = null;
    try {
      _nextFrame = await _codec!.getNextFrame();
    } catch (exception, stack) {
      reportError(
        context: ErrorDescription('resolving an image frame'),
        exception: exception,
        stack: stack,
        informationCollector: _informationCollector,
        silent: true,
      );
      return;
    }
    if (_codec!.frameCount == 1) {

      if (!hasListeners) {
        return;
      }
      _emitFrame(ImageInfo(
        image: _nextFrame!.image.clone(),
        scale: _scale,
        debugLabel: debugLabel,
      ));
      _nextFrame!.image.dispose();
      _nextFrame = null;
      return;
    }
    _scheduleAppFrame();
  }
}
```

codec 的异步方法执行完成后会调用 `_handleCodecReady` 函数，方法中会将 codec 对象保存起来，然后调用 `_decodeNextFrameAndSchedule` 解码图片帧。

如果图片不是动画格式，则执行 `_emitFrame` 函数，从帧数据中拿到图片帧对象根据缩放比例创建 `ImageInfo` 对象，然后设置图片信息

```dart
void _emitFrame(ImageInfo imageInfo) {
  setImage(imageInfo);
  _framesEmitted += 1;
}
  @protected
  @pragma('vm:notify-debugger-on-exception')
  void setImage(ImageInfo image) {
    _checkDisposed();
    _currentImage?.dispose();
    _currentImage = image;

    if (_listeners.isEmpty)
      return;
    // Make a copy to allow for concurrent modification.
    final List<ImageStreamListener> localListeners =
        List<ImageStreamListener>.from(_listeners);
    for (final ImageStreamListener listener in localListeners) {
      try {
        listener.onImage(image.clone(), false);
      } catch (exception, stack) {
        reportError(
          context: ErrorDescription('by an image listener'),
          exception: exception,
          stack: stack,
        );
      }
    }
  }
```

这个时候就会根据监听器来通知一个新的图片需要渲染。那么这个监听器是什么时候添加的呢，我们回头看一下 `didChangeDependencies` 方法中，执行完 `_resolveImage` 方法后就会执行 `_listenToStream `方法。

##### ImageState.__updateSourceStream

将 _imageStream 更新为 newStream，并将流侦听器注册从旧流移动到新流（如果已注册侦听器）。

newStream 为 `_resolveImage()` 方法中创建的 ImageStream。 

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

在上面的 `MultiFrameImageStreamCompleter` 类中，图片被处理完成之后被封装成了 `ImageInfo` 对象，然后就会通过添加的监听器进行通知。

该监听器就是通过 `_listenToStream` 方法添加的。 `_listenToStream` 方法会在 `didChangeDependencies` 方法中的 `_resolveImage` 方法执行完成后执行。具体如下：

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

该方法向 _imageStream 对象中添加了监听器 ，该监听器通过 _getListener 进行获取。

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

Listener 中处理 ImageInfo 回调的部分，当有新的需要渲染时，该监听方法就会被调用，最终调用 `setState()` 方法通知界面刷新

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

#### ImageCache 的三种缓存：

- _liveImage

  LiveImage 缓存是用来保证流存活，创建时候会创建一个 `ImageStreamCompleterHandler` ，当流没有其他 Listener 的时候，会释放掉 `ImageStreamCompleterHandler`，并从缓存 map 中移除。

  在加载的图片没有缓存的时候，会通过 loader 进行加载，然后会调用 `_trackLiveImage`  存入缓存。

  ```dart
  try {
    result = loader();
    _trackLiveImage(key, result, null);
  } catch (error, stackTrace) {
    //....
  }
  
  ```

  ```dart
  void _trackLiveImage(Object key, ImageStreamCompleter completer, int? sizeBytes) {
    //避免向完成者添加不必要的回调
    _liveImages.putIfAbsent(key, () {
      //即使 ImageProvider.resolve 的调用者没有侦听流，缓存也会侦听流，并且一旦图像完成将其从挂起移动到 keepAlive，它将自行删除。即使缓存大小为 0，我们仍然添加这个跟踪器，它将为流添加一个保持活动句柄。
      return _LiveImage(
        completer,
        () {
          _liveImages.remove(key);
        },
      );
    }).sizeBytes ??= sizeBytes;
  }
  ```

  _LiveImage：

  ```dart
  class _LiveImage extends _CachedImageBase {
    _LiveImage(ImageStreamCompleter completer, VoidCallback handleRemove, {int? sizeBytes})
        //父类会创建 ImageStreamCompleterHandler
        : super(completer, sizeBytes: sizeBytes) {
      _handleRemove = () {
        handleRemove();//从缓存 map 中删除自身
        dispose();
      };
      // Listener 为空时候回调      
      completer.addOnLastListenerRemovedCallback(_handleRemove);
    }
  
    late VoidCallback _handleRemove;
  
    @override
    void dispose() {
      completer.removeOnLastListenerRemovedCallback(_handleRemove);
      super.dispose();//释放 ImageStreamCompleterHandle
    }
  
    @override
    String toString() => describeIdentity(this);
  }
  ```

- CacheImage

  此 CacheImage 用来记录已经加载完成的图片流。当图片加载完后会调用 `_touch` 方法将其添加到 cache 缓存中，

  ```dart
  class _CachedImage extends _CachedImageBase {
    _CachedImage(ImageStreamCompleter completer, {int? sizeBytes})
        //此 Cache 中添加的是已经加载完的图片流
        : super(completer, sizeBytes: sizeBytes);
  }
  ```

- PendingImage

   此缓存用来记录加载中的图片流。

  ```dart
  class _PendingImage {
    _PendingImage(this.completer, this.listener);
  
    final ImageStreamCompleter completer;
    final ImageStreamListener listener;
  
    void removeListener() {
      completer.removeListener(listener);
    }
  }
  ```

`_LiveImage` 和  `CacheImage` 的基类

```dart
abstract class _CachedImageBase {
  _CachedImageBase(
    this.completer, {
    this.sizeBytes,
  }) : assert(completer != null),
       //创建 ImageStreamCompleter 以保证流不被 dispose
       handle = completer.keepAlive();

  final ImageStreamCompleter completer;
  int? sizeBytes;
  ImageStreamCompleterHandle? handle;

  @mustCallSuper
  void dispose() {
    assert(handle != null);
    // Give any interested parties a chance to listen to the stream before we
    // potentially dispose it.
    SchedulerBinding.instance!.addPostFrameCallback((Duration timeStamp) {
      assert(handle != null);
      handle?.dispose();
      handle = null;
    });
  }
}
```

在构造方法中会创建 ImageStreamCompleterHandler ，在 dispose 的时候进行释放。

#### 缓存优化

ImageCache 提供最大图片缓存的设置方法，默认数量为 1000 张图片，同时最大的内存占用设置，默认为 100MB，同时还有基本的 putIfAbsent，evict，clear 方法。

如果需要降低图片的占用内存的时候，我们可以按照需求清理 `ImageCache` 中的缓存，例如列表中的 `Image` 被 dispose 的时候，我们可以尝试移除他的缓存，如下：

```dart
@override
void dispose() {
  //..
  if (widget.evictCachedImageWhenDisposed) {
    _imagepProvider.obtainKey(ImageConfiguration.empty).then(
      (key) {
        ImageCacheStatus statusForKey =
            PaintingBinding.instance.imageCache.statusForKey(key);
        if (statusForKey?.keepAlive ?? false) {
          //只有已完成的evict
          _imagepProvider.evict();
        }
      },
    );
  }
  super.dispose();
}
```

一般来说，`ImageCache` 使用 _imagepProvider.obtainKey 方法的返回值来当做 `key`，当图片缓存需要被移除的时候，我们获取到缓存的 key，并从 `ImageCache` 中移除。

> 需要注意的是，`未完成加载的图片缓存不能清除`。这是因为 ImageStreamCompleter 的实现类中监听了异步加载的事件流，当异步加载完成后，就会调用 reportImageChunkEvent 方法，此方法内部会调用 _checkDisposed 方法。此时如果图片被 dispose，则会抛出异常。

> 清除内存缓存就是一种 时间换空间的方式，图片展示将需要额外的加载和解码耗时。我们需要谨慎使用。

### 优化思路

- 修改缓存的大小

  ```java
  //修改缓存最大值
  const int _kDefaultSize = 100;
  const int _kDefaultSizeBytes = 50 << 20;  
  ```

- 降低内存中的图片尺寸

  在 Android 中，在将图片加载到内存之前，可以采用 `BitmapFactory` 来加载原始的宽高数据，然后通过降低采样率的方式来达到降低占用内存的效果

  在 Flutter 中，这种思想也是可行的，在原始图片被解码成 `Image` 之前，我们可以给其指定一个合适的尺寸，可以比较显著的降低内存占用。

  官方其实已经为我们提供了一个 `ResizeImage` 来降低解码后的 `Image`，但是需要我们提前为 `Image` 指定缓存的宽或者高。如果指定之后，图片就会被按比例缩放。

  ResizeImage 的实现原理并不复杂，它本身就相当于是一个代理，在加载图片的时候，他会代理原始的加载操作，如下：

  ```dart
  Image.network(
    //.....
    Map<String, String>? headers,
    int? cacheWidth,
    int? cacheHeight,
  }) : image = ResizeImage.resizeIfNeeded(cacheWidth, cacheHeight, NetworkImage(src, scale: scale, headers: headers)),
       assert(alignment != null),
       assert(repeat != null),
       assert(matchTextDirection != null),
       assert(cacheWidth == null || cacheWidth > 0),
       assert(cacheHeight == null || cacheHeight > 0),
       assert(isAntiAlias != null),
       super(key: key);
  ```

  ```dart
  static ImageProvider<Object> resizeIfNeeded(int? cacheWidth, int? cacheHeight, ImageProvider<Object> provider) {
    if (cacheWidth != null || cacheHeight != null) {
      return ResizeImage(provider, width: cacheWidth, height: cacheHeight);
    }
    return provider;
  }
  ```

  ```dart
  @override
  ImageStreamCompleter load(ResizeImageKey key, DecoderCallback decode) {
    Future<ui.Codec> decodeResize(Uint8List bytes, {int? cacheWidth, int? cacheHeight, bool? allowUpscaling}) {
      assert(
        cacheWidth == null && cacheHeight == null && allowUpscaling == null,
        'ResizeImage cannot be composed with another ImageProvider that applies '
        'cacheWidth, cacheHeight, or allowUpscaling.',
      );
      return decode(bytes, cacheWidth: width, cacheHeight: height, allowUpscaling: this.allowUpscaling);
    }
    final ImageStreamCompleter completer = imageProvider.load(key._providerCacheKey, decodeResize);
    if (!kReleaseMode) {
      completer.debugLabel = '${completer.debugLabel} - Resized(${key._width}×${key._height})';
    }
    return completer;
  }
  ```

  如上面代码所示，在加载网络图片的时候，会调用 `resizeIfNeeded` 方法，在其中会判断如果使用了缓存宽高，就会返回 `ResizeImage`，否则就会直接返回 `NetworkImage`。

  如果使用了缓存宽高，在加载图片的时候就会走到上面的 load 方法中，load 方法中会为 decode 做一层装饰，传入缓存的宽高等。最后在调用 `imageProvider(这里表示的是 NetworkImage)` 的 load 加载图片，最终解码为我们设置缓存的大小。

   deocde 的来源是 ：PaintingBinding.instance!.instantiateImageCodec。具体实现如下：

  ```dart
  Future<ui.Codec> instantiateImageCodec(
    Uint8List bytes, {
    int? cacheWidth,
    int? cacheHeight,
    bool allowUpscaling = false,
  }) {
    assert(cacheWidth == null || cacheWidth > 0);
    assert(cacheHeight == null || cacheHeight > 0);
    assert(allowUpscaling != null);
    return ui.instantiateImageCodec(
      bytes,
      targetWidth: cacheWidth,
      targetHeight: cacheHeight,
      allowUpscaling: allowUpscaling,
    );
  }
  ```

  ```dart
  Future<Codec> instantiateImageCodec(
    Uint8List list, {
    int? targetWidth,
    int? targetHeight,
    bool allowUpscaling = true,
  }) async {
    final ImmutableBuffer buffer = await ImmutableBuffer.fromUint8List(list);
    final ImageDescriptor descriptor = await ImageDescriptor.encoded(buffer);
    if (!allowUpscaling) {
      if (targetWidth != null && targetWidth > descriptor.width) {
        targetWidth = descriptor.width;
      }
      if (targetHeight != null && targetHeight > descriptor.height) {
        targetHeight = descriptor.height;
      }
    }
    buffer.dispose();
    ////指定需要的宽高  
    return descriptor.instantiateCodec(
      targetWidth: targetWidth,
      targetHeight: targetHeight,
    );
  }
  ```

  我们可以看到缓存宽高最终影响到的是 `targetWidth` 和 `targetHeight` 属性。

  到这里我们应该已经知道如何通过限制尺寸的方式来优化内存大小了，不过每次加载图片的时候都弄一个缓存宽高也挺麻烦的，这里推荐一个大佬写的 `autu_resize_image`，使用起来比较省事，[有需要的话可以参考一下](https://github.com/cr1992/auto_resize_image/blob/master/auto_resize_image.dart) 

- 增加磁盘缓存

  ```dart
  Future<ui.Codec> _loadAsync(NetworkImage key,StreamController<ImageChunkEvent> chunkEvents, image_provider.DecoderCallback decode,) async {
      try {
        assert(key == this);
     //--------新增代码1 begin--------------
     // 判断是否有本地缓存
      final Uint8List cacheImageBytes = await ImageCacheUtil.getImageBytes(key.url);
      if(cacheImageBytes != null) {
        return decode(cacheImageBytes);
      }
     //--------新增代码1 end--------------
  
      //...省略
        if (bytes.lengthInBytes == 0)
          throw Exception('NetworkImage is an empty file: $resolved');
  
          //--------新增代码2 begin--------------
         // 缓存图片数据到本地，需要定制具体的缓存策略
         await ImageCacheUtil.saveImageBytesToLocal(key.url, bytes);
         //--------新增代码2 end--------------
  
        return decode(bytes);
      } finally {
        chunkEvents.close();
      }
    }
  ```

  只需要改进一下加载图片的方法即可。不过这种会侵入到 Image 的源码中，不太推荐使用。

- 清理内存缓存

  ```dart
   PaintingBinding.instance.imageCache.clear();
  ```

  这种方式可以按照自己的需求处理，无非就是时间换空间的方式，使用之后，加载图片会重新下载解码。

- 使用第三方库

  [flutter_cached_network_image](https://github.com/Baseflow/flutter_cached_network_image)，这个库实现了本地的图片缓存，有需要的可以了解一下。

### 写在最后

到这里整片文章也就完了，说实话最开始的时候也是一知半解，最后也是通过查看资料和博客进行学习理解，然后梳理整个流程，最后写成一篇文章，方便自己回忆，也方便别人理解。

如果本文有帮助到你的地方，不胜荣幸，如有文章中有错误和疑问，欢迎大家提出!

### 参考资料

> [Flutter图片加载优化探索](https://juejin.cn/post/6940549272320344071#heading-20)
>
> [Flutter 图片加载](https://juejin.cn/post/6844904057681739783#heading-29)
>
> 省略.....

### 推荐阅读

- [一文搞懂 BuildContext](https://juejin.cn/post/6991734742672474119#heading-0)
- [Key 的原理和使用](https://juejin.cn/post/6976069994270588941)
- [Flutter 三棵树的构建流程分析](https://juejin.cn/post/7031858186567188511#heading-13)
- [启动，渲染，setState 流程](https://juejin.cn/post/7036936085326266382)‘
- [Flutter 布局流程](https://juejin.cn/post/7049563669562146846#heading-13)
- [Android 集成 Flutter | 与交互](https://juejin.cn/post/7054743801520193543)

