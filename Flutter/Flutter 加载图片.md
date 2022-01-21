

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

##### _resolveImage

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
    final ImageStreamCompleter? completer = PaintingBinding.instance!.imageCache!.putIfAbsent(
      key,
      () => stream.completer!,
      onError: handleError,
    );
    assert(identical(completer, stream.completer));
    return;
  }
  final ImageStreamCompleter? completer = PaintingBinding.instance!.imageCache!.putIfAbsent(
    key,
    () => load(key, PaintingBinding.instance!.instantiateImageCodec),
    onError: handleError,
  );
  if (completer != null) {
    stream.setCompleter(completer);
  }
}
```
