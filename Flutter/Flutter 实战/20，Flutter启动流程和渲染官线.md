### 启动流程

Flutter 的启动入口在 `lib/main.dart` 里的 `main()` 函数中，他是 `Dart` 应用程序的起点，main 函数中最简单的实现如下：

```dart
void main() => runApp(MyApp());
```

可以看到，main 函数中只调用了 `runApp()` 方法，我们看看它里面都干了什么：

```dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

接收了一个 widget 参数，它是 `Flutter` 启动后要展示的第一个组件，而 `WidgetsFlutterBinding` 正是绑定 `widget` 和 `Flutter` 引擎的桥梁，定义如下：

```dart
/// 基于 Widgets 框架的应用程序的具体绑定。
class WidgetsFlutterBinding extends BindingBase with GestureBinding, SchedulerBinding, ServicesBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {

  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance!;
  }
}
```

可以看到 `WidgetsFlutterBinding` 继承自 `BindingBase` ，并且混入了很多 `Binding`，在介绍这些 `Binding` 之前我们先介绍一下 `Window` ，下面是 `Window` 的官方解释：

> The most basic interface to the host operating system's user interface.
>
> 主机操作系统用户界面的最基本界面。

很明显，`Window` 正是 `Flutter Framework` 连接宿主操作系统的接口，

我们看一下 `Window 类的部分定义`：

```dart
@Native("Window,DOMWindow")
class Window extends EventTarget  implements WindowEventHandlers,  WindowBase  GlobalEventHandlers,
        _WindowTimers, WindowBase64 {
          
  // 当前设备的DPI，即一个逻辑像素显示多少物理像素，数字越大，显示效果就越精细保真。
  // DPI是设备屏幕的固件属性，如Nexus 6的屏幕DPI为3.5 
  double get devicePixelRatio => _devicePixelRatio;
  
  // Flutter UI绘制区域的大小
  Size get physicalSize => _physicalSize;

  // 当前系统默认的语言Locale
  Locale get locale;
    
  // 当前系统字体缩放比例。  
  double get textScaleFactor => _textScaleFactor;  
    
  // 当绘制区域大小改变回调
  VoidCallback get onMetricsChanged => _onMetricsChanged;  
  // Locale发生变化回调
  VoidCallback get onLocaleChanged => _onLocaleChanged;
  // 系统字体缩放变化回调
  VoidCallback get onTextScaleFactorChanged => _onTextScaleFactorChanged;
  // 绘制前回调，一般会受显示器的垂直同步信号VSync驱动，当屏幕刷新时就会被调用
  FrameCallback get onBeginFrame => _onBeginFrame;
  // 绘制回调  
  VoidCallback get onDrawFrame => _onDrawFrame;
  // 点击或指针事件回调
  PointerDataPacketCallback get onPointerDataPacket => _onPointerDataPacket;
  // 调度Frame，该方法执行后，onBeginFrame和onDrawFrame将紧接着会在合适时机被调用，
  // 此方法会直接调用Flutter engine的Window_scheduleFrame方法
  void scheduleFrame() native 'Window_scheduleFrame';
  // 更新应用在GPU上的渲染,此方法会直接调用Flutter engine的Window_render方法
  void render(Scene scene) native 'Window_render';

  // 发送平台消息
  void sendPlatformMessage(String name,
                           ByteData data,
                           PlatformMessageResponseCallback callback) ;
  // 平台通道消息处理回调  
  PlatformMessageCallback get onPlatformMessage => _onPlatformMessage;
  
  ... //其它属性及回调
    
}        
```

可以看到 `Window` 中包含了当前设备和系统的一些信息和 `Flutter Engine` 的一些回调。

现在回过头来看一下 `WidgetsFlutterBinding` 混入的各种 `Binding`。通过查看这些 `Binding` 的源码，我们可以发现这些 `Binding` 中基本都是监听并处理 `Window` 对象中的一些事件，然后将这些事件安装 `Framework` 的模型进行包装，抽象后然后进行分发。可以看到 `WidgetsFlutterBinding` 正是粘连 `Flutter engine` 与上层  Framework 的胶水。

- GestureBinding：提供了 `window.onPointerDataPacket` 回调，绑定 Fragment 手势子系统，是 Framework 事件模型与底层事件的绑定入口。

  ```dart
  mixin GestureBinding on BindingBase implements HitTestable, HitTestDispatcher, HitTestTarget {
    @override
    void initInstances() {
      super.initInstances();
      _instance = this;
      window.onPointerDataPacket = _handlePointerDataPacket;
    }
  }
  ```

- ServiceBinidng：提供了 `window.onPlatformMessage` 回调，用户绑定平台消息通道(message channel) ，主要处理原生和 Flutter 通信。

  ```dart
  mixin SchedulerBinding on BindingBase {
    @override
    void initInstances() {
      super.initInstances();
      _instance = this;
      if (!kReleaseMode) {
        addTimingsCallback((List<FrameTiming> timings) {
          timings.forEach(_profileFramePostEvent);
        });
      }
    }
  ```

- SchedulerBinding：提供了 `window.onBeginFrame` 和 `window.onDrawFrame` 回调，监听刷新事件，绑定 Framework 绘制调度子系统。

- PaintingBinding ：绑定绘制库，主要用户处理图片缓存

- SemanticsBidning：语义化层与 Flutter engine 的桥梁，主要是辅助功能的底层支持。

- RendererBinding：提供了 `window.onMetricsChanged` ，`window.onTextScaleFactorChanged` 等回调。他是渲染树与 Flutter engine 的桥梁。

- WidgetsBinding：提供了 `window.onLocaleChange`，`onBulidScheduled` 等回调。他是 Flutter widget 层与 engine 的桥梁。

`widgetsFlutterBinding.ensureInitiallized()` 负责初始化一个 `widgetsBinding` 的全局单例，紧接着会调用 `WidgetBinding` 的  `attachRootwWidget` 方法，该方法负责将根 Widget 添加到 `RenderView ` 上，代码如下：

```dart
void scheduleAttachRootWidget(Widget rootWidget) {
  Timer.run(() {
    attachRootWidget(rootWidget);
  });
}
```

```dart
void attachRootWidget(Widget rootWidget) {
  final bool isBootstrapFrame = renderViewElement == null;
  _readyToProduceFrames = true;
  _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
    container: renderView,
    debugShortDescription: '[root]',
    child: rootWidget,
  ).attachToRenderTree(buildOwner!, renderViewElement as RenderObjectToWidgetElement<RenderBox>?);
  if (isBootstrapFrame) {
    SchedulerBinding.instance!.ensureVisualUpdate();
  }
}
```

注意，代码中有 `renderView`和 `renderViewElement` 两个变量，`renderView` 是一个 `Renderobject` ，他是渲染树的根。而 `renderViewElement` 是 renderView 对应的 Element 对象。

可见该方法主要完成了根 `widget` 到根`RenderObject` 再到根 `Element` 的整个关联过程，我们在看看 `attachToRenderTree` 的源码实现过程：

```dart
RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [ RenderObjectToWidgetElement<T>? element ]) {
  if (element == null) {
    owner.lockState(() {
      element = createElement();
      assert(element != null);
      element!.assignOwner(owner);
    });
    owner.buildScope(element!, () {
      element!.mount(null, null);
    });
  } else {
    element._newWidget = this;
    element.markNeedsBuild();
  }
  return element!;
}
```

该方法负责创建根 element，即 `RenderObjectToWidgetElement` ，并且将 element 与 widget 进行关联，即创建出 widget 树对应的 element 树。

如果 element 创建过了，则将根 element 中关联的 widget 设为新的，由此可以看出 element 只会创建一次，后面会进行复用。那么 BuildOwner 是什么呢？，其实他就是 widget framework 的管理类，它跟踪哪些 widget 需要重新构建。

组件树在构建完毕后，回到 runApp 的实现中，当调完 `attachRootWidget` 后，最后一行会调用 `WidgetsFlutterBainding` 实例的 `scheduleWarmUpFrame()` 方法，该方法的是现在 SchedulerBinding 中，他被调用后会立即进行一次绘制，在此次绘制结束前，该方法就会锁定事件分发，也就是说在本次绘制结束完成之前 Flutter 不会响应各种事件，这可以保证在绘制过程中不会触发新的重绘。

#### 总结

通过上面上面的分析我们可以知道 `WidgetsFlutterBinding` 就像是一个胶水，它里面会监听并处理 `window` 对象的事件，并且将这些事件按照 `framework`的模型进行包装并且分发。所以说 `widgetsFlutterBinding` 正是连接 Flutter engine 与上传 Framework 的胶水。

```
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
```

- ensureInitialized ：负责初始化 `WidgetsFlutterBinding` ，并且监听 window 的事件进行包装分发。
- scheduleAttachRootWidget：在该方法的后续中，会创建根 `Element` ，调用 `mount` 完成  `element` 和 `RenderObject` 树的创建
- scheduleWarmUpFrame：开始绘制第一帧

### 渲染官线

#### Frame

一次绘制过程，我们可以将其称为一帧(frame)，我们知道 flutter 可以实现 60 fps，就是指 1 秒中可以进行60次重绘，FPS 越大，界面就会越流畅。

这里需要说明的是 Flutter 中的 frame 并不等于屏幕的刷新帧，因为 Flutter UI 框架并不是每次屏幕刷新都会触发，这是因为，如果 UI 在一段时间不变，那么每次重新走一遍渲染流程是不必要的，因此 Flutter 在第一帧渲染结束后会采取一种主动请求 frame 的方式来实现只有当 UI 可能会改变时才会重新走渲染流程。

1，Flutter 会在 `window` 上注册一个 `onBeginFrame` 和一个 `onDrawFrame`回调，在 onDrawFrame 回调中最终会调用 `drawFrame`。

2，当我们调用 `window.scheduleFrame` 方法之后，Flutter 引擎会在合适时机(可以认为是在屏幕下一次刷新之前，具体取决于 Flutter 引擎实现) 来调用 `onBeginFrame 和 onDrawFrame`。

在调用 `window.scheduleFrame` 之前会对 onBeginFrame 和 onDrawFrame 进行注册，如下所示：

```dart
void scheduleFrame() {
  if (_hasScheduledFrame || !framesEnabled)
    return;
  assert(() {
    if (debugPrintScheduleFrameStacks)
      debugPrintStack(label: 'scheduleFrame() called. Current phase is $schedulerPhase.');
    return true;
  }());
  ensureFrameCallbacksRegistered();
  window.scheduleFrame();
  _hasScheduledFrame = true;
}

 void ensureFrameCallbacksRegistered() {
    window.onBeginFrame ??= _handleBeginFrame;
    window.onDrawFrame ??= _handleDrawFrame;
  }
