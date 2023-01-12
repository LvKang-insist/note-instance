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

### IMS 的创建和启动

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

1. 创建InputDispatcher ，该类主要用于对原始事件的分发，传递给 WMS

2. 创建 InputReader，使用智能指针创建 EventHub 并传入。inputreader 会不断的读取 EventHub 中的原始信息进行加工并交给 InputDispatcher ，InputDispatcher 中保存了 Window 的信息（WMS 会将窗口的信息实时更新到 InputDispatcher 中），可以将事件信息派发到合适的窗口，InputReader 和 InputDispatcher 都是耗时操作，会在单独线程中执行。

    **EventHub 通过 Linux 内核的 Notify 与 Epoll 机制监听设备节点，通过 EventHub 的 getEvent 函数读取设备节点的增删事件和原始输入事件。**

**总结一下**

1. 在 IMS 构造方法中，先创建了一个处于 `android.display` 的 Handler 对象。接着调用 native 层函数 `nativeInit` 创建了 NativeInputManager ，并且将该对象的地址转成 long 类型返回给了java 层。
2. 在 NativeInputMnager 构造方法中保存了 IMS 的实例，并创建了 InputMnanager 对象。
3. InputManager 中创建了 InputDispatcher 和 InputReader 对象，分别用于读取事件和分发事件。InputReader 会不断的从 EventHub 中读取原始事件信息并加工交给 InputDispatcher ，inputDispatcher 会将事件分发给合适的 Window。

<img src="https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202301101029652.png" alt="image-20230110102957616" style="zoom:50%;" />

#### IMS 的启动

在 IMS 创建完成之后，就会调用他的 `start` 方法进行启动

```java
public void start() {
    nativeStart(mPtr);
    //.....
}
```

这里调用了 nativeStart 方法，又是一个 native 方法，不过这里传入了一个参数 mPtr，通过上面的分析可以知道，该参数是 native 层 NativeInputManager 对象的指针，在这里又将其传入了 native 层。

```c++
static void nativeStart(JNIEnv* env, jclass /* clazz */, jlong ptr) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    status_t result = im->getInputManager()->start();
    if (result) {
        jniThrowRuntimeException(env, "Input manager could not be started.");
    }
}
```

