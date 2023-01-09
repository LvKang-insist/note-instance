### 前言

一般情况下很多同学对于点击事件的认识都只存在于从 Activity 开始的，然后从 Window 中进行分发，甚至有些人可能也只知道 `onTouchEvent` 和 `dispatchTouchEvetn` 这几个方法，对于 View 层的了解都不属性。

自从对于应用层面的分发过程了解清楚之后，就一直忍不住想知道只个事件到底是怎么产生的，到底是从哪里来，要往哪里去，具体的派发机制是怎样的。虽然在开发的过程中搞懂应用层面的也就够了，但是在好奇心的驱使下，我还是忍不住的开始了......

### 组成部分

说起组成部分，我们最熟悉的就是  View 中的事件传递了。除此之外，还有两个大部分，第一个就是输入系统部分，也就是 InputManagerService(IMS) 和 WindowManagerService(WMS) 部分。如下图所示：

![image-20230109160549376](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202301091605418.png)

#### 输入系统

输入系统主要分为输入子系统和 IMS 组成。Android 中的输入设备有很多，例如屏幕，鼠标，键盘等都是输入设备，对于应用开发者，接触最多的也就是屏幕了。当输入设备可用时，Linux会在 /dev/input 中创建对应的设备节点。用户操作输入设备就会产生各种事件，**这些事件的原始信息就会被 Linux内核中的输入子系统采集**，原始信息由 `Kernel space` 的驱动层一直传递到设备结点。

IMS 的工作就是监听 /dev/input 下所有设备的节点，当有数据时就会进行加工处理，并交给 WMS 派发给合适的 Window。

#### WMS 处理

WMS 本身的主要职责就是 Window(窗口) 管理等，具体的如下图。

![202210191446431.webp](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202301091200340.awebp)

因为 WMS 的主要职责就是窗口管理，而事件最终也是要交给合适的 Window 来往下派发的，所以说 WMS 就是输入系统的中转站了，WMS 作为 Window 的管理者，会将输入事件交给合适的 Window来处理。

#### View 处理

这个应该是我们最熟悉的了，一般情况下，输入事件最终都会交给 View 进行处理。

### IMS 的创建与启动

http://aospxref.com/android-12.0.0_r3/xref/frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp

与 WMS 一样，IMS 也是在 `SystemServer` 中创建的 ，在 run 方法中调用了 `startOtherServers` 方法

```java
private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
		 inputManager = new InputManagerService(context);

     wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
            new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
          
     inputManager.start();
}
```

上面通过构造方法创建了 IMS ，接着调用 WMS 的 mian 方法创建了 WMS ,并且传入了 IMS ，我们直达 WMS 是事件的 中转站，所以传入 IMS 并不会感到意外。最后调用了 IMS 的 start 方法。

#### IMS 构造方法

```java
public InputManagerService(Context context) {
    this.mContext = context;
    //1
    this.mHandler = new InputManagerHandler(DisplayThread.get().getLooper());

    mStaticAssociations = loadStaticInputPortAssociations();
    mUseDevInputEventForAudioJack =
            context.getResources().getBoolean(R.bool.config_useDevInputEventForAudioJack);
    Slog.i(TAG, "Initializing input manager, mUseDevInputEventForAudioJack="
            + mUseDevInputEventForAudioJack);
    //2
    mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());
}
```

在注释1出创建了处于 `android.display` 线程的 handler，这样通过 handlerr 发送的消息对应的线程执行。`android.display` 线程是系统共享的单例前台线程，可以用做一些低延时显示的相关操作，WMS 的创建也是在 `android.display` 中创建的。

注释2 调用了调用了 nativeInit 方法，可以看出来这是个 JNI 调用

```c++
static jlong nativeInit(JNIEnv* env, jclass /* clazz */,
        jobject serviceObj, jobject contextObj, jobject messageQueueObj) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == nullptr) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }
		//1
    NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,
            messageQueue->getLooper());
    im->incStrong(0);
    return reinterpret_cast<jlong>(im);
}
```

注释1处创建了 NativeInputManager ，并将该对象指针返回给了 java framwork 层，下次需要使用 native 层  NativeInputManager 对象的时候 ，直接传递这个指针就可以访问了。

接着继续看一下 NativeInputManager 构造方法

```c++
NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper), mInteractive(true) {
    JNIEnv* env = jniEnv();
		......
    //1      
    mServiceObj = env->NewGlobalRef(serviceObj);
 		
    //2      
    InputManager* im = new InputManager(this, this);
    mInputManager = im;
    defaultServiceManager()->addService(String16("inputflinger"), im);
}
```

注释1，将 java 层传下来的 IMS 对象保存在 mServiceObj 中。

注释2，创建 InputManager 对象：

```c++
InputManager::InputManager(
        const sp<InputReaderPolicyInterface>& readerPolicy,
        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
    //1
    mDispatcher = createInputDispatcher(dispatcherPolicy);
    mClassifier = new InputClassifier(mDispatcher);
    //2
    mReader = createInputReader(readerPolicy, mClassifier);
}

sp<InputReaderInterface> createInputReader(const sp<InputReaderPolicyInterface>& policy,
                                           const sp<InputListenerInterface>& listener) {
    return new InputReader(std::make_unique<EventHub>(), policy, listener);
}
```

InputManager 构造方法中：

1. 创建InputDispatcher ，该类主要用于对原始时间的分发，传递给 WMS

2. 创建 InputReader，并传入 EventHub 对象，inputreader 会不断的读取 EventHub 中的原始信息进行加工并交给 InputDispatcher ，InputDispatcher 中保存了 Window 的信息（WMS 会将窗口的信息实时更新到 InputDispatcher 中），可以将事件信息派发到合适的窗口，InputReader 和 InputDispatcher 都是耗时操作，会在单独线程中执行。

    EventHub 