```

可以看见，只有主动调用 scheduleFrame 之后，才会调用 drawFrame(该方法是注册的回调)。

**所以我们在 Flutter 中提到 frame 时，如无特别说明，则是和 drawFrame() 相互对应，而不是和屏幕的刷新相对应。**



#### Frame 处理流程

当有新的 frame 到来时，开始调用 `SchedulerBinding.handleDrawFrame`  来处理 frame，具体过程就是执行四个任务队列：transientCallbacks，midFrameMicotasks，persistentCallbacks，postFrameCallbacks。当四个任务队列执行完毕后当前 frame 结束。

综上，Flutter 将整个生命周期分为 5 种状态，通过 SchedulerPhase 来表示他们：

```dart
enum SchedulerPhase {
  /// 空闲状态，并没有 frame 在处理，这种状态表示页面未发生变化，并不需要重新渲染
  /// 如果页面发生变化，需要调用 scheduleFrame 来请求 frame。
  /// 注意，空闲状态只是代表没有 frame 在处理。通常微任务，定时器回调或者用户回调事件都有可能被执行
  /// 比如监听了 tap 事件，用户点击后我们 onTap回调就是在 onTap 执行的
  idle,

  /// 执行 临时 回调任务，临时回调任务只能被执行一次，执行后会被移出临时任务队列。
  /// 典型代表就是动画回调会在该阶段执行
  transientCallbacks,