将传过来的 ptr 转成了 NativeInputManager 对象，并且获取到 InputManager` ，再调用 `start` 函数。

```c++
status_t InputManager::start() {
    //1
    status_t result = mDispatcher->start();
    if (result) {
        ALOGE("Could not start InputDispatcher thread due to error %d.", result);
        return result;
    }
		//2
    result = mReader->start();
    if (result) {
        ALOGE("Could not start InputReader due to error %d.", result);
        mDispatcher->stop();
        return result;
    }
    return OK;
}
```

注释一调用了 IputDispatcher 的 start 方法，用于对事件进行分发

注释2 调用了 InputReader 的 start 方法，用于从 EventHub 中获取原始事件进行处理。

至于具体的流程我们下面在做分析

### InputReader 读取事件

IMS 启动的时候会调用 native 层，通过 InputManager 来创建 InputReader 来读取事件，下面我们来具体分析一下

```c++
status_t InputReader::start() {
    if (mThread) {
        return ALREADY_EXISTS;
    }
    mThread = std::make_unique<InputThread>(
            "InputReader", [this]() { loopOnce(); }, [this]() { mEventHub->wake(); });
    return OK;
}
```

线程中执行的是 loopOnce 函数

```c++
void InputReader::loopOnce() {
    ......
    //1
    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);

    { // acquire lock
        if (count) {
            //2
            processEventsLocked(mEventBuffer, count);
        }
    } // release lock
    ......
}
```

注释一：调用 EventHub 的 getEvents 函数来获取设备节点的信息到 mEventBuffer 中，事件信息主要有两种，一种是设备的增删事件（设备事件），一种是原始的输入事件

注释二：对 mEventBuffer 中的输入事件信息进行加工处理，**加工处理后的事件会交给 InputDispatcher 来处理**

```c++
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
  for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        size_t batchSize = 1;
        //1
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            int32_t deviceId = rawEvent->deviceId;
            while (batchSize < count) {
                if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT ||
                    rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1;
            }
            //2
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            switch (rawEvent->type) {//3
                case EventHubInterface::DEVICE_ADDED:
                    addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                    break;
                case EventHubInterface::DEVICE_REMOVED:
                    removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                    break;
                case EventHubInterface::FINISHED_DEVICE_SCAN:
                    handleConfigurationChangedLocked(rawEvent->when);
                    break;
                default:
                    ALOG_ASSERT(false); // can't happen
                    break;
            }
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
void InputReader::addDeviceLocked(nsecs_t when, int32_t eventHubId) {
	...
	std::shared_ptr<InputDevice> device = createDeviceLocked(eventHubId, identifier);
	...
	mDevices.emplace(eventHubId, device);
	...
}
```

上面代码中遍历所有的输入事件，这些事件用 RawEvent 对象来表示，将原始事件和设备事件分开处理，其中设备事件分为三种类型，这些事件都是在 EventHub 的 getEvent 函数中生成的。

如果是 DEVICE_ADDED(设备添加事件)，**InputReader 会新建一个 InputDevice 对象，用来存储设备信息，并且会将 InputDevice 存储在 KeyedVector 类型的容器 mDevices 中。** 

注释一判断事件类型，true 表示原始输入事件，false 表示设备事件

注释二处理 deviceId 所对应设备的原始输入事件

注释三判断设备事件类型，根据具体情况进行处理

我们重点关注一下原始事件的处理，也就是 `processEventsForDeviceLocked` 函数

```c++
void InputReader::processEventsForDeviceLocked(int32_t eventHubId, const RawEvent* rawEvents,
                                               size_t count) {
    //1
    auto deviceIt = mDevices.find(eventHubId);
    if (deviceIt == mDevices.end()) {
        ALOGW("Discarding event for unknown eventHubId %d.", eventHubId);
        return;
    }
	  //2
    std::shared_ptr<InputDevice>& device = deviceIt->second;
    if (device->isIgnored()) {
        // ALOGD("Discarding event for ignored deviceId %d.", deviceId);
        return;
    }

    device->process(rawEvents, count);
}
```

mDevices 中保存 key 是 eventHub，而 value 就是 设备InputDevice

注释1处根据 eventHubId 查询 map 中设备的索引，然后根据索引的设备对象 InputDevice 调用 process 方法继续处理。

```C++
void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count != 0; rawEvent++) {
       //默认为 false，如果设备输入事件缓冲区溢出，这个值为 true 
       if (mDropUntilNextSync) {
            if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
                mDropUntilNextSync = false;
            } else {
              .....
            }
        } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
            //缓冲区溢出
            ALOGI("Detected input event buffer overrun for device %s.", getName().c_str());
            mDropUntilNextSync = true;
            reset(rawEvent->when);
        } else {
            for_each_mapper_in_subdevice(rawEvent->deviceId, [rawEvent](InputMapper& mapper) {
                mapper.process(rawEvent);
            });
        }
        --count;
    }
}
inline void for_each_mapper_in_subdevice(int32_t eventHubDevice,
                                         std::function<void(InputMapper&)> f) {
  auto deviceIt = mDevices.find(eventHubDevice);
  if (deviceIt != mDevices.end()) {
     auto& devicePair = deviceIt->second;
     auto& mappers = devicePair.second;
     for (auto& mapperPtr : mappers) {
       f(*mapperPtr);
     }
   }
}

```

**真正加工原始输入事件的是 InputMapper 对象**，由于原始输入事件的类型很多，因此 InputMapper 有很多子类，用于加工不同的原始输入事件，例如 TouchInputMapper 用于处理触摸输入事件，KeyboardInputMapper 处理键盘输入事件等。

上面代码遍历每一个事件，然后再通过 `for_each_mapper_in_subdevice` 函数遍历 Mapper 类型，获取到合适的 InputMapper 子类对象，然后调用 process 方法：

- ##### 栗子一：键盘输入事件

    ```C++
    void KeyboardInputMapper::process(const RawEvent* rawEvent) {
        switch (rawEvent->type) {
            case EV_KEY: { //1
                int32_t scanCode = rawEvent->code;
                int32_t usageCode = mCurrentHidUsage;
                mCurrentHidUsage = 0;
    
                if (isKeyboardOrGamepadKey(scanCode)) {
                    //2
                    processKey(rawEvent->when, rawEvent->readTime, rawEvent->value != 0, scanCode,usageCode);
                }
                break;
            }
            //3
            case EV_MSC: {
                if (rawEvent->code == MSC_SCAN) {
                    mCurrentHidUsage = rawEvent->value;
                }
                break;
            }
            //4
            case EV_SYN: {
                if (rawEvent->code == SYN_REPORT) {
                    mCurrentHidUsage = 0;
                }
            }
        }
    }
    ```

    注释一，如果事件类型是按键类型的事件，就会调用注释二处的 `processKey` 函数

    注释三处为其他事件的处理，注释四为同步事件处理。

    我们主要来看一下 processkey 函数的处理

    ```c++
    void KeyboardInputMapper::processKey(nsecs_t when, nsecs_t readTime, bool down, int32_t scanCode,nt32_t usageCode) {
      
        NotifyKeyArgs args(getContext()->getNextId(), when, readTime, getDeviceId(), mSource,
                           getDisplayId(), policyFlags,
                           down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
                           AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, keyMetaState, downTime);
        getListener()->notifyKey(&args);
    }
    ```

    processKey 会将加工后的键盘输入事件封装为 NotifyKeyArgs ，接着将其通知给 InputListenerInterface ，InputDispatcher 继承了 InputListenerInterface ，**因此这里实际上调用了 InputDispatcher 的 notifyKey 函数**，然后进行处理。

    ```c++
    void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    		//...
        bool needWake;
        { // acquire lock
            mLock.lock();
            if (shouldSendKeyToInputFilterLocked(args)) {
                mLock.unlock();
                policyFlags |= POLICY_FLAG_FILTERED;
                if (!mPolicy->filterInputEvent(&event, policyFlags)) {
                    return; // event was consumed by the filter
                }
                mLock.lock();
            }
            //1
            std::unique_ptr<KeyEntry> newEntry =
                    std::make_unique<KeyEntry>(args->id, args->eventTime, args->deviceId, args->source,
                                               args->displayId, policyFlags, args->action, flags,
                                               keyCode, args->scanCode, metaState, repeatCount,
                                               args->downTime);
            //2
            needWake = enqueueInboundEventLocked(std::move(newEntry));
            mLock.unlock();
        } // release lock
    
        if (needWake) {
            mLooper->wake();
        }
    }
    ```

    上面代码采用Mutex互斥锁的形式，注释一处根据 args 重新封装一个 KeyEvent ，代表一次按键数据，注释二根据 KeyEvetn 来判断是否需要将 InputDispatcher 唤醒，如果需要，就调用 wake 进行唤醒，InputDispatcher 被唤醒就会重新对输入事件进行分发。

- 🌰2：触摸输入事件

    ```C++
    void TouchInputMapper::process(const RawEvent* rawEvent) {
        mCursorButtonAccumulator.process(rawEvent); //处理鼠标事件，type == EV_KEY 进行处理
        mCursorScrollAccumulator.process(rawEvent); //处理鼠标滚轮事件，type == EV_REL 进行处理
        mTouchButtonAccumulator.process(rawEvent);  //处理屏幕触摸事件，type == EV_KEY 进行处理
    
        if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
            sync(rawEvent->when, rawEvent->readTime);
        }
    }
    ```

    上面代码中，前三个是对事件进行判断，然后再进行处理，其中 EV_KEY 是键盘事件，EV_REL 是相对坐标事件。如果符合对应的条件，则会抽取对应的信息，例如坐标数据等

    接着进行判断，如果是 EV_SYN (同步事件)，并且事件 code 等于 SYN_REPORT 则执行同步函数

    ```C++
    void TouchInputMapper::sync(nsecs_t when, nsecs_t readTime) {
        // Push a new state.
        mRawStatesPending.emplace_back();
    
        RawState& next = mRawStatesPending.back();
        next.clear();
        next.when = when;
        next.readTime = readTime;
    
        // Sync button state.
        next.buttonState =
                mTouchButtonAccumulator.getButtonState() | mCursorButtonAccumulator.getButtonState();
    
        // 同步滑动事件
        next.rawVScroll = mCursorScrollAccumulator.getRelativeVWheel();
        next.rawHScroll = mCursorScrollAccumulator.getRelativeHWheel();
        mCursorScrollAccumulator.finishSync();
    
        // 同步触摸事件
        syncTouch(when, &next);
    
        //.....
    
        //继续处理事件
        processRawTouches(false /*timeout*/);
    }
    ```

    继续看 `processRawTouches` 函数

    ```C++
    void TouchInputMapper::processRawTouches(bool timeout) {
        //....
        cookAndDispatch(when, readTime);
        //....   
    }
    ```

    ```C++
    void TouchInputMapper::cookAndDispatch(nsecs_t when, nsecs_t readTime) {
     		 //...
         //进行分发
         dispatchTouches(when, readTime, policyFlags);  
    		 //...
    }
    ```

    ```C++
    void TouchInputMapper::dispatchTouches(nsecs_t when, nsecs_t readTime, uint32_t policyFlags) {
        BitSet32 currentIdBits = mCurrentCookedState.cookedPointerData.touchingIdBits;
        BitSet32 lastIdBits = mLastCookedState.cookedPointerData.touchingIdBits;
        int32_t metaState = getContext()->getGlobalMetaState();
        int32_t buttonState = mCurrentCookedState.buttonState;
    
       
            // Dispatch pointer up events.
            while (!upIdBits.isEmpty()) {
                //.....
                dispatchMotion(when, readTime, policyFlags, mSource, AMOTION_EVENT_ACTION_POINTER_UP, 0,
                               isCanceled ? AMOTION_EVENT_FLAG_CANCELED : 0, metaState, buttonState, 0,
                               mLastCookedState.cookedPointerData.pointerProperties,
                               mLastCookedState.cookedPointerData.pointerCoords,
                               mLastCookedState.cookedPointerData.idToIndex, dispatchedIdBits, upId,
                               mOrientedXPrecision, mOrientedYPrecision, mDownTime);
                dispatchedIdBits.clearBit(upId);
                mCurrentCookedState.cookedPointerData.canceledIdBits.clearBit(upId);
            }
    
            // Dispatch move events if any of the remaining pointers moved from their old locations.
            // Although applications receive new locations as part of individual pointer up
            // events, they do not generally handle them except when presented in a move event.
            if (moveNeeded && !moveIdBits.isEmpty()) {
                //...
                dispatchMotion(when, readTime, policyFlags, mSource, AMOTION_EVENT_ACTION_MOVE, 0, 0,
                               metaState, buttonState, 0,
                               mCurrentCookedState.cookedPointerData.pointerProperties,
                               mCurrentCookedState.cookedPointerData.pointerCoords,
                               mCurrentCookedState.cookedPointerData.idToIndex, dispatchedIdBits, -1,
                               mOrientedXPrecision, mOrientedYPrecision, mDownTime);
            }
    
            // Dispatch pointer down events using the new pointer locations.
            while (!downIdBits.isEmpty()) {
                 //...
                dispatchMotion(when, readTime, policyFlags, mSource, AMOTION_EVENT_ACTION_POINTER_DOWN,
                               0, 0, metaState, buttonState, 0,
                               mCurrentCookedState.cookedPointerData.pointerProperties,
                               mCurrentCookedState.cookedPointerData.pointerCoords,
                               mCurrentCookedState.cookedPointerData.idToIndex, dispatchedIdBits,
                               downId, mOrientedXPrecision, mOrientedYPrecision, mDownTime);
            }
        }
    }
    ```

    上面代码中，会根据记录的上一次触摸位置，对事件的类型进行判断，然后做相应的分发，事件类型有抬起，下落，移动等，然后进行对应的分发，无论是何种类型，最终调用的都是 `dispatchMoion`方法

    ```C++
    void TouchInputMapper::dispatchMotion(nsecs_t when, nsecs_t readTime, uint32_t policyFlags,
                                          uint32_t source, int32_t action, int32_t actionButton,
                                          int32_t flags, int32_t metaState, int32_t buttonState,
                                          int32_t edgeFlags, const PointerProperties* properties,
                                          const PointerCoords* coords, const uint32_t* idToIndex,
                                          BitSet32 idBits, int32_t changedId, float xPrecision,
                                          float yPrecision, nsecs_t downTime) {
    
    
        //最终形成的 NoteifyMotionArgs 对象
        NotifyMotionArgs args(getContext()->getNextId(), when, readTime, deviceId, source, displayId,
                              policyFlags, action, actionButton, flags, metaState, buttonState,
                              MotionClassification::NONE, edgeFlags, pointerCount, pointerProperties,
                              pointerCoords, xPrecision, yPrecision, xCursorPosition, yCursorPosition,downTime, std::move(frames));
        //回调到 InputDispatcher 的 notifyMotion 方法中
        getListener()->notifyMotion(&args);
    }
    ```

​	 

### inputDispatcher->start()

```c++
status_t InputDispatcher::start() {
    if (mThread) {
        return ALREADY_EXISTS;
    }
    mThread = std::make_unique<InputThread>(
            "InputDispatcher", [this]() { dispatchOnce(); }, [this]() { mLooper->wake(); });
    return OK;
}
```

创建了单独的线程运行，InputThread 接收三个参数，第一个是线程名字，第二个是执行 threadLoop 时的回调函数，第三个是线程销毁前唤醒线程的回调。

```c++
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    { // acquire lock
        std::scoped_lock _l(mLock);
        mDispatcherIsAlive.notify_all();

        //1
        if (!haveCommandsLocked()) {
           //2
           dispatchOnceInnerLocked(&nextWakeupTime);
        }
        if (runCommandsLockedInterruptible()) {
            nextWakeupTime = LONG_LONG_MIN;
        }
        const nsecs_t nextAnrCheck = processAnrsLocked();
        nextWakeupTime = std::min(nextWakeupTime, nextAnrCheck);

        // We are about to enter an infinitely long sleep, because we have no commands or
        // pending or queued events
        if (nextWakeupTime == LONG_LONG_MAX) {
            mDispatcherEnteredIdle.notify_all();
        }
    } // release lock

    // 3
    nsecs_t currentTime = now();
    //4
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    mLooper->pollOnce(timeoutMillis);
}
```

注释一 ：用于检查 InputDispatcher 的缓存队列中是否有等待处理的命令，没有就会执行执行注释二

注释二 ：将输入事件分发给合适的 Window

注释三 ：获取当前时间

注释四 ：计算需要睡眠的时间，调用 pollOnce 进入休眠，当 InputReader 有输入事件时，会唤醒 InputDispatcher，重新进行事件分发		