  /// 在执行临时任务是可能会产生一下新的微任务，比如在执行第一个临时任务时创建了一个 Fluture，
  /// 且这个 Future 在所有任务执行完毕前就已经 resolve
  /// 这种情况 Future 的回调将会在 [midFrameMicrotasks] 阶段执行
  midFrameMicrotasks,

  /// 执行一些持久的任务(每一个 frame 都要执行的任务)，比如渲染官线(构建，布局，绘制)
  /// 就是在该任务队列执行的
  persistentCallbacks,

  /// 在当前 frame 在结束之前将会执行 postFrameCallbacks，通常进行一些清理工作和请求新的 frame
  postFrameCallbacks,
}
```

需要注意，接下来需要重点介绍的渲染管线就是在 `persistentCallbacks` 中执行的。



#### 渲染管线(rendering pipline)

当我们页面需要发生变化时，我们需要调用 scheduleFrame() 方法去请求 frame，该方法中会注册 `_handleBeginFrame`和 `_handleDrawFrame`。 当 frame 到来时就会执行 _handleDrawFrame，代码如下：

```dart
void _handleDrawFrame() {
  //判断当前 frame 是否需要推迟，这里的推迟原因是当前坑是预热帧	
  if (_rescheduleAfterWarmUpFrame) {
    _rescheduleAfterWarmUpFrame = false;
    //添加一个回调，该回调会在当前帧结束后执行
    addPostFrameCallback((Duration timeStamp) {
      _hasScheduledFrame = false;
      //重新请求 frame。
      scheduleFrame();
    });
    return;
  }
  handleDrawFrame();
}
```

```dart
void handleDrawFrame() {
  assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
  Timeline.finishSync(); // end the "Animate" phase
  try {
    // 切换当前生命周期状态	
    _schedulerPhase = SchedulerPhase.persistentCallbacks;
     // 执行持久任务的回调，  
    for (final FrameCallback callback in _persistentCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp!);

    // postFrame 回调
    _schedulerPhase = SchedulerPhase.postFrameCallbacks;
    final List<FrameCallback> localPostFrameCallbacks =
        List<FrameCallback>.from(_postFrameCallbacks);
    _postFrameCallbacks.clear();
    for (final FrameCallback callback in localPostFrameCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp!);
  } finally {
     // 将状态改为空闲状态
    _schedulerPhase = SchedulerPhase.idle;
    Timeline.finishSync(); // end the Frame
     //....
    _currentFrameTimeStamp = null;
  }
}
```

在上面的代码中，对持久任务进行了遍历，并且进行回调，对应的是 `_persistentCallbacks` ,通过对调用栈的分析，发现该回调是在初始化 `RendererBinding` 的时候被添加到 `_persistentCallbacks` 中的：

```dart
mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, SemanticsBinding, HitTestable {
  @override
  void initInstances() {
    super.initInstances();
    //添加持久任务回调......
    addPersistentFrameCallback(_handlePersistentFrameCallback);
    initMouseTracker();
    if (kIsWeb) {
      //添加 postFrame 任务回调
      addPostFrameCallback(_handleWebFirstFrame);
    }
  }
  void addPersistentFrameCallback(FrameCallback callback) {
    _persistentCallbacks.add(callback);
  }
```

所以最终的回调就是 `_handlePersistentFrameCallback`

```dart
void _handlePersistentFrameCallback(Duration timeStamp) {
  drawFrame();
  _scheduleMouseTrackerUpdate();
}
```

 在上面代码中，调用到了 drawFrame 方法。

___

通过上面的分析之后，我们知道了当 frame 到来时，会调用到  drawFrame 中，由于 drawFrame 有一个实现方法，所以首先会调用到 WidgetsBinding 的 `drawFrame()` 方法，如下：

```dart
void drawFrame() {
  .....//省略无关
  try {
    if (renderViewElement != null)
      buildOwner!.buildScope(renderViewElement!);
    super.drawFrame();
    buildOwner!.finalizeTree();
  } 
}
```